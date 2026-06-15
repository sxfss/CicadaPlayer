# Decode / Sync / Render 执行链：frame 如何按 clock 输出

这一章从 frame queue 开始，继续追普通 URL/MP4 的播放循环。目标是看清：decoder 产出的 frame
不是马上无条件 render，而是先经过 audio/video 不同路径，再由 master clock 决定视频等待、渲染或丢弃。

主链是：

```text
ProcessVideoLoop()
  -> doDeCode()
  -> mAudioFrameQue / mVideoFrameQue
  -> setUpAVPath()
  -> DoCheckBufferPass()
  -> doRender()
  -> RenderAudio()
  -> RenderVideo()
  -> SMPAVDeviceManager::renderAudioFrame/renderVideoFrame()
  -> audio/video render listener
  -> SuperMediaPlayer::RenderCallback()
```

## 这一层由谁持有？

`SuperMediaPlayer` 持有 frame queue 和 clock：

```text
std::queue<unique_ptr<IAFFrame>> mVideoFrameQue
std::deque<unique_ptr<IAFFrame>> mAudioFrameQue
SystemReferClock mMasterClock
```

`SuperMediaPlayer` 不直接持有 decoder/render 实现，而是通过：

```text
std::unique_ptr<SMPAVDeviceManager> mAVDeviceManager
```

`SMPAVDeviceManager` 持有：

```text
DecoderHandle mAudioDecoder
DecoderHandle mVideoDecoder
std::unique_ptr<IAudioRender> mAudioRender
std::unique_ptr<IVideoRender> mVideoRender
```

所以职责分布是：

```text
SuperMediaPlayer：决定什么时候 decode、什么时候 render、怎么同步。
SMPAVDeviceManager：管理具体 decoder/render 对象。
IDecoder：packet -> frame。
IAudioRender/IVideoRender：frame -> 设备或显示。
```

## 输入是什么？输出是什么？

输入：

```text
mAudioFrameQue.front()
mVideoFrameQue.front()
```

输出：

```text
音频：送入 audio render，audio render 反馈播放位置。
视频：按 master clock 判断是否 render，送入 video render 或 drop。
事件：render callback 反馈 rendered / first frame / statistics。
```

这里要分清：

```text
audio render 影响 master clock
video render 服从 master clock
```

有音频时，音频通常是更可信的播放时间来源。无音频时，视频会自己校准 clock。

## 主循环什么时候调用它？

主循环依然是：

```text
mainService()
  -> ProcessVideoLoop()
```

`ProcessVideoLoop()` 的后半段顺序是：

```text
doDeCode()
  -> 尝试产出 audio/video frame

setUpAVPath()
  -> 有 packet/meta 后创建 decoder
  -> 有 audio frame 后创建 audio render
  -> 有 video packet/meta 后创建 video render + decoder

DoCheckBufferPass()
  -> 判断是否可以结束 buffering / 是否要暂停渲染

doRender()
  -> RenderAudio()
  -> RenderVideo()
```

所以渲染不是单独线程里随便跑，而是在 `SuperMediaPlayer` 主循环里按状态推进。

## setUpAVPath：为什么不是一开始就创建所有东西？

`SetUpAudioPath()` 会先等 audio packet 出现：

```text
if audio packet queue empty:
    return 0
GetStreamMeta()
setUpAudioDecoder()
```

audio render 还要等第一帧 audio frame 出来，因为 render 需要真实音频格式：

```text
if mAudioFrameQue not empty and audio render invalid:
    setUpAudioRender(mAudioFrameQue.front()->getInfo().audio)
```

`SetUpVideoPath()` 也会先等 video packet、video meta、interlaced parser 结果，再创建 render 和 decoder。

这说明播放器对象不是构造函数里一次性全建好，而是数据流推进到某个阶段后逐步建立：

```text
有 URL 才能建 data source
有 probe bytes 才能选 demuxer
有 stream meta/packet 才能建 decoder
有 audio frame format 才能建 audio render
```

## decoderFactory 和 SMPAVDeviceManager 的关系

`SMPAVDeviceManager::setUpDecoder()` 会调用 `decoderFactory::create()`，返回 `unique_ptr<IDecoder>`。

简化关系：

```text
SuperMediaPlayer::setUpAudioDecoder()/CreateVideoDecoder()
  -> SMPAVDeviceManager::setUpDecoder()
  -> decoderFactory::create(meta, flags, ...)
  -> codecPrototype / built-in decoder
  -> IDecoder
```

`SuperMediaPlayer` 只关心“我有一个 audio/video decoder 可用”；具体是 FFmpeg 软解、平台硬解还是其他
prototype，实现细节都藏在 factory 后面。

第一轮读源码时，不要被硬解 fallback 带偏。先看 `IDecoder::send_packet()` 和 `getFrame()` 这条
共同接口。

## RenderAudio：音频如何成为 clock 参考

`RenderAudio()` 从 `mAudioFrameQue` 取 frame，调用：

```text
mAVDeviceManager->renderAudioFrame(mAudioFrameQue.front(), 0)
```

成功后，它会设置 master clock 的参考：

```text
mMasterClock.setReferenceClock(getAudioPlayTimeStampCB, this)
```

这表示 master clock 后续可以通过 callback 获取音频播放时间。音频 render 的队列时长、设备播放位置，
都会影响“当前到底播放到哪里”。

理解音频路径时按这个顺序：

```text
mAudioFrameQue.front()
  -> renderAudioFrame()
  -> IAudioRender::renderFrame()
  -> audio device queue
  -> getAudioPlayTimeStampCB
  -> mMasterClock.GetTime()
```

所以音频不仅是输出，它还是时间基准。

## RenderVideo：视频为什么会等待或丢帧

`RenderVideo()` 从 `mVideoFrameQue.front()` 取视频 frame，拿它的 pts 和 master clock 比较：

```text
masterPlayedTime = mMasterClock.GetTime()
videoLateUs = masterPlayedTime - videoPts
videoLateUs -= mVideoDelayTime
```

然后做三类决策：

```text
videoLateUs < -10ms
  -> 视频太早，不 render，不 pop，下一轮再看

videoLateUs 在可接受范围
  -> SendVideoFrameToRender()
  -> pop frame

videoLateUs 太大
  -> drop frame 或清理更早 packet
  -> pop frame / flush video path
```

这就是同步逻辑的核心：视频不是按最快速度显示，而是跟着 master clock。

无音频时，视频自己可以建立 clock：

```text
if (!HAVE_AUDIO && first video rendered)
  -> mMasterClock.setTime(videoPts)
  -> mMasterClock.setReferenceClock(mClockRef, mCRArg)
```

所以 audio/video 同步第一轮只要掌握：

```text
有音频：audio clock 做主，video 对齐它。
无音频：video 推进 clock。
```

## SendVideoFrameToRender 和 render callback

视频真正输出走：

```text
SendVideoFrameToRender(frame)
  -> mAVDeviceManager->renderVideoFrame(frame)
  -> IVideoRender::renderFrame()
```

render 完成或丢帧后，listener 会回到：

```text
ApsaraVideoRenderListener::onFrameInfoUpdate()
  -> SuperMediaPlayer::RenderCallback()
```

音频也类似：

```text
ApsaraAudioRenderCallback::onFrameInfoUpdate()
  -> SuperMediaPlayer::RenderCallback()
```

这个 callback 用来反馈 rendered、first frame、统计和内部消息。它不是 data path 的入口，而是 render
结果回到控制面/统计面的出口。

## DoCheckBufferPass 在渲染前做什么？

`DoCheckBufferPass()` 位于 decode 和 render 之间。它决定：

- 当前是否还在 buffering。
- packet 缓存是否达到起播水位。
- 播放中缓存耗尽时是否暂停 audio render 和 master clock。
- 实时流积压过多时是否清旧 packet 追赶。

所以 `doRender()` 不是每轮必定执行。只有 buffering 状态允许，才会继续 render。

这点能把 `07-buffering-and-qoe-study.md` 和本章串起来：

```text
packet queue 决定能不能继续播放；
frame queue 决定当前有没有可渲染帧；
clock 决定视频帧是否该现在显示。
```

## 下一层是谁？

render 是普通播放主链的末端。之后只有事件回流：

```text
rendered frame
  -> render listener
  -> RenderCallback()
  -> PlayerMessageControl internal message / notifier / stats
```

如果你继续读，可以转向两个方向：

- clock 细节：`mediaPlayer/system_refer_clock.*`
- render 实现：`framework/render/audio/*`、`framework/render/video/*`

第一轮建议先读 clock，不要先追平台 render。

## 读源码时按哪些函数跳？

建议按这个顺序：

1. `SuperMediaPlayer::ProcessVideoLoop()`
2. `SuperMediaPlayer::setUpAVPath()`
3. `SuperMediaPlayer::SetUpAudioPath()`
4. `SuperMediaPlayer::SetUpVideoPath()`
5. `SMPAVDeviceManager::setUpDecoder()`
6. `decoderFactory::create()`
7. `SuperMediaPlayer::doDeCode()`
8. `SuperMediaPlayer::FillVideoFrame()`
9. `SuperMediaPlayer::DecodeAudio()`
10. `SuperMediaPlayer::DoCheckBufferPass()`
11. `SuperMediaPlayer::doRender()`
12. `SuperMediaPlayer::RenderAudio()`
13. `SuperMediaPlayer::RenderVideo()`
14. `SuperMediaPlayer::SendVideoFrameToRender()`
15. `SuperMediaPlayer::RenderCallback()`

读的时候每一步问：

```text
它消费的是 packet 还是 frame？
它更新的是 queue、clock、render device，还是状态通知？
```

## 这一章你需要真正掌握什么

- decoder/render 对象由 `SMPAVDeviceManager` 聚合管理。
- `SuperMediaPlayer` 主循环决定 decode、buffering、render 的推进节奏。
- audio render 不只是输出，还提供 master clock 参考。
- video render 要根据 `mMasterClock.GetTime()` 判断等待、渲染或丢帧。
- render callback 是结果回流，不是拉流/解码入口。

