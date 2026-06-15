# Build / Run Probe：当前环境能不能跑 CicadaPlayer

这份记录只做轻量探测，不启动完整 external 大编译。目标是判断当前机器是否能直接构建
`cmdline/cicadaPlayer`，以及如果不能，需要补哪些依赖。

## 探测结论

当前环境可以完成 CMake 配置，但不能完成 `cicadaPlayer` 编译。

已具备：

```text
cmake: 3.22.1
c++: g++ 11.4.0
pkg-config: 0.29.2
SDL2: 2.0.20
```

当前缺失：

```text
external/install/ffmpeg/Linux/x86_64/include
external/install/curl/Linux/x86_64/include
external/install/openssl/Linux/x86_64/include
external/install/libxml2/Linux/x86_64/include
external/boost/boost/lockfree/spsc_queue.hpp
```

实际失败点：

```text
framework/data_source/ffmpeg_data_source.h: libavformat/url.h not found
framework/base/media/spsc_queue.h: boost/lockfree/spsc_queue.hpp not found
```

这说明当前 checkout 缺少 CicadaPlayer 自己期望的 external 安装产物。尤其
`libavformat/url.h` 是 FFmpeg 内部头文件，不是系统 `libavformat-dev` 一定会提供的稳定公共头。
所以不建议简单用系统 FFmpeg 包硬凑。

## 已执行的轻量命令

```bash
cmake -S /home/sxf/Code/CicadaPlayer/cmdline -B /tmp/cicada-cmdline-build
cmake --build /tmp/cicada-cmdline-build --target cicadaPlayer -j2
```

配置成功，构建失败。`/tmp/cicada-cmdline-build` 是临时目录，不进入仓库。

## 后续要真正跑起来怎么做

官方 Linux 编译路径在 `doc/compile_Linux.md`：

```bash
cd /home/sxf/Code/CicadaPlayer
. setup.env
cd external
./build_external.sh Linux

cd ../cmdline
mkdir build
cd build
cmake ../
make cicadaPlayer
```

这里的关键不是 `cmdline` 本身，而是先把 external 产物准备好。完整 external 编译可能会拉源码、
编译 FFmpeg/curl/openssl/libxml2/boost 相关依赖，耗时和网络风险都比较高，适合单独开一轮处理。

## 跑通以后要观察什么日志

如果后续 `cicadaPlayer` 能运行，建议优先观察这些阶段，而不是盯着所有日志：

```text
MediaPlayer::SetDataSource
SuperMediaPlayer::mainService
SuperMediaPlayer::ProcessVideoLoop
SuperMediaPlayer::doReadPacket
SuperMediaPlayer::ReadPacket
demuxer_service::readPacket
avFormatDemuxer::ReadPacket
SuperMediaPlayer::doDeCode
SuperMediaPlayer::setUpAVPath
SuperMediaPlayer::DoCheckBufferPass
SuperMediaPlayer::render
SuperMediaPlayer::RenderAudio
SuperMediaPlayer::RenderVideo
```

每条日志都按三类归档：

```text
control：命令、状态、seek、stop、release
data：bytes、packet、frame、queue
time：buffer duration、audio clock、video late/early
```

这样跑出来的日志才能反哺学习文档，而不是变成一大段难读的输出。
