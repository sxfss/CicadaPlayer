# 播放入口链路

这一章从外部 API 追到播放器核心循环。第一轮只看控制主线，不展开每个底层模块实现。

## 总图

```text
用户代码
  -> Cicada::MediaPlayer
  -> media_player_api.cpp 的 Cicada* 函数
  -> playerHandle
  -> ICicadaPlayer
  -> SuperMediaPlayer
  -> PlayerMessageControl
  -> SuperMediaPlayer::mainService()
  -> ProcessVideoLoop()
  -> doReadPacket() / doDeCode() / doRender()
```

## 1. 外层 C++ SDK 类：MediaPlayer

入口文件：

- `mediaPlayer/MediaPlayer.h`
- `mediaPlayer/MediaPlayer.cpp`

`MediaPlayer` 更像 SDK 用户直接接触的门面层。它做几类事情：

- 创建底层播放器 handle：构造函数里调用 `CicadaCreatePlayer()`。
- 注册回调：构造函数里把 prepared、completion、first frame、error、seek、buffer 等回调塞进 `playerListener`。
- 挂业务能力：ABR、analytics、cache config、外部字幕、截图等。
- 把用户操作转发到 C API：例如 `CicadaPreparePlayer()`、`CicadaStartPlayer()`、`CicadaSeekToTime()`。

第一轮读 `MediaPlayer` 时，不要把它当播放器核心。它是“对外 API + 业务适配 + 回调桥接”。

## 2. C API handle：media_player_api

入口文件：

- `mediaPlayer/media_player_api.h`
- `mediaPlayer/media_player_api.cpp`

关键结构：

```cpp
typedef struct playerHandle_t {
    ICicadaPlayer *pPlayer;
} playerHandle_t;
```

这层把 C++ 对象藏在 `playerHandle` 里，对外暴露一组 `Cicada*` 函数。典型链路：

```text
MediaPlayer::Prepare()
  -> CicadaPreparePlayer(handle)
  -> handle->pPlayer->Prepare()
```

这层值得学的 C++ 点：

- SDK 边界有时需要 C 风格 handle，方便跨语言或 ABI 包装。
- 真实实现仍然通过 C++ 虚接口 `ICicadaPlayer` 分发。
- `CicadaReleasePlayer()` 负责释放 `ICicadaPlayer` 和 handle，这是明确的所有权边界。

## 3. 播放器实现选择：CicadaPlayerPrototype

入口文件：

- `mediaPlayer/CicadaPlayerPrototype.h`
- `mediaPlayer/CicadaPlayerPrototype.cpp`
- `mediaPlayer/SuperMediaPlayer.cpp`

创建链路：

```text
CicadaCreatePlayer(opts)
  -> 解析 opts
  -> CicadaPlayerPrototype::create(&createOpt)
  -> 按 probeScore() 选择具体播放器
  -> 默认返回 SuperMediaPlayer
```

这里的设计重点不是“播放器有几个实现”，而是变化点被留出来了：

- 默认播放器是 `SuperMediaPlayer`。
- Apple 平台可能选择 `AppleAVPlayer`。
- 其他实现可以通过 prototype 注册和 `probeScore()` 参与选择。

这是一个典型工程模式：公共 API 不直接依赖具体实现，具体实现由创建器集中选择。

## 4. 核心实现：SuperMediaPlayer

入口文件：

- `mediaPlayer/ICicadaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.h`
- `mediaPlayer/SuperMediaPlayer.cpp`

`SuperMediaPlayer` 是第一轮最重要的类。你要先抓住这些方法：

- `SetDataSource(const char *url)`
- `Prepare()`
- `Start()`
- `Pause()`
- `SeekTo(int64_t pos, bool bAccurate)`
- `Stop()`
- `mainService()`
- `ProcessVideoLoop()`
- `doReadPacket()`

典型用户操作会被转换成消息：

```text
SuperMediaPlayer::SetDataSource()
  -> 构造 MsgDataSourceParam
  -> putMsg(MSG_SETDATASOURCE, param)

SuperMediaPlayer::Prepare()
  -> putMsg(MSG_PREPARE, dummyMsg)
  -> mApsaraThread->start()

SuperMediaPlayer::Start()
  -> putMsg(MSG_START, dummyMsg)

SuperMediaPlayer::SeekTo()
  -> putMsg(MSG_SEEKTO, param)
  -> 记录 mSeekPos / mSeekNeedCatch
```

这和 ffplay 的直接函数式流程不同，更接近 ijkplayer 的消息驱动。

## 5. 消息队列：PlayerMessageControl

入口文件：

- `mediaPlayer/player_msg_control.h`
- `mediaPlayer/player_msg_control.cpp`

`PlayerMessageControl` 管三件事：

- `putMsg()`：接收外部操作，并处理重复消息策略。
- `processMsg()`：从队列取出可处理消息。
- `OnPlayerMsgProcessor()`：分发到 listener 的具体处理函数。

例如：

```text
MSG_PREPARE
  -> ProcessPrepareMsg()

MSG_START
  -> ProcessStartMsg()

MSG_PAUSE
  -> ProcessPauseMsg()

MSG_SETDATASOURCE
  -> ProcessSetDataSourceMsg(url)

MSG_SEEKTO
  -> ProcessSeekToMsg(pos, accurate)
```

这部分要重点看两个工程点：

- 重复消息压缩：某些消息会 `REPLACE_ALL` 或 `REPLACE_LAST`，避免队列堆积无效操作。
- 参数所有权：`MSG_SETDATASOURCE` 里 url 是 `new string`，最后由 `recycleMsg()` 删除。

第二点不是现代 C++ 的理想写法，但非常适合训练你识别“谁创建，谁释放，异常路径会不会泄漏”。

## 6. 主循环：mainService

入口文件：

- `mediaPlayer/SuperMediaPlayer.cpp`

`mainService()` 的骨架可以这样理解：

```text
如果已经取消，退出
  -> 通知工具模块 loop
  -> 发送 DCA message
  -> 如果没有消息，或消息处理完
       -> ProcessVideoLoop()
       -> 计算下次 loop 间隔
       -> 条件变量等待
```

`ProcessVideoLoop()` 是实际播放流水线调度：

```text
检查状态
  -> doReadPacket()
  -> doDeCode()
  -> setUpAVPath()
  -> DoCheckBufferPass()
  -> startRendering()
  -> doRender()
  -> LiveTimeSync()
  -> checkEOS()
  -> updateBufferInfo()
  -> OnTimer()
```

这就是 CicadaPlayer 对 ffplay 主循环的工程化拆分。ffplay 把很多逻辑集中在少量函数中，CicadaPlayer 把它拆成更多对象和更细函数。

## 7. 第一轮对照 ffplay/ijk

| 问题 | ffplay | ijkplayer | CicadaPlayer |
| --- | --- | --- | --- |
| 外部 API | 命令行和全局结构 | JNI/FFPlayer API | `MediaPlayer` + C API handle |
| 命令调度 | 函数直接驱动 | 消息队列 | `PlayerMessageControl` |
| 核心循环 | read/decode/render 结构较集中 | read thread + decoder/render | `mainService()` + `ProcessVideoLoop()` |
| 模块替换 | 少 | 有平台差异 | 大量接口/factory/prototype |
| 学习重点 | 播放器原理 | seek/serial/flush | C++ 工程拆分 |

## 本章练习

1. 只用源码回答：`MediaPlayer::Start()` 到 `SuperMediaPlayer::Start()` 中间经过了哪些函数。
2. 找出 `MSG_SEEKTO` 在 `player_msg_control.cpp` 里最终分发到哪个处理函数。
3. 画出 `mainService()` 一轮循环，不要超过 8 个节点。
4. 对比 ffplay：`doReadPacket()` 和 ffplay 的 `read_thread()` 有哪些相似点，哪些职责被拆走了。

