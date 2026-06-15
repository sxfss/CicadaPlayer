# Buffering 与 QoE：从缓存时长到播放体验

这一章专门看 CicadaPlayer 的 buffering 思路。重点不是“队列里有几个 packet”，而是：

```text
packet 被读入
  -> 按音频/视频/字幕分别进入 BufferController
  -> MediaPacketQueue 维护缓存时长
  -> SuperMediaPlayer 根据缓存时长决定是否 loading
  -> demuxer 收到 client buffer level
  -> ABR / cache / analytics 从旁路读取状态
```

这条链路很适合迁移到 AVPlayer。它不是巨型功能，但能把播放器原理中的数据面、
控制面和体验指标串起来。

## 1. 源码入口

主要文件：

- `mediaPlayer/buffer_controller.h`
- `mediaPlayer/buffer_controller.cpp`
- `mediaPlayer/media_packet_queue.h`
- `mediaPlayer/media_packet_queue.cpp`
- `mediaPlayer/SuperMediaPlayer.cpp`
- `mediaPlayer/abr/AbrBufferAlgoStrategy.cpp`
- `mediaPlayer/abr/AbrBufferRefererData.cpp`
- `framework/cacheModule/CacheManager.cpp`

第一轮只看这些函数：

- `BufferController::AddPacket()`
- `MediaPacketQueue::AddPacket()`
- `MediaPacketQueue::GetDuration()`
- `SuperMediaPlayer::getPlayerBufferDuration()`
- `SuperMediaPlayer::DoCheckBufferPass()`
- `SuperMediaPlayer::PostBufferPositionMsg()`
- `AbrBufferAlgoStrategy::ComputeBufferTrend()`
- `CacheManager::init()`
- `CacheManager::sendMediaFrame()`

## 2. 播放器问题是什么

buffering 不是简单的“没数据了”。

播放器至少要回答这些问题：

- 当前音频、视频、字幕各自还有多少可播放时长？
- 起播时要攒多少数据才算 ready？
- 播放中缓存耗尽时，什么时候进入 loading？
- loading 后要攒到多少数据才能恢复播放？
- 缓存太多时，demuxer / 网络下载是否应该降速或暂停？
- 直播延迟太大时，是追帧、丢帧，还是继续堆积？
- 卡顿、缓冲进度、buffer position 怎么通知外部？

这些问题都不能只靠 packet 数量解决。packet 数量和可播放时长不是一回事：

```text
同样 20 个 packet
  -> 音频可能覆盖几百毫秒
  -> 视频可能覆盖一秒多
  -> 不同码率和帧率下意义也不同
```

所以这一章的关键指标是 **buffer duration**。

## 3. BufferController：按媒体类型管理 packet 队列

`BufferController` 内部有三条队列：

```text
mVideoPacketQueue
mAudioPacketQueue
mSubtitlePacketQueue
```

外部通过 `BUFFER_TYPE_VIDEO / AUDIO / SUBTITLE / AV / ALL` 选择操作对象。

它解决的问题是：播放器主循环不用直接操作三条具体队列，而是通过一个入口完成：

- add packet
- get packet
- clear packet
- 查询 duration
- 查询 pts / timePosition
- 按时间清理旧 packet
- 找关键帧位置

典型链路：

```text
ReadPacket()
  -> 识别 streamIndex
  -> mBufferController->AddPacket(packet, BUFFER_TYPE_VIDEO/AUDIO/SUBTITLE)

DecodeAudio / DecodeVideo
  -> mBufferController->getPacket(BUFFER_TYPE_AUDIO/VIDEO)
```

这个设计值得迁移的是“按媒体类型隔离队列 + 统一查询接口”，不是具体实现细节。

## 4. MediaPacketQueue：缓存时长从哪里来

`MediaPacketQueue::AddPacket()` 会根据 packet duration 更新内部时长：

```text
如果 packet 自带 duration
  -> 累加到 mDuration 和 mTotalDuration

如果 packet 没有 duration，但之前推断过 mPacketDuration
  -> 用 mPacketDuration 补齐
```

这里能看到一个播放器工程里的现实问题：并不是所有 packet 都天然带有可靠 duration。

CicadaPlayer 的做法是维护一个 `mPacketDuration`：

- 第一次遇到有效 duration 时记下来。
- 后续缺 duration 的 packet 可以用它补。
- `SetOnePacketDuration()` 还能把已有队列里缺失的 duration 补上。

这不是完美算法，但体现了一个重要原则：

```text
buffering 判断要尽量基于可播放时长，而不是队列长度。
```

## 5. getPlayerBufferDuration：真正用于决策的缓存时长

`SuperMediaPlayer::getPlayerBufferDuration()` 不只看 `BufferController`。

它会把几类数据合在一起：

- `BufferController` 里的 packet duration。
- demuxer 内部还没交出来的 buffer duration。
- decoder input padding 里还没解出的数据。
- 音频、视频、字幕不同流的 duration。

简化理解：

```text
audio buffered duration
video buffered duration
subtitle buffered duration
decoder padding duration
demuxer inner buffer duration
  -> 取一个用于播放决策的 duration
```

这里有一个重要点：播放器不能只看“读线程已经放进 packet queue 的数据”。

真实播放链路里，数据可能卡在不同层：

```text
demuxer 内部
packet queue
decoder input
frame queue
render 设备
```

第一版 AVPlayer 不需要全部做完，但可以保留这个建模思路：

```text
BufferSnapshot
  -> packetQueueDuration
  -> demuxerBufferedDuration
  -> decoderPendingDuration
  -> eof
```

## 6. DoCheckBufferPass：buffering 状态怎么切

`SuperMediaPlayer::DoCheckBufferPass()` 是 buffering 决策的核心入口。

它大致做四类事情：

### 6.1 通知 demuxer 当前 buffer level

当缓存低于 `highLevelBufferDuration` 时，如果正在播放，会把 demuxer 的 client
buffer level 设为 low。接近 `maxBufferDuration` 时，会设为 low full。

这个设计说明：buffering 不是只影响播放器状态，也会反向影响读取和下载策略。

### 6.2 起播和首缓冲

首次缓冲使用 `startBufferDuration` 作为阈值。准备阶段只有缓存达到阈值，或者 EOF 等特殊情况，才会进入 prepared。

这里要学的是：起播阈值和播放中恢复阈值可以不同。

```text
startBufferDuration
  -> 起播门槛

highLevelBufferDuration
  -> 播放中恢复门槛

maxBufferDuration
  -> 缓存上限附近的反压信号
```

### 6.3 播放中缓存耗尽

当 `cur_buffer_duration <= 0`，且当前处于 playing 或 paused，播放器会进入 buffering：

```text
mBufferingFlag = true
NotifyLoading(start)
pause master clock
pause audio render
```

这说明 loading 不是一个 UI 事件而已，它会影响 clock 和 audio render。

### 6.4 缓冲恢复

当缓存重新超过阈值，或者 EOF 到达，播放器会退出 buffering：

```text
NotifyLoading(end)
restart master clock
resume audio render
mBufferingFlag = false
```

这是一套完整闭环：

```text
buffer duration 低
  -> loading start
  -> clock/render pause

buffer duration 恢复
  -> loading end
  -> clock/render resume
```

## 7. Buffer position：外部看到的缓存进度

`PostBufferPositionMsg()` 会把当前播放位置加上 buffer duration，得到 buffer position：

```text
bufferPosition = currentPosition + bufferDuration
```

如果 EOF 到达，则 buffer position 直接等于 duration。

这就是用户界面里常见的“已缓冲到哪里”。它不是队列长度，而是播放时间轴上的位置。

AVPlayer 可以把它作为最小 QoE 指标之一：

- current position
- buffer position
- buffered duration
- buffering state

## 8. ABR buffer trend：不只看当前值，也看趋势

`AbrBufferAlgoStrategy::ComputeBufferTrend()` 体现了另一个思路：不要只看当前 buffer duration，
还要看一段时间内缓存是在变多还是变少。

它维护：

- `mLastBufferDuration`
- `mBufferStatics`
- `mDownloadSpeed`
- `mIsUpHistory`

简化逻辑：

```text
如果正在 buffering
  -> 记为下降趋势

如果当前 buffer duration 比上次大，或者已经接近满
  -> 记为上升趋势

否则
  -> 记为下降趋势
```

然后根据趋势和阈值决定是否切码率：

```text
低缓存 + 连续下降
  -> 降码率

高缓存 + 连续上升
  -> 升码率
```

AVPlayer 当前不需要做 ABR，但可以先借鉴一个小点：

```text
BufferTrend
  -> Rising
  -> Falling
  -> Stable
```

它可以用于日志和 metrics，不必马上驱动码率选择。

## 9. CacheManager：缓存是旁路，不是播放主线

`CacheManager` 的接入方式值得看：

```text
set cache config
set source url
init()
  -> 如果已有缓存文件，返回本地路径
  -> 如果可缓存，打开 frame processing

sendMediaFrame()
  -> 首次设置媒体信息和 stream meta
  -> 启动 cache module
  -> 把后续 packet/frame 送给 cache module
```

它说明缓存能力可以作为旁路模块挂在播放链上：

```text
播放主链
  -> 正常读 packet、解码、渲染

缓存旁路
  -> 观察媒体帧
  -> 写入缓存模块
  -> 成功/失败用 callback 上报
```

这个思想适合迁移。不要一开始做完整缓存系统，可以先做：

- `PlaybackMetrics` 记录 buffer position。
- `CacheProbe` 或 `FrameTrace` 旁路观察 packet。
- 后续再决定是否写文件缓存。

## 10. 适合迁移到 AVPlayer 的最小版本

第一版只做 `BufferSnapshot + BufferController + Metrics`：

```text
PacketQueue
  -> 维护 packet count
  -> 维护 buffered duration

BufferSnapshot
  -> audioBufferedDuration
  -> videoBufferedDuration
  -> minBufferedDuration
  -> maxBufferedDuration
  -> eof

BufferController
  -> 根据 low/high threshold 做状态判断

PlayerCore
  -> 根据 BufferDecision 投递 InternalBufferLow / InternalBufferEnough

Metrics
  -> rebuffer count
  -> rebuffer total duration
  -> max rebuffer duration
  -> buffer position
```

推荐先验证三个场景：

- 起播前缓存不足，不进入 ready。
- 播放中缓存耗尽，进入 buffering。
- 缓存恢复到 high watermark，退出 buffering。

## 11. 哪些地方不要照抄

不建议直接照搬：

- `SuperMediaPlayer::DoCheckBufferPass()` 过大，混合了准备、buffering、直播追帧、drop、通知和错误处理。
- `BufferController` 暴露的查询和清理接口很多，第一版 AVPlayer 不需要这么宽。
- `MediaPacketQueue` 使用 `recursive_mutex`，新代码里应尽量避免把递归锁作为默认选择。
- 缓存、ABR、loading、demuxer buffer level 混在一个大控制循环里，学习时要拆成概念。
- `CacheManager` 内部仍有手动释放和裸指针，适合学习生命周期审查，不适合作为新写法模板。

## 12. 可公开的技术表达

可以这样概括这一章：

```text
CicadaPlayer 的 buffering 设计不是只看 packet queue 是否为空，而是围绕 buffer duration
建立播放决策。packet queue 负责维护可播放时长，SuperMediaPlayer 根据起播阈值、
播放中阈值和缓存上限切换 loading 状态，并把 buffer level 反馈给 demuxer。
ABR 和 cache 作为旁路模块消费这些状态或媒体帧。迁移到 AVPlayer 时，可以先实现
BufferSnapshot、low/high watermark、rebuffer metrics 和 buffer position。
```

