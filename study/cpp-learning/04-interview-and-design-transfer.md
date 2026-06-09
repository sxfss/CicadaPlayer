# 面试表达与设计迁移

这一章把 CicadaPlayer 的源码学习转成两类输出：

1. 面试时能讲清楚的播放器工程设计。
2. 后续能迁移到 AVPlayerLab 或你自己的播放器中的 C++17 设计草图。

注意：迁移的是设计思想，不是照搬 CicadaPlayer 的代码风格。

## 1. 总体面试主线

可以这样讲 CicadaPlayer 的学习价值：

```text
ffplay 帮我建立了播放器最小闭环：
open input -> demux -> packet queue -> decode -> frame queue -> render -> clock。

ijkplayer 让我理解了工程播放器里的 seek、flush、serial 和消息控制链。

CicadaPlayer 更像一个播放器 SDK：
它把对外 API、C ABI handle、核心播放器、数据源、解封装、解码、渲染、
缓存、ABR、analytics 拆成多层模块。

我重点学习的是它的工程边界，而不是照抄它的 C++11 历史写法。
```

这段话能避免面试官觉得你只是“看过很多源码”，而是能说清每个项目给你补了什么能力。

## 2. 设计点迁移表

| CicadaPlayer 设计点 | C++ 概念 | C++17 推荐表达 | AVPlayerLab 可落点 | 面试表达 |
| --- | --- | --- | --- | --- |
| `MediaPlayer -> ICicadaPlayer -> SuperMediaPlayer` | facade、接口、多态、职责分离 | `Player` facade + `IPlayerEngine` + `FfmpegPlayerEngine` | 未来 public facade 和当前 `PlayerCore` 解耦 | 对外 API 稳定，内部 engine 可替换 |
| `media_player_api.cpp` 的 C handle | ABI 边界、opaque handle | C API 层保留 handle，内部用 RAII 管 C++ 对象 | 以后做 C API/跨语言绑定时再引入 | SDK 边界可以是 C，内部仍保持 C++ 设计 |
| `PlayerMessageControl` | command queue、控制面串行化 | `enum class PlayerCommandType` + value command struct | 已对应 `PlayerCommandQueue` 和 command thread | API 不直接做重活，命令进入内部线程串行处理 |
| `MSG_SEEKTO` 替换策略 | latest-wins、命令压缩 | 队列层合并 pending seek | `PlayerCore::seekTo()` latest-wins | 高频 seek 只保留最后意图，减少无效工作 |
| `IDataSource/IDemuxer/IDecoder/Render` | 接口隔离、运行时多态 | `std::unique_ptr<I*>` 表达独占实现 | 后续 demuxer/decoder/render 边界 | 上层调度依赖抽象，不依赖平台实现 |
| `probeScore + clone/create` | factory/prototype、策略集中 | factory 返回 `std::unique_ptr`，score 用 `enum class` | decoder/data source factory | 创建策略集中，调用方不用到处写格式判断 |
| 硬解/软解 fallback | 能力探测、失败降级 | `DecoderFactory::create()` 明确 fallback 链 | 后续 decoder 模块 | 优先硬解，失败可控地降级软解 |
| cache/ABR/analytics | 业务旁路模块 | `CacheController` / `AbrController` / `MetricsCollector` | `BufferController`、`MetricsCollector` | 业务能力找主链接入点，不污染 decode/render |
| `FRAMEWORK_ERR_* + NotifyError` | 错误传播、用户可见回调 | `enum class PlayerError + Result + callback` | `PostStatus` 后续扩展 error model | 内部错误要最终转成上层可观察状态 |

## 3. 面试题表达模板

### 为什么播放器 API 不直接执行播放动作？

回答重点：

```text
播放器 API 可能从 UI 线程或业务线程调用，如果在调用栈里直接 open、seek、decode，
会造成阻塞和状态竞争。更稳的做法是把外部请求转成 command，进入播放器内部线程串行处理。
这样 start/pause/seek/stop 的顺序、优先级和取消语义都能统一管理。
```

可对照源码：

- `MediaPlayer` 对外提供 API。
- `SuperMediaPlayer::Start/SeekTo` 投递消息。
- `PlayerMessageControl` 做消息入队、替换和分发。
- `mainService()`/`ProcessVideoLoop()` 负责内部推进。

不要照抄点：

- CicadaPlayer 的消息参数里有 `void *` 和手动 `new string`。
- 你自己的 C++17 写法应优先用 value command struct。

### seek 命令怎么设计？

回答重点：

```text
seek 是用户高频操作，不能每个 seek 都完整执行到底。
控制面需要 latest-wins，只保留最后一次有效 seek。
数据面需要 flush/serial/generation，避免旧 packet、旧 frame 或旧 worker 回调污染新时间线。
```

迁移到 AVPlayerLab：

- `PlayerCore::seekTo()` 投递 `PlayerCommandType::Seek`。
- 队列层做 latest-wins。
- 后续 worker 用 `generation/workSerial/seekSerial` 过滤旧结果。

### 硬解/软解 fallback 怎么设计？

回答重点：

```text
不要在业务调用处散落硬解/软解判断。
应该把创建策略放进 DecoderFactory：先根据 stream meta、平台能力、用户 flags 探测，
优先选择硬解；硬解不可用或创建失败时，明确 fallback 到软解。
返回值用 unique_ptr<IDecoder> 表达独占所有权。
```

可对照源码：

- `framework/codec/decoderFactory.cpp`
- `codecPrototype::create()`
- `createBuildIn()`
- `DECFLAG_HW` / `DECFLAG_SW`

不要照抄点：

- 新代码不要用裸整数表达复杂选择结果。
- 工厂返回对象时优先把所有权写在类型里。

### 缓存、ABR、analytics 为什么不放进 decoder？

回答重点：

```text
decoder 的职责是 packet 到 frame，不应该知道缓存、ABR 和业务指标。
缓存需要接触 URL 和媒体数据回调，ABR 需要码率/缓冲状态，analytics 需要状态和错误事件。
这些能力应该作为 Player facade 或 core 周边的 controller 接入主链，而不是塞进底层 codec。
```

迁移到 AVPlayerLab：

- `BufferController` 负责缓冲策略。
- `MetricsCollector` 负责状态、耗时、错误、卡顿统计。
- `PlayerCore` 只负责控制面编排和状态推进。

## 4. C++17 迁移草图

### 4.1 Player facade 和 engine 分离

```cpp
class IPlayerEngine {
public:
    virtual ~IPlayerEngine() = default;
    virtual PostStatus setDataSource(std::string url) = 0;
    virtual PostStatus prepareAsync() = 0;
    virtual PostStatus start() = 0;
    virtual PostStatus pause() = 0;
    virtual PostStatus seekTo(std::int64_t positionMs) = 0;
    virtual void release() = 0;
};

class Player {
public:
    explicit Player(std::unique_ptr<IPlayerEngine> engine)
        : engine_(std::move(engine))
    {
    }

private:
    std::unique_ptr<IPlayerEngine> engine_;
};
```

这个草图对应 CicadaPlayer 的 `MediaPlayer -> ICicadaPlayer`，但所有权更清楚。

### 4.2 控制命令用值类型

```cpp
enum class PlayerCommandType {
    SetDataSource,
    Prepare,
    Start,
    Pause,
    Seek,
    Stop,
    Release,
};

struct PlayerCommand {
    PlayerCommandType type = PlayerCommandType::Start;
    std::string url;
    std::int64_t positionMs = 0;
    std::uint64_t sequence = 0;
};
```

这个草图对应 CicadaPlayer 的 `MSG_*`，但避免了 `void *` 参数和手动释放。

### 4.3 decoder factory 返回 unique_ptr

```cpp
class DecoderFactory {
public:
    std::unique_ptr<IDecoder> create(const StreamMeta& meta, DecodePreference preference)
    {
        if (preference.allowHardware) {
            if (auto decoder = tryCreateHardware(meta)) {
                return decoder;
            }
        }

        if (preference.allowSoftware) {
            return tryCreateSoftware(meta);
        }

        return nullptr;
    }
};
```

这里借鉴 CicadaPlayer 的硬解/软解 fallback，但避免让调用方承担释放责任。

### 4.4 错误用明确类型

```cpp
enum class PlayerError {
    None,
    EmptyUrl,
    OpenInputFailed,
    DemuxerCreateFailed,
    DecoderCreateFailed,
    Interrupted,
};

struct PlayerResult {
    PlayerError error = PlayerError::None;
    std::string message;

    bool ok() const { return error == PlayerError::None; }
};
```

这个草图对应 CicadaPlayer 的 `FRAMEWORK_ERR_*` 和 `NotifyError`，但更适合学习和面试表达。

## 5. 不要陷进去的地方

第一轮不建议深挖：

- HLS/DASH 协议细节。
- Android/Apple 平台 SDK 封装。
- DRM。
- analytics 指标体系全量字段。
- 宏和 CMake 平台适配细节。

看到这些内容时只问：

```text
它挂在哪一层？
它和播放主链通过什么接口交互？
它是否影响控制面、数据面、错误上报或资源释放？
```

## 6. 后续专题固定模板

以后每看一个 CicadaPlayer 专题，都按这个模板补笔记：

```text
## 这个设计解决什么问题

## C++ 概念是什么

## 哪些地方值得借鉴

## 哪些地方不要照抄

## C++17 可以怎么写

## 可以迁移到我的播放器哪里

## 面试怎么表达
```

这样能保证你不会陷入源码细节，而是持续产出可复用的工程能力。

## 7. 最小学习闭环

每个专题完成后，只交付三样东西：

1. 一张调用链图。
2. 一个职责/所有权表。
3. 一个 C++17 改写草图。

能做到这三点，就比泛泛读完几千行源码更有价值。
