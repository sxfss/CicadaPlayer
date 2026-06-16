# CicadaPlayer 播放器学习路线

这个目录是给你自己的二次学习材料，不替代仓库原有的 `doc/code_learning.zh.md` 和
`framework/code_learning.zh.md`。原文档适合作为官方入口，这里更关注：已经学过 ffplay 和
ijkplayer 之后，如何把 CicadaPlayer 当成播放器 SDK 工程来读。

## 当前学习重点

现在不要再把文档当成一组平铺笔记读。主线应该是：

```text
对象关系
  -> 普通 URL/MP4 播放循环
  -> data source
  -> demuxer
  -> packet/frame queue
  -> decode/sync/render
```

CicadaPlayer 的价值不只是“某个模块怎么写”，而是这些模块如何被 `SuperMediaPlayer::mainService()` 串成
一个可工作的播放器。

如果你是第一轮阅读，先不要打开所有文档。直接按
[00-orientation/02-first-pass-reading-guide.md](00-orientation/02-first-pass-reading-guide.md)
里的 5 篇主线读，把普通 URL/MP4 的执行链路读透。

## 目录结构

### 00 Orientation

入口：[00-orientation/README.md](00-orientation/README.md)

解决问题：

- 仓库目录怎么分层。
- 第一轮先看哪些源码。
- 当前本地能不能直接编译运行 cmdline。
- 第一轮只读哪几篇，读完要产出什么。

### 01 Entry / Control Plane

入口：[01-entry-control-plane/README.md](01-entry-control-plane/README.md)

解决问题：

- `MediaPlayer` API 怎么进入 C API handle。
- `ICicadaPlayer` 和 `SuperMediaPlayer` 的关系。
- `PlayerMessageControl` 怎么处理 prepare/start/pause/seek/stop。
- seek 这种控制命令如何影响后续数据面。

这一组只解决控制面，不等于已经理解拉流、解复用、解码和渲染。

### 02 Runtime Pipeline

入口：[02-runtime-pipeline/README.md](02-runtime-pipeline/README.md)

这是当前最重要的一组。推荐先读：

1. [02-runtime-pipeline/00-object-model-and-module-wiring.md](02-runtime-pipeline/00-object-model-and-module-wiring.md)
2. [02-runtime-pipeline/01-normal-mp4-playback-loop.md](02-runtime-pipeline/01-normal-mp4-playback-loop.md)

然后再分层读 data source、demuxer、packet/frame queue、decode/sync/render。第一轮优先读
`04-packet-frame-queue.md` 和 `05-decode-sync-render.md`，它们是理解反压和音视频同步的关键。

这组要回答的问题是：

```text
一个普通 URL/MP4 是怎么被拉流的？
bytes 怎么变成 packet？
packet 为什么要先进 BufferController？
decoder 为什么有 pending packet 和内部队列？
frame queue 和 packet queue 有什么区别？
audio clock 为什么通常是 master？
video frame 为什么会等待、渲染或丢弃？
线程、状态、callback 和 stop/release 的边界在哪里？
```

### 03 Playback Features

入口：[03-playback-features/README.md](03-playback-features/README.md)

解决问题：

- buffering/QoE 如何影响读取和渲染。
- 哪些小而美的播放器机制适合迁移到 AVPlayer。
- 起播、首帧、seek、弱网、EOF、错误、低内存、A/V sync 这些场景分别解决什么需求。

这些专题必须挂回运行主线理解，不能当孤立功能读。

### 04 C++ Engineering

入口：[04-cpp-engineering/README.md](04-cpp-engineering/README.md)

解决问题：

- 用 CicadaPlayer 训练对象职责、所有权、生命周期、接口、多态、工厂。
- 区分值得借鉴的工程设计和不建议照抄的 C++11 历史写法。
- 用固定审查清单判断：是否需要指针、是否需要多态、析构是否安全、错误是否类型化。

C++ 是辅助视角。除非主题涉及所有权、生命周期、线程同步、接口边界和错误传播，不强行做 C++17 改写。

### 99 Legacy

入口：[99-legacy/README.md](99-legacy/README.md)

保存早期历史材料，不作为日常学习主线。

## 和前置学习的关系

ffplay 给你的是播放器最小主干：

```text
open input -> read packet -> packet queue -> decode -> frame queue -> render -> clock
```

ijkplayer 给你的是工程化播放器里更成熟的控制链：

```text
API/message -> read thread -> seek/flush/serial -> decoder/render callback
```

CicadaPlayer 要重点看更像 SDK 的组织方式：

```text
MediaPlayer API
  -> C API handle
  -> ICicadaPlayer
  -> SuperMediaPlayer message loop
  -> data_source / demuxer / codec / render / cache
```

## 阅读输出模板

每看完一个阶段，不要只记函数名，固定产出四件东西：

1. 一张机制图。
2. 一个核心类职责表。
3. 一个优缺点评价。
4. 一个 AVPlayer 最小迁移草图。

只有涉及所有权、接口边界、线程同步、错误传播时，再补 C++ 工程注意点。不要一上来写大而全的源码索引。

推荐格式：

```text
场景/模块：
  它解决什么播放器问题？

机制图：
  input -> module -> output

核心对象：
  ClassA：负责什么，拥有谁，被谁调用。
  ClassB：负责什么，生命周期在哪里结束。

值得学习：
  这个设计为什么在播放器工程里有价值？

不建议照抄：
  哪些是历史写法、职责过大、所有权不清或类型表达不足？

AVPlayer 最小迁移：
  第一版只迁移哪个小机制？如何验证它工作？
```

## 暂时不建议深挖

- `platform/Android`、`platform/Apple`：平台 SDK 封装多，容易偏。
- `external/`：第三方依赖，不是当前主线。
- `framework/demuxer/dash` 全量细节：协议结构复杂，后面作为 DASH 专题处理。
- `drm/`：业务和平台依赖较重，除非你要准备 DRM 方向。
- `analytics/`：第一轮只理解它挂在外层和主循环旁路，不深入指标体系。
