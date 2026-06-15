# 01 Entry / Control Plane

这一组讲外部 API 怎么进入播放器核心，以及控制命令如何进入主循环。

推荐顺序：

1. [01-player-entry-flow.md](01-player-entry-flow.md)
2. [02-player-message-control.md](02-player-message-control.md)
3. [03-seek-control.md](03-seek-control.md)

这一组只解决控制面。读完以后，你应该知道 `Prepare/Start/Seek/Stop` 怎么投递命令，但还不能算理解
拉流、解复用、解码、同步和渲染。数据面继续看
[../02-runtime-pipeline/README.md](../02-runtime-pipeline/README.md)。
