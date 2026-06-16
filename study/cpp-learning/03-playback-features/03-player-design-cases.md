# 播放器设计案例：从需求看 CicadaPlayer 的亮点

这一章不按源码文件组织，而是按播放器真实问题组织。目标是把你看到的代码变成可复用的设计判断：

```text
播放器需求
  -> 工程难点
  -> CicadaPlayer 的机制
  -> 值得学习的设计
  -> 不建议照抄的部分
  -> AVPlayer 最小迁移版
```

先只追普通 URL/MP4 的主线。HLS/DASH、DRM、平台硬解只作为旁路提示，不在本章展开。

## 0. 场景总图

```text
SetDataSource / Prepare / Start
  -> PlayerMessageControl
  -> mainService / ProcessVideoLoop
  -> read packet
  -> buffer gate
  -> decode
  -> setup audio/video path
  -> render audio/video
  -> callback / QoE event
```

关键源码入口：

- [SuperMediaPlayer.cpp](../../../mediaPlayer/SuperMediaPlayer.cpp)：播放主循环、buffering、seek、render、错误处理。
- [SMPMessageControllerListener.cpp](../../../mediaPlayer/SMPMessageControllerListener.cpp)：控制命令如何改变播放器状态。
- [buffer_controller.cpp](../../../mediaPlayer/buffer_controller.cpp)：packet queue 和 buffer duration。
- [ActiveDecoder.cpp](../../../framework/codec/ActiveDecoder.cpp)：decoder 内部输入/输出队列和异步解码。
- [SMPAVDeviceManager.cpp](../../../mediaPlayer/SMPAVDeviceManager.cpp)：decoder/render 聚合和 flush 边界。

## 1. 起播和首帧：不是 start 后立刻 render

### 需求是什么

用户调用 `start()` 后，希望尽快看到首帧、听到声音，同时又不能因为缓存太少马上卡顿。

### 难点是什么

起播需要同时满足几件事：

- data source 和 demuxer 已经打开。
- 至少拿到有效 stream meta 或 packet。
- decoder 可以建立。
- audio/video render 需要真实 frame 格式。
- buffer 达到起播门槛，或者 EOF 等特殊条件允许继续。

所以首帧不是单点事件，而是 data path 多个阶段陆续准备好的结果。

### CicadaPlayer 怎么做

主线可以这样看：

```text
Prepare / Start command
  -> ProcessPrepareMsg / ProcessStartMsg
  -> demuxer_service start
  -> ProcessVideoLoop
      -> doReadPacket
      -> doDeCode
      -> setUpAVPath
      -> DoCheckBufferPass
      -> render
      -> RenderCallback
      -> NotifyFirstFrame
```

`SetUpAudioPath()` 会等 audio packet 和 audio frame 信息，`SetUpVideoPath()` 会等 video packet/meta/parser
结果。`DoCheckBufferPass()` 会在 render 前判断起播 buffer 是否足够。

### 值得学习

- 起播是一个流水线成熟过程，不是一个函数同步完成。
- 首帧通知从 render callback 往外发，比较接近“用户真的看到/听到”的时间点。
- decoder/render 延迟创建，依赖真实流信息，避免 prepare 阶段过早假设格式。

### 不建议照抄

- 起播相关状态散落在 `SuperMediaPlayer`、message listener、render callback 中，阅读成本高。
- AVPlayer 里可以把首帧拆成更清晰的事件：`DemuxReady`、`DecoderReady`、`FirstFrameDecoded`、`FirstFrameRendered`。

### AVPlayer 最小迁移版

```text
StartupTracker
  demuxReady
  firstPacketRead
  decoderReady
  firstFrameDecoded
  firstFrameRendered
```

第一版先不做复杂 QoE，只记录这些时间点并打印 timeline log，就能明显提升调试能力。

## 2. Seek：真正难的是清旧数据和等新数据

### 需求是什么

用户拖动进度条后，播放器要尽快跳到目标位置，并且不能播放 seek 前的旧 packet/frame。

### 难点是什么

seek 不是只调用 demuxer 的 `Seek()`：

- packet queue 里可能有旧 packet。
- decoder 内部可能有旧输入和旧输出。
- frame queue 里可能有旧 frame。
- audio render 设备队列里可能还有旧音频。
- render callback 可能晚到，导致 position 回退。
- 暂停态 seek 也要更新位置和首帧预览。

### CicadaPlayer 怎么做

主线入口在 `SMPMessageControllerListener::ProcessSeekToMsg()` 和 `SuperMediaPlayer` 的 seek 分支：

```text
SeekTo()
  -> put MSG_SEEKTO
  -> ProcessSeekToMsg
      -> set seek position / seek flag
      -> check SeekInCache
      -> flush packet queue
      -> demuxer seek when needed
      -> reset clock
  -> ProcessVideoLoop
      -> drop stale packet/frame
      -> wait first valid frame
      -> ResetSeekStatus
      -> NotifySeekEnd
```

### 值得学习

- seek 被放进控制消息队列，避免 API 线程直接做重活。
- 有 `mSeekFlag`、`mSeekNeedCatch`、`mSeekInCache` 这类状态来区分 seek 阶段。
- seek 后会清 packet、decoder、frame、render 相关状态，避免旧数据漏出来。

### 不建议照抄

- seek 状态变量较多，语义需要结合上下文理解。
- AVPlayer 应该显式引入 `seekSerial` 或 `generation`，让旧回调天然失效，而不是只靠多个 bool。

### AVPlayer 最小迁移版

```text
struct SeekContext {
  int64_t targetUs;
  uint64_t serial;
  bool accurate;
};

onSeek:
  serial++
  clear packet queues
  flush decoders
  clear frame queues
  reset clock to target

onFrame:
  if frame.serial != currentSerial:
    drop
```

## 3. Buffering 和弱网：核心指标是可播放时长

### 需求是什么

网络抖动时，播放器既不能无限缓存，也不能刚有一点数据就播放导致反复 loading。

### 难点是什么

packet 数量不是可靠指标：

```text
20 个 audio packet 和 20 个 video packet
  -> 覆盖的播放时长可能完全不同
```

播放器更应该关心 buffer duration。

### CicadaPlayer 怎么做

```text
ReadPacket
  -> BufferController::AddPacket
  -> MediaPacketQueue records duration
  -> getPlayerBufferDuration
  -> DoCheckBufferPass
      -> start loading
      -> end loading
      -> notify demuxer client buffer level
```

`doReadPacket()` 根据 buffer 是否满、低内存、读取超时来控制拉包。`DoCheckBufferPass()` 决定 render 前是否放行。

### 值得学习

- 用 buffer duration 而不是 packet count 做播放决策。
- 起播水位、恢复水位、最大水位是不同概念。
- buffer 状态反向影响 demuxer 读取和下载策略，这是完整播放器的工程亮点。

### 不建议照抄

- buffer 策略和播放器状态耦合较深。
- AVPlayer 可以先抽成 `BufferPolicy`，输入 buffer snapshot，输出 `canRead`、`canRender`、`shouldNotifyLoading`。

### AVPlayer 最小迁移版

```text
BufferSnapshot:
  audioPacketDurationUs
  videoPacketDurationUs
  demuxerBufferedUs
  decoderPendingUs
  eof

BufferPolicy:
  canRead()
  canRender()
  loadingTransition()
```

## 4. EOF 和完成：不是 demuxer EOF 就立即 completion

### 需求是什么

文件读到结尾后，播放器要等已经读出来的数据都播放完，再通知 completion。

### 难点是什么

EOF 只说明 demuxer 没有更多 packet，不代表：

- packet queue 空了。
- decoder 没有缓存帧。
- frame queue 空了。
- audio render 设备已经播完。

### CicadaPlayer 怎么做

`doReadPacket()` 看到 EOF 后会记录 `mEof` 并通知 demuxer EOF 事件。后续 `doDeCode()`、`RenderAudio()`、
`RenderVideo()` 继续消费剩余 packet/frame。最终在合适时机切到 `PLAYER_COMPLETION`。

### 值得学习

- 把 demux EOF 和 playback completion 区分开。
- completion 应该由“数据面被消费完”驱动，而不是由“读取结束”直接驱动。

### 不建议照抄

- EOF 分支和 seek、prepare、decoder EOS 混在一起，第一轮阅读容易迷路。
- AVPlayer 可以定义明确的完成条件：`demuxEof && packetQueuesEmpty && decodersDrained && frameQueuesEmpty && renderDrained`。

## 5. 错误恢复：错误要分层，不要全变成 unknown

### 需求是什么

播放器要把网络、demux、decoder、render 的错误转成外部能理解的错误或事件。

### 难点是什么

不同层的错误语义不同：

- 网络错误可能可以重试。
- demux open stream 失败可能是格式或 URL 问题。
- decoder 失败可能可以 fallback 到软解。
- audio render 打不开设备，通常需要通知上层。

### CicadaPlayer 怎么做

`NotifyError()` 会把 framework error 映射成 media player error。代码里也能看到 codec not support、render init error、
network retry、system low memory 等事件。

### 值得学习

- 错误不是只有一个 `-1`，最终要映射到播放器外部 API 能理解的错误域。
- 有些问题是 fatal error，有些是 event，比如硬解失败后切软解。

### 不建议照抄

- 内部仍有不少 `int ret` 和 magic status 组合，读起来不够类型安全。
- AVPlayer 可以用 `enum class ErrorDomain`、`enum class ErrorCode` 和小型 `Result<T>` 表达。

## 6. 低内存：播放器要能降级，而不是继续堆缓存

### 需求是什么

端侧播放器必须考虑内存压力。低内存时继续拉流、堆 packet、堆 frame，可能直接导致 OOM。

### CicadaPlayer 怎么做

`doReadPacket()` 里会检查低内存，并调整 `highLevelBufferDuration`、`startBufferDuration` 等缓存目标，同时发出
`MEDIA_PLAYER_EVENT_SYSTEM_LOW_MEMORY`。

### 值得学习

- 低内存不是单纯通知 UI，而是会改变读取策略。
- 缓存策略应该能动态降级。

### AVPlayer 最小迁移版

```text
onLowMemory:
  maxBufferDurationUs = min(maxBufferDurationUs, lowMemoryCap)
  startBufferDurationUs = min(startBufferDurationUs, lowMemoryStartCap)
  dropNonEssentialCaches()
  notify event
```

## 7. A/V Sync 和丢帧：video 不应该按最快速度显示

### 需求是什么

视频要跟音频同步。视频解码出来得早，要等；来得晚，要考虑丢帧追赶。

### CicadaPlayer 怎么做

```text
RenderAudio
  -> audio render
  -> getAudioPlayTimeStampCB
  -> master clock

RenderVideo
  -> compare video pts with master clock
  -> wait / render / drop
```

### 值得学习

- audio render 的实际播放位置比系统时间更适合作为 master clock。
- video render 是调度问题，不是“解出来就显示”。

### 不建议照抄

- 同步分支、drop 分支、seek catch 分支交织在大函数里。
- AVPlayer 可以先实现一个 `RenderScheduler`，只负责输入 `videoPts` 和 `clockNow`，输出 `Wait/Render/Drop`。

## 8. 这一章怎么用

每读一个播放器场景，都输出四件东西：

```text
需求一句话
机制图
优秀设计 / 风险点
AVPlayer 最小迁移版
```

不要先追所有源码细节。先把场景讲清，再回到源码验证。
