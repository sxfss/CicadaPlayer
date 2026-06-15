# 源码地图

这一章只解决一个问题：CicadaPlayer 这么大，先看哪里，后看哪里。

## 顶层结构

| 目录 | 学习价值 | 第一轮建议 |
| --- | --- | --- |
| `mediaPlayer/` | 播放器 SDK 外层和核心控制入口 | 必看 |
| `framework/` | 数据源、解封装、解码、渲染、工具、缓存等基础模块 | 必看骨架 |
| `cmdline/` | 命令行使用入口和测试工具 | 需要跑 demo 时再看 |
| `platform/` | Android、Apple、Flutter 平台封装 | 第一轮跳过 |
| `plugin/` | 扩展数据源等插件 | 第二轮看数据源时补 |
| `external/` | 第三方依赖 | 跳过 |
| `build_tools/` | 跨平台构建脚本 | 构建失败时再看 |

## 第一优先级：播放器入口

重点文件：

- `mediaPlayer/MediaPlayer.h`
- `mediaPlayer/MediaPlayer.cpp`
- `mediaPlayer/media_player_api.h`
- `mediaPlayer/media_player_api.cpp`
- `mediaPlayer/ICicadaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.cpp`
- `mediaPlayer/player_msg_control.h`
- `mediaPlayer/player_msg_control.cpp`
- `mediaPlayer/CicadaPlayerPrototype.h`
- `mediaPlayer/CicadaPlayerPrototype.cpp`

读法：

```text
MediaPlayer 公共方法
  -> Cicada* C API
  -> ICicadaPlayer 虚接口
  -> SuperMediaPlayer 实现
  -> PlayerMessageControl 消息队列
  -> SuperMediaPlayer::mainService()
```

这一层最重要，因为它决定了所有底层模块是怎么被调度起来的。

## 第二优先级：framework 抽象接口

这些接口先读声明，不急着读每个实现：

- `framework/data_source/IDataSource.h`
- `framework/demuxer/IDemuxer.h`
- `framework/codec/IDecoder.h`
- `framework/render/audio/IAudioRender.h`
- `framework/render/video/IVideoRender.h`

你要关注的问题：

- 每个接口代表播放器流水线里的哪一段。
- 谁创建它。
- 谁持有它。
- 生命周期由谁结束。
- 错误和状态如何向上返回。

## 第三优先级：创建和选择实现

重点文件：

- `framework/base/prototype.h`
- `framework/data_source/dataSourcePrototype.h`
- `framework/data_source/dataSourcePrototype.cpp`
- `framework/demuxer/demuxerPrototype.h`
- `framework/demuxer/demuxerPrototype.cpp`
- `framework/codec/codecPrototype.h`
- `framework/codec/decoderFactory.h`
- `framework/codec/decoderFactory.cpp`
- `framework/render/renderFactory.h`
- `framework/render/renderFactory.cpp`

这里是 CicadaPlayer 很适合学 C++ 的地方：它不是直接写死一种实现，而是通过 `probeScore()`、`clone()`、factory fallback 去选合适实现。

典型模式：

```text
注册候选实现
  -> 按输入参数 probe
  -> 选择 score 最高的实现
  -> clone/create
  -> 返回接口指针或 unique_ptr
```

## 第四优先级：数据路径

推荐从短链路开始，不要直接读 HLS/DASH 全量协议。

```text
IDataSource
  -> dataSourcePrototype
  -> file/curl/ffmpeg data source
  -> IDemuxer
  -> demuxer_service
  -> avFormatDemuxer 或 playList_demuxer
```

重点文件：

- `framework/data_source/IDataSource.h`
- `framework/data_source/dataSourcePrototype.cpp`
- `framework/data_source/file_data_source.*`
- `framework/data_source/ffmpeg_data_source.*`
- `framework/demuxer/demuxer_service.*`
- `framework/demuxer/avFormatDemuxer.*`
- `framework/demuxer/play_list/playList_demuxer.*`

## 第五优先级：解码和渲染

解码：

- `framework/codec/IDecoder.h`
- `framework/codec/decoderFactory.cpp`
- `framework/codec/avcodecDecoder.*`
- `framework/codec/ActiveDecoder.*`

渲染：

- `framework/render/audio/IAudioRender.h`
- `framework/render/video/IVideoRender.h`
- `framework/render/audio/SdlAFAudioRender*.h`
- `framework/render/video/SdlAFVideoRender.*`
- `framework/render/video/AFActiveVideoRender.*`

读这部分时要带着 ffplay 对照：

- ffplay 里队列和渲染更集中。
- CicadaPlayer 里 decoder/render 是接口化模块，由 `SuperMediaPlayer` 组织调度。

## 第六优先级：业务扩展模块

第一轮只知道它们挂在哪里：

- `mediaPlayer/abr/`：自适应码率。
- `mediaPlayer/analytics/`：播放指标采集。
- `mediaPlayer/subTitle/`：字幕。
- `framework/cacheModule/`：边播边缓存。
- `framework/drm/`：DRM。
- `framework/filter/`：音视频滤镜。

其中你当前可以优先选 `framework/cacheModule/CacheManager.cpp`，因为它和播放主链关系明确：

```text
MediaPlayer::SetDataSource()
  -> CacheManager::init()
  -> 可能返回本地缓存路径
  -> MediaPlayer::mediaFrameCallback()
  -> CacheManager::sendMediaFrame()
```

## 第一轮阅读检查点

读完第一轮后，应该能回答这些问题：

- `MediaPlayer` 和 `SuperMediaPlayer` 为什么不是同一个类。
- `media_player_api.cpp` 为什么还要包一层 C API handle。
- `SetDataSource/Prepare/Start/SeekTo` 哪些是同步入口，哪些进入消息队列。
- `mainService()` 一轮循环做了哪些事。
- `IDataSource`、`IDemuxer`、`IDecoder`、`IAudioRender`、`IVideoRender` 分别对应 ffplay 哪些概念。
- factory/prototype 在这个项目里解决了什么变化点。

