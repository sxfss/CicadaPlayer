# C++ 工程模式学习点

这一章不按播放器流程读，而是把 CicadaPlayer 里值得学习的 C++ 工程写法抽出来。

## 1. 门面层和核心实现分离

代表文件：

- `mediaPlayer/MediaPlayer.h`
- `mediaPlayer/MediaPlayer.cpp`
- `mediaPlayer/ICicadaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.cpp`

结构：

```text
MediaPlayer
  -> 面向 SDK 用户
  -> 管配置、回调、ABR、analytics、cache
  -> 不直接做播放主循环

ICicadaPlayer
  -> 核心播放器抽象接口

SuperMediaPlayer
  -> 核心播放器实现
  -> 管消息、线程、demux、decode、render、buffer
```

学习点：

- 不要把所有能力塞进一个类。
- 外层 API 可以稳定，内部实现可以替换。
- 业务扩展能力适合挂在门面层或专门 manager，不一定进入核心循环。

## 2. C handle 包 C++ 对象

代表文件：

- `mediaPlayer/media_player_api.h`
- `mediaPlayer/media_player_api.cpp`

结构：

```cpp
typedef struct playerHandle_t {
    ICicadaPlayer *pPlayer;
} playerHandle_t;
```

学习点：

- C API 可以作为 ABI 边界。
- C++ 对象可以藏在 opaque handle 后面。
- 创建和释放必须成对：`CicadaCreatePlayer()` / `CicadaReleasePlayer()`。
- handle 层不应该承载复杂业务逻辑，它主要做转发和边界适配。

这个模式在 SDK、插件系统、跨语言绑定里很常见。

## 3. 接口隔离

代表接口：

- `framework/data_source/IDataSource.h`
- `framework/demuxer/IDemuxer.h`
- `framework/codec/IDecoder.h`
- `framework/render/audio/IAudioRender.h`
- `framework/render/video/IVideoRender.h`

这些接口对应播放器流水线：

```text
IDataSource  -> 读字节
IDemuxer     -> 拆 packet / stream meta
IDecoder     -> packet 到 frame
IAudioRender -> 音频输出
IVideoRender -> 视频输出
```

学习点：

- 接口按职责切，不按文件类型切。
- 接口应描述“调用方需要什么”，不是暴露实现内部细节。
- 多平台实现、FFmpeg 实现、SDL 实现都可以挂到同一接口后面。

阅读方法：

先读接口声明，再找一个最常用实现。不要一次读完所有实现。

## 4. prototype + probeScore

代表文件：

- `framework/base/prototype.h`
- `mediaPlayer/CicadaPlayerPrototype.cpp`
- `framework/data_source/dataSourcePrototype.cpp`
- `framework/demuxer/demuxerPrototype.cpp`

共同模式：

```text
候选实现注册到数组
  -> create() 遍历候选实现
  -> 每个实现 probeScore()
  -> 选择分数最高的
  -> clone()
```

`PROBE_SCORE` 定义了大致分数语义：

- `SUPPORT_NOT`
- `SUPPORT_DEFAULT`
- `SUPPORT_MAX`

学习点：

- 这比一堆 `if/else` 更适合扩展。
- `probeScore()` 把“这个实现能不能处理当前输入”的判断交给实现自己。
- `clone()` 让注册对象像原型一样生产真实对象。

需要警惕的点：

- 当前实现里有静态数组和 `_nextSlot`，要关注容量、初始化顺序和线程安全。
- 注册机制读起来方便，但如果边界不清，后期也容易隐藏依赖关系。

## 5. factory fallback

代表文件：

- `framework/codec/decoderFactory.cpp`

典型链路：

```text
decoderFactory::create()
  -> codecPrototype::create()
  -> 如果没有外部/注册实现
  -> createBuildIn()
  -> 按 DECFLAG_HW / DECFLAG_SW 选择硬解或软解
```

学习点：

- factory 不只是 `new` 对象，还能集中处理策略。
- fallback 让系统在插件实现不可用时还能选择内置实现。
- `unique_ptr<IDecoder>` 明确表达了返回对象的所有权。

这部分可以和你之前 PlayerCore/Decoder 设计题对照：硬解、软解、fallback 不是只靠类名解决，关键是创建策略和失败路径。

## 6. 消息队列和控制面

代表文件：

- `mediaPlayer/player_msg_control.h`
- `mediaPlayer/player_msg_control.cpp`
- `mediaPlayer/SuperMediaPlayer.cpp`

核心模式：

```text
外部线程调用 API
  -> putMsg()
  -> 内部主循环 processMsg()
  -> 分发到 Process*Msg()
```

学习点：

- 控制面和数据面分离：API 调用不直接做 read/decode/render。
- 重复消息可以压缩：例如只保留最后一个 seek、view update 或配置变化。
- 消息参数要有明确所有权，尤其是 `new string` 这种手动管理对象。

对照 ijkplayer：

- ijk 的 `FFP_REQ_SEEK` 是控制消息。
- CicadaPlayer 的 `MSG_SEEKTO` 也是控制消息。
- 两者都不是在 API 调用栈里直接完成 seek。

## 7. 资源所有权

这个项目同时有现代 C++ 和旧式 C/C++ 混合写法。

值得注意的正向例子：

- `decoderFactory::create()` 返回 `unique_ptr<IDecoder>`。
- `demuxerPrototype::create()` 接收 `unique_ptr<DemuxerMeta>` 并通过 `move` 传递。
- `CacheManager` 析构函数释放 `mDataSource`。

需要主动训练警觉的例子：

- `PlayerMessageControl` 里某些消息参数通过 `new string` 创建，再由 `recycleMsg()` 删除。
- `MediaPlayer` 构造函数里有多个 `new` 出来的 manager，需要看析构是否成对释放。
- FFmpeg 相关对象通常需要专门 release/free 函数，不能只看 C++ 析构。

阅读时固定问三句话：

1. 这个对象是谁创建的。
2. 所有权有没有转移。
3. 失败、stop、seek、析构路径会不会释放。

## 8. 缓存模块作为业务能力接入

代表文件：

- `mediaPlayer/MediaPlayer.cpp`
- `mediaPlayer/PlayerCacheDataSource.*`
- `framework/cacheModule/CacheManager.h`
- `framework/cacheModule/CacheManager.cpp`
- `framework/cacheModule/CacheModule.*`

第一轮只抓接入点：

```text
MediaPlayer::SetDataSource()
  -> 如果启用 cache
  -> 创建 CacheManager
  -> CacheManager::init()
  -> 可能把播放 URL 替换成本地缓存文件

MediaPlayer::mediaFrameCallback()
  -> CacheManager::sendMediaFrame()
  -> CacheModule 写缓存
```

学习点：

- 缓存不是播放器主循环本身，但它影响 data source 和 media frame 回调。
- 业务能力通常需要同时接触“入口参数”和“播放过程中的数据回调”。
- 这类模块很适合训练边界设计：不要让缓存逻辑散落到所有 decode/render 代码里。

## 9. 建议做的 C++ 小练习

这些练习不需要改源码，可以写在学习笔记里：

1. 画 `MediaPlayer`、`playerHandle`、`ICicadaPlayer`、`SuperMediaPlayer` 的所有权关系。
2. 把 `PlayerMessageControl::putMsg()` 的重复消息策略整理成表。
3. 对 `dataSourcePrototype::create()` 写伪代码，标出 fallback 分支。
4. 对 `decoderFactory::create()` 写伪代码，标出硬解、软解、外部 prototype 的优先级。
5. 找一个 `new` 出来的对象，追它在哪个析构或 cleanup 路径释放。
6. 找一个 `unique_ptr` 返回值，说明为什么这里比裸指针更清晰。

## 第一轮判断标准

读完这一章后，你不需要记住所有类名，但应该能说清：

- 为什么 CicadaPlayer 比 ffplay 更适合学 C++ 工程结构。
- 哪些地方是接口隔离，哪些地方是工厂选择。
- 消息队列为什么能降低 API 调用和播放器主循环之间的耦合。
- 哪些旧式资源管理写法需要你重点检查生命周期。

