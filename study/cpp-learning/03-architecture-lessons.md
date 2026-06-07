# 可借鉴的 C++ 工程设计点

这一章只总结 CicadaPlayer 里适合作为 C++ 开发借鉴的部分。它不是现代 C++ 风格范本，很多裸指针、宏、头文件 `using namespace` 都不建议照抄；但它的播放器 SDK 分层、接口边界、控制消息和模块接入方式很值得学。

## 总判断

CicadaPlayer 适合学习：

- 播放器 SDK 怎么拆层。
- 对外 API 怎么和内部核心隔离。
- 大型媒体模块怎么通过接口和 factory/prototype 组织。
- 控制命令怎么进入内部线程和主循环。
- 业务能力如缓存、ABR、analytics 怎么挂到播放主链旁边。
- C/C++/FFmpeg 混合工程里怎么审查资源生命周期。

不适合作为直接代码风格模板：

- 它是 C++11 基线，不是 C++17/20 风格。
- 核心播放链路基本不用异常，主要靠 `int ret`、错误码和 callback。
- 裸指针和手动 `delete` 较多。
- 宏和 C 风格 handle 较多。
- 部分头文件直接 `using namespace std`，这个不要学。

更准确的学习姿势是：学架构边界和工程问题处理方式，用现代 C++17 的所有权规则去反向改写它。

## 1. SDK 门面层和核心实现分离

代表文件：

- `mediaPlayer/MediaPlayer.h`
- `mediaPlayer/MediaPlayer.cpp`
- `mediaPlayer/media_player_api.cpp`
- `mediaPlayer/ICicadaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.cpp`

结构：

```text
MediaPlayer
  -> 面向 SDK 使用者
  -> 负责配置、回调、ABR、analytics、cache 接入

media_player_api.cpp
  -> C API handle 层
  -> 把 C++ 对象藏在 playerHandle 后面

ICicadaPlayer
  -> 核心播放器抽象接口

SuperMediaPlayer
  -> 默认核心实现
  -> 负责消息、主循环、数据读取、解码、渲染、buffer
```

可以借鉴：

- 外部 API 不直接依赖具体核心实现。
- 门面层可以挂业务能力，核心层专注播放状态和媒体流水线。
- C API handle 可以作为 ABI 边界，适合 SDK 或跨语言封装。

用 C++17 写自己的项目时，可以这样转化：

```text
Player
  -> public API / facade

IPlayerEngine
  -> 内部抽象接口

FfmpegPlayerEngine
  -> 具体实现

PlayerHandle 或 unique_ptr<IPlayerEngine>
  -> 明确所有权和释放路径
```

## 2. 控制面和数据面分离

代表文件：

- `mediaPlayer/player_msg_control.h`
- `mediaPlayer/player_msg_control.cpp`
- `mediaPlayer/SuperMediaPlayer.cpp`

CicadaPlayer 不是在 API 调用栈里直接完成播放动作，而是把外部命令投递成消息：

```text
SetDataSource()
  -> MSG_SETDATASOURCE

Prepare()
  -> MSG_PREPARE

Start()
  -> MSG_START

SeekTo()
  -> MSG_SEEKTO

内部主循环
  -> processMsg()
  -> Process*Msg()
  -> ProcessVideoLoop()
```

可以借鉴：

- API 线程和播放器工作线程解耦。
- seek、start、pause 等控制命令可以串行化。
- 可以做重复消息压缩，例如最后一次 seek 生效。
- 控制命令和 read/decode/render 数据路径分开，复杂度更可控。

用 C++17 写自己的项目时，可以这样转化：

```cpp
enum class PlayerCommandType {
    SetSource,
    Prepare,
    Start,
    Pause,
    Seek,
    Stop,
};

struct PlayerCommand {
    PlayerCommandType type{};
    int64_t positionMs{};
    std::string url;
};
```

然后用一个线程安全队列或状态机处理它。重点不是照抄 `PlayerMessageControl`，而是借鉴“外部命令进入内部串行控制面”的设计。

## 3. 接口隔离和模块边界

代表接口：

- `framework/data_source/IDataSource.h`
- `framework/demuxer/IDemuxer.h`
- `framework/codec/IDecoder.h`
- `framework/render/audio/IAudioRender.h`
- `framework/render/video/IVideoRender.h`

这些接口刚好对应播放器流水线：

```text
IDataSource
  -> 读字节

IDemuxer
  -> 拆 packet 和 stream meta

IDecoder
  -> packet 到 frame

IAudioRender / IVideoRender
  -> 输出音频/视频
```

可以借鉴：

- 按职责拆接口，而不是按“一个大播放器类”堆功能。
- 平台差异、FFmpeg 实现、SDL 实现都藏在接口后面。
- 上层调度代码只关心接口，不关心具体实现。

用 C++17 写自己的项目时，接口可以更简洁：

```cpp
class IDecoder {
public:
    virtual ~IDecoder() = default;
    virtual DecodeResult send(Packet packet) = 0;
    virtual std::optional<Frame> receive() = 0;
};
```

如果对象需要运行时多态，使用：

```cpp
std::unique_ptr<IDecoder> decoder;
```

如果不需要多态、不需要可空，就优先用值成员。

## 4. factory/prototype 选择实现

代表文件：

- `mediaPlayer/CicadaPlayerPrototype.cpp`
- `framework/data_source/dataSourcePrototype.cpp`
- `framework/demuxer/demuxerPrototype.cpp`
- `framework/codec/decoderFactory.cpp`

典型逻辑：

```text
候选实现注册
  -> 按输入 probeScore()
  -> 选择最高分实现
  -> clone/create
  -> fallback 到内置实现
```

可以借鉴：

- 让具体实现自己判断是否支持当前输入。
- 工厂集中处理选择策略，调用方不用写一堆平台/格式判断。
- fallback 逻辑很适合解码器：优先硬解，失败时软解。

不建议照抄：

- 静态数组注册和 `_nextSlot` 这种写法比较老。
- `clone()` 返回裸指针，调用方必须清楚释放责任。

C++17 改写方向：

```cpp
class IDecoderFactory {
public:
    virtual ~IDecoderFactory() = default;
    virtual int probe(const StreamMeta& meta) const = 0;
    virtual std::unique_ptr<IDecoder> create(const StreamMeta& meta) const = 0;
};
```

返回值直接用 `std::unique_ptr`，让所有权写在类型里。

## 5. 业务能力作为旁路模块接入

代表文件：

- `mediaPlayer/MediaPlayer.cpp`
- `mediaPlayer/abr/`
- `mediaPlayer/analytics/`
- `framework/cacheModule/CacheManager.cpp`
- `mediaPlayer/PlayerCacheDataSource.*`

CicadaPlayer 的业务能力没有全部塞进核心播放循环。比如缓存大致是：

```text
MediaPlayer::SetDataSource()
  -> 如果启用 cache
  -> CacheManager::init()
  -> 可能把 url 替换成本地缓存路径

MediaPlayer::mediaFrameCallback()
  -> CacheManager::sendMediaFrame()
  -> 写缓存
```

可以借鉴：

- 业务能力应找清楚接入点，而不是污染所有模块。
- 缓存既需要入口 URL，也需要播放过程中的 frame 回调。
- ABR 和 analytics 更像门面层的附加能力，不应该强行变成 decoder/render 的职责。

用 C++17 写自己的项目时，可以把这种能力设计成小组件：

```text
Player
  -> owns CacheController
  -> owns AnalyticsReporter
  -> owns AbrController

PlayerCore
  -> focuses on command/state/data pipeline
```

这样能避免核心类承担太多业务细节。

## 6. 错误处理模型

代表现象：

- 核心播放链路基本不用 `try/catch`。
- 大量函数返回 `int`，使用 `0`、`-1`、`FRAMEWORK_ERR_*`。
- 错误通过日志和 callback 上报。

可以借鉴：

- 媒体热路径里少用异常是常见选择。
- 跨 C API/FFmpeg/平台 SDK 时，错误码模型更容易对接。
- 错误最终要能转成用户可见 callback 或状态。

不建议照抄：

- 到处返回 `int`，但语义不统一，会增加阅读成本。
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
    PlayerError error{PlayerError::None};
    std::string message;

    bool ok() const { return error == PlayerError::None; }
};
```

如果接口很多，可以先不用引入复杂模板，保持“boring C++17”：明确 enum、明确 result struct、明确 callback。

## 7. 所有权和生命周期审查

代表文件：

- `mediaPlayer/MediaPlayer.cpp`
- `mediaPlayer/MediaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.h`
- `framework/codec/decoderFactory.cpp`
- `framework/base/media/AVAFPacket.*`

可以借鉴：

- `unique_ptr<IAFPacket>`、`unique_ptr<IAFFrame>` 很适合表达媒体数据所有权转移。
- `unique_ptr<IDecoder>` 很适合表达 factory 创建后的独占所有权。
- `SuperMediaPlayer` 中大量成员用 `unique_ptr`，能让析构路径更清楚。

不建议照抄：

- `MediaPlayer` 里 `mConfig`、`mQueryListener`、`mAbrManager` 等裸指针加手动 `delete`。
- 消息参数里 `new string`，再靠 `recycleMsg()` 删除，容易漏。
- factory 创建、factory 销毁的对象需要额外规则，不能假装普通裸指针很清楚。

C++17 判断规则：

```text
必定存在、生命周期跟宿主绑定
  -> 值成员

独占拥有、可空、延迟创建、运行时多态
  -> std::unique_ptr

只观察、不拥有
  -> 裸指针或引用，但命名/注释要清楚

共享所有权
  -> std::shared_ptr，但要谨慎

C handle 或 factory destroy
  -> RAII wrapper 或 unique_ptr + custom deleter
```

以 `MediaPlayerConfig` 为例，更现代的方向不是一定用 `unique_ptr`，而是优先考虑值成员：

```cpp
class MediaPlayer {
private:
    MediaPlayerConfig mConfig{};
};
```

如果函数需要指针，传 `&mConfig`。值成员的意思是对象嵌在宿主对象内部，宿主对象在堆上时，值成员也在堆上，不等于一定占栈。

## 8. 线程封装和 shutdown

代表文件：

- `framework/utils/afThread.h`
- `framework/utils/afThread.cpp`
- `mediaPlayer/SuperMediaPlayer.cpp`
- `framework/codec/ActiveDecoder.cpp`
- `mediaPlayer/abr/AbrManager.cpp`

可以借鉴：

- 把线程启动、暂停、停止封装起来。
- 用 `std::condition_variable` 控制等待，而不是忙等。
- 用 `atomic` 表达跨线程状态。
- shutdown 路径要显式：停止线程、唤醒等待、释放队列、释放底层资源。

不建议照抄：

- `afThread` 内部仍有裸 `std::thread *`。
- 线程状态、pause/stop 语义比较复杂，学习时要画状态图。

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

## 9. 不要照抄的风格清单

这些点适合识别，不适合模仿：

- 头文件 `using namespace std`。
- 对拥有关系使用裸指针。
- 手写 `new/delete`，但没有异常安全和早退路径保护。
- C 风格宏包装复杂逻辑。
- `void *` 到处传，除非这是明确的 ABI 边界。
- `int` 错误码没有统一 enum 或 result 结构。
- `catch (...)` 吞掉异常，只返回默认值。

看到这些不要简单否定项目，而是问：

```text
它是不是 SDK/平台/C API 边界导致的？
它是不是历史 C++11 代码导致的？
如果用 C++17 重写，所有权和错误语义能不能更清楚？
```

## 10. 适合迁移到自己项目的学习模板

如果你要把 CicadaPlayer 的思路迁移到 AVPlayerLab 或自己的播放器练习里，可以保留这些设计问题：

```text
1. 对外 API 和内部核心是否分离？
2. 命令是否进入串行控制面？
3. 播放状态和用户意图是否分开？
4. data source / demuxer / decoder / render 是否有清晰接口？
5. factory 是否只负责创建策略，而不是掺杂业务流程？
6. 业务模块是否通过明确接入点接入？
7. 每个对象的所有权是否能从类型看出来？
8. stop/release 是否能唤醒所有等待并关闭所有线程？
9. 错误是否能从底层错误码转成上层状态或 callback？
10. 哪些地方是为了平台/SDK 边界才保留 C 风格写法？
```

这才是 CicadaPlayer 对 C++ 开发最有价值的部分：不是语法现代，而是工程边界真实、模块关系复杂、很适合训练架构判断。

