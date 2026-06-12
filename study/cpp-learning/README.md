# CicadaPlayer 播放器与 C++ 学习路线

这个目录是给你自己的二次学习材料，不替代仓库原有的 `doc/code_learning.zh.md` 和
`framework/code_learning.zh.md`。原文档适合作为官方入口，这里更关注：已经学过
ffplay 和 ijkplayer 之后，如何把 CicadaPlayer 当成播放器 SDK 工程来读。

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
3. 读 [02-cpp-concepts-through-cicada.md](02-cpp-concepts-through-cicada.md)
   - 目标：用 CicadaPlayer 训练资源管理、对象职责、所有权、多态、工厂和错误处理。
4. 读 [03-engineering-patterns-and-lessons.md](03-engineering-patterns-and-lessons.md)
   - 目标：把接口、工厂、消息队列、业务接入这些工程模式抽出来，并判断哪些值得借鉴。
5. 读 [04-interview-and-design-transfer.md](04-interview-and-design-transfer.md)
   - 目标：把源码设计转成可公开的技术表达和 AVPlayerLab 迁移草图。

第二轮再进入专题：

- `control message + seek`：先读 [05-player-message-control-study.md](05-player-message-control-study.md)，从 `PlayerMessageControl` 入手。
- `AVPlayer 迁移候选`：读 [06-avplayer-transfer-candidates.md](06-avplayer-transfer-candidates.md)，筛选适合迁移的小型播放器设计亮点。
- `buffering + QoE`：读 [07-buffering-and-qoe-study.md](07-buffering-and-qoe-study.md)，理解缓存时长、水位、卡顿状态和旁路统计。
- `seek control`：读 [08-seek-control-study.md](08-seek-control-study.md)，理解连续 seek、缓存内 seek、flush 和完成通知。
- `data_source + cache`：读 [09-data-source-cache-study.md](09-data-source-cache-study.md)，理解 URL、数据源选择和缓存旁路。
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
3. 一个播放器机制总结；只有涉及所有权、接口边界、线程同步、错误传播时，再补 C++ 工程注意点。

不要一上来写大而全的源码索引。CicadaPlayer 文件很多，源码索引很容易看起来完整，但对学习没有帮助。

## 后续专题固定模板

以后每看一个源码专题，都固定回答这些问题，避免陷进细节：

```text
这个设计解决什么播放器问题？
控制面 / 数据面 / 线程模型是什么？
哪些地方值得迁移？
哪些地方不要照抄？
如果涉及 C++ 工程问题，应该注意什么？
可公开的技术表达是什么？
是否需要另存为本地私有表达？
```

尤其要及时标出不适合照抄的历史写法：裸 owning pointer、手动 `new/delete`、
头文件 `using namespace`、宽泛 `catch (...)`、不清晰的 `void *` 所有权和裸
`int` 错误流。
