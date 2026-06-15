# 控制面消息队列：PlayerMessageControl

这一章专门看 CicadaPlayer 的控制面消息队列。它和你正在做的 AVPlayerLab
`PlayerCore` 很贴近：外部 API 不直接做重活，而是把用户意图转成命令，交给内部线程串行处理。

第一轮读这部分，不要陷进所有消息类型。先抓三件事：

```text
消息怎么进队列？
重复消息怎么压缩？
消息参数谁创建、谁释放？
```

## 1. 源码入口

主要文件：

- `mediaPlayer/player_msg_control.h`
- `mediaPlayer/player_msg_control.cpp`
- `mediaPlayer/SuperMediaPlayer.cpp`
- `mediaPlayer/SMPMessageControllerListener.cpp`

关键函数：

- `SuperMediaPlayer::putMsg()`
- `PlayerMessageControl::putMsg()`
- `PlayerMessageControl::processMsg()`
- `PlayerMessageControl::recycleMsg()`
- `PlayerMessageControl::OnPlayerMsgProcessor()`
- `SMPMessageControllerListener::OnPlayerMsgIsPadding()`

## 2. 总体链路

典型控制命令链路：

```text
外部 API
  -> SuperMediaPlayer::SetDataSource / Prepare / Start / Pause / SeekTo
  -> SuperMediaPlayer::putMsg()
  -> PlayerMessageControl::putMsg()
  -> SuperMediaPlayer::mainService()
  -> PlayerMessageControl::processMsg()
  -> PlayerMessageControl::OnPlayerMsgProcessor()
  -> SMPMessageControllerListener::Process*Msg()
```

比如 seek：

```text
SuperMediaPlayer::SeekTo(pos, accurate)
  -> 构造 MsgSeekParam
  -> putMsg(MSG_SEEKTO, param)
  -> PlayerMessageControl::putMsg()
  -> mainService() 里 processMsg()
  -> ProcessSeekToMsg(pos, accurate)
```

这个设计的核心价值是：API 调用线程只表达“我要做什么”，播放器内部线程决定“什么时候做、按什么顺序做”。

## 3. 消息结构

`player_msg_control.h` 里定义了三层结构：

```text
PlayMsgType
  -> 消息类型，例如 MSG_START、MSG_PAUSE、MSG_SEEKTO

MsgParam
  -> union，承载不同消息参数

QueueMsgStruct
  -> msgType + msgParam + msgTime
```

可以把消息大致分成四类：

| 类型 | 例子 | 学习点 |
| --- | --- | --- |
| 播放控制 | `MSG_PREPARE`、`MSG_START`、`MSG_PAUSE`、`MSG_SEEKTO` | 控制面主线 |
| 配置变化 | `MSG_SET_SPEED`、显示模式、旋转、镜像 | 重复消息需要压缩 |
| 资源入口 | `MSG_SETDATASOURCE`、`MSG_SET_BITSTREAM`、外部字幕 | 参数所有权要看清 |
| 内部事件 | `MSG_INTERNAL_RENDERED`、clean frame、hold video | 内部回投，不等同外部 API |

## 4. 入队和唤醒

`SuperMediaPlayer::putMsg()` 做两件事：

```text
mMessageControl->putMsg(type, param)
如果 trigger 为 true，唤醒 mPlayerCondition
```

这说明外部 API 不直接驱动播放循环，而是：

```text
投递消息
  -> 唤醒 mainService
  -> mainService 决定处理消息还是推进播放循环
```

`MSG_INTERNAL_RENDERED` 会用 `trigger = false` 投递，说明不是所有内部事件都需要立即唤醒主循环。

## 5. 重复消息策略

`getRepeatTimeMS()` 决定同类消息如何处理：

| 策略 | 典型消息 | 含义 |
| --- | --- | --- |
| `REPLACE_ALL` | `MSG_SETDATASOURCE`、`MSG_PREPARE`、切流、显示配置 | 队列里已有同类消息全部删掉，只保留新的 |
| `REPLACE_LAST` | `MSG_START`、`MSG_PAUSE`、`MSG_SET_SPEED` | 如果队尾是同类消息，删掉队尾再入队 |
| `SEEK_REPEAT_TIME` | `MSG_SEEKTO` | 500ms 内靠近的 seek 会合并，并限制队列里残留的 seek 数量 |
| `REPLACE_NONE` | rendered、外部字幕选择、video clean/hold | 不压缩 |

seek 这里不是简单“永远只保留最后一个”。它大致做了两层控制：

```text
如果新 seek 距离上一个同类 seek 小于 500ms
  -> 删除最后一个旧 seek

如果队列里同类 seek 已经有 2 个或以上
  -> 删除最早的 seek
```

可以理解成：保留最新意图，同时避免队列里积累太多过期 seek。

## 6. processMsg 的处理方式

`processMsg()` 有一个值得学的点：它不会在持锁状态下直接调用业务处理函数。

逻辑可以这样理解：

```text
加锁
  -> 遍历 mMsgQueue
  -> 如果消息不是 padding，搬到本地 processQueue
  -> 从 mMsgQueue 删除
解锁

遍历 processQueue
  -> OnPlayerMsgProcessor()
  -> recycleMsg()
  -> 统计外部消息数量
```

这样做的好处：

- 队列锁只保护队列结构。
- 真实业务处理不占用队列锁。
- 避免 `Process*Msg()` 里复杂逻辑扩大锁范围。

需要注意：

- 消息从 `mMsgQueue` 搬到 `processQueue` 后，原队列节点不调用 `recycleMsg()`。
- 因为参数所有权已经跟着 `QueueMsgStruct` 副本进入 `processQueue`。
- 最终由处理后的 `processQueue` 节点调用 `recycleMsg()`。

这也是为什么手动所有权读起来很累：你必须知道“这个指针现在跟着哪个消息副本走”。

## 7. padding：暂时不能处理的消息

`SMPMessageControllerListener::OnPlayerMsgIsPadding()` 会让部分消息暂时留在队列里。

典型规则：

```text
正在切视频流
  -> 新的视频切流消息先 padding

正在切音频流
  -> 新的音频切流消息先 padding

正在切字幕流
  -> 新的字幕切流消息先 padding

正在 seek
  -> 新的 MSG_SEEKTO 先 padding
```

这里的思路是：某些动作不是瞬时完成的。比如 seek 已经开始但还没结束，新 seek
不能直接和旧 seek 的处理过程交错，否则状态会混乱。

对应到 `SuperMediaPlayer::doRender()` 和 `playCompleted()`，它们在结束 seek 时还会检查：

```text
如果队列里没有新的 MSG_SEEKTO
  -> NotifySeekEnd
  -> ResetSeekStatus
```

这说明 `findMsgByType(MSG_SEEKTO)` 参与了 seek 完成语义：如果后面还有 pending seek，就不要急着宣告 seek 彻底结束。

## 8. 参数所有权风险

这部分是你当前最应该训练的 C++ 眼力。

`SuperMediaPlayer::SetDataSource()` 会这样做：

```text
new string(url)
  -> 放到 MsgDataSourceParam.url
  -> putMsg(MSG_SETDATASOURCE)
```

`addExtSubtitle()` 也会为 URL 创建 `new string`。

释放位置在 `PlayerMessageControl::recycleMsg()`：

```text
如果消息是 MSG_SETDATASOURCE 或 MSG_ADD_EXT_SUBTITLE
  -> delete msg.msgParam.dataSourceParam.url
```

这套写法能工作，但不是现代 C++ 学习模板。风险在于：

- `MsgParam` 是 union，类型安全弱。
- URL 是裸指针，所有权不写在类型里。
- 替换消息、清空队列、处理完成都必须记得调用 `recycleMsg()`。
- 新增一个带堆对象的消息时，必须同步改 `recycleMsg()`，否则容易泄漏。

也要区分哪些 `void *` 只是借用：

```text
MSG_SETVIEW 的 view
MSG_SET_BITSTREAM 的 arg
MSG_INTERNAL_RENDERED 的 userData
```

这类参数通常不是 `PlayerMessageControl` 拥有，不应该在这里释放。

## 9. C++17 可以怎么写

如果只做学习项目，先不要一上来引入复杂模板或复杂 `std::variant`。第一版可以用简单值类型：

```cpp
enum class PlayerCommandType {
    SetDataSource,
    Prepare,
    Start,
    Pause,
    Seek,
    Stop,
    SetSpeed,
};

struct PlayerCommand {
    PlayerCommandType type = PlayerCommandType::Start;
    std::string url;
    std::int64_t positionMs = 0;
    bool accurate = false;
    float speed = 1.0f;
    std::uint64_t sequence = 0;
};
```

重复消息策略可以单独表达：

```cpp
enum class RepeatPolicy {
    KeepAll,
    ReplaceAll,
    ReplaceLast,
    CoalesceSeek,
};
```

这样比 `#define REPLACE_ALL (-1)` 更容易讲清：

```text
命令是什么
命令携带什么参数
重复命令如何处理
谁拥有命令参数
```

如果以后命令参数差异变得很大，再考虑：

```cpp
using PlayerCommandPayload = std::variant<
    SetDataSourcePayload,
    SeekPayload,
    SetSpeedPayload
>;
```

但你当前阶段先把“值类型命令 + 明确策略 + 队列所有权”讲清楚更重要。

## 10. 和 AVPlayerLab 的对应关系

CicadaPlayer：

```text
MediaPlayer/SuperMediaPlayer API
  -> PlayMsgType + MsgParam
  -> PlayerMessageControl
  -> SMPMessageControllerListener
  -> SuperMediaPlayer 状态和播放循环
```

AVPlayerLab 当前方向：

```text
PlayerCore public API
  -> PlayerCommandType + PlayerCommand
  -> PlayerCommandQueue
  -> command thread
  -> PlayerStateMachine
```

可以迁移的设计：

- API 只投递命令，不直接改核心状态。
- command thread 串行处理状态迁移。
- seek 做 latest-wins 或 pending 合并。
- release/stop 要能唤醒等待线程并清理队列。

不迁移的写法：

- 不用 `void *` 承载命令参数。
- 不用手动 `new string` + `recycleMsg()` 管理命令 payload。
- 不用宏整数表达重复消息策略。

## 11. 面试怎么表达

可以这样讲：

```text
我看 CicadaPlayer 时重点关注它的控制面。
它不是在 API 调用栈里直接完成 prepare/start/seek，而是把外部请求转成消息，
放进 PlayerMessageControl，由内部 mainService 统一处理。

这个设计的好处是 API 线程和播放器内部线程解耦，seek/start/pause 等命令可以串行化，
高频 seek 还能做合并，避免无效工作堆积。

但它的消息参数使用 union、void* 和手动 new/delete，这不是我会照抄的现代 C++ 写法。
如果用 C++17 重写，我会使用 enum class 表达命令类型，用值类型 struct 表达 payload，
用明确的 RepeatPolicy 表达压缩策略，让所有权从类型上可见。
```

## 12. 本章练习

1. 画出 `SuperMediaPlayer::SeekTo()` 到 `ProcessSeekToMsg()` 的调用链。
2. 整理 `getRepeatTimeMS()` 的消息策略表。
3. 解释为什么 `processMsg()` 要先搬到 `processQueue` 再处理。
4. 找出哪些消息参数由 `PlayerMessageControl` 拥有，哪些只是借用。
5. 用 C++17 写一个简化版 `PlayerCommand`。
6. 对照 AVPlayerLab，说明 `PlayerCommandQueue` 应该避免哪些 CicadaPlayer 旧写法。
