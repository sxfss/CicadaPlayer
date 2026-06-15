# Demuxer 执行链：从字节流到 IAFPacket

这一章继续普通 URL/MP4 主线。上一章 data source 只提供 bytes；这一章 demuxer 把 bytes
解析成 packet。

主链是：

```text
demuxer_service::initOpen()
  -> demuxer_service::createDemuxer()
  -> read probe bytes from IDataSource
  -> demuxerPrototype::create()
  -> avFormatDemuxer::Open()
  -> avformat_open_input()
  -> av_read_frame()
  -> AVPacket
  -> IAFPacket
  -> SuperMediaPlayer::ReadPacket()
```

## 这一层由谁持有？

`SuperMediaPlayer` 持有：

```text
std::unique_ptr<demuxer_service> mDemuxerService
```

`demuxer_service` 持有：

```text
std::unique_ptr<IDemuxer> mDemuxerPtr
IDataSource *mPDataSource
```

这里的关系是：

```text
SuperMediaPlayer owns demuxer_service
demuxer_service owns concrete IDemuxer
demuxer_service refers to IDataSource
```

`IDemuxer` 是接口。普通 MP4 通常落到 `avFormatDemuxer`，HLS/DASH 会走播放列表 demuxer。第一轮
只追普通 MP4。

## 输入是什么？输出是什么？

输入：

```text
IDataSource::Read() 提供的 bytes
probe buffer 里的前几百到 1024 字节
URL / options / demuxer meta
```

输出：

```text
std::unique_ptr<IAFPacket>
```

`IAFPacket` 里带着后面播放链需要的信息：

- stream index：属于 audio/video/subtitle 哪一路。
- pts/dts/timePosition：后续同步、seek、buffering 都依赖它。
- duration：用于缓存时长估算。
- flags：关键帧判断会用到。
- data/size：decoder 真正要吃的数据。

## 主循环什么时候调用它？

还是从主循环看：

```text
mainService()
  -> ProcessVideoLoop()
  -> doReadPacket()
  -> ReadPacket()
  -> mDemuxerService->readPacket(pMedia_Frame, index)
```

`demuxer_service::readPacket()` 只是转发：

```text
mDemuxerPtr->ReadPacket(packet, index)
```

普通 MP4 下，具体实现就是 `avFormatDemuxer::ReadPacket()`。如果没有启用 demuxer 内部线程，
它直接走 `ReadPacketInternal()`；如果内部线程在跑，则从它自己的 packet queue 里取。

## demuxer_service 怎么选择具体 demuxer？

源码入口：

- `framework/demuxer/demuxer_service.cpp`
- `framework/demuxer/demuxerPrototype.cpp`
- `framework/demuxer/avFormatDemuxer.cpp`
- `framework/demuxer/play_list/playList_demuxer.cpp`

选择过程是：

```text
createDemuxer()
  -> 读取 probe buffer
  -> demuxerPrototype::create(url, probeBuffer, size, meta, opts)
  -> 每个 prototype 判断 is_supported / probeScore
  -> 返回具体 IDemuxer
```

普通 MP4 会由 `avFormatDemuxer::is_supported()` 通过 FFmpeg probe 判断支持。HLS/DASH 会被播放列表
相关 probe 抢走，所以不要把 HLS 细节混进普通 MP4 主线。

## avFormatDemuxer 怎么打开输入？

`avFormatDemuxer::Open()` 会进入 `open(nullptr)`。

关键步骤：

```text
if (mReadCb != nullptr)
  -> avio_alloc_context(read_buffer, ..., avio_callback_read, avio_callback_seek)
  -> mCtx->pb = mPInPutPb

avformat_open_input(&mCtx, filename, in_fmt, opts)
```

这说明 FFmpeg 并不一定自己直接打开 URL。CicadaPlayer 可以给 FFmpeg 一个自定义 AVIO，
FFmpeg 要数据时会调用：

```text
avFormatDemuxer::avio_callback_read()
  -> mReadCb(mUserArg, buffer, size)
  -> demuxer_service::read_callback()
  -> IDataSource::Read()
```

这就是 data source 和 FFmpeg demux 的连接点。

## av_read_frame 如何变成 IAFPacket？

`avFormatDemuxer::ReadPacketInternal()` 的核心是：

```text
AVPacket *pkt = av_packet_alloc()
av_read_frame(mCtx, pkt)
检查 stream 是否 opened
把 AVPacket 包装成 CicadaPlayer 的 IAFPacket
返回给调用者
```

你读这段时要盯住三个信息：

- `pkt->stream_index`：后面 `SuperMediaPlayer::ReadPacket()` 用它分流到 audio/video/subtitle。
- `pkt->pts / pkt->dts`：后面同步和渲染调度使用。
- `pkt->duration`：后面 packet queue 统计 buffer duration 使用。

如果 `av_read_frame()` 返回 EOF，CicadaPlayer 这里约定 `0` 表示 EOS；如果是 `EAGAIN`，说明暂时没读到，
主循环后面会继续轮询。

## 下一层是谁？

demuxer 的下一层不是 decoder，而是 `SuperMediaPlayer::ReadPacket()`。

`ReadPacket()` 会根据 packet 的 stream index 分流：

```text
video packet
  -> mBufferController->AddPacket(packet, BUFFER_TYPE_VIDEO)

audio packet
  -> mBufferController->AddPacket(packet, BUFFER_TYPE_AUDIO)

subtitle packet
  -> mBufferController->AddPacket(packet, BUFFER_TYPE_SUBTITLE)
```

所以 demuxer 只完成“解复用”，不负责缓存策略、不负责 decoder 调度、不负责 A/V sync。

## 读源码时按哪些函数跳？

建议按这个顺序：

1. `SuperMediaPlayer::ReadPacket()`
2. `demuxer_service::readPacket()`
3. `avFormatDemuxer::ReadPacket()`
4. `avFormatDemuxer::ReadPacketInternal()`
5. `av_read_frame()`
6. `avFormatDemuxer::avio_callback_read()`
7. `demuxer_service::read_callback()`
8. `IDataSource::Read()`

这个顺序故意从 player loop 往下追，再从 FFmpeg callback 反向追回 data source。这样能看清：

```text
player 要 packet
demuxer 要 bytes
data source 提供 bytes
demuxer 返回 packet
player 把 packet 入队
```

## 和 HLS/DASH 的关系

HLS/DASH 第一轮只记一句：

```text
它们也要实现 IDemuxer::ReadPacket()，所以对 SuperMediaPlayer 来说仍然是 packet 来源。
```

不同的是，播放列表 demuxer 内部还要管理 playlist、segment、子 demuxer、码率切换。那些是协议层复杂度，
不是普通 MP4 主链必需知识。

## 这一章你需要真正掌握什么

- `demuxer_service` 是 `SuperMediaPlayer` 和具体 demuxer 之间的外壳。
- `demuxerPrototype` 决定普通容器还是播放列表 demuxer。
- 普通 MP4 核心实现是 `avFormatDemuxer + av_read_frame`。
- `IDemuxer` 输出 packet，不直接输出 frame。
- `ReadPacket()` 拿到 packet 后，下一步是进 `BufferController`，不是直接 render。

