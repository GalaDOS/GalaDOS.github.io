---
title: GCC带宽探测原理
author: GalaDOS
date: 2022-01-18 20:16:00 +0800
categories: [public]
tags: [RTC, public]
mermaid: true
math: true
---

## 前言
拥塞控制是RTC系统中非常重要的一个功能模块，直接关系到通信质量。而在谷歌开源的WebRTC中，就有着名为GCC（Google Congestion Control）的优秀实现供我们参考学习。但是由于其代码经历大量迭代，兼容和实验性质的代码较多，不利于阅读。本文将结合实际运行的例子，从代码层分析GCC的带宽探测过程。**使用的WebRTC版本为72**，分析内容不包含默认不开启的实验代码和废弃代码。阅读本文需要对RTP协议有一定了解。

## TransportSequenceNumber 与 TransportCC
根据协商结果的不同，WebRTC会采用不同的方式评估带宽。当上层协商启用RTP拓展`AbsSendTime`以及RTCP类型`REMB`时，带宽的计算工作主要集中在接收端；而当启用RTP拓展`TransportSequenceNumber`以及RTCP类型`TransportCC`时，计算工作则几乎全在发送端。由于后者相对更新，并且按照WebRTC开发人员的说法也相对更好一些[[1]](#ref1)，所以本文仅分析发送端计算带宽的情况。

如果读者对`TransportSequenceNumber`和`TransportCC`不甚了解，可以将前者理解为附加在每个媒体包后的一个递增的编号，而后者则是接收端每隔一段时间发送的汇总反馈。该反馈中包含接收到的编号以及对应的时间信息。

## GCC上层代码结构
首先我们看一下GCC的上层代码结构：

![GCC上层代码结构](/posts/2022-01-18/GCC_UML.jpg)
_GCC上层代码结构_

- **SendSideCongestionController**: 发送端拥塞控制的总控制类。该类在72版本中仍处于重构阶段，有两个同名实现，所属不同的命名空间，并且在将来会被移除。不过它的移除并没有影响下层逻辑，所以对本文影响不大。
- **NetworkControllerInterface**: 该接口接收各类网络事件，并返回带宽探测的结果。
- **GoogCcNetworkController**: 基于GCC的拥塞控制模块，与它平级可选的还有BbrNetworkController和PccNetworkController，只不过代码中没有启用。另外，新的WebRTC已经移除了BBR，原因是`perform badly for WebRTC purposes`。
- **CongestionControlHandler**: 负责将GCC的评估结果发送到Pacer和BitrateAllocator。Pacer将数据包以一定速率均匀地发送至网络，防止引发拥塞和丢包。同时Pacer还具备发送padding包和历史包的能力。历史包可以在丢包时重传，也可以当作padding包单纯用来辅助带宽探测。BitrateAllocator负责将总带宽根据事先配置的优先级和上下限分配给各个数据流。
- **ProbeController**: 控制探测数据包的发送。在特定情况下会以指数递增形式发送探测包，从而快速探测出当前带宽。
- **ProbeBitrateEstimator**: 评估探测包的接收码率，作为后续评估的参考。
- **AlrDetector**: Alr全称Application limited region。该类评估当前应用产生的数据量是否受限（即是否未达到分配给它的码率）。若存在受限，则会通过padding等手段填充数据，防止探测出来的带宽过低。Alr一般是由于画面内容过于静止，视频编码器的输出码率无法达到目标码率导致的。
- **AcknowledgedBitrateEstimator**: 计算对端实际接收到的码率，作为后续评估的参考。
- **DelayBasedBwe**: 根据时延变化的趋势评估带宽，作为后续评估的参考。
- **SendSideBandwidthEstimation**: 该类汇总delay based bandwidth和loss rate等信息，得出GCC评估带宽的最终结果。

顺便一提，其实GCC下还有一个`LossBasedBwe`，即基于丢包统计的带宽估计，和`DelayBasedBwe`相呼应。不过它默认是不开的，所以本文就不介绍了。下图表达了一个典型的，由TransportCC触发网络模型更新的例子，读者可以大致感受一下它们是怎么配合工作的。各个子模块的细节将在接下来几节中介绍。

![GCC网络模型更新流程的例子](/posts/2022-01-18/GCC_example.jpg)
_GCC网络模型更新流程的例子_

## 带宽的主动探测（ProbeController与ProbeBitrateEstimator）
RTC中的探测（probe）指的是使用大量探测包，主动探测当前带宽上限的行为。一般有以下几个原因会触发probe询问事件：

- 会话刚刚建立
- 当前使用的网络连接发生切换
- 用户重新配置了可用的最大带宽
- 应用从网络拥塞中恢复（即网络时延由高位下降至一个稳定值时）
- 探测到的带宽发生变化
- GCC本身的定期处理（默认25ms一次）

发生这些事件后，GCC会向ProbeController询问是否需要主动探测带宽，如果需要，那具体要使用多大的码率。接着GCC再将结果发送给Pacer。Pacer则根据这个码率发送探测包。每次探测一般持续数十毫秒，并且pacer会优先使用有意义的媒体数据包，如果不够，再用纯0的padding包补充。

### 探测包的分配
ProbeController的内部结构是一个简单的状态机：

![ProbeController状态机](/posts/2022-01-18/probe_controller.jpg)
_ProbeController状态机_

在不同的状态下，ProbeController会对不同的询问事件做不同的处理，下面仅列举几个比较重要的规则：
- 当处于`kInit`状态时，进行两次探测。第一次的探测码率是用户配置的初始码率的3倍，第二次是6倍，同时状态迁移至`kWaitingForProbingResult`。
- 当处于`kWaitingForProbingResult`状态，并且触发询问事件的原因是探测到的带宽发生了变化时，若这个新带宽大于之前发送的探测包码率的0.7倍，则以新带宽的2倍进行下一次探测；若低于0.7倍，则不做处理。
- 当处于`kWaitingForProbingResult`状态，并且距离上一次探测到的带宽发生变化已经过去了1秒以上时，状态迁移至`kProbingComplete`。
- 当处于`kProbingComplete`状态，并且触发询问事件的原因是应用从网络拥塞中恢复时，查询最近是否有进入alr，如果有，则以当前带宽的2倍进行一次探测。

### 通过探测包的反馈估计带宽
Pacer得到目标探测码率后，开始发送探测包。接收端不需要识别是否是探测包，统一根据Transport CC协议返回feedback即可。发送端收到feedback后，根据id判断它对应的是探测包还是普通媒体包。如果是探测包，则传给ProbeBitrateEstimator，用于计算主动探测的带宽。ProbeBitrateEstimator首先用该组探测包的总数据量除以首包和尾包的时间间隔得到码率，再按照下图流程进行调整。图中接收码率低于发送码率的0.9倍时，算法认为发送码率已经达到网络通道的瓶颈，为了防止发生网络拥塞，所以额外降低了码率。

![主动探测带宽的计算过程](/posts/2022-01-18/probe_bitrate.jpg)
_主动探测带宽的计算过程_

可以看到整个主动探测过程使用了大量的经验值，实现并不美观，令人担心其普适性。幸运的是主动探测的主要作用仅仅是在对话开启或网络发生状况时快速上探带宽，防止画面长时间模糊。上探完成后，GCC有更精细的算法去进一步调整评估结果。

## 对端接收码率评估（AcknowledgedBitrateEstimator）
该类利用Transport CC的feedback计算对端实际接收到的码率，乍看之下和上节的ProbeBitrateEstimator功能类似。但实际上，由于probe过程是分组进行的，而媒体数据的发送却是连续不断的，所以后者的接收码率计算方式可以设计得更鲁棒一些。实际上，该类是通过贝叶斯估计的方式不断更新码率的，这里直接上源码，还是挺好读的：
```c++
  // 以150ms为周期，计算该时间段内的码率，作为一个码率样本 bitrate_sample
  // .....

  // 利用此样本更新码率bitrate_estimate_以及其他参数
  float sample_uncertainty =
      10.0f * std::abs(bitrate_estimate_ - bitrate_sample) / bitrate_estimate_;
  float sample_var = sample_uncertainty * sample_uncertainty;
  // Update a bayesian estimate of the rate, weighting it lower if the sample
  // uncertainty is large.
  // The bitrate estimate uncertainty is increased with each update to model
  // that the bitrate changes over time.
  float pred_bitrate_estimate_var = bitrate_estimate_var_ + 5.f;
  bitrate_estimate_ = (sample_var * bitrate_estimate_ +
                       pred_bitrate_estimate_var * bitrate_sample) /
                      (sample_var + pred_bitrate_estimate_var);
  bitrate_estimate_var_ = sample_var * pred_bitrate_estimate_var /
                          (sample_var + pred_bitrate_estimate_var);
```

## 基于时延的带宽估计（DelayBasedBwe）
该类是GCC最重要的模块，包含两个主要功能。一是根据transport cc反馈的时间信息来探测当前网络状态
；二是根据得到的网络状态调整预测带宽。前者由成员`TrendlineEstimator`实现，后者由成员`AimdRateControl`实现：

```mermaid
classDiagram
    direction LR
    DelayBasedBwe *-- DelayIncreaseDetectorInterface
    DelayBasedBwe *-- AimdRateControl
    DelayIncreaseDetectorInterface <|.. TrendlineEstimator
    class DelayBasedBwe{
        +IncomingPacketFeedbackVector(transport_cc)
        +LatestEstimate(DataRate*)
    }
    class DelayIncreaseDetectorInterface{
        +Update(recv_delta, send_delta, arrival_time)
        +State()
    }
    class TrendlineEstimator{
        -deque"pair'double, double'" delay_hist_
        -BandwidthUsage hypothesis_
    }
    class AimdRateControl{
        +TimeToReduceFurther(Timestamp, DataRate) : bool
        +LatestEstimate()
        +Update(RateControlInput, Timestamp)
        +SetRtt(TimeDelta)
    }
```

### 网络拥塞状态的探测（TrendlineEstimator）
TCP传输一般是根据是否丢包来探测拥塞的（不全是，比如BBR策略）。但是发生丢包时，传输时延已经由于网络节点的buffer缓存而增加了。所以对于时延敏感的RTC应用来说，需要更早的探测出拥塞状态。而在TransportCC中存储着一组媒体包的对端接收时间（arrive_time, AT）。利用它和本地的发送历史（send time，ST），可以计算出每个媒体包的网络传输耗时的变化：

$$ delta = (AT_{2} - ST_{2}) - (AT_{1} - ST_{1}) $$

当这个delta变大时，我们就可以认为网络发生了拥塞。在`TrendlineEstimator`出现之前，GCC是使用卡尔曼滤波来评估网络状态的。现在谷歌换成了更简单的Trendline，其原理是对delta样本进行一阶线性拟合，根据结果的斜率来判断网络状态。斜率大于0时是Overusing，小于0时是Underusing，约等于0时是Normal。**这里要注意Underusing这个状态，他比较容易造成误解。一般来说，只有先发生拥塞，时延才有下降的可能性。也就是说，Underusing表达的是网络正在从拥塞中恢复，而不是网络带宽没有被充分利用。**

### 根据网络状态调整带宽（AimdRateControl）
熟悉TCP传输的读者应该对AIMD很熟悉。它的意思是加性增，乘性减。其目的是在网络正常时缓慢增涨传输速率，在网络拥塞时快速减少，从而在避免拥塞的同时充分利用带宽。AimdRateControl先根据Trendline的探测结果迁移内部状态机的状态，再结合该状态与rtt、acknowledged bitrate等信息决策带宽：

```mermaid
stateDiagram-v2
    kRcHold --> kRcIncrease : network normal
    kRcHold --> kRcDecrease : network overusing
    kRcIncrease --> kRcDecrease : network overusing
    kRcIncrease --> kRcHold : network underusing
    kRcDecrease --> kRcHold: network underusing
    kRcDecrease --> kRcHold: bitrate reduced once
```

```mermaid
flowchart TD
    A[Calculate rtt accroding to feedback] --> B[/is network overuse?\]
    B -- Yes --> C[/time_now - last_redution_time > rtt?\]
    B -- No --> D[update state machine]
    C -- Yes --> D
    D -- kRcHold --> E(Return current_bitrate)
    D -- kRcIncrease --> F(Return current_bitrate + 1200Bytes * duration)
    D -- kRcDecrease --> G[new_bitrate = acknowledged_bitrate * 0.85]
    G --> H(Return min current_bitrate, new_bitrate)
    C -- No --> I[/acknowledged_bitrate < current_bitrate * 0.5?\]
    I -- Yes --> D
    I -- No --> E
```

## 带宽的最终决策（SendSideBandwidthEstimation）
To be continued.

## 参考文献
<span id = "ref1">[1] [https://groups.google.com/g/discuss-webrtc/c/ZyKcu3E9XgA/m/hF0saddeLgAJ](https://groups.google.com/g/discuss-webrtc/c/ZyKcu3E9XgA/m/hF0saddeLgAJ) </span>