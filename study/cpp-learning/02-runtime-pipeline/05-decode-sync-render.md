# Decode / Sync / Render：frame 为什么不能解出来就显示

这一章接在 [01-normal-mp4-playback-loop.md](01-normal-mp4-playback-loop.md) 后面。它只追普通 URL/MP4 的
audio/video 输出，不展开平台 render 细节和硬解实现。

这一章要回答的问题是：

```text
decoder 已经产出 frame 之后，播放器如何决定：
  audio 什么时候送设备？
  video 什么时候等待？
  video 什么时候渲染？
  video 什么时候丢弃？
  render 完成后事件怎么回流？
```

读完你应该能画出两张图：

- sync 图：`audio render -> master clock -> video wait/render/drop`
- ownership 图：`SMPAVDeviceManager` 管 decoder/render，`SuperMediaPlayer` 管调度和 frame queue

## 0. 先看主图

```text
ProcessVideoLoop()
  -> doDeCode()
      -> DecodeAudio()
      -> DecodeVideoPacket()
      -> mAudioFrameQue / mVideoFrameQue
  -> setUpAVPath()
      -> SetUpAudioPath()
      -> SetUpVideoPath()
      -> SMPAVDeviceManager
  -> DoCheckBufferPass()
      -> decide buffering or render allowed
  -> doRender()
      -> render()
          -> RenderAudio()
              -> IAudioRender
              -> getAudioPlayTimeStampCB
              -> SystemReferClock
          -> RenderVideo()
              -> compare video PTS with master clock
              -> wait / render / drop
              -> IVideoRender
          -> RenderSubtitle()
  -> RenderCallback()
      -> MSG_INTERNAL_RENDERED
      -> first frame / rendered / position / stats
```

关键源码入口集中在 [mediaPlayer/SuperMediaPlayer.cpp](../../../mediaPlayer/SuperMediaPlayer.cpp)：

- `doDeCode()`，当前约 `1946` 行：把 packet 推进到 decoder 和 frame queue。
- `render()`，当前约 `2329` 行：统一调度 audio、video、subtitle render。
- `RenderAudio()`，当前约 `2369` 行：送音频 frame，并建立 audio master clock。
- `RenderVideo()`，当前约 `2507` 行：按 master clock 处理 video frame。
- `RenderCallback()`，当前约 `4084` 行：render 结果回流到内部消息和外部通知。

## 1. 这一层解决什么播放器问题？

frame queue 里有 frame，不代表播放器可以立即显示它。

真实播放器要同时处理：

- audio render 设备有自己的队列和播放进度。
- video frame 解码可能早于播放时间，需要等。
- video frame 可能已经晚了，需要丢帧追赶。
- seek 后旧 callback 可能晚到，不能污染 position。
- 首帧通知应该接近用户真实看到/听到的时刻。
- buffer 不足时，即使有 frame，也可能不能继续 render。

所以 decode/sync/render 不是“decode 完就显示”，而是一套调度策略：

```text
decoded frame
  -> buffer gate
  -> clock decision
  -> render or wait or drop
  -> callback / stats
```

## 2. 这一层由谁持有？

`SuperMediaPlayer` 持有调度状态、frame queue 和 clock：

```text
std::deque<std::unique_ptr<IAFFrame>> mAudioFrameQue
std::queue<std::unique_ptr<IAFFrame>> mVideoFrameQue
SystemReferClock mMasterClock
```

`SMPAVDeviceManager` 聚合具体 decoder/render：

```text
DecoderHandle mAudioDecoder
DecoderHandle mVideoDecoder
std::unique_ptr<IAudioRender> mAudioRender
std::unique_ptr<IVideoRender> mVideoRender
```

职责分布：

```text
SuperMediaPlayer
  -> 决定什么时候 decode、什么时候 render、怎么同步。

SMPAVDeviceManager
  -> 管理 decoder/render 对象，提供 sendPacket/getFrame/renderFrame/flushDevice。

IDecoder
  -> packet -> frame。

IAudioRender / IVideoRender
  -> frame -> 设备或显示。
```

这是一条重要边界：`SuperMediaPlayer` 不直接知道具体是 SDL、Android、Apple 还是 GL render，它通过设备管理层和接口调用。

## 3. setUpAVPath：输出路径不是 prepare 时一次性建好

很多初学者会以为 prepare 阶段就应该建好所有 decoder/render。CicadaPlayer 不是这样。

audio path 依赖：

```text
audio packet
  -> stream meta
  -> audio decoder
  -> first audio frame
  -> audio render
```

video path 依赖：

```text
video packet
  -> stream meta / parser result
  -> video decoder
  -> video render
```

原因是 render 需要真实格式。例如 audio render 需要 sample rate、channels、sample format；这些信息在第一帧出来后更可靠。

值得学习：

- 延迟创建 decoder/render，依赖真实流信息。
- 主流程只依赖接口，具体实现交给 factory/prototype。

不建议照抄：

- setup 逻辑散落在大类多个函数里。AVPlayer 可以把它收进 `AvPathBuilder` 或 `OutputPathController`。

## 4. RenderAudio：audio 不只是输出，还是 master clock

`RenderAudio()` 的主链：

```text
mAudioFrameQue.front()
  -> SMPAVDeviceManager::renderAudioFrame()
  -> IAudioRender::renderFrame()
  -> audio device queue
  -> getAudioPlayTimeStampCB()
  -> mMasterClock.GetTime()
```

音频送入 render 后，`mMasterClock` 会绑定音频播放时间：

```text
mMasterClock.setReferenceClock(getAudioPlayTimeStampCB, this)
```

这意味着后续判断“当前播放到哪里”，优先相信 audio render 的播放进度，而不是简单相信系统时间或 video PTS。

### RenderAudio 条件表

| 条件 | CicadaPlayer 行为 | 学习重点 |
| --- | --- | --- |
| `mAudioFrameQue.empty()` | 没有 audio frame，本轮不渲染；如果 audio EOS 且 render 队列也空，会取消 master reference | audio queue 空不等于播放结束，还要看 decoder EOS 和设备队列 |
| audio frame `pts == INT64_MIN` | 丢掉这个异常 frame | PTS 无效的 frame 不能参与 clock |
| audio render 返回格式不支持 | 如果设备队列为空，按当前 frame 格式重建 audio render | render 依赖真实 frame 格式，格式变化要能恢复 |
| audio 设备打开失败且有视频 | 关闭 audio，发 open audio device failed event，继续 video | audio 失败不一定必须让播放器整体失败 |
| audio 设备打开失败且无视频 | 切 `PLAYER_ERROR` 并通知错误 | 纯音频场景没有 fallback 输出 |
| 第一帧有效 audio | 初始化 `mAudioTime`，设置 audio master clock | audio render 成为时间基准 |
| seek 中第一帧 audio | 先更新 `mCurrentPos`，避免异步 callback 导致位置不准 | seek 后 position 更新要防旧状态干扰 |

## 5. RenderVideo：video 是调度问题，不是输出函数

`RenderVideo()` 的核心是比较 video PTS 和 master clock：

```text
masterPlayedTime = mMasterClock.GetTime()
videoLateUs = masterPlayedTime - videoPts
videoLateUs -= mVideoDelayTime
```

含义：

```text
videoLateUs < 0
  -> video 早于 master clock，还没到显示时间。

videoLateUs 接近 0
  -> 可以显示。

videoLateUs 很大
  -> video 已经晚了，需要丢帧或追赶。
```

### RenderVideo 条件表

| 条件 | CicadaPlayer 行为 | 学习重点 |
| --- | --- | --- |
| video render 无效 | 直接返回，不渲染 | render path 必须先建立 |
| `mVideoFrameQue.empty()` | 直接返回，不渲染 | 没有 frame 和不能播放是两个不同问题 |
| video frame 为空或 PTS 异常 | 保护性处理，避免非法 PTS 破坏同步 | 媒体数据要防御异常时间戳 |
| master clock 不存在或失效，且偏差过大 | 用 video PTS 校准 master clock | 无 audio 或 master 异常时，video 可以临时校准 clock |
| `videoLateUs < -10ms` | video 太早，本轮不 pop，下一轮再看 | 早到的 frame 要等，不是丢 |
| `videoLateUs >= 500ms` 且非 PTS reverting | 尝试清理更早 video packet，flush video path，进入 catching up | 晚太多时继续按顺序播会越来越延迟 |
| `dropLateVideoFrames` 且 video 仍晚 | 丢弃当前 frame 或继续追赶 | 追帧状态要持续到延迟恢复 |
| `videoLateUs < 500ms` | 允许 render | 小范围 late 可以直接显示 |
| 长时间没有渲染 | 即使较晚也尝试 render | 避免长时间黑屏或画面停住 |
| 无 audio 且首次 video render | 用 video PTS 建立 master clock | 无音频时 video 自己推进时间 |

### 三类结果

```text
wait:
  video 太早，不 pop frame。

render:
  SendVideoFrameToRender()
  pop frame
  update played video pts

drop:
  mark discard
  RenderCallback(rendered=false)
  pop frame
  enter catching up
```

这就是 A/V sync 的主线：audio 推进时间，video 服从时间。

## 6. RenderCallback：结果回流，而不是数据入口

render callback 的路径：

```text
IAudioRender / IVideoRender
  -> ApsaraAudioRenderCallback / ApsaraVideoRenderListener
  -> SuperMediaPlayer::RenderCallback()
  -> put MSG_INTERNAL_RENDERED
  -> PlayerMessageControl
  -> notifier / stats / first frame
```

`RenderCallback()` 先过滤状态：

```text
if canceled:
  ignore

if status not in PREPARED / PAUSED / PLAYING:
  ignore
```

### RenderCallback 事件表

| 输入 | 行为 | 学习重点 |
| --- | --- | --- |
| `mCanceled == true` | 直接忽略 | stop/release 后不要继续更新状态 |
| 当前状态不是 prepared/paused/playing | 直接忽略 | callback 可能晚到，必须过滤生命周期状态 |
| rendered audio/video frame | 投递 `MSG_INTERNAL_RENDERED` | callback 不直接改全部状态，回到消息系统处理 |
| first frame | 由后续内部消息触发 first frame 通知 | 首帧应接近真实 render，而不是 decode 完成 |
| rendered=false | 仍回流统计或状态 | 丢帧也是 QoE 信号 |

这个 callback 是结果回流，不是拉流入口。不要把它和 data source callback 混在一起。

## 7. DoCheckBufferPass 和 render 的关系

`DoCheckBufferPass()` 位于 decode 和 render 之间：

```text
doDeCode()
  -> frame queue maybe has frames
DoCheckBufferPass()
  -> decide can render
doRender()
  -> RenderAudio / RenderVideo
```

它说明一个关键点：

```text
有 frame
  != 可以 render
```

render 前还要看：

- 是否正在 buffering。
- 起播或恢复播放水位是否够。
- seek 状态是否还没结束。
- live 场景是否需要追帧清缓存。

因此同步渲染要和 buffering 一起理解。

## 8. 值得学习的设计

- audio master clock：用实际音频播放位置做时间基准，比系统时间更贴近用户听到的进度。
- video wait/render/drop：video render 是调度决策，不是简单输出。
- render callback 回流：首帧、rendered、drop 都能回到事件和统计链路。
- decoder/render 接口隔离：主循环不绑定具体平台实现。
- buffer gate 在 render 前：避免刚解出少量 frame 就播放导致频繁卡顿。

## 9. 不建议直接照抄的地方

- `RenderVideo()` 分支较多，PTS reverting、drop late、catch up、seek 等逻辑交织，读起来困难。
- `SuperMediaPlayer` 同时管理 frame queue、clock、render、通知、统计，职责偏大。
- render callback 的状态过滤依赖多个外部状态，现代实现最好引入 generation/serial。
- status 和错误仍然偏 C 风格，类型表达不够清楚。

## 10. AVPlayer 最小迁移版

第一版不要照搬整套 render。可以拆成三个小对象：

```text
AudioClockSource
  -> 从 audio render 获取播放位置

RenderScheduler
  -> input: videoPts, clockNow, thresholds
  -> output: Wait / Render / Drop

RenderEventSink
  -> rendered / dropped / firstFrame
  -> 回到 PlayerCore 线程过滤 serial
```

最小验证：

```text
1. audio frame render 后 clock 开始前进。
2. videoPts 早于 clock 10ms 以上时等待。
3. videoPts 晚于 clock 500ms 以上时进入 drop/catch-up。
4. seekSerial 变化后，旧 render callback 被忽略。
5. first frame 只在真正 render 后通知。
```

## 11. 读源码时按哪些函数跳？

第一轮只按这个顺序读：

```text
SuperMediaPlayer::doDeCode()
SuperMediaPlayer::setUpAVPath()
SuperMediaPlayer::SetUpAudioPath()
SuperMediaPlayer::SetUpVideoPath()
SuperMediaPlayer::render()
SuperMediaPlayer::RenderAudio()
SuperMediaPlayer::RenderVideo()
SuperMediaPlayer::SendVideoFrameToRender()
SuperMediaPlayer::RenderCallback()
SMPAVDeviceManager::renderAudioFrame()
SMPAVDeviceManager::renderVideoFrame()
```

每看一个函数，只问：

```text
它消费的是 packet 还是 frame？
它更新的是 queue、clock、render device，还是事件？
它会让 frame 等待、渲染、丢弃，还是只回调通知？
```
