# 普通 URL/MP4 播放循环：一轮 mainService 到底做了什么

这一章只追普通 URL/MP4，不展开 HLS/DASH、DRM、平台硬解。目标是把你现在看到的模块串成一条
真实执行主线：

```text
API command
  -> PlayerMessageControl
  -> SuperMediaPlayer::mainService()
  -> ProcessVideoLoop()
  -> bytes / packet / frame / render / clock
```

如果只看 `PlayerMessageControl`，你会知道命令如何进入播放器；但真正拉流、解复用、解码、同步、
渲染是在 `ProcessVideoLoop()` 里按阶段推进的。

## 先看一轮循环的骨架

源码入口：`mediaPlayer/SuperMediaPlayer.cpp`

```text
mainService()                      around line 1157
  -> processMsg()                  around line 1170
  -> ProcessVideoLoop()            around line 1171
      -> doReadPacket()            around line 1237
      -> doDeCode()                around line 1238
      -> setUpAVPath()             around line 1245
      -> DoCheckBufferPass()       around line 1247
      -> startRendering()          around line 1251
      -> doRender()                around line 1259
```

这不是简单的“读一帧、解一帧、显示一帧”。它更像一个小调度器：每轮尽量推进几个阶段，但每个阶段
都可能因为状态、水位、队列、clock 或设备未准备好而停下来。

## mainService：先处理控制面，再推进数据面

`mainService()` 每轮先看消息队列：

```text
if message queue empty:
  run ProcessVideoLoop()
else if processMsg() returns 0:
  run ProcessVideoLoop()
```

这里的含义是：控制命令优先级高，但命令处理完以后，主循环仍然要继续推进数据面。比如 seek 命令本身
只是改变目标位置、flush 旧数据、设置状态；seek 后真正读到新 packet、解出新 frame、重新 render，
还是靠后续 `ProcessVideoLoop()` 一轮轮推进。

`updateLoopGap()` 和 `wait_for()` 决定下一轮睡多久。它不是固定 40ms 睡眠：追帧、seek、buffer 有数据时
会更积极地跑，普通状态下会留出等待时间，避免空转。

## ProcessVideoLoop：播放器数据面的调度顺序

`ProcessVideoLoop()` 先检查状态：

```text
not preparing/prepared/playing/paused/completion:
  only timer tick, then return

no demuxer service:
  only timer tick, then return
```

这说明 `SuperMediaPlayer` 的主循环一直存在，但不是所有状态都会推进数据面。只有播放器已经进入可工作状态，
并且 demuxer 已经打开后，才会走下面的主路径。

主路径顺序是固定的：

```text
read packet first
decode packet second
set up decoder/render path third
check buffering before render
render at the end
```

这个顺序背后的播放器原理是：

- 先读包：保证 packet queue 有输入，buffering 和 decoder 才有材料。
- 再解码：把 packet 转成 frame，让 render 前有已解码帧。
- 再 setup：有些设备必须等到 packet/meta/frame 出现后才能创建。
- 再检查 buffer：不是有 frame 就一定能播，缓存不足时要停住渲染。
- 最后 render：render 必须服从 clock，尤其是 video 不能按最快速度显示。

## 阶段一：doReadPacket 是 pull 模型，不是 data source 主动推

`doReadPacket()` 大致逻辑：

```text
if EOF:
  return

while true:
  if player buffer is full:
    break
  if low memory:
    shrink buffer target and break
  ret = ReadPacket()
  if EAGAIN / EOF / error:
    break
```

这里体现了播放器里的 pull 模型：不是网络线程主动把数据推满所有队列，而是 player loop 根据当前 buffer
水位主动向 demuxer 要 packet。

关键约束：

- `maxBufferDuration` 控制最多读多少。
- `startBufferDuration` 保证起播或恢复播放前至少保留一定缓存。
- `mBufferIsFull` 表示 player 侧 packet queue 已经够多，下一轮等消费掉一些再读。
- 低内存时会降低 `highLevelBufferDuration` 和 `startBufferDuration`，避免继续堆数据。

所以 `doReadPacket()` 不只是“调用 av_read_frame”。它是读取节流器：决定什么时候继续拉，什么时候停。

## 阶段二：ReadPacket 把 demuxer 输出接进 BufferController

`ReadPacket()` 的关键点：

```text
demuxer_service::readPacket(unique_ptr<IAFPacket>&)
  -> returns one IAFPacket
  -> inspect streamIndex / pts / timePosition
  -> add to BufferController audio/video/subtitle queue
```

`ReadPacket()` 拿到的是 `IAFPacket`，还不是 frame。它会做几类事情：

- 初始化 PTS discontinuity 阈值，用于后面判断时间戳跳变。
- 记录 `timePosition`，用于当前位置、seek catch、统计。
- 根据 stream index 判断 audio/video/subtitle。
- 把 packet 放入 `BufferController` 对应队列。

这里要分清两层：

```text
demuxer_service / avFormatDemuxer：从 bytes 解析 packet
BufferController：保存 packet，服务 buffering / seek / drop / decode input
```

普通 MP4 的字节读取发生在更下层：`avFormatDemuxer` 通过 FFmpeg AVIO callback 拉
`IDataSource::Read()`。所以你在 `SuperMediaPlayer::ReadPacket()` 里看不到直接网络读取，这是正常的。

## 阶段三：doDeCode 不是无限解码，它受 frame queue 和 decoder 背压限制

`doDeCode()` 分 video 和 audio 两段。

video 侧先看 frame queue 是否还能放：

```text
if video frame queue size < max cache size:
  get packet from BufferController if no pending packet
  FillVideoFrame()
  DecodeVideoPacket(pending packet)
  if decoder says STATUS_RETRY_IN:
    keep pending packet and break
```

这里最重要的是 pending packet：`mVideoPacket` / `mAudioPacket` 表示“已经从 packet queue 取出，但
decoder 还没成功吃掉”。如果 decoder 输入队列满了，不能把这个 packet 丢掉，只能下轮继续送。

audio 侧更保守：

```text
while audio frame queue size < 2:
  get audio packet
  DecodeAudio(pending packet)
```

原因是 audio render 自己也有设备队列，`SuperMediaPlayer` 不需要在 frame queue 里堆太多 audio frame。
video 则要考虑画面缓存、追帧和丢帧。

这一层的背压链路是：

```text
render 慢
  -> frame queue 变满
  -> doDeCode 停止取 packet
  -> BufferController packet queue 消费变慢
  -> doReadPacket 看到 buffer duration 变高
  -> 停止从 demuxer 拉包
```

这就是模块关联性：render 的速度会反向影响拉流速度。

## 阶段四：setUpAVPath 为什么放在 decode 后

`setUpAVPath()` 会分别尝试 `SetUpAudioPath()` 和 `SetUpVideoPath()`。

audio path 的约束：

```text
no audio packet:
  cannot get reliable stream meta yet

no audio frame:
  cannot create audio render yet
```

源码里 `SetUpAudioPath()` 先等 audio packet，再创建 audio decoder；audio render 则等第一帧 audio frame，
因为 render 需要真实采样率、声道、sample format 等 frame 信息。

video path 的约束：

```text
no video packet:
  wait

video interlaced type unknown:
  wait parser result

meta ready:
  create video decoder/render
```

所以 decoder/render 不是 prepare 时一次性全创建。播放器经常要等真实流信息出现后再建路径，尤其是
HLS/DASH 或码流信息延迟出现时。普通 MP4 也沿用这套机制。

## 阶段五：DoCheckBufferPass 是 render 前的闸门

`DoCheckBufferPass()` 的核心不是“更新一下 buffer 状态”，而是决定本轮是否允许进入 render。

它会看：

- 当前 packet buffer duration。
- 起播阶段用 `startBufferDuration`，普通播放用 `highLevelBufferDuration`。
- demuxer client buffer level：low / normal / low_full。
- 首次 buffering 时是否要清掉 seek 后过晚或过早的数据。
- prepare 状态是否可以变成 prepared。

如果它返回 `false`，`ProcessVideoLoop()` 会直接 return，本轮不会 render。

这解释了一个常见问题：为什么 frame queue 里可能已经有帧，但播放器还不显示？因为播放不是只看 frame。
它还要看整体 buffer 水位和状态机。否则网络抖动时会刚有一点数据就播，很快又卡住。

## 阶段六：render 先 audio 后 video，video 服从 clock

`render()` 的顺序：

```text
RenderAudio()
RenderVideo()
RenderSubtitle()
```

audio 先被送入 audio render。首次有效 audio frame 后，master clock 会绑定 audio render 的播放时间：

```text
mMasterClock.setReferenceClock(getAudioPlayTimeStampCB, this)
```

这就是 audio master：真正从声卡播放出去的进度，比单纯系统时间更适合作为“现在播放到哪里”的基准。

video render 时拿视频 PTS 和 master clock 比：

```text
videoLateUs = masterClockTime - videoPts - videoDelay
```

然后分三种情况：

```text
video too early:
  return false, next loop try again

video acceptable:
  SendVideoFrameToRender()

video too late:
  drop frame or clear older packet, enter catching-up
```

这就是同步原理的主线：audio 推进时间，video 对齐时间。video 不应该因为 decode 已经完成就立刻显示。

## 用一个 frame 串起来

一帧普通 MP4 视频大致经历：

```text
IDataSource::Read()
  -> avFormatDemuxer / av_read_frame()
  -> IAFPacket(video)
  -> BufferController video packet queue
  -> mVideoPacket pending packet
  -> IDecoder::send_packet()
  -> ActiveDecoder input queue
  -> decoder thread
  -> ActiveDecoder output queue
  -> mVideoFrameQue
  -> RenderVideo()
  -> compare PTS with master clock
  -> SendVideoFrameToRender()
  -> RenderCallback()
```

一帧 audio 类似，但 render 后还会成为 master clock 的参考。

## 这条主线和后续文档怎么对应

```text
对象关系：
  00-object-model-and-module-wiring.md

bytes 从哪里来：
  02-data-source-cache.md

packet 怎么产生：
  03-demuxer-prototype.md

packet/frame queue 怎么连接：
  04-packet-frame-queue.md

clock/render 怎么输出：
  05-decode-sync-render.md
```

读源码时不要再按目录乱跳。先把函数放回这一轮循环里，再判断它属于 read、decode、setup、buffer gate、
render 里的哪一段。

## 这一章真正要掌握什么

- `mainService()` 是控制面和数据面的汇合点。
- `ProcessVideoLoop()` 是普通播放主循环，不是单纯 video loop。
- 拉流是 player loop pull demuxer，demuxer 再通过 callback pull data source。
- `BufferController` 是 packet 层缓冲，不是 frame queue。
- decoder 有自己的输入/输出队列，pending packet 用来处理 decoder 背压。
- `DoCheckBufferPass()` 是 render 前闸门，buffer 不够时即使有 frame 也可能不 render。
- audio render 提供 master clock，video 根据 clock 等待、渲染或丢帧。
