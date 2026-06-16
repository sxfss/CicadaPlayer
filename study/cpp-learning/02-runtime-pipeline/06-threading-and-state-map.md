# 线程与状态图：播放器稳定性的底层边界

这一章补的是“为什么播放器容易出竞态”的视角。你已经能看到 `PlayerMessageControl` 和
`ProcessVideoLoop()`，但还需要把线程、状态、队列和回调边界画出来。

第一轮只建立普通 URL/MP4 的主图，不展开所有平台 render 线程。

## 0. 线程总图

```text
API caller thread
  -> MediaPlayer / ICicadaPlayer API
  -> SuperMediaPlayer::putMsg()
  -> PlayerMessageControl queue

player main thread: afThread
  -> SuperMediaPlayer::mainService()
      -> processMsg()
      -> ProcessVideoLoop()
      -> read / decode / buffer / render scheduling

decoder worker thread: ActiveDecoder
  -> input queue
  -> real decoder
  -> output queue

render callback thread
  -> audio/video render callback
  -> SuperMediaPlayer::RenderCallback()
  -> notify first frame / rendered / position

notifier / app callback side
  -> PlayerNotifier
  -> user listener callbacks
```

关键源码入口：

- [SuperMediaPlayer.cpp](../../../mediaPlayer/SuperMediaPlayer.cpp)：`putMsg()`、`mainService()`、`Stop()`、`RenderCallback()`。
- [SMPMessageControllerListener.cpp](../../../mediaPlayer/SMPMessageControllerListener.cpp)：prepare/start/pause/seek 命令处理。
- [ActiveDecoder.cpp](../../../framework/codec/ActiveDecoder.cpp)：decoder 输入队列、输出队列、解码线程。
- [SMPAVDeviceManager.cpp](../../../mediaPlayer/SMPAVDeviceManager.cpp)：flush decoder/render 的聚合层。
- [player_notifier.cpp](../../../mediaPlayer/player_notifier.cpp)：外部事件通知。

## 1. API 线程：只表达意图，不做重活

API 调用线程的理想职责是：

```text
SetDataSource / Prepare / Start / Pause / Seek / Stop
  -> 检查基本状态
  -> 组装 MsgParam
  -> putMsg()
  -> 尽快返回
```

CicadaPlayer 大部分控制命令都走 `putMsg()`。这能避免 API 调用直接卡在网络读取、demux seek、decoder flush 或
render 设备操作上。

值得学习：

- 用户线程和播放线程解耦。
- 所有控制意图进入同一条消息队列，状态变化更容易串行化。

需要警惕：

- 仍有少数 TODO 表示希望移动到 `mainService` 线程，说明历史代码里线程边界不是完全干净。
- 读源码时要问：这个函数是在 API 线程执行，还是在 player main thread 执行？

## 2. mainService 线程：控制面和数据面汇合

`mApsaraThread` 运行 `SuperMediaPlayer::mainService()`。它每轮做两件事：

```text
processMsg()
ProcessVideoLoop()
```

这让播放器拥有一个中心调度点：

- 控制命令改变状态、seek 目标、flush 行为。
- 数据面继续推进 read、decode、buffer、render。
- loop gap 根据追帧、seek、buffer 情况变化，避免固定 sleep。

值得学习：

- 用一个主循环串起控制面和数据面，便于处理 seek/stop/release 这类跨层动作。
- 复杂播放器不要让多个线程随意改核心状态。

需要警惕：

- `SuperMediaPlayer` 同时承担太多职责。学习时要把成员变量按阶段分组，而不是逐个背字段。

## 3. ActiveDecoder 线程：decoder 有自己的背压边界

`ActiveDecoder` 不是简单同步函数。它大致有三层：

```text
SuperMediaPlayer::DecodeVideoPacket
  -> SMPAVDeviceManager::sendPacket
  -> ActiveDecoder::send_packet
  -> input queue
  -> decode thread
  -> output queue
  -> ActiveDecoder::getFrame
  -> SuperMediaPlayer frame queue
```

这个结构解决的问题是：

- decoder 可能比较慢，不能阻塞 player main loop 太久。
- decoder 输入满时，要返回 `STATUS_RETRY_IN`。
- packet 已经从 `BufferController` 取出但 decoder 没吃掉时，要用 pending packet 保留。

值得学习：

- decoder input queue 和 player packet queue 是两层，不要混为一谈。
- pending packet 是背压语义：取出了，但还没被下游消费。

需要警惕：

- status 位组合不如强类型结果清晰。
- 现代 C++17 可以给 decoder 输入结果定义 `Accepted / Retry / FatalError / Eos`。

## 4. render callback：它是结果回流，不是数据入口

render 之后会通过 audio/video listener 回到 `SuperMediaPlayer::RenderCallback()`：

```text
IAudioRender / IVideoRender
  -> ApsaraAudioRenderCallback / ApsaraVideoRenderListener
  -> SuperMediaPlayer::RenderCallback()
  -> NotifyFirstFrame / rendered event / position update
```

这里要注意两个点：

- callback 可能异步到达，所以 seek 过程中要避免旧 callback 把 position 更新回旧时间。
- first frame 更接近 render 完成后的用户可见事件，不应该只用 decode 完成代替。

值得学习：

- 把“数据已经渲染”作为外部事件来源，比“数据已经解码”更贴近播放体验。
- render callback 是 QoE 统计的重要入口。

需要警惕：

- callback 线程边界要非常清楚。AVPlayer 里可以让 callback 只投递事件，由主线程过滤 serial/generation 后再更新状态。

## 5. 状态图：不要只看 PlayerStatus

播放器稳定性不是一个 `PlayerStatus` 就能表达完。CicadaPlayer 里至少有几类状态：

```text
生命周期状态:
  IDLE / INITIALIZED / PREPARING / PREPARED / PLAYING / PAUSED / COMPLETION / STOPPED / ERROR

数据面状态:
  eof / buffering / first buffer / buffer full / low memory

seek 状态:
  seek flag / seek target / accurate catch / seek in cache

render 状态:
  render started / first frame / audio clock ready / video catching up

线程状态:
  main thread running / decoder thread running / render callback active
```

读源码时不要把这些都混成“播放器状态”。它们分别回答不同问题：

```text
能不能接受 API 命令？
能不能继续读 packet？
能不能继续 decode？
能不能 render？
能不能通知外部状态？
```

## 6. Stop / Release：要先停线程，再释放对象

停止播放器时最容易出的问题是回调和线程还在使用对象。

CicadaPlayer 的 `Stop()` 体现了几个重要顺序：

```text
preStop demuxer
stop player thread
clear / flush audio path
clear / flush video path
stop demuxer service
release data source / render / parser
clear message queue
```

值得学习：

- stop 不是只改状态，还要让 read、decode、render、callback 都停止或清理。
- 析构里先 `Stop()`，再释放 notifier，避免后台线程继续使用已释放对象。

需要警惕：

- 原代码里仍有裸指针和手动 delete，生命周期审查成本高。
- AVPlayer 可以用 RAII wrapper 让线程对象析构时自动 request stop + join。

## 7. Seek / Flush：跨越所有数据层

seek 需要清理的不是一条队列，而是整条数据路径：

```text
packet queue
  -> decoder input queue
  -> decoder output queue
  -> frame queue
  -> render device queue
  -> clock / position state
```

CicadaPlayer 分别通过 `BufferController`、`SMPAVDeviceManager::flushDevice()`、`FlushAudioPath()`、
`FlushVideoPath()`、render flush 和 seek 状态处理这些层。

值得学习：

- flush 是跨模块协议，不是某个类内部小操作。
- seek 后旧数据过滤应该和 seek 状态绑定。

AVPlayer 最小迁移版：

```text
seekSerial++
packetQueues.clear()
decoders.flush()
frameQueues.clear()
renderers.flush()
clock.setTime(target)

onPacket/onFrame/onCallback:
  if serial is stale:
    ignore
```

## 8. C++ 设计提醒

线程和状态设计里，现代 C++17 最值得坚持的是：

- 线程拥有者要明确，析构必须能停线程。
- callback 不应该持有不清晰的裸对象生命周期。
- 跨线程状态尽量集中在少数对象里，不要散落在大类的几十个 bool。
- 状态变化最好有明确事件流，不要靠多个隐式条件推断。

这不是要求你把 CicadaPlayer 全部改掉，而是读源码时要能区分：

```text
这是播放器工程上值得学习的机制
这是历史代码里不够现代的表达方式
```
