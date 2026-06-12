# Decoder / Render / Clock 边界：从 packet 到 frame 再到输出

这一章看解码、渲染和 clock 的职责边界。重点不是平台硬解细节，而是理解播放器如何把
packet 送进 decoder，把 frame 交给 render，再用 clock 决定什么时候输出、什么时候等待、
什么时候丢帧。

可以把这条链路先简化成：

```text
demuxer packet
  -> packet queue
  -> IDecoder / ActiveDecoder
  -> frame queue
  -> audio render / video render
  -> render callback
  -> clock / position / first-frame / stats
```

## 源码入口

先看这些文件，不需要一开始追进每个平台实现：

- `framework/codec/IDecoder.h`
- `framework/codec/decoderFactory.h`
- `framework/codec/decoderFactory.cpp`
- `framework/codec/ActiveDecoder.h`
- `framework/codec/ActiveDecoder.cpp`
- `mediaPlayer/SMPAVDeviceManager.h`
- `mediaPlayer/SMPAVDeviceManager.cpp`
- `mediaPlayer/SuperMediaPlayer.cpp`
- `framework/render/audio/IAudioRender.h`
- `framework/render/video/IVideoRender.h`

重点函数：

- `decoderFactory::create()`
- `decoderFactory::createBuildIn()`
- `ActiveDecoder::open()`
- `ActiveDecoder::send_packet()`
- `ActiveDecoder::getFrame()`
- `ActiveDecoder::flush()`
- `SMPAVDeviceManager::setUpDecoder()`
- `SuperMediaPlayer::SetUpAudioPath()`
- `SuperMediaPlayer::SetUpVideoPath()`
- `SuperMediaPlayer::RenderAudio()`
- `SuperMediaPlayer::RenderVideo()`
- `SuperMediaPlayer::SendVideoFrameToRender()`
- `SuperMediaPlayer::RenderCallback()`

## 这个设计解决什么播放器问题？

解码和渲染不是简单的函数调用。播放器要同时处理这些问题：

- 同一种 stream 可能有多个 decoder 实现：内置软解、平台硬解、外部注册 codec。
- 解码通常不能阻塞控制面：packet 输入、frame 输出和 codec 内部工作需要隔离。
- 音频和视频的输出节奏不同：音频常作为主时钟，视频要根据 master clock 等待或丢帧。
- render 不是只负责“画出来”：它还要反馈首帧、已渲染、设备异常、耗时统计等事件。
- seek、pause、flush、stop 会打断 packet/frame 链路，decoder 和 render 都要有清晰边界。

CicadaPlayer 的价值在于：它不是把 FFmpeg decode 和 SDL render 直接写在一个循环里，而是把
decoder、render、clock、callback 拆成可以替换和扩展的 SDK 模块。

## decoderFactory：选择 decoder 实现

`decoderFactory::create()` 的主逻辑是：

```text
meta + flags
  -> 先询问 codecPrototype 是否有外部实现
  -> 如果没有，再 createBuildIn()
  -> 根据 DECFLAG_HW / DECFLAG_SW 选择硬解或软解
```

这里有两个值得学的点。

第一，播放器核心不应该直接 `new avcodecDecoder` 写死所有路径。用 factory 后，核心只依赖
`IDecoder`，后续可以接入平台硬解、DRM decoder、测试 fake decoder。

第二，硬解是一个策略选择，不是播放器主链路的必需条件。`createBuildIn()` 会根据 flag 和平台
条件选择 Android / Apple 硬解，失败后也可以回到软件 decoder。这对 AVPlayer 很重要：先把
软件解码链路做稳，再考虑硬解 fallback。

适合迁移到 AVPlayer 的最小版本：

```text
struct DecoderConfig {
    MediaType type;
    CodecId codec;
    bool preferHardware;
};

class IDecoder {
public:
    virtual ~IDecoder() = default;
    virtual DecodeResult sendPacket(Packet packet) = 0;
    virtual FrameResult receiveFrame() = 0;
    virtual void flush() = 0;
};

class IDecoderFactory {
public:
    virtual ~IDecoderFactory() = default;
    virtual std::unique_ptr<IDecoder> create(const DecoderConfig& config) = 0;
};
```

注意：AVPlayer 的第一版不需要完整 prototype 注册体系。先有一个清晰的 `IDecoderFactory`
接口和 FFmpeg 实现，就已经能支撑单元测试和后续扩展。

## ActiveDecoder：把 codec 工作和播放器主循环隔开

`ActiveDecoder` 也实现 `IDecoder`，但它不是具体 codec，而是一个 active wrapper。它内部持有：

- 真实 decoder。
- input packet queue。
- output frame queue。
- decode thread。
- pause / flush / running 等控制状态。

它对外仍然暴露 `send_packet()`、`getFrame()`、`flush()` 这类 decoder 接口。这样播放器主循环
不用关心 codec 内部是否有线程，只把 packet 交进去，再从 frame queue 取 frame。

这个设计解决的是播放器工程里的一个常见矛盾：

```text
控制面希望 decoder 像普通对象一样可调用；
数据面又希望 decode 工作能异步执行，避免阻塞主循环。
```

值得迁移的不是它的具体线程实现，而是这个边界：

```text
PlayerCore
  -> IDecoder
      -> SyncFfmpegDecoder
      -> AsyncDecoderWorker
```

AVPlayer 初期可以先做同步 decoder，等 packet/frame 队列稳定后，再加一个异步 wrapper。不要
一开始就把线程、flush、seek serial、硬解 fallback 全塞进 decoder。

## 音频路径：先解码，再建立 render 和 clock 参考

`SetUpAudioPath()` 的逻辑可以理解为：

```text
等到有 audio packet / stream meta
  -> setUpAudioDecoder()
  -> decoder 输出第一批 frame 信息
  -> setUpAudioRender()
  -> RenderAudio() 持续推 frame
```

`RenderAudio()` 的重点不是“播放声音”这一行，而是这些边界：

- 从 frame queue 取音频 frame。
- 必要时根据 frame format 重新创建 audio render。
- 把 frame 交给 audio render。
- 根据 render 反馈更新音频 clock / position。
- 通过 listener / callback 把已渲染事件送回播放器内部。

这说明 audio render 在播放器里通常不只是一个 sink。它还是 clock 的重要来源。音频设备实际消费
了多少数据，决定了当前播放时间的可信参考。

## 视频路径：解码输出 frame，再由 clock 决定输出时机

`SetUpVideoPath()` 的逻辑可以理解为：

```text
等到 video packet / stream meta
  -> CreateVideoRender()
  -> CreateVideoDecoder()
  -> 如果硬解失败，回退软解
  -> RenderVideo() 根据 master clock 调度 frame
```

`RenderVideo()` 的核心是调度，而不只是调用 render：

- frame 太早：等待。
- frame 正常：送给 video render。
- frame 太晚：根据策略丢帧或清理积压。
- render 成功后更新 video pts 和渲染统计。

这就是 ffplay/ijk 里视频同步逻辑在 SDK 工程中的形态：解码器负责产出 frame，render 负责输出，
clock/sync 逻辑决定这个 frame 是否该现在输出。

AVPlayer 可以先迁移一个很小的版本：

```text
Frame frame = videoFrames.peek();
auto delay = frame.pts - clock.now();

if (delay > threshold) {
    wait(delay);
} else if (delay < -dropThreshold) {
    videoFrames.pop();
    stats.droppedVideoFrames++;
} else {
    videoRender.render(frame);
    videoFrames.pop();
}
```

这里先不追求复杂策略。关键是把“decode 出 frame”和“决定 frame 什么时候显示”拆开。

## RenderCallback：render 结果回到控制面

CicadaPlayer 的 audio/video render listener 最终会走到 `SuperMediaPlayer::RenderCallback()`。
它把 render 事件包装成内部消息，例如已渲染、首帧、耗时信息，再投递回播放器主循环。

这个点很值得学：render 回调不应该随便直接改播放器状态。更稳的做法是：

```text
render thread callback
  -> package event
  -> post internal message
  -> player loop handles state/statistics/notification
```

这样可以减少跨线程状态修改，也方便把 first-frame、seek complete、QoE 统计统一放在控制面处理。

## Clock 边界：谁决定“现在播放到哪里”

播放器里 clock 的核心问题不是变量怎么存，而是谁有资格定义当前播放时间。

常见规则是：

- 音频正常输出时，audio clock 通常最可信。
- 视频根据 master clock 判断早到、准时、迟到。
- seek / pause / buffering 会改变 clock 的有效性。
- render callback 可以更新统计，但不应该绕开 clock 直接决定全局状态。

对 AVPlayer 来说，第一版可以先做一个很小的 clock 抽象：

```text
class PlaybackClock {
public:
    void reset(int64_t positionUs);
    void start();
    void pause();
    int64_t nowUs() const;
};
```

等真实 audio render 接入后，再让 audio device position 成为 master clock 的输入。不要一开始就
做多 clock 竞争、外部时钟、变速、低延迟等完整模型。

## 适合迁移到 AVPlayer 的最小版本

建议按这个顺序迁移，不要一次做大：

1. `IDecoder` 接口：`sendPacket()`、`receiveFrame()`、`flush()`。
2. `FfmpegDecoder` 或 fake decoder：先证明 packet -> frame 的边界。
3. `FrameQueue`：让 decoder 输出和 render 输入解耦。
4. `IRenderSink`：先做 fake audio/video sink，用于测试状态和时序。
5. `PlaybackClock`：先做单一 master clock。
6. `VideoFrameScheduler`：只负责 wait / render / drop。
7. `RenderEvent`：把 first-frame、rendered、drop 变成内部事件。

这套最小链路比完整硬解和平台渲染更适合当前阶段。它能直接训练播放器原理：packet/frame 边界、
音视频同步、渲染回调、flush 后旧数据清理。

## 哪些地方不要照抄？

- 不要把 `SuperMediaPlayer` 的大类形态照搬到 AVPlayer。它同时承担路径搭建、解码调度、渲染调度、
  同步、丢帧、通知和统计，学习时要拆职责。
- 不要过早引入平台硬解细节。硬解涉及 surface、DRM、颜色格式、生命周期、fallback，应该等软解链路
  稳定后再做。
- 不要把 decoder、render、clock 的错误都混成 `int` 返回值。AVPlayer 可以先用清晰的 enum 和
  结构化结果。
- 不要让 render callback 直接修改多个跨线程状态。优先转成内部事件，由播放器主循环处理。
- 不要为了“现代 C++”强行复杂化。这里最重要的是边界清楚、生命周期清楚、flush 语义清楚。

## 可公开的技术表达

CicadaPlayer 的解码渲染链路可以总结为：用 `decoderFactory` 隔离具体 decoder 选择，用
`ActiveDecoder` 把 codec 工作和播放器主循环解耦，用 audio/video render callback 把输出结果
回传给控制面，再由 clock/sync 逻辑决定视频 frame 的等待、渲染和丢弃。这个设计比最小播放器多了一层
SDK 化边界，适合用来学习播放器模块化、异步解码和音视频同步。

