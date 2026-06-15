# Packet / Frame Queue 执行链：packet 如何进入 decoder

上一章 demuxer 输出 `IAFPacket`。这一章看 packet 怎么被 `SuperMediaPlayer` 接住、缓存、再送进
decoder。

主链是：

```text
SuperMediaPlayer::ReadPacket()
  -> BufferController::AddPacket()
  -> MediaPacketQueue::AddPacket()
  -> doDeCode()
  -> BufferController::getPacket()
  -> mVideoPacket / mAudioPacket
  -> DecodeVideoPacket() / DecodeAudio()
  -> SMPAVDeviceManager::sendPacket()
  -> ActiveDecoder::send_packet()
  -> ActiveDecoder input queue
  -> ActiveDecoder output queue
  -> mVideoFrameQue / mAudioFrameQue
```

## 这一层由谁持有？

`SuperMediaPlayer` 持有 packet 队列聚合器：

```text
std::unique_ptr<BufferController> mBufferController
```

`BufferController` 持有三条 packet queue：

```text
MediaPacketQueue mVideoPacketQueue
MediaPacketQueue mAudioPacketQueue
MediaPacketQueue mSubtitlePacketQueue
```

`SuperMediaPlayer` 还持有两个 pending packet：

```text
std::unique_ptr<IAFPacket> mVideoPacket
std::unique_ptr<IAFPacket> mAudioPacket
```

这两个 pending packet 很重要。它们表示“已经从 packet queue 取出，但 decoder 还不一定成功消费”的
packet。

解码后的 frame queue 也在 `SuperMediaPlayer` 里：

```text
std::queue<unique_ptr<IAFFrame>> mVideoFrameQue
std::deque<unique_ptr<IAFFrame>> mAudioFrameQue
```

## 输入是什么？输出是什么？

输入：

```text
demuxer 输出的 IAFPacket
```

输出有两层：

```text
送入 decoder 的 IAFPacket
decoder 输出的 IAFFrame
```

这里最容易混淆的是 packet queue 和 frame queue：

```text
packet queue：还没解码，服务 buffering / seek / drop / 缓存统计
frame queue：已经解码，服务 render / clock / 首帧
```

## 主循环什么时候调用它？

主循环的入口仍然是：

```text
mainService()
  -> ProcessVideoLoop()
```

`ProcessVideoLoop()` 里和队列有关的顺序是：

```text
doReadPacket()
  -> ReadPacket()
  -> packet 入 BufferController

doDeCode()
  -> 从 BufferController 取 packet
  -> 送 ActiveDecoder
  -> 从 ActiveDecoder 取 frame
  -> frame 入 mAudioFrameQue / mVideoFrameQue

doRender()
  -> 消费 frame queue
```

所以这一层是 demuxer 和 decoder/render 之间的数据面胶水。

## ReadPacket 如何把 packet 入队？

`SuperMediaPlayer::ReadPacket()` 调用：

```text
mDemuxerService->readPacket(pMedia_Frame, index)
```

拿到 packet 后，根据 `pFrame->getInfo().streamIndex` 分流：

```text
if video:
    mBufferController->AddPacket(move(pMedia_Frame), BUFFER_TYPE_VIDEO)

if audio:
    mBufferController->AddPacket(move(pMedia_Frame), BUFFER_TYPE_AUDIO)

if subtitle:
    mBufferController->AddPacket(move(pMedia_Frame), BUFFER_TYPE_SUBTITLE)
```

这一步的职责不是解码，而是把 packet 放到对应媒体类型的缓存里。

## MediaPacketQueue 为什么不是普通 FIFO？

`MediaPacketQueue` 内部有：

```text
std::list<unique_ptr<IAFPacket>> mQueue
iterator mCurrent
int64_t mDuration
int64_t mTotalDuration
int64_t mPacketDuration
```

它不仅能 `AddPacket()` / `getPacket()`，还支持：

- `GetDuration()`：计算当前还剩多少 packet 缓存。
- `GetPacketPts()` / `GetLastPTS()`：提供当前位置和尾部时间。
- `GetKeyTimePositionBefore()`：找 seek 或丢帧需要的关键帧。
- `ClearPacketBeforeTimePos()`：清理目标时间之前的数据。
- `Rewind()`：缓存内 seek 时重置读取位置。

普通 FIFO 只关心先入先出；播放器 packet queue 还要服务 buffering、seek、低延迟追赶和旧数据失效。

## packet 怎么进入 decoder？

`doDeCode()` 里视频大致是：

```text
if (mVideoPacket == nullptr)
    mVideoPacket = mBufferController->getPacket(BUFFER_TYPE_VIDEO)

FillVideoFrame()
DecodeVideoPacket(mVideoPacket)
```

音频大致是：

```text
if (mAudioPacket == nullptr)
    mAudioPacket = mBufferController->getPacket(BUFFER_TYPE_AUDIO)

DecodeAudio(mAudioPacket)
```

这里的关键不是函数名，而是所有权：

```text
从 BufferController 取出 packet 后，packet 暂时归 mVideoPacket/mAudioPacket。
只有 decoder 成功接收后，unique_ptr 才会被置空。
```

如果 decoder 暂时满了，返回 `STATUS_RETRY_IN`，pending packet 不能丢。下一轮主循环还会继续尝试。

## ActiveDecoder 是哪一层队列？

`SMPAVDeviceManager::sendPacket()` 只是转发到当前 audio/video decoder：

```text
decoderHandle->decoder->send_packet(packet, timeOut)
```

具体 `ActiveDecoder::send_packet()` 会进入 decoder 内部队列：

```text
如果 mInputQueue 或 mOutputQueue 满
  -> 返回 STATUS_RETRY_IN
否则
  -> packet.release()
  -> push 到 mInputQueue
  -> 唤醒 decode thread
```

所以有两层背压：

```text
BufferController 太满 -> doReadPacket 停止继续读包
ActiveDecoder 太满 -> doDeCode 停止继续送包，pending packet 保留
```

这就是播放器避免无限堆内存的核心机制之一。

## frame 怎么回到 SuperMediaPlayer？

`ActiveDecoder` 的 decode thread 会把输出放进 `mOutputQueue`。主循环再调用：

```text
mAVDeviceManager->getFrame(frame, DEVICE_TYPE_VIDEO, 0)
mAVDeviceManager->getFrame(frame, DEVICE_TYPE_AUDIO, 0)
```

视频在 `FillVideoFrame()` 里进入：

```text
mVideoFrameQue.push(move(frame))
```

音频在 `DecodeAudio()` 里进入：

```text
mAudioFrameQue.push_back(move(frame))
```

到这里，数据已经从 packet 变成 frame。下一层不再关心 demuxer packet，而是由 render 和 clock 判断
frame 什么时候输出。

## flush 时哪些数据要失效？

seek、stop、实时流追赶都会触发清理。要分三层看：

```text
BufferController::ClearPacket()
  -> 清 packet queue

SMPAVDeviceManager::flushDevice()
  -> decoder->flush()
  -> render flush

FlushAudioPath() / FlushVideoPath()
  -> 清 mAudioFrameQue / mVideoFrameQue
  -> 清 pending packet
  -> 重置 played pts / EOS / decoder full 状态
```

只清 packet queue 不够，因为 decoder 里可能还有旧 frame；只 flush decoder 也不够，因为
`mVideoFrameQue` 里可能已经有旧 frame 等待 render。

这也是你后面做 AVPlayer 时要特别注意的点：seek 不是一个 demuxer 操作，而是一组数据面失效操作。

## 下一层是谁？

packet/frame queue 的下一层是同步和渲染：

```text
mAudioFrameQue -> RenderAudio()
mVideoFrameQue -> RenderVideo()
```

下一章 `11-decoder-render-boundary-study.md` 会从 frame queue 开始，继续看：

```text
audio frame 如何送 audio render
audio render 如何成为 master clock 参考
video frame 如何根据 master clock 等待、渲染或丢弃
render callback 如何回到 SuperMediaPlayer
```

## 读源码时按哪些函数跳？

建议按这个顺序：

1. `SuperMediaPlayer::ProcessVideoLoop()`
2. `SuperMediaPlayer::doReadPacket()`
3. `SuperMediaPlayer::ReadPacket()`
4. `BufferController::AddPacket()`
5. `MediaPacketQueue::AddPacket()`
6. `SuperMediaPlayer::doDeCode()`
7. `BufferController::getPacket()`
8. `SuperMediaPlayer::DecodeVideoPacket()` / `SuperMediaPlayer::DecodeAudio()`
9. `SMPAVDeviceManager::sendPacket()`
10. `ActiveDecoder::send_packet()`
11. `SMPAVDeviceManager::getFrame()`
12. `SuperMediaPlayer::FillVideoFrame()` / `DecodeAudio()` frame 入队部分

读这条链时只问两个问题：

```text
这个对象现在拥有 packet/frame 吗？
如果下游暂时不能接收，这个 packet/frame 会不会丢？
```

## 这一章你需要真正掌握什么

- demuxer 输出 packet 后，先进入 `BufferController`，不是直接 decode/render。
- `MediaPacketQueue` 服务 buffering、seek、drop，不只是 FIFO。
- `mVideoPacket/mAudioPacket` 是 pending packet，用来处理 decoder retry。
- `ActiveDecoder` 内部还有 input/output queue，是 decoder 异步边界。
- frame queue 是 render 前的队列，职责和 packet queue 不一样。

