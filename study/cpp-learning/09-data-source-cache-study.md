# Data Source 与 Cache：从 URL 到真实播放源

这一章看 CicadaPlayer 的数据源和缓存旁路。重点不是某个 HTTP 实现，而是理解：

```text
外部 URL
  -> MediaPlayer::SetDataSource()
  -> 可选 CacheManager 改写 playUrl
  -> CicadaSetDataSourceWithUrl()
  -> SuperMediaPlayer / demuxer_service
  -> dataSourcePrototype 选择 IDataSource 实现
  -> IDataSource 提供 Open / Read / Seek / Close
```

数据源是播放器数据面的入口。缓存则是旁路能力：它不应该改变播放主链的职责，但可以影响
最终播放 URL，也可以观察媒体帧并写入缓存。

## 1. 源码入口

主要文件：

- `mediaPlayer/MediaPlayer.cpp`
- `framework/cacheModule/CacheManager.cpp`
- `framework/cacheModule/CacheManager.h`
- `framework/data_source/IDataSource.h`
- `framework/data_source/dataSourcePrototype.h`
- `framework/data_source/dataSourcePrototype.cpp`
- `framework/data_source/ffmpeg_data_source.*`
- `framework/data_source/curl/*`

第一轮只看这些函数：

- `MediaPlayer::SetDataSource()`
- `MediaPlayer::mediaFrameCallback()`
- `CacheManager::init()`
- `CacheManager::sendMediaFrame()`
- `dataSourcePrototype::create()`
- `IDataSource::Open() / Read() / Seek() / Close()`

## 2. 播放器问题是什么

数据源层要解决的问题是：播放器上层不应该关心 URL 背后到底是什么。

同样是一个 `url`，它可能代表：

- 本地文件。
- HTTP/HTTPS 资源。
- 已经缓存好的本地文件。
- 需要边播边缓存的远端资源。
- HLS/DASH 里的分片 URL。
- 插件或平台特殊 source。

如果 `SuperMediaPlayer` 到处判断这些类型，播放主循环会被数据源细节污染。

所以数据源层要提供统一能力：

```text
Open
Read
Seek
Close
Interrupt
GetOption / Set_config
```

这和 ffplay 的 `AVIOContext` 思想类似：上层按“读字节流”使用，下层决定字节来自哪里。

## 3. SetDataSource：先处理缓存，再下发最终 playUrl

`MediaPlayer::SetDataSource()` 是外层入口。

当 cache module 开启时，它不会直接把原始 URL 下发给核心播放器，而是先构造 `CacheManager`：

```text
new CacheManager
  -> setCacheConfig()
  -> setSourceUrl(url)
  -> setDescription()
  -> set callbacks
  -> setDataSource(PlayerCacheDataSource)
  -> playUrl = CacheManager::init()
```

然后再统一下发：

```text
CicadaSetDataSourceWithUrl(handle, playUrl.c_str())
```

这里的关键点是 `playUrl` 可能不是原始 URL：

```text
如果已有缓存文件
  -> playUrl = 本地缓存路径

如果没有缓存或不能缓存
  -> playUrl = 原始 URL
```

这说明 cache 能影响“最终播放源”，但它仍然在 `MediaPlayer` facade 层处理，不直接塞进
decode/render 热路径。

## 4. CacheManager::init：缓存命中和缓存资格判断

`CacheManager::init()` 的逻辑可以简化成：

```text
如果 cache 没启用
  -> 返回原始 URL

设置 cache config / source URL / description
查询是否已有缓存文件
  -> 有则返回缓存文件路径

检查当前 URL 是否可以缓存
  -> 可以则打开 mNeedProcessFrame
  -> 返回原始 URL
```

它同时处理两件事：

- **cache hit**：已经缓存过，直接播放本地文件。
- **cache collect**：还没缓存，但可以边播边写缓存。

这两个概念要分开：

```text
cache hit
  -> 改写播放 URL

cache collect
  -> 播放仍走原 URL
  -> 旁路接收媒体帧写缓存
```

## 5. sendMediaFrame：缓存旁路如何接入播放链

`MediaPlayer::mediaFrameCallback()` 会把媒体帧传给 `CacheManager::sendMediaFrame()`。

第一次收到 frame 时，`CacheManager` 会补齐缓存模块需要的元信息：

```text
getStreamSize()
getDuration()
getStreamMeta(video)
getStreamMeta(audio)
setMediaInfo()
setStreamMeta()
set callbacks
start cache module
```

之后才持续写入：

```text
mCacheModule.addFrame(frame, type)
```

这说明缓存不是主播放链的一部分，而是一个观察者式旁路：

```text
播放主链
  -> 正常读 packet / decode / render

缓存旁路
  -> 观察 packet/frame
  -> 写 cache module
  -> 用 callback 报告成功或失败
```

这个模式很适合 AVPlayer：先做 frame trace 或 metrics 旁路，再决定是否真的写文件缓存。

## 6. dataSourcePrototype：根据输入选择具体 IDataSource

`dataSourcePrototype::create()` 是数据源选择入口。

它会遍历已注册的 prototype：

```text
for each prototype
  -> probeScore(uri, opts, flags)
  -> 选择 score 最高的实现
  -> clone(uri)
```

如果没有 prototype 命中，会按编译配置和全局设置 fallback：

```text
CurlDataSource2 / CurlDataSource
  -> ffmpegDataSource
```

这个设计解决的问题是：上层不需要关心当前 URL 应该用 curl、ffmpeg 还是插件 source。

值得迁移的是：

```text
选择策略集中在 factory/prototype
播放主链只依赖 IDataSource
```

第一版 AVPlayer 可以更简单：

```text
createDataSource(url)
  -> file source
  -> http source
  -> fallback source
```

等真实 source 类型变多后，再引入 probe score。

## 7. IDataSource：播放器看到的数据源边界

`IDataSource` 把不同 source 统一成一组能力：

```text
Open(flags)
Read(buf, size)
Seek(offset, whence)
Close()
Interrupt()
Set_config()
GetOption()
getBufferDuration()
enableCache()
```

这些接口说明数据源不只是“读文件”：

- 网络 source 需要超时、重试、代理、header、interrupt。
- HLS/DASH 分片 source 可能需要 segment list。
- cache source 可能需要 getBufferDuration。
- 播放器 stop/release 时必须能 interrupt 阻塞读。

AVPlayer 最小版本可以先保留四个核心能力：

```text
open
read
seek
interrupt
```

不要一开始把 CicadaPlayer 的全部配置项搬过去。

## 8. 适合迁移到 AVPlayer 的最小版本

第一版建议拆成三个小对象：

```text
MediaSource
  -> 保存原始 URL、最终 URL、source type

IDataSource
  -> open/read/seek/interrupt

DataSourceFactory
  -> 根据 URL 创建具体 source
```

如果要模拟 cache，可以先做轻量旁路：

```text
FrameTraceSink
  -> 接收 packet/frame meta
  -> 记录 stream type、pts、duration、size
```

不要第一版就写完整磁盘缓存。先验证这条链：

```text
SetDataSource(url)
  -> resolve final source
  -> create data source
  -> demuxer reads through IDataSource
  -> optional observer receives packet meta
```

## 9. 哪些地方不要照抄

不建议直接照搬：

- `MediaPlayer::SetDataSource()` 里同时处理 cache、description、callback、URL 改写，facade 压力偏大。
- `CacheManager` 持有裸指针 `ICacheDataSource *` 并在析构里手动 delete。
- `dataSourcePrototype` 使用静态数组和 `_nextSlot`，容量、初始化顺序和线程安全都要靠约定。
- `create()` 返回裸 `IDataSource *`，所有权需要靠调用方记住。
- `IDataSource` 接口很宽，第一版 AVPlayer 不应该一次复制所有能力。

## 10. 可公开的技术表达

可以这样概括这一章：

```text
CicadaPlayer 把数据源选择和播放主链拆开：MediaPlayer 先处理缓存配置，CacheManager
可能把原始 URL 改写成本地缓存路径，也可能作为旁路观察媒体帧写缓存。真正的数据读取
通过 IDataSource 边界完成，dataSourcePrototype 根据 URL、配置和 flags 选择具体
实现。迁移到 AVPlayer 时，可以先做 DataSourceFactory + IDataSource 的最小边界，
缓存能力先做成 frame trace 或 packet meta 旁路，不急着实现完整磁盘缓存。
```

