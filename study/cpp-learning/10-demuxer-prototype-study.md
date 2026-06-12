# Demuxer 边界与 Prototype：普通容器和播放列表如何统一

这一章看 CicadaPlayer 的 demuxer 组织方式。重点不是深入 HLS/DASH 协议，而是理解：

```text
IDataSource
  -> demuxer_service
  -> 读取 probe buffer
  -> demuxerPrototype 选择 IDemuxer 实现
  -> avFormatDemuxer 或 playList_demuxer
  -> 对上层统一提供 ReadPacket / Seek / stream meta
```

demuxer 是播放器从“字节流”进入“媒体流”的边界。它负责把容器、播放列表、stream meta、
packet 读取和 seek 统一起来。

## 1. 源码入口

主要文件：

- `framework/demuxer/IDemuxer.h`
- `framework/demuxer/demuxer_service.h`
- `framework/demuxer/demuxer_service.cpp`
- `framework/demuxer/demuxerPrototype.h`
- `framework/demuxer/demuxerPrototype.cpp`
- `framework/demuxer/avFormatDemuxer.*`
- `framework/demuxer/play_list/playList_demuxer.*`
- `framework/demuxer/play_list/HLSManager.*`
- `framework/demuxer/dash/DashManager.*`

第一轮只看这些函数：

- `demuxer_service::createDemuxer()`
- `demuxer_service::initOpen()`
- `demuxer_service::readPacket()`
- `demuxer_service::Seek()`
- `demuxerPrototype::create()`
- `avFormatDemuxer::Open()`
- `playList_demuxer::Open()`
- `playList_demuxer::ReadPacket()`

## 2. 播放器问题是什么

播放器不能只支持一种容器。

同一个播放主链可能面对：

- 本地 MP4 / FLV / TS。
- HTTP MP4。
- HLS m3u8。
- DASH mpd。
- 外挂字幕 demuxer。
- sample decrypt 包装后的 demuxer。

如果 `SuperMediaPlayer` 直接判断这些格式，播放主循环会越来越复杂。

demuxer 层要提供统一边界：

```text
Open
ReadPacket
Seek
GetMediaMeta
GetStreamMeta
OpenStream / CloseStream
getBufferDuration
setClientBufferLevel
```

这样上层只关心“拿 packet 和 stream 信息”，不关心底层是 FFmpeg 还是 playlist manager。

## 3. demuxer_service：上层看到的 demuxer 外壳

`demuxer_service` 是 `SuperMediaPlayer` 主要接触的对象。

它内部持有：

```text
std::unique_ptr<IDemuxer> mDemuxerPtr
IDataSource *mPDataSource
probe buffer
read / seek / open / interrupt callbacks
```

它的职责不是亲自解析所有格式，而是：

- 从 `IDataSource` 或 callback 读取 probe 数据。
- 调用 `demuxerPrototype::create()` 选择具体 demuxer。
- 给具体 demuxer 设置 read/seek/open/interrupt 回调。
- 对外转发 `ReadPacket / Seek / GetStreamMeta / OpenStream`。

可以把它理解成 demuxer 的 facade：

```text
SuperMediaPlayer
  -> demuxer_service
  -> IDemuxer
```

## 4. probe buffer：先看一点数据再选实现

`createDemuxer()` 会准备一段 probe buffer，默认最多 1024 字节。

它会从两个来源读取：

```text
IDataSource::Read()
或者外部 read callback
```

如果开头像 MPD，会扩大 probe size，因为 DASH 识别需要更多 XML 内容。

然后调用：

```text
demuxerPrototype::create(url, buffer, size, meta, opts)
```

这个设计说明：demuxer 选择不应该只看扩展名，也要看实际数据头和 meta。

AVPlayer 第一版可以简化：

```text
先根据 URL 后缀和少量 header 判断
后续再引入 probe score
```

## 5. demuxerPrototype：选择具体 IDemuxer

`demuxerPrototype::create()` 会遍历注册的 prototype：

```text
for each demuxer prototype
  -> probeScore(uri, buffer, size, meta, opts)
  -> 记录最高分
  -> score 足够高时提前结束
  -> clone(uri, type, meta)
```

这个机制把“格式判断”和“播放主链”拆开。

值得迁移的是：

```text
播放主链依赖 IDemuxer
格式选择集中在 factory/prototype
每个实现自己声明支持什么输入
```

暂时不要照搬的是静态数组注册和裸指针返回。AVPlayer 第一版可以先用普通工厂函数。

## 6. avFormatDemuxer：普通容器的 FFmpeg 包装

`avFormatDemuxer` 负责普通容器和 FFmpeg 支持的输入。

`Open()` 里可以看到两种模式：

```text
有 read callback
  -> 创建 AVIOContext
  -> FFmpeg 通过 callback 读数据

没有 read callback
  -> 直接用 filename / URL 打开
```

它还会设置 FFmpeg 输入参数，例如 protocol whitelist、safe、usetoc 等。

这里值得学的是：FFmpeg demuxer 不一定直接打开 URL。播放器可以把自己的 `IDataSource`
适配成 FFmpeg 的 read/seek callback，从而统一网络、缓存、加密、统计等能力。

AVPlayer 迁移时，最小设计可以是：

```text
FFmpegDemuxer
  -> 可直接 avformat_open_input(url)
  -> 后续再支持 custom AVIO callback
```

不要第一版就把所有 AVIO callback 和协议处理做全。

## 7. playList_demuxer：HLS/DASH 不是普通容器

`playList_demuxer::Open()` 的流程和 `avFormatDemuxer` 不一样：

```text
创建 proxyDataSource
  -> parser parse(mPath)
  -> 根据 type 创建 HLSManager 或 DashManager
  -> 设置 options / data source config / header merge / url hash callback
  -> playlistManager->init()
```

它对外仍然实现 `IDemuxer`：

```text
ReadPacket()
Seek()
GetStreamMeta()
getBufferDuration()
setClientBufferLevel()
```

但内部是 playlist manager，而不是单个 FFmpeg format context。

学习时要抓住这个边界：

```text
普通容器
  -> avFormatDemuxer

HLS/DASH
  -> playList_demuxer
  -> HLSManager / DashManager

上层
  -> 都通过 IDemuxer 使用
```

## 8. buffer level 反馈到 demuxer

前面 `07-buffering-and-qoe-study.md` 里看到，`SuperMediaPlayer::DoCheckBufferPass()` 会调用：

```text
demuxerHandle->setClientBufferLevel(...)
```

`playList_demuxer` 会把这个状态传给 playlist manager。

这说明 demuxer 不是纯粹“被动吐 packet”。在 HLS/DASH 这类协议里，客户端 buffer level
可能影响后续分片读取、下载节奏或 manager 策略。

AVPlayer 可以先记录这个接口位置：

```text
BufferController
  -> DemuxerClientBufferLevel
  -> Demuxer / Source policy
```

不需要第一版就做真实网络调度。

## 9. 适合迁移到 AVPlayer 的最小版本

第一版只需要一个清晰的 `IDemuxer` 边界：

```text
class IDemuxer
  -> open()
  -> readPacket()
  -> seek()
  -> getStreamInfo()
  -> close()
```

实现顺序建议：

```text
FakeDemuxer
  -> 用于 PlayerCore 和状态测试

FFmpegDemuxer
  -> 普通本地文件 / URL

DemuxerFactory
  -> 后续再决定是否引入 probe
```

等普通文件链路稳定后，再考虑：

- HLS playlist demuxer。
- DASH。
- custom AVIO。
- buffer level 反馈。

## 10. 哪些地方不要照抄

不建议直接照搬：

- `demuxerPrototype` 静态数组和裸指针返回。
- `demuxer_service` 同时负责 probe、callback 适配、demuxer 生命周期、接口转发，职责偏重。
- HLS/DASH 内部复杂度很高，第一轮不要深入所有 segment tracker 和 parser 细节。
- `IDemuxer` 能力很宽，AVPlayer 第一版不要一次性复制全部接口。
- 不要因为 CicadaPlayer 有 prototype，就在只有一个 FFmpeg 实现时过早做复杂插件系统。

## 11. 可公开的技术表达

可以这样概括这一章：

```text
CicadaPlayer 用 demuxer_service 隔离上层播放器和具体 demuxer。demuxer_service 先从
IDataSource 读取少量 probe 数据，再通过 demuxerPrototype 选择具体 IDemuxer。
普通容器走 avFormatDemuxer，HLS/DASH 走 playList_demuxer，但对上层都表现为统一的
ReadPacket / Seek / StreamMeta 边界。迁移到 AVPlayer 时，应先建立最小 IDemuxer
接口和 FFmpegDemuxer，再在真实变化出现后引入 probe/factory。
```

