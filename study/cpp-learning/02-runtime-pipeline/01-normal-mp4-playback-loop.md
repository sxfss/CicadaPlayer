# 普通 URL/MP4 播放循环：从命令到渲染的一轮执行

这一章只追普通 URL/MP4，不展开 HLS/DASH、DRM、平台硬解。它的目标不是把
`SuperMediaPlayer.cpp` 里的函数名列一遍，而是回答一个更核心的问题：

```text
用户调用 play 之后，播放器到底如何一轮轮把网络/文件里的 bytes
变成 packet、frame，并最终按 clock 渲染出去？
```

读完这一章，你应该能画出三张图：

- 执行流图：控制命令如何进入主循环，主循环如何推进 read/decode/render。
- 数据形态图：`bytes -> packet -> frame -> render` 分别由哪个模块负责。
- 反压图：render 慢为什么会反过来让 read packet 停下来。

## 0. 先看主图

普通 URL/MP4 的主路径可以先压缩成这一张图：

```text
MediaPlayer API
  -> C API handle
  -> ICicadaPlayer / SuperMediaPlayer
  -> PlayerMessageControl
  -> SuperMediaPlayer::mainService()
      -> processMsg()
      -> ProcessVideoLoop()
          -> doReadPacket()
              -> demuxer_service::readPacket()
              -> avFormatDemuxer / av_read_frame()
              -> IDataSource::Read()
              -> BufferController::AddPacket()
          -> doDeCode()
              -> BufferController::getPacket()
              -> IDecoder::send_packet()
              -> IDecoder::getFrame()
              -> mAudioFrameQue / mVideoFrameQue
          -> setUpAVPath()
              -> setup decoder / audio render / video render
          -> DoCheckBufferPass()
              -> decide buffering or render allowed
          -> doRender()
              -> RenderAudio()
              -> RenderVideo()
              -> RenderCallback()
```

这张图里最重要的点是：`PlayerMessageControl` 只是控制面入口。真正的拉流、解复用、解码、同步和渲染，
是在 `ProcessVideoLoop()` 里按阶段推进的。

源码入口集中在 [mediaPlayer/SuperMediaPlayer.cpp](../../../mediaPlayer/SuperMediaPlayer.cpp)。
VS Code 本地 Markdown 预览对行号跳转支持不稳定，所以这里保留可打开文件的相对链接，并把当前行号作为阅读提示：

- `SuperMediaPlayer::mainService()`，当前约 `1157` 行：播放主线程入口，先处理控制面，再推进数据面。
- `SuperMediaPlayer::ProcessVideoLoop()`，当前约 `1204` 行：普通播放数据面的主调度函数。
- `SuperMediaPlayer::doReadPacket()`，当前约 `1314` 行：按 buffer 水位和状态决定是否继续拉 packet。
- `SuperMediaPlayer::ReadPacket()`，当前约 `2959` 行：把 demuxer 输出的 `IAFPacket` 接入 `BufferController`。
- `SuperMediaPlayer::doDeCode()`，当前约 `1946` 行：从 packet queue 推进到 decoder 和 frame queue。
- `SuperMediaPlayer::SetUpAudioPath()`，当前约 `3594` 行：在 audio packet/frame 信息可用后建立音频路径。
- `SuperMediaPlayer::SetUpVideoPath()`，当前约 `3690` 行：在 video packet/meta 信息可用后建立视频路径。
- `SuperMediaPlayer::DoCheckBufferPass()`，当前约 `1452` 行：render 前的 buffering 闸门。
- `SuperMediaPlayer::render()`，当前约 `2329` 行：统一调度 audio/video/subtitle render。
- `SuperMediaPlayer::RenderAudio()`，当前约 `2369` 行：送入音频渲染，并让 audio render 成为 master clock 参考。
- `SuperMediaPlayer::RenderVideo()`，当前约 `2507` 行：按 master clock 判断 video frame 是等待、渲染还是丢弃。

## 1. 这一轮循环解决什么播放器问题？

播放器主循环不能写成简单的：

```text
while playing:
  read one packet
  decode one packet
  render one frame
```

真实播放器至少要同时处理这些问题：

- 控制命令随时可能插进来，比如 pause、seek、stop。
- data source 或 demuxer 可能暂时没数据，不能卡死主线程。
- packet queue 可能已经足够多，继续拉流会浪费内存。
- decoder 可能暂时吃不下 packet，需要保留 pending packet。
- frame queue 可能满了，继续 decode 没意义。
- 有 frame 不代表能显示，还要看 buffering 状态和 master clock。
- video 不能按 decode 速度显示，必须跟 audio clock 对齐。

所以 `ProcessVideoLoop()` 更像一个小调度器：每轮尝试推进多个阶段，但每个阶段都可能因为状态、队列、
水位、clock 或设备未准备好而暂停。

## 2. mainService：控制面和数据面的交汇点

播放器 API 不是直接在调用线程里做重活，而是把命令投递给 `PlayerMessageControl`。主线程函数
`mainService()` 每轮先处理控制命令，再推进数据面。

简化逻辑是：

```text
if message queue has message:
  processMsg()

if message processing allows data loop:
  ProcessVideoLoop()

wait next loop gap
```

这解释了 seek 这类命令的执行方式：

```text
seek command
  -> 更新目标时间
  -> flush 旧 packet/frame/decoder 状态
  -> 调整播放状态
  -> 后续 ProcessVideoLoop() 再读新位置的数据
```

seek 本身不是“立刻解出目标帧”。它只是改变主循环的意图和状态。真正读到新 packet、解出新 frame、
重新渲染，仍然靠后续一轮轮数据面推进。

设计点评：

- 值得学习：API 线程和播放主循环解耦，避免用户调用 `seek()` 时直接阻塞在 demux/decode/render 上。
- 需要警惕：控制状态分散在较大的 `SuperMediaPlayer` 里，读源码时要按阶段分组，不要按成员变量平铺记忆。

## 3. ProcessVideoLoop：不是 video loop，而是播放数据面调度器

名字叫 `ProcessVideoLoop()`，但它并不只处理 video。它负责普通播放数据面的主要推进顺序：

```text
read packet
  -> decode packet
  -> setup decoder/render path
  -> check buffering gate
  -> render audio/video/subtitle
```

每个阶段的意义不同：

```text
doReadPacket()
  让 demuxer 继续吐 packet，补充 packet queue。

doDeCode()
  从 packet queue 取 packet，推进 decoder 产出 frame。

setUpAVPath()
  等真实 packet/frame/meta 到位后再创建 decoder/render。

DoCheckBufferPass()
  判断当前缓存是否允许播放，必要时停在 buffering。

doRender()
  把 frame 按 master clock 输出到 audio/video render。
```

这套顺序的播放器含义是：先补充输入，再尽量产出 frame，然后在渲染前统一做 buffer gate 和 clock gate。

不要把它理解为“每轮一定完成一整套 read-decode-render”。更准确的理解是：

```text
每轮只推进能推进的部分；
推进不了就停在当前阶段；
下一轮根据新状态继续尝试。
```

## 4. doReadPacket：pull 模型和读取反压

### 播放器问题

data source 不应该无限把数据往播放器里推。否则网络快、解码慢或渲染慢时，packet 会越堆越多，内存不可控。

播放器需要一个读取节流点：

```text
缓存不够 -> 继续向 demuxer 要 packet
缓存足够 -> 暂停读取
内存紧张 -> 降低缓存目标
EOF/EAGAIN/error -> 停止本轮读取
```

### CicadaPlayer 做法

`doReadPacket()` 大致是一个 pull loop：

```text
while true:
  if EOF:
    break

  if player buffer is full:
    break

  if low memory:
    shrink buffer target
    break

  ret = ReadPacket()

  if ret is EAGAIN / EOF / error:
    break
```

这里的重点是：player loop 主动向 demuxer 拉 packet。demuxer 再通过 FFmpeg/AVIO callback 向
`IDataSource::Read()` 拉 bytes。普通 URL/MP4 下，它不是一个 data source 主动 push 到 render 的模型。

```text
ProcessVideoLoop
  pulls demuxer_service::readPacket()
    pulls avFormatDemuxer / av_read_frame()
      pulls IDataSource::Read()
```

### 源码轨迹

```text
doReadPacket()
  -> ReadPacket()
      -> demuxer_service::readPacket(unique_ptr<IAFPacket>&)
      -> inspect stream index / pts / timePosition
      -> BufferController::AddPacket()
```

`ReadPacket()` 拿到的是 `IAFPacket`，不是 decoded frame。它会按 stream index 放入 audio/video/subtitle
packet queue。

### 关键条件表

| 条件 | CicadaPlayer 行为 | 学习重点 |
| --- | --- | --- |
| `mEof == true` | 直接返回，不再读 demuxer | demux EOF 后不代表播放完成，只是读取阶段停止 |
| `mBufferIsFull` 且 buffer 仍接近上限 | 停止本轮读取 | 下游没消费掉足够数据前，不继续拉流 |
| `cur_buffer_duration > maxBufferDuration` 且起播 buffer 仍够 | 设置 `mBufferIsFull`，停止读取 | `maxBufferDuration` 是读取反压阈值 |
| 系统低内存 | 降低 `highLevelBufferDuration` / `startBufferDuration`，发低内存事件，停止读取 | 低内存会改变缓存策略，不只是通知 UI |
| `ReadPacket()` 返回 `-EAGAIN` | 本轮停止读取，等待后续数据 | 网络/直播暂时没数据，不应该进入错误态 |
| `ReadPacket()` 返回 `0` | 标记 `mEof`，通知 demuxer EOF | demux EOF 是数据源结束，不等于 render 已完成 |
| `ReadPacket()` 返回格式不支持或其他 fatal error | `NotifyError()` 或停止本轮读取 | 读取错误要分层映射到播放器错误 |
| 单轮读取超过 timeout | 停止本轮读取 | 主循环不能长时间卡在读取阶段 |

### 设计点评

- 值得学习：把读取节流放在 player loop 里，可以用同一套 buffer duration、EOF、low memory、seek 状态来控制拉流节奏。
- 不要照抄：`SuperMediaPlayer` 同时承担太多判断。你自己的 AVPlayer 可以把 buffer snapshot 和 read throttle 单独抽出来，主循环只消费结果。

## 5. ReadPacket：bytes 到 packet 的边界

这一层容易误解：你在 `SuperMediaPlayer::ReadPacket()` 里看不到直接网络读取，并不代表它没有拉流。

普通 MP4 的数据形态变化是：

```text
IDataSource::Read()
  -> bytes
  -> avFormatDemuxer / av_read_frame()
  -> IAFPacket
  -> BufferController
```

职责边界是：

```text
IDataSource
  负责从 URL/file/cache 读取 bytes。

IDemuxer / avFormatDemuxer
  负责理解容器格式，把 bytes 切成 audio/video/subtitle packet。

BufferController
  负责保存 packet，并提供 buffering、seek、drop、decode input 所需的查询和取包能力。
```

这也是为什么后续文档要分开看 data source、demuxer、packet queue。它们不是同一个概念。

## 6. doDeCode：受 frame queue 和 decoder 背压限制

### 播放器问题

decode 也不能无限跑。原因有两个：

- frame queue 满了，继续 decode 只会堆 frame。
- decoder 可能暂时不能接受更多 packet，需要下轮再送同一个 packet。

### CicadaPlayer 做法

video 侧大致是：

```text
if video frame queue has room:
  if no pending video packet:
    get packet from BufferController

  FillVideoFrame()
  DecodeVideoPacket(pending packet)

  if decoder says STATUS_RETRY_IN:
    keep pending packet for next loop
```

audio 侧通常更保守：

```text
while audio frame queue size < small threshold:
  get audio packet
  DecodeAudio(pending packet)
```

这里的关键概念是 pending packet：

```text
mVideoPacket / mAudioPacket
  = 已经从 BufferController 取出
  = 但还没有被 decoder 成功消费
  = 不能丢，只能下轮继续送
```

### 关键条件表

| 条件 | CicadaPlayer 行为 | 学习重点 |
| --- | --- | --- |
| video decoder 无效或 video EOS | 不推进 video decode | decoder path 未建立或已结束时，不应继续取 video packet |
| video frame queue 未满 | 才尝试取 video packet 并 decode | frame queue 是 decode 的下游背压 |
| `mVideoPacket == nullptr` | 从 `BufferController` 取一个 video packet | packet 所有权从 packet queue 移到 pending packet |
| decoder 返回 `STATUS_RETRY_IN` | 保留 `mVideoPacket`，下轮继续送 | decoder 没吃掉 packet 时不能丢 |
| 单轮 video decode 超过约 50ms | 停止本轮 decode | 主循环要避免被 decode 阶段长期占用 |
| audio frame queue 小于小阈值 | 才继续取 audio packet decode | audio render 设备也有队列，不需要在 frame queue 堆太多 |
| audio packet 为空且未 EOF | 停止 audio decode | 暂时没有输入，不是错误 |
| audio packet 为空且 EOF | 送空 packet 推进 decoder drain | EOF 后还要把 decoder 内部缓存 frame 排出来 |

### 反压图

这一段最值得学习的是反压链路：

```text
render 慢
  -> frame queue 变满
  -> doDeCode 停止取 packet
  -> BufferController packet queue 消费变慢
  -> packet buffer duration 变高
  -> doReadPacket 停止从 demuxer 拉包
  -> IDataSource 读取自然降速
```

这就是播放器模块之间真正的关联性。它不是“read、decode、render 三个函数排队调用”这么简单，而是下游消费速度
会反过来控制上游读取速度。

### 设计点评

- 值得学习：用 packet queue、frame queue、pending packet 表达不同阶段的所有权和背压。
- 不要照抄：如果你自己实现，可以把 pending packet 的语义封装成更小的 decoder input adapter，减少主循环里散落的状态判断。

## 7. setUpAVPath：为什么不是 prepare 时一次性建好

很多初学播放器时会以为：

```text
prepare
  -> open demuxer
  -> create decoder
  -> create render
```

但真实工程里经常不能这么简单。原因是一些关键信息要等数据流推进后才可靠：

```text
audio packet/meta
  -> 才能创建 audio decoder

first audio frame
  -> 才知道真实 sample rate / channels / sample format
  -> 才能创建 audio render

video packet/meta/parser result
  -> 才能创建 video decoder/render
```

所以 `setUpAVPath()` 放在 decode 附近是有意义的。它不是纯初始化函数，而是随着数据面逐步成熟，补齐 decoder
和 render 路径。

设计点评：

- 值得学习：延迟创建 decoder/render，让路径建立依赖真实流信息，而不是只依赖外部传入 URL。
- 需要警惕：延迟创建会让状态机更复杂。文档和代码里必须讲清“哪类对象何时可用”，否则读者会以为 prepare 后所有对象都已存在。

## 8. DoCheckBufferPass：有 frame 也不一定能播

### 播放器问题

如果 frame queue 里刚有一点数据就立刻播放，弱网下很容易出现：

```text
刚显示几帧
  -> 数据耗尽
  -> loading
  -> 又攒一点
  -> 又播放
  -> 反复卡顿
```

所以播放器要在 render 前做一次 buffer gate。

### CicadaPlayer 做法

`DoCheckBufferPass()` 会综合判断：

- 当前 packet buffer duration。
- 起播阶段的 `startBufferDuration`。
- 播放中恢复用的 `highLevelBufferDuration`。
- 接近上限时的 `maxBufferDuration`。
- demuxer client buffer level。
- seek 后是否需要清理过早或过晚的数据。
- prepared/loading 状态是否需要切换。

如果它返回 `false`，本轮 `ProcessVideoLoop()` 不会继续 render。

### 关键条件表

| 条件 | CicadaPlayer 行为 | 学习重点 |
| --- | --- | --- |
| buffer duration 低于 high level 且正在播放 | demuxer client buffer level 设为 low | 播放器 buffer 状态会反向影响 demuxer/下载策略 |
| buffer 接近 max buffer | demuxer client buffer level 设为 low full | 缓存太多也要反馈给上游 |
| `mFirstBufferFlag` 且未 EOF | 使用 `startBufferDuration` 作为本轮阈值 | 起播水位和播放中恢复水位可以不同 |
| preparing 且 buffer 达到阈值或 EOF | 切到 `PLAYER_PREPARED`，通知 prepared | prepared 不是只由 open 成功决定，还看缓冲/流状态 |
| 播放中 buffer duration `<= 0` | 进入 buffering，暂停 master clock 和 audio render，返回 false | loading 是播放状态变化，不只是 UI 事件 |
| buffering 中 buffer 重新超过阈值或 EOF | 结束 loading，恢复 clock/audio render | 退出 buffering 要同时恢复时间基准和设备 |
| real-time 流延迟过大 | 清旧 packet、flush audio/video path、追赶 live | 直播/实时流不能无限堆延迟 |
| seek 仍未结束 | 即使 buffer 够，也可能继续等待 | seek 状态会压住 buffering 恢复和 render 放行 |

这说明播放器里有两个不同的问题：

```text
有没有 frame？
  -> decode 是否已经产出可渲染数据。

能不能播放？
  -> buffer、状态机、clock 是否允许进入 render。
```

### 设计点评

- 值得学习：把 buffer gate 放在 render 前，可以统一控制起播、卡顿恢复和 demuxer buffer level。
- 不要照抄：buffering 判断混在大类里会比较难测。AVPlayer 可以先做一个 `BufferPolicy`，输入 buffer snapshot，输出 `canRender / shouldBuffer / notifyLoading`。

## 9. doRender：audio 推进 clock，video 服从 clock

render 阶段的顺序是：

```text
RenderAudio()
RenderVideo()
RenderSubtitle()
```

音频不仅是输出，它还是 master clock 的来源。首次有效 audio frame 送入 audio render 后，master clock 会绑定
audio render 的播放时间：

```text
mMasterClock.setReferenceClock(getAudioPlayTimeStampCB, this)
```

这代表播放器后续判断“现在播放到哪里”，优先相信音频设备的播放进度，而不是简单相信系统时间。

video render 时会拿 video PTS 和 master clock 比较：

```text
videoLateUs = masterClockTime - videoPts - videoDelay
```

然后做三类决策：

```text
video too early
  -> 不 render，不 pop，下一轮再看

video on time
  -> SendVideoFrameToRender()
  -> pop frame

video too late
  -> drop frame 或清理更早数据
  -> 进入追帧逻辑
```

这就是音视频同步的最小主线：

```text
有音频：
  audio render 推进 master clock
  video 对齐 audio clock

无音频：
  video 自己推进或校准 clock
```

设计点评：

- 值得学习：把 audio render 的真实播放位置作为时间基准，比单纯用解码 PTS 或系统时间更贴近用户听到的进度。
- 需要警惕：clock、render callback、frame drop 这几块强耦合，后续阅读时要单独画同步状态图，不能只看函数调用顺序。

## 10. 用一帧视频串起来

把上面的阶段合成一条视频帧生命周期：

```text
IDataSource::Read()
  -> bytes
  -> avFormatDemuxer / av_read_frame()
  -> IAFPacket(video)
  -> BufferController video packet queue
  -> mVideoPacket pending packet
  -> IDecoder::send_packet()
  -> ActiveDecoder input queue
  -> decoder thread
  -> ActiveDecoder output queue
  -> mVideoFrameQue
  -> DoCheckBufferPass()
  -> RenderVideo()
  -> compare video PTS with master clock
  -> wait / render / drop
  -> IVideoRender::renderFrame()
  -> RenderCallback()
```

一帧 audio 类似，但 audio render 之后还会参与 master clock：

```text
audio frame
  -> IAudioRender::renderFrame()
  -> audio device queue
  -> getAudioPlayTimeStampCB()
  -> SystemReferClock
  -> video sync decision
```

## 11. 这一章的优秀设计

可以重点借鉴这些设计：

- 控制面和数据面分离：API 投递命令，主循环统一处理状态和数据推进。
- pull 模型读取：由 player loop 根据 buffer 和下游消费速度决定是否继续读。
- packet queue 和 frame queue 分层：packet 服务 buffering/seek/drop，frame 服务 render/sync。
- pending packet：清楚表达“取出但还没被 decoder 消费”的中间状态。
- render 前 buffer gate：避免刚有少量 frame 就立刻播放导致频繁卡顿。
- audio master clock：用实际音频播放进度驱动 video 同步。

## 12. 不建议直接照抄的地方

这篇不是说 CicadaPlayer 每个实现都适合作为现代 C++ 模板。读源码时要注意：

- `SuperMediaPlayer` 职责很大，控制、读包、解码、buffering、同步、通知都在里面，学习时要按阶段拆开。
- 状态变量很多，部分语义需要结合上下文理解，不适合直接搬到小项目。
- buffer policy、read throttle、render gate 没有被拆成很小的可测试对象。
- C++11 历史写法和 C/FFmpeg 风格较多，学习设计时不要顺手照搬裸指针、宏和宽泛状态散落。

更好的学习方式是：借鉴它的播放器机制和模块边界，但在 AVPlayer 里用更小的对象表达这些策略。

## 13. 迁移到 AVPlayer 的最小版本

不要一开始照着 CicadaPlayer 做完整 SDK。可以先迁移一个最小闭环：

```text
PlayerCore main loop
  -> CommandQueue
  -> ReadController
  -> PacketQueue(audio/video)
  -> DecoderAdapter
  -> FrameQueue(audio/video)
  -> BufferPolicy
  -> Clock
  -> RenderScheduler
```

第一版只需要做到：

- 命令不直接做重活，而是进入主循环处理。
- read 阶段根据 packet buffer duration 决定是否继续拉包。
- decode 阶段受 frame queue 上限和 pending packet 约束。
- render 阶段先过 buffer gate，再按 clock 输出。
- 每个阶段能打印一条 timeline log，方便验证执行流。

这比直接迁移 CicadaPlayer 的大类更适合学习，也更容易测试。

## 14. 下一步怎么读

这篇只负责主线。后续按这个顺序继续拆：

```text
00-object-model-and-module-wiring.md
  -> 谁持有谁，哪些对象长期存在。

02-data-source-cache.md
  -> bytes 从 URL/file/cache 怎么进入 demuxer。

03-demuxer-prototype.md
  -> bytes 怎么被容器层切成 IAFPacket。

04-packet-frame-queue.md
  -> packet queue、decoder queue、frame queue 有什么区别。

05-decode-sync-render.md
  -> clock、等待、渲染、丢帧怎么做得更细。
```

读源码时始终先问一句：

```text
这个函数是在 read、decode、setup、buffer gate、render 里的哪一段？
```

能回答这个问题，CicadaPlayer 的大类就不会再是一团散乱函数。
