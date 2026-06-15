# AVPlayer 迁移候选：从 CicadaPlayer 筛选播放器亮点

这一章只做一件事：从 CicadaPlayer 里挑出适合迁移到 AVPlayer 的小型设计亮点。

这里不追求“把 CicadaPlayer 搬过去”。CicadaPlayer 是一个完整 SDK，里面有平台适配、
协议、缓存、ABR、analytics、render 等大量历史工程内容。AVPlayer 当前更需要的是：
围绕播放控制、seek、buffering、诊断和模块扩展，提炼几个小而完整、能被测试验证的功能包。

阅读时先看播放器问题，再看工程设计。C++ 只在所有权、线程、接口、错误传播这些地方作为辅助视角。

## 选择规则

一个候选点适合迁移，至少要满足三条：

- 能解决真实播放器问题，例如命令乱序、seek 堆积、buffering 抖动、状态不可观测。
- 能拆成小功能包，不需要一次引入完整 SDK 能力。
- 能用测试、日志或 timeline 验证，而不是只停留在架构图上。

暂时不选这些：

- 完整 HLS/DASH 协议实现。
- 完整 ABR 算法。
- DRM、字幕、平台硬解全量适配。
- 大规模源码现代化改造。
- 只为了展示 C++ 语法而存在的抽象。

## 推荐优先级

| 优先级 | 功能包 | 迁移价值 | 复杂度 |
| --- | --- | --- | --- |
| P0 | Playback control stability | 稳定播放器控制面，支撑后续模块 | 中 |
| P0 | Seek experience | 解决连续拖动、旧 seek 完成污染状态 | 中 |
| P1 | Buffering and QoE | 把队列水位、缓存时长、卡顿状态串起来 | 中 |
| P1 | Observer and timeline diagnostics | 让播放器行为可解释、可复盘 | 低 |
| P2 | SDK extensibility | 为 data source、demuxer、decoder 替换留边界 | 中 |

## 1. Playback control stability

### CicadaPlayer 观察点

相关入口：

- `mediaPlayer/MediaPlayer.cpp`
- `mediaPlayer/ICicadaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.cpp`
- `mediaPlayer/player_msg_control.cpp`

CicadaPlayer 的外部 API 不是直接在调用线程里做重活，而是把用户动作转成 `MSG_*`
消息，交给 `SuperMediaPlayer::mainService()` 所在的内部循环处理：

```text
MediaPlayer API
  -> ICicadaPlayer
  -> SuperMediaPlayer::putMsg()
  -> PlayerMessageControl::putMsg()
  -> mainService()
  -> processMsg()
```

### 播放器问题是什么

播放器 API 常见问题不是“函数能不能调通”，而是：

- `prepare/start/pause/seek/stop/release` 可能来自不同线程。
- API 调用很快，真实媒体操作很慢。
- 旧的异步结果可能在新命令之后才回来。
- release 期间还可能有 worker 回调、render 回调或内部消息。

如果 API 直接执行重活，状态机会很快变成“谁最后写状态，谁说了算”。

### 这个设计解决什么

控制面消息队列把问题拆成两层：

```text
API 线程
  -> 只表达用户意图

播放器内部线程
  -> 串行解释意图
  -> 决定状态迁移
  -> 发起或取消后台任务
```

这个设计的价值不是“用了队列”，而是让状态修改有一个主语：播放器控制线程。

### 适合迁移到 AVPlayer 的最小版本

第一版只需要这些能力：

- public API 投递 `Prepare/Start/Pause/Seek/Stop/Release` 命令。
- 单一 command thread 串行消费命令。
- command thread 独占状态机写权限。
- worker 只回投 `Prepared/SeekDone/Error` 这类内部事件。
- 每个异步任务携带 `generation` 或 `workSerial`，旧结果被丢弃。
- `release` 进入后，新命令拒绝或只允许幂等收尾。

### 优先级和复杂度

- 优先级：P0。
- 复杂度：中。
- 原因：它是后续 seek、buffering、metrics 的地基。

### C++ 工程注意点

这里需要关注 C++，但重点不是语法，而是线程和生命周期：

- command queue 的 `abort` 必须唤醒等待线程。
- command thread 析构前必须可停止、可 join。
- 状态机最好只在一个线程写。
- observer 回调不要在锁内调用。

## 2. Seek experience

### CicadaPlayer 观察点

相关入口：

- `mediaPlayer/SuperMediaPlayer.cpp`
- `mediaPlayer/player_msg_control.cpp`
- `mediaPlayer/SMPMessageControllerListener.cpp`

`PlayerMessageControl` 对 `MSG_SEEKTO` 不是简单无限入队。它会结合 repeat policy、
seek padding 和 `findMsgByType(MSG_SEEKTO)` 控制连续 seek 的处理时机。

可以理解成三件事：

```text
新 seek 进来
  -> 尽量压缩过期 seek
  -> 如果当前 seek 还没完成，后续 seek 可能 padding
  -> seek 完成时检查队列里是否还有 pending seek
```

### 播放器问题是什么

用户拖动进度条时，播放器会短时间收到大量 seek：

- 如果每个 seek 都执行，解封装和解码会被大量无效工作拖垮。
- 如果旧 seek 完成后直接通知 UI，用户会看到过期位置。
- 如果暂停态 seek 完成后误触发播放，状态语义会错。
- 如果 stop/release 与 seek 交错，旧回调可能污染已释放状态。

### 这个设计解决什么

seek 设计的核心不是“跳到某个时间点”，而是维护用户最新意图：

```text
连续 seek
  -> 旧意图可以丢
  -> 最新意图必须保留
  -> 完成事件必须属于当前 seek
```

### 适合迁移到 AVPlayer 的最小版本

建议拆成一个 `SeekController` 或放在 `PlayerCore` 的 seek 策略里：

- 最新 seek 覆盖旧 pending seek。
- 每个 seek 分配 `seekSerial`。
- worker 回来的 `SeekDone(serial)` 必须匹配当前 serial。
- 暂停态 seek 完成后保持暂停态。
- release/stop 后丢弃所有未完成 seek 结果。
- timeline 记录 `SeekRequested/SeekStarted/SeekDropped/SeekDone`。

### 优先级和复杂度

- 优先级：P0。
- 复杂度：中。
- 原因：seek 是播放器控制面最容易暴露竞态的操作，也是最适合做小闭环验证的功能。

### C++ 工程注意点

这里主要注意序列号和所有权：

- `seekSerial` 使用单调递增整数即可，不需要复杂对象。
- seek 参数尽量用值传递，例如 position、accurate、serial。
- 不要把 seek 参数做成裸指针消息 payload。

## 3. Buffering and QoE

### CicadaPlayer 观察点

相关入口：

- `mediaPlayer/SuperMediaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.cpp`
- `mediaPlayer/buffer_controller.*`
- `mediaPlayer/abr/AbrBufferAlgoStrategy.cpp`
- `framework/cacheModule/CacheManager.cpp`

CicadaPlayer 里 buffering 不是孤立状态。它和 buffer duration、buffer position、
ABR、cache、render loop 都有关：

```text
packet/frame 消费速度
  -> 当前缓存时长
  -> buffering 状态
  -> ABR 或下载策略
  -> 对外状态和统计
```

`AbrBufferAlgoStrategy` 里还会观察 buffer duration 的趋势，而不是只看单次值。

### 播放器问题是什么

只靠“队列是否为空”判断 buffering 很粗糙：

- packet 数量不能代表时长，音视频码率和帧率不同。
- low/high 阈值如果一样，会造成 buffering 状态频繁抖动。
- 卡顿发生后，如果没有统计，就无法判断是网络、解码还是渲染问题。
- cache 和 buffering 如果没有统一指标，很难解释播放体验。

### 这个设计解决什么

更好的思路是把 buffering 看成一个策略决策：

```text
BufferSnapshot
  -> audio/video buffered duration
  -> queue count
  -> eof
  -> download/cache status

BufferController
  -> BufferLow / BufferEnough / BufferFull / KeepCurrent
```

### 适合迁移到 AVPlayer 的最小版本

第一版不做完整 cache，也不做 ABR。只做：

- PacketQueue 统计 packet count 和 buffered duration。
- BufferSnapshot 汇总音频、视频、总缓存时长。
- BufferController 使用 low/high watermark。
- low/high 之间保持当前状态，避免频繁切换。
- Metrics 统计 rebuffer count、rebuffer duration、max rebuffer duration。
- timeline 记录 `BufferLow/BufferEnough/RebufferStart/RebufferEnd`。

### 优先级和复杂度

- 优先级：P1。
- 复杂度：中。
- 原因：依赖前面的控制面和队列基础，但实现范围可以很小。

### C++ 工程注意点

这里的 C++ 重点是快照和线程边界：

- 数据线程更新 queue 状态。
- 控制线程读取不可变 `BufferSnapshot`。
- 不要让 UI 或 observer 直接读内部队列。
- snapshot 用值类型更直观。

## 4. Observer and timeline diagnostics

### CicadaPlayer 观察点

相关入口：

- `mediaPlayer/MediaPlayer.cpp`
- `mediaPlayer/native_cicada_player_def.h`
- `mediaPlayer/analytics/`
- `mediaPlayer/MediaPlayerAnalyticsUtil.*`

CicadaPlayer 的外层 `MediaPlayer` 会接入 listener、analytics collector、ABR manager
等旁路模块。这些模块不应该挤进 decode/render 热路径，而是围绕播放事件和状态变化工作。

### 播放器问题是什么

没有 observer 和 timeline 的播放器很难调试：

- API 返回成功，不代表内部已经完成。
- 用户看到卡顿，但日志里不知道是 buffering、seek 还是 decode 阻塞。
- release 崩溃或竞态通常发生在旧回调、旧线程和旧状态交错时。
- 单看最终状态，看不出命令处理顺序。

### 这个设计解决什么

诊断系统应该回答：

```text
什么时候收到命令？
什么时候开始执行？
什么时候状态变化？
什么时候 worker 回调？
什么时候丢弃旧事件？
什么时候进入/退出 buffering？
```

### 适合迁移到 AVPlayer 的最小版本

第一版只需要轻量 observer 和 timeline：

- `IPlayerObserver` 接收 state、error、buffering、seek done。
- `TimelineEvent` 记录 time、type、serial、state、detail。
- command thread 统一产生 timeline。
- 测试可以断言 timeline 里出现关键事件。
- 日志输出只作为辅助，不作为唯一验证方式。

### 优先级和复杂度

- 优先级：P1。
- 复杂度：低。
- 原因：实现小，但能显著提升项目可解释性。

### C++ 工程注意点

- observer 回调不要持锁调用。
- callback 生命周期要能处理对象释放。
- timeline event 用值类型，避免保存外部对象指针。

## 5. SDK extensibility

### CicadaPlayer 观察点

相关入口：

- `framework/data_source/IDataSource.h`
- `framework/demuxer/IDemuxer.h`
- `framework/codec/IDecoder.h`
- `framework/data_source/dataSourcePrototype.cpp`
- `framework/demuxer/demuxerPrototype.cpp`
- `framework/codec/decoderFactory.cpp`

CicadaPlayer 的 data source、demuxer、decoder、render 都不是直接写死在一个大函数里。
它通过接口、prototype、factory、probe score 组织实现选择。

典型模式：

```text
候选实现注册
  -> probe 当前输入
  -> 选择合适实现
  -> create/clone
  -> 上层只拿接口使用
```

### 播放器问题是什么

播放器天然会遇到多种变化点：

- 本地文件、HTTP、缓存数据源。
- 普通容器、HLS、DASH。
- 软解、硬解、平台解码器。
- SDL、Android、Apple 等不同 render。

如果这些判断散落在播放主循环里，后续扩展会越来越难。

### 这个设计解决什么

factory/prototype 把“如何选择实现”从“如何播放”里拆出去：

```text
播放主线
  -> 只依赖 IDataSource / IDemuxer / IDecoder

选择策略
  -> 根据 URL、header、stream meta、平台能力决定具体实现
```

### 适合迁移到 AVPlayer 的最小版本

AVPlayer 不需要马上做完整插件系统。先做三个小边界：

- `IMediaSource` 或 `IDataSource`：文件源、后续 HTTP 源。
- `IDemuxer`：先只有 FFmpeg 实现，但上层依赖接口。
- `IDecoder`：先只有 fake 或 FFmpeg decoder，后续再考虑平台实现。

factory 第一版可以很简单：

```text
createFfmpegDemuxer(source)
createDefaultDecoder(streamMeta)
```

等出现第二个真实实现后，再引入 probe score。

### 优先级和复杂度

- 优先级：P2。
- 复杂度：中。
- 原因：设计价值高，但过早抽象会拖慢当前播放控制主线。

### C++ 工程注意点

这里才适合补充 C++ 所有权规则：

- factory 返回独占所有权时，用 `unique_ptr`。
- 上层只持有接口指针，不知道具体实现类型。
- 不要为了未来插件化提前做复杂注册表。

## 不建议照抄的地方

这些点适合拿来做“生命周期审查”，但不适合作为 AVPlayer 的新代码模板：

- `void *` 消息参数承载复杂 payload。
- 裸 owning pointer 和手动 `new/delete`。
- 需要在 `recycleMsg()` 里按消息类型手动释放参数。
- 过度依赖裸 `int` 表达错误、状态和 score。
- 大类同时承担 facade、业务模块、状态转发和资源释放。
- 平台差异、业务能力和播放主循环耦合过深。

## 推荐迁移顺序

第一阶段先做控制面：

```text
Playback control stability
  -> Seek experience
  -> Observer and timeline diagnostics
```

原因是这三项互相支撑：

- command thread 让状态写入有序。
- seek serial 验证旧异步事件过滤能力。
- timeline 让行为可观察、可测试。

第二阶段再做数据面体验：

```text
PacketQueue duration
  -> BufferSnapshot
  -> BufferController
  -> Rebuffer metrics
```

第三阶段再考虑扩展点：

```text
IDataSource
  -> IDemuxer
  -> IDecoder
  -> factory/probe score
```

## 公开技术表达

这份文档对外只表达技术学习结论：

```text
CicadaPlayer 的价值不在于照抄源码，而在于提供一个播放器 SDK 工程样本。
适合迁移的是控制面串行化、seek 意图压缩、buffering 决策、播放事件诊断、
以及 data source / demuxer / decoder 的模块边界。
AVPlayer 可以把这些拆成小功能包逐步实现，每个功能包都用测试和 timeline 验证。
```

如果后续需要写私人用途的表达材料，应另存到本地私有目录，不写进仓库。
