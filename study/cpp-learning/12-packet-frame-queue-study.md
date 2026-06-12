# Packet / Frame Queue：数据面如何连接 demuxer、decoder 和 render

前面已经分别看了 data source、demuxer、decoder/render。中间还缺一个关键层：队列。

播放器不是从 demuxer 读一个 packet 就立刻同步 decode/render。真实播放链路里，packet 要先进入
按媒体类型拆开的缓存队列；decoder 再从队列取 packet，输出 frame；render 再从 frame 队列按 clock
调度输出。

可以先记住这条主线：

```text
demuxer_service::readPacket()
  -> SuperMediaPlayer::ReadPacket()
  -> BufferController::AddPacket()
  -> MediaPacketQueue
  -> BufferController::getPacket()
  -> SMPAVDeviceManager::sendPacket()
  -> ActiveDecoder input/output queue
  -> SuperMediaPlayer mAudioFrameQue / mVideoFrameQue
  -> RenderAudio() / RenderVideo()
```

这一章重点不是把每个队列实现背下来，而是看清：哪些队列负责缓存，哪些队列负责解码异步，哪些队列负责
渲染调度，seek/flush/drop 时哪些旧数据必须失效。

## 1. 源码入口

先看这些文件：

- `mediaPlayer/media_packet_queue.h`
- `mediaPlayer/media_packet_queue.cpp`
- `mediaPlayer/buffer_controller.h`
- `mediaPlayer/buffer_controller.cpp`
- `mediaPlayer/SuperMediaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.cpp`
- `mediaPlayer/SMPAVDeviceManager.cpp`
- `framework/codec/ActiveDecoder.h`
- `framework/codec/ActiveDecoder.cpp`

重点函数：

- `SuperMediaPlayer::doReadPacket()`
- `SuperMediaPlayer::ReadPacket()`
- `BufferController::AddPacket()`
- `BufferController::getPacket()`
- `MediaPacketQueue::AddPacket()`
- `MediaPacketQueue::getPacket()`
- `MediaPacketQueue::ClearPacketBeforeTimePos()`
- `MediaPacketQueue::ClearPacketBeforePTS()`
- `MediaPacketQueue::ClearPacketAfterTimePosition()`
- `SuperMediaPlayer::DecodeVideoPacket()`
- `SuperMediaPlayer::DecodeAudio()`
- `SuperMediaPlayer::FillVideoFrame()`
- `SuperMediaPlayer::FlushAudioPath()`
- `SuperMediaPlayer::FlushVideoPath()`
- `ActiveDecoder::send_packet()`
- `ActiveDecoder::getFrame()`
- `ActiveDecoder::flush()`

## 2. 播放器问题是什么？

队列在播放器里解决的是数据面稳定性问题：

- 网络和 demuxer 读包是突发的，decoder/render 消费是按节奏的，二者速度不一致。
- audio、video、subtitle 的缓存要分开统计，否则无法判断谁缺数据。
- seek 后旧 packet、旧 frame、decoder 内部缓存都可能还在，必须明确失效。
- 实时流延迟过大时，不能只继续堆积，要能按关键帧清理旧 packet。
- decoder 可能临时满，packet 不能因为一次 `EAGAIN` 就丢失。
- render 只应该处理已经解码好的 frame，不能直接依赖 demuxer packet。

这就是为什么播放器里通常会有多层队列，而不是一个全局大队列。

## 3. 三层队列分别负责什么？

CicadaPlayer 里可以按职责分成三层。

第一层是 packet 缓存队列：

```text
BufferController
  -> audio MediaPacketQueue
  -> video MediaPacketQueue
  -> subtitle MediaPacketQueue
```

这一层存的是 demuxer 读出来的 `IAFPacket`。它负责缓存时长统计、packet 数量、关键帧位置、seek
缓存命中、实时流丢包、buffering 判断。

第二层是 decoder 内部队列：

```text
ActiveDecoder
  -> mInputQueue
  -> decode thread
  -> mOutputQueue
```

这一层解决 decoder 异步执行。播放器主循环把 packet 送进去，decoder thread 输出 frame。

第三层是播放器渲染前 frame 队列：

```text
SuperMediaPlayer
  -> mAudioFrameQue
  -> mVideoFrameQue
```

这一层存的是已经解码出的 `IAFFrame`。audio render 从 `mAudioFrameQue` 取 frame，video render 从
`mVideoFrameQue` 取 frame，再根据 clock 判断是否输出。

这三层不要混在一起理解：

```text
packet queue：还没解码，主要服务 buffering / seek / drop。
decoder queue：codec 异步边界，主要服务吞吐和非阻塞。
frame queue：已经解码，主要服务 render / clock / first-frame。
```

## 4. ReadPacket：demuxer packet 进入 BufferController

`SuperMediaPlayer::doReadPacket()` 会根据当前缓存时长、最大缓存、内存情况决定是否继续读包。真正读包在
`ReadPacket()` 里完成。

`ReadPacket()` 的简化逻辑是：

```text
pMediaFrame = demuxer_service::readPacket()
  -> 判断 streamIndex 属于 video/audio/subtitle
  -> 更新 first pts、timePosition、stream change 等状态
  -> BufferController::AddPacket(packet, BUFFER_TYPE_*)
  -> 更新 V_FRAME_RECEIVE / A_FRAME_RECEIVE 等统计点
```

这里有一个重要边界：demuxer 只负责产出 packet，播放器在 `ReadPacket()` 里把 packet 分流到对应媒体
队列。这样后续 buffering、seek、实时流追赶都可以按 audio/video/subtitle 分别处理。

`doReadPacket()` 不会无限读。它会看 `getPlayerBufferDuration()`、`maxBufferDuration`、
`startBufferDuration` 和低内存状态。这个逻辑和 `07-buffering-and-qoe-study.md` 是连在一起的：
缓存不是文件读得越多越好，而是要在流畅度、延迟和内存之间折中。

## 5. MediaPacketQueue 不是普通 FIFO

`MediaPacketQueue` 内部是 `std::list<unique_ptr<IAFPacket>>`，并且维护一个 `mCurrent` 迭代器。
这说明它不只是简单的 push/pop。

它还负责这些信息：

- `mDuration`：从当前消费位置到队尾的可播放缓存时长。
- `mTotalDuration`：队列整体累计时长。
- `mPacketDuration`：当 packet 本身没有 duration 时，用估计值补齐。
- `mMAXBackwardDuration`：允许保留一段已经读过的数据，支持回退/缓存内 seek。
- `GetKeyTimePositionBefore()`：找某个位置之前的关键帧。
- `ClearPacketBeforeTimePos()` / `ClearPacketBeforePTS()`：清理旧数据。
- `ClearPacketAfterTimePosition()`：切流或无缝切换时清理后段数据。

普通 FIFO 只回答“下一个 packet 是什么”。这里的 `MediaPacketQueue` 还要回答：

```text
当前缓存了多久？
最近的关键帧在哪里？
seek 位置是否还在缓存里？
哪些旧 packet 可以丢？
实时流延迟过大时清到哪里？
```

这就是播放器队列和普通任务队列的区别。

## 6. BufferController：按媒体类型统一管理 packet

`BufferController` 本身不做复杂算法，它是三个 `MediaPacketQueue` 的门面：

```text
BUFFER_TYPE_AUDIO    -> mAudioPacketQueue
BUFFER_TYPE_VIDEO    -> mVideoPacketQueue
BUFFER_TYPE_SUBTITLE -> mSubtitlePacketQueue
BUFFER_TYPE_AV       -> audio + video
BUFFER_TYPE_ALL      -> audio + video + subtitle
```

它的价值是让 `SuperMediaPlayer` 不直接操作多个队列实例，而是用统一接口表达意图：

- `AddPacket()`：读包后按类型加入。
- `getPacket()`：送 decoder 前取出。
- `GetPacketDuration()`：buffering 和 QoE 决策。
- `GetPacketSize()`：EOS 判断和调试。
- `ClearPacket()`：普通 seek 或 stop/release 时清空。
- `ClearPacketBeforeTimePos()`：缓存内 seek、低延迟追赶、丢旧数据。
- `GetKeyTimePositionBefore()`：找能安全继续解码的关键帧位置。
- `Rewind()`：缓存内 seek 时重置读取位置。

这个门面层值得借鉴。AVPlayer 里可以先有一个很小的 `PacketBuffer`，内部按 stream type 管理队列，
对外只暴露播放器需要的能力。

## 7. Packet 进入 decoder：失败时不能误删

从 `BufferController::getPacket()` 取出的 packet 会先放到 `mVideoPacket` 或 `mAudioPacket` 临时成员里，
再调用 decoder：

```text
if (mVideoPacket == nullptr) {
    mVideoPacket = mBufferController->getPacket(BUFFER_TYPE_VIDEO);
}

ret = DecodeVideoPacket(mVideoPacket);
```

这个写法有一个关键点：`mVideoPacket` 是“当前正在尝试送入 decoder 的 packet”。如果 decoder 返回
`STATUS_RETRY_IN`，说明 decoder input/output queue 暂时满了，这个 packet 不能丢，下一轮还要继续送。

只有当 `sendPacket()` 成功消费 packet 后，`unique_ptr` 才会被置空：

```text
ret = mAVDeviceManager->sendPacket(pVideoPacket, DEVICE_TYPE_VIDEO, 0);
if (!(ret & STATUS_RETRY_IN)) {
    assert(pVideoPacket == nullptr);
}
```

这就是所有权和背压在播放器里的结合：

```text
packet 取出队列后，不等于一定被 decoder 消费；
decoder 满了，packet 必须留在当前变量里等待重试。
```

AVPlayer 做这块时要特别注意：不要把 `popPacket()` 和 `sendPacket()` 绑定成不可回滚的一步。更清晰的
模型是：

```text
peek or take pending packet
  -> try send to decoder
  -> success: consume pending
  -> retry: keep pending
  -> fatal error: report and drop/reset according to policy
```

## 8. ActiveDecoder：第二层异步队列

`ActiveDecoder::send_packet()` 会把 packet 放进 decoder 的 `mInputQueue`。如果 `mInputQueue` 或
`mOutputQueue` 太满，就返回 `STATUS_RETRY_IN`。

简化逻辑是：

```text
send_packet(packet)
  -> 如果 input/output queue 满，返回 STATUS_RETRY_IN
  -> 否则 packet.release() 进入 mInputQueue
  -> decode thread 被唤醒

getFrame()
  -> 如果 mOutputQueue 有 frame，弹出 frame
  -> 如果 decoder EOS，返回 STATUS_EOS
  -> 否则返回 EAGAIN
```

这里可以看到第二层背压：

- packet queue 太大时，读包会停。
- decoder input/output queue 太大时，送解码会停。
- frame queue 太大时，播放器不会继续大量拉 decoder 输出。

多层背压合起来，播放器才不会在弱网、卡顿、后台、低端机上无限堆内存。

## 9. Frame queue：解码后等待 render

`SuperMediaPlayer` 自己维护两个 frame 队列：

```text
std::queue<unique_ptr<IAFFrame>> mVideoFrameQue;
std::deque<unique_ptr<IAFFrame>> mAudioFrameQue;
```

video 解码输出由 `FillVideoFrame()` 推入 `mVideoFrameQue`。audio 解码输出由 `DecodeAudio()` 推入
`mAudioFrameQue`。

frame queue 的职责和 packet queue 不一样：

- packet queue 关注缓存和可解码性。
- frame queue 关注输出时机和 render 节奏。

`RenderVideo()` 会从 `mVideoFrameQue.front()` 看视频 frame 的 pts，再和 master clock 比较：

- 视频太早：不 pop，下一轮再判断。
- 视频正常：送 render，然后 pop。
- 视频太晚：标记 discard、统计 drop，然后 pop。

`RenderAudio()` 则更多依赖 audio render 的消费能力和设备队列。audio frame 一旦进入 render，就会影响
audio clock 和播放位置。

## 10. Flush：packet、decoder、frame 都要清

seek、stop、切流、实时追赶都可能触发 flush。播放器数据面里最容易出错的点就是只清了一层队列。

CicadaPlayer 的 flush 相关层次是：

```text
BufferController::ClearPacket()
  -> 清 packet queue

SMPAVDeviceManager::flushDevice()
  -> decoder->flush()
  -> render flush

SuperMediaPlayer::FlushAudioPath()
  -> flush device
  -> clear mAudioFrameQue
  -> reset mAudioPacket / audio pts / audio EOS

SuperMediaPlayer::FlushVideoPath()
  -> flush device
  -> discard and clear mVideoFrameQue
  -> reset mVideoPacket / video pts / decoder full / video EOS
```

`ProcessSeekToMsg()` 里如果不是缓存内 seek，会先 `ClearPacket(BUFFER_TYPE_ALL)`，再让 demuxer seek。
缓存内 seek 则会尝试用 `ClearPacketBeforeTimePos()` 保留可用缓存，只清理 seek 位置之前的数据。

这里的学习重点是：

```text
旧 packet 清了，不代表 decoder 里没有旧 frame；
decoder flush 了，不代表 render 设备队列没有旧音频；
frame queue 清了，不代表当前 pending packet 已经失效。
```

所以播放器需要成套地定义 flush 语义，而不是随手调用某一个 `clear()`。

## 11. Seek、缓存内回退和旧数据失效

`SeekInCache()` 会利用 `BufferController` 的能力判断 seek 位置是否在现有 packet 缓存里。如果能命中，
它可以避免真正的 demuxer seek，直接调整 packet 队列消费位置和清理范围。

这里依赖几个能力：

- `Rewind()`：把 `mCurrent` 回到队列开头。
- `GetPacketFirstTimePos()` / `GetPacketLastTimePos()`：判断目标位置是否在缓存覆盖范围。
- `GetKeyTimePositionBefore()`：找到目标位置前的关键帧。
- `ClearPacketBeforeTimePos()`：清掉不再需要的旧 packet。

这块很适合对照 ffplay/ijk 的 serial 思想。CicadaPlayer 这里更多是“清队列 + flush 路径 + seek 状态”
组合，并没有把 serial 抽成一个特别清晰的公共机制。学习时要抓播放器问题，不必照抄实现形态。

AVPlayer 里更建议把 seek generation 显式化：

```text
struct Packet {
    StreamType type;
    int64_t ptsUs;
    int64_t timePositionUs;
    uint64_t generation;
};

struct Frame {
    StreamType type;
    int64_t ptsUs;
    uint64_t generation;
};
```

seek 后 generation 增加。旧 generation 的 packet/frame 即使漏出来，也能被 decoder/render 层过滤掉。
这比只靠清队列更容易测试。

## 12. 实时流和低延迟：队列不能只会增长

`DoCheckBufferPass()` 里能看到实时流延迟控制：当缓存超过 `RTMaxDelayTime` 太多时，会根据最后的
audio/video 位置和关键帧位置清理旧 packet，并 flush 对应路径。

简化理解：

```text
如果实时流积压太多：
  -> 找一个安全的时间点
  -> video 尽量落在关键帧
  -> 清理这个点之前的 audio/video packet
  -> flush decoder/frame path
  -> 更新 master clock 或播放状态
```

这个设计解决的是直播低延迟问题：如果网络或设备短时间卡住，播放器不能永远慢慢追历史数据。对直播来说，
“追上当前时间”有时比“不丢任何帧”更重要。

## 13. 适合迁移到 AVPlayer 的最小版本

AVPlayer 不需要一开始复刻 CicadaPlayer 的所有队列能力。建议先做这几个小模块：

```text
PacketQueue
  - push(Packet)
  - take()
  - duration()
  - clear()
  - clearBefore(positionUs)

PacketBuffer
  - audio PacketQueue
  - video PacketQueue
  - getBufferedDuration()
  - clear(StreamMask)

PendingPacket
  - 保存已经从 PacketQueue 取出、但 decoder 还没成功消费的 packet

FrameQueue
  - push(Frame)
  - peek()
  - pop()
  - clear(generation)
```

然后用一个最小数据面闭环验证：

```text
demuxer fake packets
  -> PacketBuffer
  -> fake decoder
  -> FrameQueue
  -> fake renderer
  -> clock scheduler
```

验收标准可以很具体：

- decoder 返回 retry 时，pending packet 不丢。
- seek 后旧 generation frame 不会 render。
- buffering duration 来自 packet queue，而不是 frame queue。
- video frame 太早时不会 pop。
- video frame 太晚时会 drop 并产生统计事件。

## 14. 哪些地方不要照抄？

- 不要把 `MediaPacketQueue` 的 `mCurrent`、backward duration、extra_data 处理一次性搬过去。先做普通
  packet queue，再按需求加缓存内 seek。
- 不要让队列同时承担太多职责。AVPlayer 可以把 queue、buffer statistics、seek cache policy 分开。
- 不要只依赖清队列保证旧数据失效。显式 generation / serial 更容易推理和测试。
- 不要忽略 retry 语义。`sendPacket()` 失败时 packet 是否还归调用者所有，必须写清楚。
- 不要让 render 层反向操作 packet queue。render 应该通过事件和状态反馈影响控制面，而不是直接清数据面。

## 15. 可公开的技术表达

CicadaPlayer 的数据面不是单队列模型，而是分成 packet 缓存队列、decoder 内部队列和 render 前 frame
队列。`BufferController` 负责按媒体类型管理 packet 缓存、缓存时长、关键帧位置和清理策略；
`ActiveDecoder` 用 input/output queue 隔离异步解码；`SuperMediaPlayer` 再用 audio/video frame
queue 做渲染调度。这个结构的核心价值是把 buffering、seek、flush、背压和音视频同步拆到合适的边界上，
避免 demuxer、decoder、render 互相耦合。

