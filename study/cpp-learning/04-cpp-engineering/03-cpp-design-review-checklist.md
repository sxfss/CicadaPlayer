# C++ 设计审查清单：用 CicadaPlayer 训练工程判断

这一章不是语法清单，而是读 C++ 播放器工程时的审查问题。每看到一个类、成员或接口，都用这些问题过一遍。

```text
谁创建？
谁拥有？
谁释放？
是否需要多态？
是否需要指针？
是否跨线程？
错误如何传播？
接口是否太大？
```

目标是：学 CicadaPlayer 的工程设计，同时识别不适合作为现代 C++17 模板的历史写法。

## 1. 这个对象是谁拥有的？

先区分三类指针：

```text
owning pointer
  -> 负责 delete / 释放资源

non-owning pointer
  -> 只是观察或回调上下文

borrowed pointer from C API
  -> 生命周期由外部协议保证
```

CicadaPlayer 例子：

- `MediaPlayer` 里有 `mConfig = new MediaPlayerConfig()`，析构时 `delete mConfig`，这是 owning pointer。
- `SuperMediaPlayer` 里大量成员已经是 `std::unique_ptr`，比如 `mBufferController`、`mMessageControl`。
- callback 里的 `void *userData` 是 C API 常见写法，本质是 non-owning context。

现代 C++17 推荐：

```cpp
class MediaPlayer {
private:
    MediaPlayerConfig config_;                 // 永远存在，用值成员
    std::unique_ptr<AbrManager> abrManager_;   // 可选或多态，用 unique_ptr
};
```

判断规则：

- 总是存在、不需要多态、不需要延迟创建：优先值成员。
- 独占拥有、可空、需要延迟创建、需要多态：`std::unique_ptr`。
- 共享生命周期很少是第一选择，除非确实有多个 owner。

## 2. 析构是否能安全停线程？

播放器类最怕对象析构了，后台线程还在访问它。

CicadaPlayer 例子：

```text
SuperMediaPlayer::~SuperMediaPlayer()
  -> Stop()
  -> mApsaraThread->stop()
  -> delete mPNotifier
```

这个顺序说明作者知道 notifier 可能被线程使用，所以要先停线程，再释放 notifier。

值得学习：

- 析构前先停止后台工作。
- stop/release 是播放器生命周期的一部分，不是简单 delete。

需要警惕：

- 手动 `new/delete` 和复杂 stop 顺序放在大类里，长期维护成本高。

现代 C++17 推荐：

```cpp
class JoiningThread {
public:
    ~JoiningThread() {
        requestStop();
        join();
    }
};
```

先把“线程会被安全 join”变成对象语义，再让播放器持有它。

## 3. 这个类是否职责过大？

`SuperMediaPlayer` 很适合学习真实播放器压力，但不适合当小项目类设计模板。

它同时承担：

- 控制命令处理。
- 状态管理。
- read packet 调度。
- decode 调度。
- buffer 判断。
- audio/video path setup。
- render 和 A/V sync。
- notifier 和 analytics。

值得学习：

- 这些职责确实都属于播放器引擎需要解决的问题。

不建议照抄：

- 全塞进一个大类会让状态关系变难测试、难复用。

AVPlayer 更好的拆法：

```text
PlayerCore
  -> CommandQueue
  -> PipelineLoop
  -> BufferPolicy
  -> DecodeController
  -> RenderScheduler
  -> Clock
  -> EventSink
```

第一版不用做得很复杂，但至少把“策略”和“执行”拆开。

## 4. 接口是否表达了稳定边界？

CicadaPlayer 值得学习的一点是接口边界比较多：

```text
IDataSource
IDemuxer
IDecoder
IAudioRender
IVideoRender
```

这些接口让 `SuperMediaPlayer` 不需要关心：

- URL 是 HTTP、本地文件还是 cache。
- demuxer 是普通容器、HLS 还是 DASH。
- decoder 是 FFmpeg 软解还是平台硬解。
- render 是 SDL、Android、Apple 还是 GL。

值得学习：

- 播放器主流程依赖抽象接口，具体实现由 factory/prototype 选择。

需要警惕：

- 接口如果过大，也会导致实现类被迫关心太多场景。
- C++ 接口要特别注意析构函数是否 virtual，所有权是否通过返回类型表达清楚。

现代 C++17 推荐：

```cpp
class IDecoder {
public:
    virtual ~IDecoder() = default;
    virtual DecodeResult send(PacketView packet) = 0;
    virtual FrameResult receive() = 0;
};

std::unique_ptr<IDecoder> createDecoder(const StreamMeta& meta);
```

## 5. 错误是否可读、可分类？

CicadaPlayer 里常见：

```text
int ret
STATUS_RETRY_IN
STATUS_HAVE_ERROR
framework error
MEDIA_PLAYER_ERROR_*
NotifyError()
```

这是 C/FFmpeg 风格工程里常见写法，兼容性强，但类型表达不够清楚。

值得学习：

- 错误需要分层映射，内部错误不能直接泄漏给外部 API。
- 有些问题是 fatal error，有些只是 event 或 fallback。

现代 C++17 推荐：

```cpp
enum class DecodeStatus {
    Accepted,
    RetryInput,
    EndOfStream,
    FatalError,
};

struct DecodeResult {
    DecodeStatus status;
    PlayerError error;
};
```

这样比 magic int 更适合学习和测试。

## 6. `void *` 是否只是 C API 边界？

CicadaPlayer 里 callback 很多：

```text
callback function pointer
void *userData
```

这在 C API、SDK ABI、跨语言绑定里很常见。它不一定是错，但要知道它牺牲了类型安全。

适合保留的场景：

- C API handle。
- 外部 SDK callback。
- 跨语言或插件边界。

不建议带进内部 C++ 设计：

```cpp
using RenderCallback = std::function<void(const RenderEvent&)>;
```

内部模块之间尽量用强类型事件，只有最外层 ABI 边界再转成 `void *`。

## 7. 是否把 namespace 放进头文件？

`SuperMediaPlayer.h` 这类头文件里能看到 `using namespace std;`。这不建议学习。

问题是：

- 污染所有包含这个头文件的翻译单元。
- 容易引入名字冲突。
- 公共头文件影响范围非常大。

现代 C++17 推荐：

```cpp
std::unique_ptr<BufferController> bufferController_;
std::queue<std::unique_ptr<Frame>> videoFrames_;
```

头文件里宁愿多写 `std::`，也不要 `using namespace`。

## 8. 是否需要多态，还是值成员就够？

不是所有对象都需要 `new` 或 `unique_ptr`。

适合值成员：

- 配置对象。
- 小型策略对象。
- 总是存在的状态对象。
- 不需要运行时替换的 helper。

适合多态指针：

- data source。
- demuxer。
- decoder。
- render。
- 平台相关实现。

判断句：

```text
如果对象类型在运行时会变化，用接口 + unique_ptr。
如果对象只是播放器固定组成部分，用值成员更简单。
```

## 9. 队列所有权是否清楚？

播放器里 packet/frame 所有权非常关键。

CicadaPlayer 里有几层：

```text
BufferController
  owns packet queues

SuperMediaPlayer
  owns pending audio/video packet
  owns audio/video frame queue

ActiveDecoder
  owns decoder input/output queues
```

值得学习：

- packet queue、decoder queue、frame queue 是不同层，不要混为一个队列。
- `std::unique_ptr<IAFPacket>` 和 `std::unique_ptr<IAFFrame>` 能表达“这个 packet/frame 当前只有一个 owner”。

现代 C++17 推荐：

```cpp
using PacketPtr = std::unique_ptr<Packet>;
using FramePtr = std::unique_ptr<Frame>;

class PacketQueue {
public:
    void push(PacketPtr packet);
    PacketPtr pop();
};
```

push/pop 都移动所有权，少用裸指针传递媒体数据。

## 10. 最小审查清单

以后读 CicadaPlayer 或写 AVPlayer，先问这 12 个问题：

```text
1. 这个类的单一职责是什么？
2. 谁创建它？
3. 谁释放它？
4. 是否总是存在？如果是，能不能做值成员？
5. 是否需要多态？如果是，接口是否足够小？
6. 是否跨线程访问？谁能改状态？
7. 析构时线程和 callback 是否已经停掉？
8. 错误是怎么传播的？有没有类型化？
9. packet/frame 的所有权是否通过 unique_ptr 或队列边界表达？
10. seek/flush 会不会漏掉某一层缓存？
11. public header 是否污染命名空间或暴露过多实现？
12. 这个设计是播放器机制值得学，还是历史写法只适合理解？
```

这份清单的核心不是“挑代码毛病”，而是训练你把 C++ 写法和播放器工程问题对应起来。
