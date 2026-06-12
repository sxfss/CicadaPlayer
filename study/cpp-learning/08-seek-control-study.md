# Seek 控制链路：从用户意图到 flush 和完成通知

这一章专门看 CicadaPlayer 的 seek 控制。seek 不是一个单点 API，而是一条横跨控制面、
buffer、demuxer、decoder/render 和通知层的链路。

第一轮读 seek 时不要陷进所有分支。先抓住这条主线：

```text
用户调用 SeekTo
  -> 转成 MSG_SEEKTO
  -> PlayerMessageControl 压缩/排队
  -> ProcessSeekToMsg 设置 seek 状态
  -> 判断是否能在缓存内 seek
  -> 必要时清空 packet 并调用 demuxer seek
  -> flush decoder/render/subtitle 相关路径
  -> 渲染到新位置后通知 seek end
```

这个专题的重点是播放器机制，不是 C++ 语法。C++ 只在消息参数、状态所有权和线程边界处作为辅助视角。

## 1. 源码入口

主要文件：

- `mediaPlayer/SuperMediaPlayer.cpp`
- `mediaPlayer/SMPMessageControllerListener.cpp`
- `mediaPlayer/player_msg_control.cpp`
- `mediaPlayer/player_msg_control.h`
- `mediaPlayer/buffer_controller.cpp`
- `mediaPlayer/media_packet_queue.cpp`

第一轮只看这些函数：

- `SuperMediaPlayer::SeekTo()`
- `PlayerMessageControl::putMsg()`
- `PlayerMessageControl::processMsg()`
- `SMPMessageControllerListener::OnPlayerMsgIsPadding()`
- `SMPMessageControllerListener::ProcessSeekToMsg()`
- `SuperMediaPlayer::SeekInCache()`
- `SuperMediaPlayer::ResetSeekStatus()`
- `SuperMediaPlayer::DoCheckBufferPass()`
- `SuperMediaPlayer::doRender()`

## 2. 播放器问题是什么

seek 难点不只是“跳到某个时间点”。

真实播放器要处理这些问题：

- 用户连续拖动进度条，短时间内产生多个 seek。
- seek 期间旧 packet、旧 decoded frame、旧 render 回调还可能存在。
- 新 seek 可能发生在上一个 seek 还没结束时。
- seek 可能命中当前缓存，也可能必须让 demuxer 重新定位。
- 暂停态 seek 完成后，位置要更新，但不应该误恢复播放。
- seek 完成通知不能太早，否则 UI 以为已经到新位置，但画面还没切过去。

所以 seek 的核心不是单次跳转，而是维护“最新用户意图”和“旧数据隔离”。

## 3. SeekTo：API 只表达用户意图

`SuperMediaPlayer::SeekTo()` 做的事情很少：

```text
把秒级 pos 转成微秒
填入 MsgSeekParam
putMsg(MSG_SEEKTO)
记录 mSeekPos
记录 mSeekNeedCatch
```

这说明外部 API 没有直接调用 demuxer seek，也没有直接清空队列。

设计价值：

- API 调用线程只提交意图。
- 内部播放线程按顺序解释意图。
- seek 和 start/pause/stop 等命令统一进入控制面。

迁移到 AVPlayer 时，public API 也应该只投递 `SeekRequested(position, accurate)`，
真实 seek 由 command thread 执行。

## 4. PlayerMessageControl：连续 seek 的第一道压缩

`PlayerMessageControl` 对 `MSG_SEEKTO` 使用 `SEEK_REPEAT_TIME = 500ms`。

它不是无限保留所有 seek，而是做两层控制：

```text
如果新 seek 距离最后一个 seek 很近
  -> 删除最后一个旧 seek

如果队列里同类 seek 已经太多
  -> 删除更早的 seek
```

这体现了 seek 的一个基本原则：

```text
拖动进度条时，旧位置通常没有执行价值，最新位置才代表用户意图。
```

这和“每个 API 调用都必须完整执行”不同。播放器控制面要允许压缩过期意图。

## 5. padding：正在 seek 时暂缓新的 seek

`SMPMessageControllerListener::OnPlayerMsgIsPadding()` 会检查：

```text
如果当前 mSeekFlag 为 true
  -> MSG_SEEKTO padding
```

也就是说，当前 seek 没有结束时，新的 seek 不会马上进入业务处理函数，而是留在消息队列里等待。

这个设计解决的是交错问题：

```text
旧 seek 正在清队列 / flush / 等首帧
新 seek 又开始清队列 / flush
  -> 状态容易互相覆盖
```

padding 让 seek 过程保持串行。再结合前面的重复消息压缩，可以减少 pending seek 的数量。

## 6. ProcessSeekToMsg：真正执行 seek 意图

`ProcessSeekToMsg()` 是 seek 业务处理的核心。

它先更新 seek 状态：

```text
mSeekNeedCatch = bAccurate
mSeekPos = seekPos
```

如果播放器还没 prepare，seek 会被记录下来，但不会马上执行。这样可以支持“prepare 前 seek”。

如果状态不允许 seek，或者 duration 非法，就重置 seek 状态。

真正开始 seek 时，会设置：

```text
mSeekFlag = true
mPlayedVideoPts = INT64_MIN
mPlayedAudioPts = INT64_MIN
mSoughtVideoPos = INT64_MIN
mCurVideoPts = INT64_MIN
```

这些状态说明 seek 要隔离旧的播放进度、旧音视频 PTS 和旧定位结果。

## 7. SeekInCache：能不重新 demux 就不重新 demux

`SeekInCache()` 判断目标位置是否已经在当前缓存范围内。

简化逻辑：

```text
如果目标位置大于当前音视频缓存的最后位置
  -> 不能缓存内 seek

如果是向后 seek
  -> rewind packet queue
  -> 检查目标位置是否早于缓存第一帧
  -> 太早则不能缓存内 seek

查找目标位置之前的关键帧
  -> 找不到则不能缓存内 seek
  -> 找到则清理关键帧之前的数据
```

缓存内 seek 的价值：

- 不需要重新请求 demuxer。
- 不需要重新读取网络或文件。
- 对短距离回退和已经缓冲的数据更快。

但它必须依赖关键帧位置，否则视频无法从任意 P 帧/B 帧直接开始解码。

## 8. 缓存外 seek：清队列并调用 demuxer

如果 `SeekInCache()` 返回 false，`ProcessSeekToMsg()` 会：

```text
ClearPacket(BUFFER_TYPE_ALL)
demuxerService->Seek(seekPos, 0, -1)
NotifyBufferPosition(seekPos)
mEof = false
```

这里要注意两点：

- 清 packet 是为了隔离旧数据。
- demuxer seek 后，后续读到的新 packet 才属于新时间线。

这和 ffplay / ijk 里的 flush 思想一致：seek 后旧队列中的数据不能继续按原时间线播放。

## 9. flush：不仅清 packet，还要清播放路径

无论 seek 是否在缓存内命中，后面都会清理播放路径：

```text
FlushVideoPath()
FlushAudioPath()
FlushSubtitleInfo()
filter clearBuffer()
subPlayer seek()
```

这说明 seek 不是只清 `BufferController`。

数据可能停在很多地方：

```text
packet queue
decoder input
decoder output
frame queue
render path
subtitle path
filter path
```

AVPlayer 第一版不一定有所有路径，但要保留这个原则：

```text
seek 后所有旧时间线的数据都必须被隔离。
```

## 10. Seek end：不能太早通知

CicadaPlayer 并不是在 demuxer seek 返回后马上 `NotifySeekEnd`。

在 `doRender()` 里，只有发生了新帧渲染相关进展后，才会处理 seek 完成：

```text
if (rendered && mSeekFlag) {
    mSeekFlag = false;

    if (!findMsgByType(MSG_SEEKTO)) {
        NotifyPosition(getCurrentPosition());
        ResetSeekStatus();
        NotifySeekEnd(mSeekInCache);
        mSeekInCache = false;
    }
}
```

这里有两个关键点：

- seek 完成更接近“新位置已经进入播放/渲染链路”，而不是 demuxer seek 调用返回。
- 如果消息队列里还有新的 `MSG_SEEKTO`，不要急着发最终 seek end。

这能避免连续 seek 时 UI 收到一堆过期完成通知。

## 11. 准确 seek：mSeekNeedCatch

`bAccurate` 会写入 `mSeekNeedCatch`。

在后续读 packet 和处理缓存内 seek 时，CicadaPlayer 会根据目标位置和实际 packet
timePosition 判断是否需要丢弃早于目标位置太多的数据。

简化理解：

```text
普通 seek
  -> 找到合适关键帧即可开始

准确 seek
  -> 还要继续追到更接近目标时间的位置
  -> 过早的数据可能被 drop
```

第一版 AVPlayer 可以先不做真正准确 seek，但要在接口和状态里保留：

```text
SeekMode::KeyFrame
SeekMode::Accurate
```

这样后续实现不会破坏 API 语义。

## 12. 适合迁移到 AVPlayer 的最小版本

建议先实现一个小闭环：

```text
PlayerCore::seek(position, mode)
  -> command queue 只保留最新 pending seek
  -> command thread 执行 seek
  -> 分配 seekSerial
  -> flush packet/frame 状态
  -> fake worker 或 demuxer 返回 SeekDone(serial)
  -> serial 匹配才更新状态和通知
```

第一版不用立刻做缓存内 seek，可以分两步：

### v1：控制面 seek

- latest-wins。
- seekSerial。
- pause 状态下 seek 完成后保持 pause。
- release/stop 后旧 seek done 被丢弃。
- timeline 记录 seek requested / started / flushed / done / dropped。

### v2：缓存内 seek

- PacketQueue 支持按 timePosition rewind / clear before。
- 查找目标位置前的关键帧。
- 命中缓存时不调用 demuxer seek。
- 未命中时走 demuxer seek。

## 13. 哪些地方不要照抄

不建议直接照搬：

- `mSeekFlag`、`mSeekPos`、`mSeekNeedCatch` 分散在大类里，读起来需要跨很多函数。
- seek 完成和 render loop、buffering、subtitle、filter 耦合在一个大流程里。
- `MSG_SEEKTO` payload 仍是 C-style union，类型安全不强。
- 没有显式 `seekSerial`，连续 seek 主要依赖 padding 和队列查找。
- 准确 seek 的语义分散在读 packet、drop 和缓存判断里，不适合直接作为新项目模板。

## 14. 可公开的技术表达

可以这样概括这一章：

```text
CicadaPlayer 的 seek 不是直接调用 demuxer seek，而是先进入控制消息队列。
消息队列会压缩连续 seek，正在 seek 时新 seek 会 padding，真正执行时先判断是否能
在缓存内定位，不能命中时才清 packet 并调用 demuxer seek。seek 后还要 flush
音视频和字幕路径，完成通知也不是 demuxer seek 返回后立刻发，而是在新位置进入
播放/渲染链路后再通知。迁移到 AVPlayer 时，可以先实现 latest-wins、seekSerial、
flush 和旧结果过滤，再扩展缓存内 seek。
```

