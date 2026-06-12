# Seek 控制链路：从用户意图到数据面重置

这一章专门看 CicadaPlayer 的 seek 控制。重点不是“调用一个 seek API”，而是理解播放器如何处理：

```text
连续 seek
  -> 只保留有价值的用户意图
  -> 避免旧 seek 和新 seek 交错
  -> 决定能否在缓存里完成
  -> 清理旧 packet / decoder / subtitle 状态
  -> 等真正渲染到新位置后再通知完成
```

seek 是播放器控制面和数据面交界最典型的功能。它同时涉及消息队列、buffer、demuxer、
decoder、render、clock 和外部通知。

## 1. 源码入口

主要文件：

- `mediaPlayer/SuperMediaPlayer.cpp`
- `mediaPlayer/SMPMessageControllerListener.cpp`
- `mediaPlayer/player_msg_control.cpp`
- `mediaPlayer/player_msg_control.h`
- `mediaPlayer/buffer_controller.*`
- `mediaPlayer/media_packet_queue.*`

第一轮只看这些函数：

- `SuperMediaPlayer::SeekTo()`
- `PlayerMessageControl::putMsg()`
- `SMPMessageControllerListener::OnPlayerMsgIsPadding()`
- `SMPMessageControllerListener::ProcessSeekToMsg()`
- `SuperMediaPlayer::SeekInCache()`
- `SuperMediaPlayer::ResetSeekStatus()`
- `SuperMediaPlayer::DoCheckBufferPass()`

## 2. 播放器问题是什么

seek 的难点不在于把 demuxer 跳到某个时间点，而在于这些边界：

- 用户拖动进度条时会连续发出很多 seek。
- seek 是慢操作，旧 seek 可能还没结束，新 seek 已经来了。
- seek 可能发生在 prepare 前、playing、paused、completion、stop 等不同状态。
- 目标位置可能还在当前缓存里，也可能必须让 demuxer 重新 seek。
- seek 后旧 packet、decoder 输入、字幕、滤镜缓存都可能变成过期数据。
- seek 完成通知不能太早，否则外部以为画面已经到新位置。

所以 seek 要维护的是“最新播放意图”和“数据面一致性”，不是单次函数调用成功。

## 3. 外部 API 只投递 seek 意图

`SuperMediaPlayer::SeekTo()` 做的事情很少：

```text
把秒级 pos 转成微秒 seekPos
  -> 构造 MsgSeekParam
  -> putMsg(MSG_SEEKTO)
  -> 记录 mSeekPos
  -> 记录 mSeekNeedCatch
```

这说明外部 API 线程不直接操作 demuxer，也不直接清空队列。它只把 seek 意图投递给内部控制面。

值得迁移的点：

- API 层只表达用户意图。
- 真正的数据面重置放到内部线程处理。
- accurate seek 不是普通 seek 的附加日志，而是会影响后续丢帧和 catch 行为。

## 4. PlayerMessageControl：连续 seek 的压缩

`PlayerMessageControl` 对 `MSG_SEEKTO` 使用 `SEEK_REPEAT_TIME = 500ms`。

简化理解：

```text
新 seek 入队
  -> 如果距离最后一个 seek 太近，删除最后一个旧 seek
  -> 如果队列里 seek 太多，删除最早的 seek
  -> 最后把新 seek 放入队列
```

这个设计的目标不是保留所有 seek，而是尽量保留用户最新意图，同时避免队列里堆积大量已经过期的目标位置。

迁移到 AVPlayer 时，第一版可以更简单：

```text
pendingSeek 只保留最新一个
每次 seek 分配 seekSerial
旧 SeekDone(serial) 直接丢弃
```

CicadaPlayer 的做法更偏历史工程，AVPlayer 可以把策略写得更清楚。

## 5. padding：正在 seek 时，新 seek 暂缓处理

`SMPMessageControllerListener::OnPlayerMsgIsPadding()` 对 `MSG_SEEKTO` 有一个关键判断：

```text
如果 mSeekFlag 为 true
  -> 新 MSG_SEEKTO padding
```

padding 的含义是：消息还留在队列里，这一轮 `processMsg()` 不处理它。

这个机制解决的问题是：当前 seek 已经进入数据面重置阶段，新 seek 不能直接和它交错执行。

但注意，它不是丢弃新 seek。新 seek 会在旧 seek 结束后继续被处理。

## 6. ProcessSeekToMsg：真正开始 seek

`SMPMessageControllerListener::ProcessSeekToMsg()` 是 seek 的核心处理点。

它大致做这些事：

```text
记录 mSeekNeedCatch / mSeekPos
  -> 如果还没 prepare，保留 mSeekPos 后返回
  -> 如果状态不允许 seek，ResetSeekStatus()
  -> 设置 mSeekFlag
  -> 重置已播放 audio/video pts
  -> 尝试 SeekInCache()
  -> NotifySeeking(mSeekInCache)
  -> 如果不在缓存里，清空 packet queue 并调用 demuxer Seek()
  -> FlushVideoPath / FlushAudioPath / FlushSubtitleInfo
```

这里有两条路径：

```text
seek in cache
  -> 不调用 demuxer seek
  -> 在已有 BufferController 里找关键帧位置并清理旧 packet

seek not in cache
  -> 清空所有 packet
  -> 调用 demuxer seek
  -> 重新读取目标位置附近数据
```

这就是 seek 的控制面和数据面分界。

## 7. SeekInCache：缓存内 seek

`SuperMediaPlayer::SeekInCache()` 判断目标位置是否还能在当前缓存中完成。

它会检查：

- 目标位置是否超过当前缓存的最后位置。
- 如果是向后 seek，目标位置是否早于当前缓存的最早位置。
- 目标位置之前是否能找到关键帧。

如果可以在缓存里完成，它会：

```text
Rewind packet queue
  -> 找目标前最近关键帧
  -> ClearPacketBeforeTimePos()
  -> 设置 mSoughtVideoPos
```

这个设计的价值是：短距离回退或缓存内跳转可以避免 demuxer 重新 seek，减少网络和解封装成本。

迁移到 AVPlayer 时，第一版不一定要实现完整缓存内 seek，但可以保留概念：

```text
SeekPlan
  -> InCache
  -> NeedDemuxerSeek
```

## 8. flush：seek 后旧数据必须失效

seek 之后，旧数据不只是 packet queue 里的 packet。

CicadaPlayer 会清理：

- `BufferController` 里的 packet。
- video path。
- audio path。
- subtitle info。
- video filter buffer。
- sub player seek 状态。

这说明 seek 是一次数据面重置：

```text
旧 demux packet
旧 decoder input
旧 decoded frame
旧 subtitle
旧 filter cache
旧 clock reference
  -> 都可能过期
```

AVPlayer 第一版可以先做最小闭环：

```text
PacketQueue flush
  -> decoder flush
  -> frame queue flush
  -> clock reset
  -> seekSerial++
```

关键不是清理多少模块，而是每个模块都知道“seek 后旧数据不能继续参与播放”。

## 9. seek 完成通知不能太早

CicadaPlayer 不是在 `ProcessSeekToMsg()` 调完 demuxer seek 后马上通知完成。

在渲染路径里，真正有 frame rendered 后才会处理：

```text
if (rendered && mSeekFlag)
  -> mSeekFlag = false
  -> 如果队列里没有新的 MSG_SEEKTO
       -> NotifyPosition(getCurrentPosition())
       -> ResetSeekStatus()
       -> NotifySeekEnd(mSeekInCache)
```

这里有两个关键点：

- seek 完成通知依赖渲染进度，不只是 demuxer seek 返回。
- 如果队列里还有新的 `MSG_SEEKTO`，就不急着通知 seek end。

这能避免外部收到一个过期 seek 的完成事件。

## 10. paused seek 的语义

seek 可能发生在 paused 状态。CicadaPlayer 在 seek end 前会更新 position：

```text
NotifyPosition(getCurrentPosition())
ResetSeekStatus()
NotifySeekEnd()
```

注释里也提到：paused 时要先更新 position，再 reset seek status，避免 `getCurrentPosition()`
返回不准确。

AVPlayer 迁移时要明确：

- paused seek 完成后仍然应该保持 paused。
- position snapshot 应更新到 seek 目标附近。
- 不应该因为 seek 完成而隐式 start。

## 11. 适合迁移到 AVPlayer 的最小版本

第一版可以只实现这套能力：

```text
PlayerCore::seek(positionMs)
  -> 投递 Seek command
  -> 覆盖 pending seek
  -> 分配 seekSerial

worker
  -> 执行 fake 或真实 demuxer seek
  -> 回投 SeekDone(seekSerial)

command thread
  -> serial 匹配才处理 SeekDone
  -> flush packet/frame/clock 状态
  -> paused 状态保持 paused
  -> 输出 timeline
```

建议先验证四个场景：

- 连续 seek 只执行最后一个目标。
- 旧 `SeekDone` 回来后被丢弃。
- paused 状态 seek 完成后仍保持 paused。
- stop/release 后 seek 回调不能改变状态。

等这个闭环稳定后，再做 cache seek。

## 12. 不建议照抄的地方

这些点适合理解历史工程，不建议直接照搬：

- `mSeekPos` 同时被 API 层和内部处理层写，语义需要结合上下文读。
- `mSeekFlag` 既参与 padding，又参与 render 后完成通知，职责偏重。
- seek 完成依赖 `findMsgByType(MSG_SEEKTO)`，逻辑分散在消息队列和 render 路径。
- `ProcessSeekToMsg()` 同时处理状态检查、cache seek、demuxer seek、flush、字幕、滤镜，函数职责很大。
- 没有显式 `seekSerial` 类型，旧结果过滤更多依赖队列和 flag。

AVPlayer 新实现应该把这些概念拆清楚：

```text
用户最新意图
当前正在执行的 seek
数据面 flush 边界
异步结果 serial
外部完成通知
```

## 13. 可公开的技术表达

可以这样概括这一章：

```text
CicadaPlayer 的 seek 不是一次简单的 demuxer seek 调用，而是一条控制面到数据面的完整链路。
API 层只投递 MSG_SEEKTO，PlayerMessageControl 负责压缩连续 seek，padding 避免 seek 交错。
ProcessSeekToMsg 决定 cache seek 还是 demuxer seek，并清理 packet、decoder、subtitle 等旧数据。
seek 完成通知要等新位置真正渲染，并且如果队列里还有新的 seek，就不通知旧 seek 完成。
迁移到 AVPlayer 时，可以先实现 latest-wins seek、seekSerial、flush 和 paused seek 语义。
```

