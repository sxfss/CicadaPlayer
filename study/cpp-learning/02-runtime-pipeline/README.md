# 02 Runtime Pipeline

这一组是当前最重要的主线：普通 URL/MP4 在 CicadaPlayer 里怎么从 bytes 走到 render。

推荐顺序：

1. [00-object-model-and-module-wiring.md](00-object-model-and-module-wiring.md)
2. [01-normal-mp4-playback-loop.md](01-normal-mp4-playback-loop.md)
3. [02-data-source-cache.md](02-data-source-cache.md)
4. [03-demuxer-prototype.md](03-demuxer-prototype.md)
5. [04-packet-frame-queue.md](04-packet-frame-queue.md)
6. [05-decode-sync-render.md](05-decode-sync-render.md)

读法：

```text
先看对象关系：谁持有谁。
再看播放循环：一轮 mainService 做什么。
最后分层深挖：bytes、packet、frame、clock、render。
```

第一轮只追普通 URL/MP4。HLS/DASH、DRM、平台硬解、ABR 先当旁路能力，不要放进主线。
