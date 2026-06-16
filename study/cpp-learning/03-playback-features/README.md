# 03 Playback Features

这一组讲主线之外、但很有学习价值的播放器能力。

推荐顺序：

1. [01-buffering-and-qoe.md](01-buffering-and-qoe.md)
2. [02-avplayer-transfer-candidates.md](02-avplayer-transfer-candidates.md)
3. [03-player-design-cases.md](03-player-design-cases.md)

这些专题要挂回主线理解：buffering 不是孤立功能，它会影响 `doReadPacket()` 的读取上限、
`DoCheckBufferPass()` 的渲染放行，以及 seek/flush 后的状态恢复。

`03-player-design-cases.md` 用场景驱动读法组织：起播、首帧、seek、buffering、弱网、EOF、错误、低内存、
A/V sync 和 render drop。它更适合用来提炼“这个播放器设计解决了什么问题”。
