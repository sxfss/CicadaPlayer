# 核心对象关系与模块装配：CicadaPlayer 是怎么串起来的

如果只看 `PlayerMessageControl`，你会知道 API 命令怎么进主线程；但继续往下看拉流、解复用、解码、
同步、渲染时，很容易迷路。原因是 CicadaPlayer 不是一条简单函数链，而是一组对象互相持有和调用。

这一章先建立对象图。后面的 `09 -> 10 -> 12 -> 11` 都挂在这张图上读。

## 先分清两张图

读这种 C++ 播放器，建议永远分两张图：

```text
ownership graph：谁创建谁，谁释放谁，谁长期持有谁。
runtime call graph：播放循环里谁调用谁，数据怎么流动。
```

这两张图不是一回事。比如 `SuperMediaPlayer` 持有 `SMPAVDeviceManager`，但运行时 render callback
又会从 render 对象回调到 `SuperMediaPlayer`。如果只按“谁调用谁”读，很容易把所有关系看成一团。

## Ownership graph：核心对象谁拥有谁

普通内置播放器大致是：

```text
MediaPlayer
  owns C playerHandle
    owns ICicadaPlayer*
      usually SuperMediaPlayer
```

`MediaPlayer` 是 SDK 外观层。它把外部 API 转成 C API handle 调用，同时挂 listener、analytics、
ABR、cache 等外层能力。

`playerHandle` 是 C API 壳。它让 C 风格接口能够持有 C++ 播放器对象。

`ICicadaPlayer` 是核心播放器接口。默认实现通常是 `SuperMediaPlayer`。

`SuperMediaPlayer` 是真正的播放引擎，它持有主要模块：

```text
SuperMediaPlayer
  owns player_type_set
  owns PlayerMessageControl
  owns SMPMessageControllerListener
  owns BufferController
  owns demuxer_service
  owns SMPAVDeviceManager
  owns SystemReferClock
  owns MediaPlayerUtil / MediaPlayerAnalyticsUtil
  owns PlayerNotifier*
  owns audio/video frame queues
  owns pending audio/video packet
  owns afThread(mainService)
```

`SMPAVDeviceManager` 再聚合 decoder/render：

```text
SMPAVDeviceManager
  owns audio DecoderHandle
    owns unique_ptr<IDecoder>
  owns video DecoderHandle
    owns unique_ptr<IDecoder>
  owns unique_ptr<IAudioRender>
  owns unique_ptr<IVideoRender>
  owns DrmManager
```

`demuxer_service` 再聚合 demuxer：

```text
demuxer_service
  owns unique_ptr<IDemuxer>
  refers IDataSource*
```

`BufferController` 再聚合 packet queue：

```text
BufferController
  owns audio MediaPacketQueue
  owns video MediaPacketQueue
  owns subtitle MediaPacketQueue
```

## Runtime call graph：一轮播放循环谁调用谁

`SuperMediaPlayer` 创建了 `afThread`，线程函数是 `mainService()`。

普通播放时，一轮主循环大致是：

```text
mainService()
  -> mMessageControl->processMsg()
  -> ProcessVideoLoop()
      -> doReadPacket()
          -> ReadPacket()
              -> demuxer_service::readPacket()
              -> BufferController::AddPacket()
      -> doDeCode()
          -> BufferController::getPacket()
          -> SMPAVDeviceManager::sendPacket()
          -> SMPAVDeviceManager::getFrame()
          -> mAudioFrameQue / mVideoFrameQue
      -> setUpAVPath()
          -> SetUpAudioPath()
          -> SetUpVideoPath()
      -> DoCheckBufferPass()
      -> doRender()
          -> RenderAudio()
          -> RenderVideo()
```

这张图比模块目录更重要。你后面读任何函数，都先放回这条主循环里：它是在读包阶段、解码阶段、建路径阶段、
buffering 判断阶段，还是渲染阶段？

## 接口和实现关系

CicadaPlayer 大量使用接口和 prototype/factory。第一轮只需要记住这几组：

```text
IDataSource
  <- fileDataSource / CurlDataSource / ffmpegDataSource / proxyDataSource

IDemuxer
  <- avFormatDemuxer / playList_demuxer / sample decrypt demuxer

IDecoder
  <- avcodecDecoder / mediaCodecDecoder / AppleVideoToolBox / ActiveDecoder wrapper

IAudioRender
  <- SDL / Android AudioTrack / Apple audio render / filter render

IVideoRender
  <- SDL / GL render / dummy render / active render
```

选择实现的入口：

```text
dataSourcePrototype::create()
demuxerPrototype::create()
decoderFactory::create()
audioRenderPrototype::create()
renderFactory / video render creation
CicadaPlayerPrototype::create()
```

这套机制的作用是：`SuperMediaPlayer` 只依赖接口，不把 HTTP、本地文件、FFmpeg demux、硬解、平台 render
写死在主流程里。

## 控制面和数据面怎么分开？

控制面：

```text
MediaPlayer API
  -> C API
  -> ICicadaPlayer
  -> SuperMediaPlayer::putMsg()
  -> PlayerMessageControl
  -> SMPMessageControllerListener
```

数据面：

```text
IDataSource bytes
  -> IDemuxer packet
  -> BufferController packet queue
  -> IDecoder frame
  -> frame queue
  -> render
```

控制面改变状态和意图，比如 prepare、start、pause、seek、stop。数据面在主循环中持续推进 bytes、packet、
frame 和 render。

你现在已经能看懂 `PlayerMessageControl`，下一步要把它理解成“入口控制器”，而不是整个播放器。真正的
播放主链在 `ProcessVideoLoop()` 里。

## SuperMediaPlayer 为什么看起来很大？

`SuperMediaPlayer` 同时做了很多事：

- 接收控制命令。
- 管理播放状态。
- 创建 data source / demuxer / decoder / render。
- 调度主循环。
- 管理 packet queue 和 frame queue。
- 做 buffering 判断。
- 做音视频同步。
- 发通知和统计。

这不是现代小类设计的好范本，但很适合学习播放器 SDK 的真实工程压力。读它时不要试图一次理解所有成员，
而是按主循环阶段给成员分组：

```text
control：mMessageControl, mMsgCtrlListener, mPlayStatus
source/demux：mDataSource, mDemuxerService
packet：mBufferController, mVideoPacket, mAudioPacket
decode/render：mAVDeviceManager, mAudioFrameQue, mVideoFrameQue
sync：mMasterClock, mPlayedAudioPts, mPlayedVideoPts
notify/stats：mPNotifier, mUtil, mMPAUtil, mRecorderSet
```

这样读大类会轻松很多。

## 普通 URL/MP4 的完整主链

把对象关系和调用关系合起来，普通 URL/MP4 的主链是：

```text
MediaPlayer::SetDataSource(url)
  -> C API handle
  -> SuperMediaPlayer stores url

Prepare/Start command
  -> PlayerMessageControl
  -> SMPMessageControllerListener
  -> create/open data source and demuxer

mainService loop
  -> doReadPacket()
      -> demuxer_service
      -> avFormatDemuxer
      -> IDataSource::Read()
      -> IAFPacket
      -> BufferController
  -> doDeCode()
      -> BufferController::getPacket()
      -> IDecoder::send_packet()
      -> IDecoder::getFrame()
      -> frame queue
  -> doRender()
      -> RenderAudio()
      -> RenderVideo()
      -> IAudioRender / IVideoRender
      -> RenderCallback()
```

这就是你后面读源码时的主地图。

## 读源码时按哪些函数跳？

建议用这条路线读一遍，不要先展开所有类：

1. `MediaPlayer::MediaPlayer()`
2. `CicadaCreatePlayer()`
3. `CicadaPlayerPrototype::create()`
4. `SuperMediaPlayer::SuperMediaPlayer()`
5. `SuperMediaPlayer::mainService()`
6. `SuperMediaPlayer::ProcessVideoLoop()`
7. `SuperMediaPlayer::doReadPacket()`
8. `SuperMediaPlayer::ReadPacket()`
9. `SuperMediaPlayer::doDeCode()`
10. `SuperMediaPlayer::setUpAVPath()`
11. `SuperMediaPlayer::DoCheckBufferPass()`
12. `SuperMediaPlayer::doRender()`

每跳一步都问：

```text
这个对象是谁持有的？
它现在处理的是控制命令、bytes、packet、frame、clock，还是通知？
它的下一层是谁？
```

## 这一章你需要真正掌握什么

- `MediaPlayer` 是外观层，不是播放主循环。
- `SuperMediaPlayer` 是播放引擎，`mainService()` 是运行时核心。
- `SMPAVDeviceManager` 把 decoder/render 聚合起来，避免主循环直接持有所有平台对象。
- `demuxer_service` 把 `IDataSource` 和 `IDemuxer` 接起来。
- `BufferController` 是 packet 缓存聚合层，frame queue 在 `SuperMediaPlayer` 里。
- 控制面入口是 `PlayerMessageControl`，数据面主线是 `ProcessVideoLoop()`。

