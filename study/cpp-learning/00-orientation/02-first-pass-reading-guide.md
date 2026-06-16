# 第一轮阅读指南：少读几篇，把主线读透

这个目录里的文档已经不少。第一轮不要追求全部读完，目标是建立一条稳定主线：

```text
对象关系
  -> 普通 URL/MP4 播放循环
  -> packet/frame 队列
  -> decode / sync / render
  -> 线程与状态边界
```

第一轮只回答一个问题：

```text
一个普通 MP4 在 CicadaPlayer 里，如何从 API 命令一路变成按 clock 输出的音视频？
```

## 1. 第一轮只读这 5 篇

### 1. 对象关系

[../02-runtime-pipeline/00-object-model-and-module-wiring.md](../02-runtime-pipeline/00-object-model-and-module-wiring.md)

读完要能画出：

```text
MediaPlayer
  -> C handle
  -> ICicadaPlayer
  -> SuperMediaPlayer
      -> PlayerMessageControl
      -> demuxer_service
      -> BufferController
      -> SMPAVDeviceManager
```

重点不是背成员变量，而是分清谁拥有谁、谁调用谁。

### 2. 普通播放循环

[../02-runtime-pipeline/01-normal-mp4-playback-loop.md](../02-runtime-pipeline/01-normal-mp4-playback-loop.md)

读完要能说清：

```text
mainService
  -> processMsg
  -> ProcessVideoLoop
      -> doReadPacket
      -> doDeCode
      -> setUpAVPath
      -> DoCheckBufferPass
      -> doRender
```

重点是理解 `ProcessVideoLoop()` 是播放器数据面调度器，不是简单 video loop。

### 3. Packet / Frame 队列

[../02-runtime-pipeline/04-packet-frame-queue.md](../02-runtime-pipeline/04-packet-frame-queue.md)

读完要能分清：

```text
packet queue
  -> buffering / seek / drop / decoder input

decoder queue
  -> decoder 内部异步输入输出

frame queue
  -> render scheduling / A-V sync
```

这是理解反压、seek flush、buffering 的桥。

### 4. Decode / Sync / Render

[../02-runtime-pipeline/05-decode-sync-render.md](../02-runtime-pipeline/05-decode-sync-render.md)

读完要能解释：

```text
audio render
  -> master clock

video frame
  -> compare PTS with master clock
  -> wait / render / drop
```

重点是：frame 不是解出来就显示，render 必须服从 buffer gate 和 clock。

### 5. 线程与状态边界

[../02-runtime-pipeline/06-threading-and-state-map.md](../02-runtime-pipeline/06-threading-and-state-map.md)

读完要能画出：

```text
API thread
  -> message queue
  -> player main thread
  -> decoder worker thread
  -> render callback
  -> notifier / app callback
```

重点是理解 stop/release、seek/flush、render callback 为什么容易出现竞态。

## 2. 第一轮暂时不读什么

先不要展开这些内容：

- DASH/HLS 细节。
- DRM。
- Android/iOS 平台 render 和硬解细节。
- analytics 全量指标。
- 复杂 ABR 策略。
- 所有工厂/prototype 的实现细节。

这些都重要，但第一轮会干扰主线。等你能完整讲清普通 URL/MP4 后，再逐个专题展开。

## 3. 每篇读完固定产出

不要只摘函数名。每篇都产出四件东西：

```text
1. 一张机制图
2. 一个核心类职责表
3. 一个优缺点评价
4. 一个 AVPlayer 最小迁移草图
```

模板：

```text
模块/场景：

机制图：
  input -> module -> output

核心类职责：
  ClassA：负责什么；拥有谁；被谁调用。
  ClassB：负责什么；生命周期在哪里结束。

值得学习：
  这个设计解决了什么播放器问题？

不建议照抄：
  哪些地方职责过大、状态分散、所有权不清或类型表达不足？

AVPlayer 最小迁移：
  第一版只迁移哪个小机制？
  用什么日志或测试验证？
```

## 4. 第一轮验收标准

读完第一轮，你应该能不看源码说清：

- API 命令为什么不直接做重活，而是投递到 message control。
- `mainService()` 如何把控制面和数据面串起来。
- packet queue、decoder queue、frame queue 为什么是三层。
- `doReadPacket()` 如何根据 buffer 和下游消费速度节流。
- `DoCheckBufferPass()` 为什么是 render 前闸门。
- audio render 为什么能成为 master clock。
- video frame 为什么会等待、渲染或丢弃。
- seek 为什么必须清 packet、decoder、frame、render 和 clock。
- `SuperMediaPlayer` 哪些机制值得学，哪些 C++ 写法不建议照抄。

如果这些能讲清，第一轮就够了。后续再看 data source、demuxer、buffering/QoE、C++ 工程审查等专题。
