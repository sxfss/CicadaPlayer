# 工程模式与架构借鉴

这一章合并原来的“C++ 工程模式”和“架构借鉴”两类内容。读它时不要把 CicadaPlayer
当成现代 C++ 风格模板，而是把它当成播放器 SDK 工程样本：

```text
先看它解决了什么工程问题
再判断哪些设计值得迁移
最后用 boring C++17 重新表达
```

## 总判断

CicadaPlayer 值得借鉴：

- 播放器 SDK 的门面层和核心实现分离。
- C API handle 作为 ABI 边界。
- `IDataSource` / `IDemuxer` / `IDecoder` / render 接口隔离。
- factory/prototype 把模块选择策略集中起来。
- 外部 API 通过消息队列进入内部控制面。
- cache、ABR、analytics 这类业务能力以旁路模块接入。
- C/C++/FFmpeg 混合工程里的资源生命周期审查方法。

不建议照抄：

- 对 owning object 使用裸指针和手动 `delete`。
- 头文件 `using namespace std`。
- 大量宏包装复杂逻辑。
- `void *` 参数所有权不清。
- 到处返回裸 `int`，但缺少统一错误语义。
- 宽泛 `catch (...)` 作为兜底，而不是清晰的业务错误模型。

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
  -> 面向 SDK 使用者
  -> 管配置、回调、ABR、analytics、cache
  -> 不直接做播放主循环

ICicadaPlayer
  -> 核心播放器抽象接口

SuperMediaPlayer
  -> 默认核心实现
  -> 管消息、线程、demux、decode、render、buffer
```

这个设计解决的问题：

- 对外 API 和内部实现解耦。
- SDK 用户不用知道底层是 `SuperMediaPlayer`、Apple player 还是其他实现。
- 业务扩展可以挂在 facade 层，不必塞进 decode/render 热路径。

值得借鉴：

- 对外稳定、对内可替换。
- facade 负责 API、配置、callback、业务接入。
- engine/core 负责状态、线程、媒体流水线。

不要照抄：

- `MediaPlayer` 已经承担了不少业务能力，继续扩张会变成大类。
- 构造/析构中存在不少裸指针和手动释放。

C++17 改写方向：

```text
Player
  -> public facade

IPlayerEngine
  -> 内部抽象接口

FfmpegPlayerEngine
  -> 具体实现

CacheController / MetricsCollector / AbrController
  -> 业务旁路模块
```

## 2. C handle 包 C++ 对象

代表文件：

- `mediaPlayer/media_player_api.h`
- `mediaPlayer/media_player_api.cpp`

典型结构：

```cpp
typedef struct playerHandle_t {
    ICicadaPlayer *pPlayer;
} playerHandle_t;
```

这个设计解决的问题：

- SDK 或跨语言场景经常需要稳定的 C ABI。
- C++ 对象可以藏在 opaque handle 后面。
- 调用方只看到 `CicadaCreatePlayer()` / `CicadaReleasePlayer()` 这类 C 函数。

值得借鉴：

- C API 可以作为边界层，内部仍然保持 C++ 对象设计。
- handle 层主要做转发和边界适配，不应该承载复杂业务。

不要照抄：

- handle 内部裸指针必须有非常清楚的 create/release 配对。
- 如果只是 C++ 内部模块通信，不需要为了“像 SDK”而引入 C handle。

C++17 改写方向：

```text
C API 边界
  -> opaque handle
  -> 内部 unique_ptr / RAII wrapper 管理真实对象

C++ 内部
  -> 直接使用 std::unique_ptr<IPlayerEngine>
```

## 3. 接口隔离和模块边界

代表接口：

- `framework/data_source/IDataSource.h`
- `framework/demuxer/IDemuxer.h`
- `framework/codec/IDecoder.h`
- `framework/render/audio/IAudioRender.h`
- `framework/render/video/IVideoRender.h`

这些接口对应播放器流水线：

```text
IDataSource
  -> 读字节

IDemuxer
  -> 拆 packet / stream meta

IDecoder
  -> packet 到 frame

IAudioRender / IVideoRender
  -> 输出音频/视频
```

这个设计解决的问题：

- 平台实现、FFmpeg 实现、插件实现可以挂到同一调用点后面。
- 上层调度代码不用直接依赖每个具体实现。
- 后续增加数据源、demuxer、decoder、render 时不需要大改主流程。

值得借鉴：

- 接口按职责切，不按文件类型或实现来源切。
- 先读接口声明，再读一个最常用实现，不要一口气读完所有实现。

不要照抄：

- 不要为了“面向对象”给所有稳定小对象都加虚接口。
- 如果没有运行时替换需求，值类型或普通类更简单。

C++17 改写方向：

```cpp
class IDecoder {
public:
    virtual ~IDecoder() = default;
    virtual DecodeResult send(Packet packet) = 0;
    virtual std::optional<Frame> receive() = 0;
};
```

## 4. factory/prototype 选择实现

代表文件：

- `framework/base/prototype.h`
- `mediaPlayer/CicadaPlayerPrototype.cpp`
- `framework/data_source/dataSourcePrototype.cpp`
- `framework/demuxer/demuxerPrototype.cpp`
- `framework/codec/decoderFactory.cpp`

共同模式：

```text
候选实现注册
  -> 每个实现 probeScore()
  -> 选择分数最高的
  -> clone/create
  -> fallback 到内置实现
```

这个设计解决的问题：

- 调用方不需要到处写平台、格式、硬解软解判断。
- 具体实现自己判断是否支持当前输入。
- 工厂集中处理创建策略和 fallback。

值得借鉴：

- `probeScore()` 很适合表达“这个实现有多适合当前输入”。
- decoder factory 的硬解/软解 fallback 很适合做面试设计题。
- `unique_ptr<IDecoder>` 明确表达了 factory 返回后的独占所有权。

不要照抄：

- 静态数组和 `_nextSlot` 需要关注容量、初始化顺序和线程安全。
- `clone()` 返回裸指针时，释放责任不够直观。
- score 用裸整数，语义需要读常量才能明白。

C++17 改写方向：

```cpp
enum class ProbeScore {
    NotSupported = 0,
    Default = 100,
    Best = 200,
};

class IDecoderFactory {
public:
    virtual ~IDecoderFactory() = default;
    virtual ProbeScore probe(const StreamMeta& meta) const = 0;
    virtual std::unique_ptr<IDecoder> create(const StreamMeta& meta) const = 0;
};
```

## 5. 消息队列和控制面

代表文件：

- `mediaPlayer/player_msg_control.h`
- `mediaPlayer/player_msg_control.cpp`
- `mediaPlayer/SuperMediaPlayer.cpp`

核心模式：

```text
外部线程调用 API
  -> putMsg(MSG_*)
  -> 内部主循环 processMsg()
  -> 分发到 Process*Msg()
  -> ProcessVideoLoop()
```

这个设计解决的问题：

- API 调用不会直接做 open/read/decode/render 重活。
- start、pause、seek、stop 等命令可以在内部串行化。
- 高频命令可以压缩，例如只保留最后一次 seek。

值得借鉴：

- 控制面和数据面分离。
- API 线程只表达意图，播放器内部线程负责执行。
- seek 这类命令需要 latest-wins 思路。

不要照抄：

- 消息参数里有 `void *` 和手动 `new string`。
- 释放依赖 `recycleMsg()`，读代码时必须追替换、处理、stop/release 三条路径。

C++17 改写方向：

```cpp
enum class PlayerCommandType {
    SetDataSource,
    Prepare,
    Start,
    Pause,
    Seek,
    Stop,
};

struct PlayerCommand {
    PlayerCommandType type{};
    std::string url;
    std::int64_t positionMs = 0;
};
```

这部分可以直接对照 AVPlayerLab 的 `PlayerCore`：public API 投递 command，command
thread 串行推进 `PlayerStateMachine`。

## 6. 业务能力作为旁路模块接入

代表文件：

- `mediaPlayer/MediaPlayer.cpp`
- `mediaPlayer/abr/`
- `mediaPlayer/analytics/`
- `framework/cacheModule/CacheManager.cpp`
- `mediaPlayer/PlayerCacheDataSource.*`

缓存大致接入点：

```text
MediaPlayer::SetDataSource()
  -> 如果启用 cache
  -> CacheManager::init()
  -> 可能把 url 替换成本地缓存路径

MediaPlayer::mediaFrameCallback()
  -> CacheManager::sendMediaFrame()
  -> 写缓存
```

这个设计解决的问题：

- cache、ABR、analytics 不是 decoder/render 的职责。
- 业务能力既需要入口参数，也可能需要播放过程中的状态、buffer、frame 或错误事件。
- 旁路模块能减少核心播放循环的业务污染。

值得借鉴：

- 先找清楚业务模块的接入点。
- cache 更靠近 data source 和 media frame callback。
- ABR/analytics 更靠近 facade、状态和指标事件。

不要照抄：

- 不要让 facade 无限膨胀成所有业务的堆放点。
- 不要让业务模块反向侵入底层 codec/render。

C++17 改写方向：

```text
Player
  -> owns CacheController
  -> owns MetricsCollector
  -> owns AbrController

PlayerCore
  -> focuses on command/state/data pipeline
```

## 7. 错误处理和上报

代表文件：

- `framework/demuxer/demuxer_service.cpp`
- `framework/demuxer/avFormatDemuxer.cpp`
- `mediaPlayer/SMPMessageControllerListener.cpp`

CicadaPlayer 常见模型：

```text
int ret
FRAMEWORK_ERR_*
AF_LOGE / AF_LOGW
NotifyError / callback
```

这个设计解决的问题：

- FFmpeg 和平台 SDK 本来就是错误码风格。
- C API 和跨语言边界更容易接错误码。
- 媒体热路径里少用异常是常见选择。

值得借鉴：

- 底层错误最终要能转成用户可见 callback 或状态。
- 不要让错误只停留在日志里。

不要照抄：

- 到处裸 `int` 会让错误语义不统一。
- `catch (...)` 只适合兜底，不适合作为主错误处理方式。

C++17 改写方向：

```cpp
enum class PlayerError {
    None,
    OpenInputFailed,
    DecoderCreateFailed,
    RenderFailed,
    Interrupted,
};

struct Result {
    PlayerError error = PlayerError::None;
    std::string message;

    bool ok() const { return error == PlayerError::None; }
};
```

## 8. 线程封装和 shutdown

代表文件：

- `framework/utils/afThread.h`
- `framework/utils/afThread.cpp`
- `mediaPlayer/SuperMediaPlayer.cpp`
- `framework/codec/ActiveDecoder.cpp`

这个设计解决的问题：

- 播放器需要多个 worker 协作，不能靠单线程同步调用完成。
- shutdown 必须停止线程、唤醒等待、清队列、释放底层资源。

值得借鉴：

- 用 condition variable 控制等待，而不是忙等。
- 用 atomic 表达简单跨线程标记。
- stop/release 路径要显式建模。

不要照抄：

- `afThread` 内部仍有裸 `std::thread *`。
- pause/stop 状态语义偏复杂，学习时要画状态图，不要直接搬。

C++17 改写方向：

```cpp
class Worker {
public:
    void start();
    void requestStop();
    void join();

private:
    std::thread thread_;
    std::atomic_bool stopRequested_{false};
    std::mutex mutex_;
    std::condition_variable cv_;
};
```

C++17 没有 `std::jthread`，所以仍要自己保证析构前 `join()` 或 `stop()`。

## 9. 阅读练习

这些练习不需要改源码：

1. 画 `MediaPlayer`、`playerHandle`、`ICicadaPlayer`、`SuperMediaPlayer` 的所有权关系。
2. 把 `PlayerMessageControl::putMsg()` 的重复消息策略整理成表。
3. 对 `dataSourcePrototype::create()` 写伪代码，标出 probe/fallback 分支。
4. 对 `decoderFactory::create()` 写伪代码，标出硬解、软解、外部 prototype 的优先级。
5. 找一个裸 owning pointer，追它在哪个析构或 cleanup 路径释放。
6. 找一个 `unique_ptr` 返回值，说明为什么这里比裸指针更清晰。

## 10. 第一轮判断标准

读完这一章后，你应该能说清：

- 为什么 CicadaPlayer 比 ffplay 更适合学 C++ 工程结构。
- 哪些地方是接口隔离，哪些地方是 factory/prototype。
- 消息队列为什么能降低 API 调用和播放器主循环之间的耦合。
- 哪些设计值得迁移到自己的播放器。
- 哪些写法只是历史风格或 SDK 边界，不应该照抄。
