# CicadaPlayer C++ 学习路线

这个目录是给你自己的二次学习材料，不替代仓库原有的 `doc/code_learning.zh.md` 和
`framework/code_learning.zh.md`。原文档适合作为官方入口，这里更关注：已经学过
ffplay 和 ijkplayer 之后，如何把 CicadaPlayer 当成 C++ 工程化项目来读。

## 定位

CicadaPlayer 不建议再按 ffplay 那种“最小播放器闭环”方式从头扫。它更适合补齐这些能力：

- 多模块 C++ 工程如何分层：API 层、播放器核心、framework 基础模块、平台适配。
- 接口和实现如何拆：`ICicadaPlayer`、`IDataSource`、`IDemuxer`、`IDecoder`、`IAudioRender`、`IVideoRender`。
- 工厂和 prototype 如何选择实现：播放器、数据源、解封装、解码器、渲染器。
- 生命周期如何管理：创建、prepare、start、pause、seek、stop、释放。
- 线程和消息如何组织：外部 API 不直接做重活，而是投递到内部消息队列和主循环。
- 业务型能力如何接入播放器主线：缓存、ABR、analytics、字幕、DRM、平台硬解。

## 和前置学习的关系

ffplay 给你的是播放器最小主干：

```text
open input -> read packet -> packet queue -> decode -> frame queue -> render -> clock
```

ijkplayer 给你的是工程化播放器里更成熟的控制链：

```text
API/message -> read thread -> seek/flush/serial -> decoder/render callback
```

CicadaPlayer 要重点看更像 SDK 的组织方式：

```text
MediaPlayer API
  -> C API handle
  -> ICicadaPlayer
  -> SuperMediaPlayer message loop
  -> data_source / demuxer / codec / render / cache
```

## 推荐顺序

第一轮只建立骨架，不要深挖所有功能。

1. 读 [00-source-map.md](00-source-map.md)
   - 目标：知道目录怎么分层，哪些先看，哪些暂时跳过。
2. 读 [01-player-entry-flow.md](01-player-entry-flow.md)
   - 目标：从 `MediaPlayer::SetDataSource/Prepare/Start/SeekTo` 追到 `SuperMediaPlayer` 主循环。
3. 读 [02-cpp-engineering-patterns.md](02-cpp-engineering-patterns.md)
   - 目标：把接口、工厂、所有权、消息队列这些 C++ 工程写法抽出来。
4. 读 [03-architecture-lessons.md](03-architecture-lessons.md)
   - 目标：区分哪些设计适合借鉴，哪些只是历史风格或 C/SDK 边界写法。

第二轮再进入专题：

- `data_source + cache`：从 `CacheManager.cpp` 和 `dataSourcePrototype.cpp` 入手。
- `demuxer + HLS/DASH`：从 `demuxer_service.cpp`、`demuxerPrototype.cpp`、`play_list/`、`dash/` 入手。
- `decoder + render + clock`：从 `decoderFactory.cpp`、`ActiveDecoder.*`、`render/`、`af_clock.*` 入手。

## 暂时不建议深挖

这些模块第一轮只做目录识别，不展开：

- `platform/Android`、`platform/Apple`：平台 SDK 封装多，容易偏。
- `external/`：第三方依赖，不是当前 C++ 主线。
- `framework/demuxer/dash` 全量细节：协议结构复杂，后面作为 DASH 专题处理。
- `drm/`：业务和平台依赖较重，除非你要准备 DRM 方向。
- `analytics/`：第一轮只理解它挂在 `MediaPlayer` 外层，不深入指标体系。

## 每轮阅读的输出

每看完一个模块，建议只产出三件东西：

1. 一张调用链图。
2. 一个“核心类职责表”。
3. 一个 C++ 学习点总结，例如所有权、接口边界、线程同步、错误传播。

不要一上来写大而全的源码索引。CicadaPlayer 文件很多，源码索引很容易看起来完整，但对学习没有帮助。
