# 通过 CicadaPlayer 培养 C++ 工程概念

这一章的目标不是夸 CicadaPlayer 写得多现代，而是把它当成一个真实 C++
工程样本：遇到好设计就提炼，遇到旧写法就练习识别风险，再用更清楚的 C++17
方式重新表达。

你现在不需要一次掌握所有语法。先围绕几个基础问题读：

```text
这个类负责什么？
谁创建它？
谁释放它？
它是否需要多态？
它是否真的需要指针？
出错时怎么传播？
多线程下谁能改它的状态？
```

## 1. 资源管理：先分清谁拥有对象

代表文件：

- `mediaPlayer/MediaPlayer.h`
- `mediaPlayer/MediaPlayer.cpp`

`MediaPlayer` 里有一些典型旧式写法：

```text
构造函数里 new MediaPlayerConfig / QueryListener / AbrManager
析构函数里 delete mQueryListener / mAbrManager / mConfig
collector 由 factory 创建，析构时再交给 factory destroy
```

这不是“无法用智能指针”，更像历史代码和 SDK 边界风格。它可以工作，但对学习者不友好：

- 构造函数中途失败时，前面已经 `new` 出来的对象要小心释放。
- 成员很多时，很难一眼看出哪个指针是拥有者，哪个只是借用。
- 外部 collector 和内部 collector 混在一个裸指针里，需要额外布尔值区分释放责任。

先把指针分成三类：

| 类型 | 含义 | CicadaPlayer 例子 | C++17 倾向 |
| --- | --- | --- | --- |
| owning pointer | 我创建，我负责释放 | `mConfig`、`mQueryListener`、`mAbrManager` | 值成员或 `std::unique_ptr` |
| non-owning pointer | 我只是借用，不释放 | listener 回调里的 `void *userData` | 裸指针可以保留，但命名/注释要写清 |
| factory-owned pointer | factory 创建，也由 factory 销毁 | `mCollector` | 用专门 owner wrapper 或清晰的 deleter |

## 2. 值成员、unique_ptr、裸指针怎么选

先用一个简单判断：

```text
对象永远存在，不需要多态，不需要延迟创建
  -> 优先值成员

对象由当前类独占，但需要延迟创建、可空、隐藏实现、或运行时多态
  -> std::unique_ptr

对象不是当前类拥有，只是回调上下文或临时借用
  -> 裸指针可以接受，但不要 delete
```

以 `MediaPlayerConfig` 为例，如果它只是 `MediaPlayer` 必有的配置对象，更清楚的写法是值成员：

```cpp
class MediaPlayer {
public:
    void setConfig(const MediaPlayerConfig& config)
    {
        config_ = config;
        configPlayer(&config_);
    }

private:
    MediaPlayerConfig config_;
};
```

如果对象真的需要可空或延迟创建，比如某个业务模块只有启用时才创建，可以用 `unique_ptr`：

```cpp
class PlayerFacade {
public:
    void enableAbr()
    {
        if (!abrManager_) {
            abrManager_ = std::make_unique<AbrManager>();
        }
        abrManager_->enable(true);
    }

private:
    std::unique_ptr<AbrManager> abrManager_;
};
```

不建议新代码这样写：

```cpp
MediaPlayerConfig* config = new MediaPlayerConfig();
// ...
delete config;
```

这种写法把“释放责任”藏在人的记忆里，而不是写在类型里。

## 3. 对象职责：不要把所有能力塞进一个类

代表文件：

- `mediaPlayer/MediaPlayer.*`
- `mediaPlayer/ICicadaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.*`

CicadaPlayer 的整体分层是值得学的：

```text
MediaPlayer
  -> SDK facade，负责对外 API、配置、回调、ABR、analytics、cache 接入

ICicadaPlayer
  -> 内部播放器抽象接口

SuperMediaPlayer
  -> 默认核心实现，负责消息、状态、demux/decode/render 调度
```

这里的好处是：SDK 用户面对的类可以稳定，内部实现可以替换。你以后设计自己的播放器时，也不要一开始就把所有逻辑塞进 `PlayerCore`。

更适合 C++17 训练的拆法：

```text
Player
  -> public facade，面向调用方

IPlayerEngine
  -> 内部抽象，表达播放器核心能力

FfmpegPlayerEngine
  -> 一个具体实现

CacheController / MetricsCollector / AbrController
  -> 业务旁路模块
```

要警惕的是：`MediaPlayer` 本身也已经承担了不少业务能力。它适合学习 SDK
门面层，但不适合照抄成一个继续膨胀的大类。

## 4. 接口和多态：先问是否真的需要运行时替换

代表接口：

- `mediaPlayer/ICicadaPlayer.h`
- `framework/data_source/IDataSource.h`
- `framework/demuxer/IDemuxer.h`
- `framework/codec/IDecoder.h`
- `framework/render/audio/IAudioRender.h`
- `framework/render/video/IVideoRender.h`

这些接口值得学，因为播放器天然有多种实现：

```text
IDataSource  -> file / curl / ffmpeg / cache / plugin
IDemuxer     -> ffmpeg demuxer / playlist demuxer / DASH/HLS
IDecoder     -> soft decoder / platform hardware decoder
Render       -> SDL / Android / Apple / null render
```

这里使用虚接口是合理的，因为调用方确实希望在运行时替换实现。

但不要把“面向对象”理解成“所有东西都要写接口”。如果一个对象没有替换需求，只是稳定的小数据结构，直接用 `struct` 或值成员更清楚。

更适合你当前阶段的接口写法：

```cpp
class IDecoder {
public:
    virtual ~IDecoder() = default;
    virtual DecodeResult send(Packet packet) = 0;
    virtual std::optional<Frame> receive() = 0;
};
```

注意三个点：

- 基类析构函数要是 `virtual`。
- 接口只暴露调用方需要的能力。
- 返回对象如果是独占所有权，优先 `std::unique_ptr<IDecoder>`。

## 5. factory/prototype：把选择策略集中起来

代表文件：

- `mediaPlayer/CicadaPlayerPrototype.cpp`
- `framework/data_source/dataSourcePrototype.cpp`
- `framework/demuxer/demuxerPrototype.cpp`
- `framework/codec/decoderFactory.cpp`

CicadaPlayer 有一个很值得借鉴的模式：

```text
注册候选实现
  -> 每个实现 probeScore()
  -> 选择分数最高的实现
  -> clone/create
  -> fallback 到内置实现
```

这解决的是“调用方不应该到处写 if/else 判断格式、平台、硬解软解”的问题。选择策略集中在 factory/prototype 里，模块实现自己报告是否支持当前输入。

但它的老写法也要警惕：

- 静态数组和 `_nextSlot` 需要关注容量和初始化顺序。
- `clone()` 多处返回裸指针，调用方必须知道谁负责释放。
- score 是裸整数，需要读常量才能理解语义。

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

这不是为了“语法更高级”，而是让支持度和所有权更容易讲清。

## 6. 消息队列：参数所有权要非常清楚

代表文件：

- `mediaPlayer/player_msg_control.h`
- `mediaPlayer/player_msg_control.cpp`
- `mediaPlayer/SuperMediaPlayer.cpp`

CicadaPlayer 的控制面值得学：

```text
外部 API
  -> putMsg(MSG_*)
  -> 内部主循环 processMsg()
  -> Process*Msg()
```

这能避免 API 线程直接执行 read/decode/render 重活，也能让 start、pause、seek、stop 在内部串行化。

但消息参数的写法有历史包袱。比如 `SetDataSource` 会创建 `new string`，通过消息传进去，再由 `recycleMsg()` 释放。这个设计能跑，但学习时要重点盯：

```text
谁 new？
消息被替换时会不会释放？
消息被处理后会不会释放？
stop/release 时队列里残留消息会不会释放？
```

更现代的 C++17 写法可以把参数放进值类型：

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

如果命令参数差异越来越大，再考虑 `std::variant`。你当前阶段先用简单
`struct` 更好，避免为了“现代”引入过多语法负担。

## 7. 错误处理：少用异常不等于到处返回 int 就好

代表文件：

- `framework/demuxer/demuxer_service.cpp`
- `framework/demuxer/avFormatDemuxer.cpp`
- `mediaPlayer/SMPMessageControllerListener.cpp`

CicadaPlayer 核心链路基本不是异常风格。它大量使用：

```text
int ret
FRAMEWORK_ERR_*
AF_LOGE / AF_LOGW
NotifyError / callback
```

这在媒体工程里有现实原因：

- FFmpeg 和平台 SDK 本来就是错误码风格。
- 热路径里不希望隐藏异常控制流。
- C API/跨语言边界更容易接错误码。

但不建议新代码到处返回裸 `int`。裸整数的问题是：调用方经常不知道每个值代表什么、是否必须上报、是否可以重试。

更适合 C++17 学习项目的写法：

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

这比复杂模板更适合你当前阶段：错误类型明确，面试也容易解释。

## 8. 多线程状态：先确认谁有写权限

代表文件：

- `mediaPlayer/SuperMediaPlayer.cpp`
- `mediaPlayer/player_msg_control.*`

播放器最容易出错的不是语法，而是状态被多个线程同时修改。读 CicadaPlayer 时，先抓住这个原则：

```text
外部线程只投递命令
内部线程串行处理命令
播放工作线程/回调只能通过受控路径影响状态
```

你在 AVPlayerLab 的 `PlayerCore` 已经沿着这个方向走：

```text
public API
  -> PlayerCommandQueue
  -> command thread
  -> PlayerStateMachine
```

这正是可以从 CicadaPlayer 借鉴、再用 C++17 写得更清楚的地方。

## 9. 读模块时的固定检查表

每看一个类，都按这个表过一遍：

| 问题 | 要写下来的答案 |
| --- | --- |
| 职责 | 它解决什么问题，不解决什么问题 |
| 创建者 | 谁 `new` / create / factory 出它 |
| 释放者 | 析构、release、factory destroy 还是外部释放 |
| 所有权 | 值成员、独占、共享、借用、factory-owned |
| 多态 | 是否真的需要虚接口 |
| 错误 | 返回值、日志、callback、异常兜底 |
| 线程 | 哪个线程能写状态，哪个线程只能读或投递命令 |
| 可借鉴点 | 架构思想是什么 |
| 不照抄点 | 老写法或风险是什么 |
| C++17 改写 | 用值成员、unique_ptr、enum class、Result、RAII 重新表达 |

## 10. 本章验收

读完这一章，不要求你能背出所有源码，但应该能说清：

- `new/delete`、值成员、`unique_ptr` 的区别。
- 如何判断一个裸指针是不是 owning pointer。
- 为什么接口适合 data source / demuxer / decoder / render。
- 为什么 factory 比到处写 `if/else` 更适合播放器模块选择。
- 为什么消息参数里手动 `new string` 是需要警惕的写法。
- 为什么错误码模型有工程原因，但裸 `int` 不是理想学习模板。
