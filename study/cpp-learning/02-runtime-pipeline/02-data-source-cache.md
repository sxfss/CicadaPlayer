# Data Source 读取链：普通 URL/MP4 的数据从哪里来

这一章只追普通 URL/MP4，不进入 HLS/DASH。目标是看清一件事：`SuperMediaPlayer` 后面
的 demuxer 不是凭空读数据，它最终会回调到 `IDataSource::Read/Seek`。

先记住这条链：

```text
MediaPlayer::SetDataSource(url)
  -> CicadaSetDataSourceWithUrl(handle, url)
  -> ICicadaPlayer::SetDataSource(url)
  -> SuperMediaPlayer stores url in player_type_set
  -> prepare/open path creates IDataSource
  -> dataSourcePrototype::create(url)
  -> IDataSource::Read/Seek
  -> demuxer_service read_callback / seek_callback
  -> avFormatDemuxer AVIO callback
```

## 这一层由谁持有？

外层 `MediaPlayer` 只持有 C handle，不直接持有数据源。真正播放引擎是
`SuperMediaPlayer`。

`SuperMediaPlayer` 里有两个和数据源相关的成员：

```text
IDataSource *mDataSource
std::unique_ptr<demuxer_service> mDemuxerService
```

这里要注意：`mDataSource` 是裸指针，说明这块是历史写法，不适合作为现代 C++ 风格照抄。学习时
重点看职责边界：数据源负责字节读取，demuxer_service 负责把字节交给 demuxer。

## 输入是什么？输出是什么？

输入是用户传进来的 URL：

```text
http://...
https://...
file://...
/local/path/file.mp4
```

输出不是 packet，而是字节流读取能力：

```text
Read(buffer, size) -> bytes
Seek(offset, whence) -> new position / file size
```

packet 是 demuxer 的输出，不是 data source 的输出。这个边界要分清：

```text
data source 输出 bytes
demuxer 输出 packet
decoder 输出 frame
render 输出到设备
```

## 主循环什么时候调用它？

`mainService()` 本身不会直接调用 `IDataSource::Read()`。主循环调用的是：

```text
mainService()
  -> ProcessVideoLoop()
  -> doReadPacket()
  -> ReadPacket()
  -> mDemuxerService->readPacket()
```

`readPacket()` 进入 demuxer 后，FFmpeg 需要更多字节时，才通过 AVIO callback 调回
`demuxer_service::read_callback()`，最终调用 `mPDataSource->Read()`。

所以真实运行时是倒着触发的：

```text
player loop asks demuxer for packet
  -> demuxer asks AVIO for bytes
  -> AVIO callback asks IDataSource for bytes
```

这也是很多人读播放器源码时容易断掉的地方：你在 `SuperMediaPlayer::ReadPacket()` 里看不到
`curl` 或 `file read`，因为读字节藏在 demuxer 的 callback 里。

## dataSourcePrototype 如何选择实现？

源码入口：

- `framework/data_source/dataSourcePrototype.cpp`
- `framework/data_source/IDataSource.h`
- `framework/data_source/file_data_source.*`
- `framework/data_source/curl/curl_data_source*.{h,cpp}`
- `framework/data_source/ffmpeg_data_source.*`

`dataSourcePrototype::create()` 的逻辑可以简化为：

```text
遍历已注册 dataSourcePrototype
  -> probeScore(uri)
  -> 选择分数最高的实现
  -> 如果没有命中，尝试 CurlDataSource / CurlDataSource2
  -> 再不行，fallback 到 ffmpegDataSource
  -> source->setOptions(opts)
```

你读这段时不要先纠结所有实现。普通 URL/MP4 第一轮只要看懂：

```text
IDataSource 是统一接口；
不同协议只是 Read/Seek 的不同实现；
上层 demuxer 不应该关心到底是 curl、file 还是 ffmpeg source。
```

## demuxer_service 怎么接上数据源？

源码入口：

- `framework/demuxer/demuxer_service.cpp`
- `demuxer_service::createDemuxer()`
- `demuxer_service::initOpen()`
- `demuxer_service::read_callback()`
- `demuxer_service::seek_callback()`

`demuxer_service` 创建时会拿到 `IDataSource *`。打开 demuxer 时，它会把读写回调交给具体
`IDemuxer`：

```text
mDemuxerPtr->SetDataCallBack(
    read_callback,
    seek_callback,
    open_callback,
    interrupt_callback,
    ...
    this)
```

然后 `read_callback` 通过 `this` 找回 `demuxer_service`，再调用：

```text
pHandle->mPDataSource->Read(buffer, size)
pHandle->mPDataSource->Seek(offset, whence)
```

这就是 data source 和 demuxer 的连接点。

## cache 在这条链上的位置

普通主链先不用把 cache 当主角。cache 是旁路能力，它会影响最终播放 URL、缓存命中和媒体帧收集，
但不是理解“拉流到 demux”的必要前提。

可以先这样放置 cache：

```text
MediaPlayer::SetDataSource(url)
  -> CacheManager 可能判断命中或改写 playUrl
  -> SuperMediaPlayer 使用最终 playUrl 创建 IDataSource
```

后面读 `CacheManager` 时再看两类能力：

- 播放前：判断本地缓存是否可用，决定最终播放 URL。
- 播放中：通过 media frame callback 收集可缓存数据。

第一轮不要让 cache 干扰主链。先把 `URL -> IDataSource -> demuxer callback` 读顺。

## 下一层是谁？

data source 的下一层是 `demuxer_service`。它拿到 bytes 后，会创建具体 `IDemuxer`，普通 MP4
一般会走 `avFormatDemuxer`。

下一章 [03-demuxer-prototype.md](03-demuxer-prototype.md) 继续追：

```text
demuxer_service
  -> probe buffer
  -> demuxerPrototype
  -> avFormatDemuxer::Open()
  -> av_read_frame()
  -> IAFPacket
```

## 读源码时按哪些函数跳？

建议按这个顺序跳，不要从目录开始扫：

1. `MediaPlayer::SetDataSource()`
2. `CicadaSetDataSourceWithUrl()`
3. `SuperMediaPlayer::SetDataSource()`
4. `dataSourcePrototype::create()`
5. `demuxer_service::demuxer_service(IDataSource*)`
6. `demuxer_service::initOpen()`
7. `demuxer_service::read_callback()`
8. 具体 `IDataSource::Read()`，例如 `CurlDataSource::Read()` 或 `ffmpegDataSource::Read()`

如果你在第 4 步之后迷路，先问自己一句：

```text
我现在看的是“读字节”，还是“解 packet”？
```

只要答案还是读字节，就不要跳到 decoder/render。

## 这一章你需要真正掌握什么

- `MediaPlayer` 不直接拉流，真正数据源在 `SuperMediaPlayer` 播放引擎里被创建和使用。
- `IDataSource` 的输出是 bytes，不是 packet。
- `demuxer_service` 通过 read/seek callback 把 demuxer 和 data source 连起来。
- 普通 URL/MP4 第一轮只需要看 `dataSourcePrototype + IDataSource + demuxer_service callback`。
- cache 是旁路，不要把它当成理解主链的第一入口。
