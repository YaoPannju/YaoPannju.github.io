# obs-studio 源码学习（三）——音频线程梳理

> 本文是一个 OBS 源码初学者的学习笔记，记录了在梳理 OBS 音频子系统时的思考和理解。文章中保留了"边问边学"的对话感，希望能帮到同样对 OBS 音频架构感兴趣的人。

---

## 先搞清楚"三大线程"之外还有什么

OBS 常说的"三大线程"指的是：

- **main 线程** — UI 主线程
- **graphics 线程** — `obs_graphics_thread`，视频渲染
- **video encoder 线程** — 视频编码

但音频也是一个完整的子系统，有自己的一套线程体系。核心是 **1 个核心音频处理线程** + N 个平台相关的**音频采集线程**。

---

## 一、核心：`audio-io: audio thread`

这是整个音频系统的心脏，全局只有**一个**。

### 在哪启动的

```
obs_reset_audio2()                          [libobs/obs.c]
  └─ obs_init_audio()
       └─ audio_output_open(&audio->audio, ai)  [libobs/media-io/audio-io.c]
            └─ pthread_create(&out->thread, NULL, audio_thread, out)
```

`audio_output_open` 不光创建了线程，还初始化了 6 个 mix（混音母线）、注册了核心回调 `audio_callback`（这个回调每次 Tick 都会被调用）、把 `set_audio_thread` 任务推入队列用于标记 TLS。

### 主循环长什么样

```c
static void *audio_thread(void *param)
{
    struct audio_output *audio = param;
    size_t rate = audio->info.samples_per_sec;       // 比如 48000
    uint64_t samples = 0;
    uint64_t start_time = os_gettime_ns();           // 启动时的墙上时间，只取一次！
    uint64_t prev_time = start_time;

    os_set_thread_name("audio-io: audio thread");

    while (os_event_try(audio->stop_event) == EAGAIN) {
        samples += 1024;                             // 每次迭代 +1024 帧
        uint64_t audio_time = start_time + audio_frames_to_ns(rate, samples);
        // 从 start_time 按采样点数算绝对时间，48kHz 下每次间隔 21.33ms

        os_sleepto_ns_fast(audio_time);              // 睡到那个绝对时刻

        input_and_output(audio, audio_time, prev_time);  // 干活
        prev_time = audio_time;
    }
    return NULL;
}
```

**关键点**：`audio_time` 始终从固定的 `start_time` 按采样点累积计算，而不是"当前时间 + 偏移"。所以即使某次迭代处理慢了，下一次的目标时刻不会被推迟——线程会少睡甚至不睡来追赶，不会累积漂移。

### 一次迭代 = 处理 1024 帧

一次迭代就是一个"音频 Tick"。48kHz 下每 Tick 约 21.33ms，每秒约 47 次。

每次迭代调用的 `input_and_output` 做了五件事：

1. **扫描活跃的 mix**：6 个 mix 中哪些有 consumer（编码器/监控挂载在上面）
2. **清零 mix buffer**：为后续累加混音做准备
3. **调用 `audio_callback`**：从各个 source 收集数据、渲染、混音
4. **钳位**：float 样本限制在 [-1.0, 1.0]，同时保留 unclamped 副本
5. **分发**：把混好的数据通过 `do_audio_output` 发给编码器和监控

---

## 二、6 个 mix × 8 个 channel

这两个数字经常同时出现，容易搞混。它们是两个正交的维度：

- **6 个 mix** = 6 条独立的混音母线（**纵轴**），对应 OBS 高级输出模式下的 6 条独立音轨。比如 track1 推流、track2 录完整混音、track3 只录麦克风……
- **8 个 channel** = 每条母线最多 8 个声道（**横轴**），对应 7.1 环绕声（FL/FR/FC/LFE/RL/RR/SL/SR）

每个 source 通过 `audio_mixers` 位掩码决定"我去哪几条 mix"。绝大多数场景只需要立体声（2 声道），但数组空间按最大值预留。

---

## 三、`input_and_output` 的核心：`audio_callback`

`audio_callback` 是 `input_and_output` 中调用的核心渲染回调。它的签名：

```c
bool audio_callback(void *param,
                    uint64_t start_ts_in,    // 本次 Tick 起始时间戳
                    uint64_t end_ts_in,      // 本次 Tick 结束时间戳
                    uint64_t *out_ts,        // [输出] 实际使用的时间戳
                    uint32_t mixers,         // 位掩码：哪些 mix 活跃
                    struct audio_output_data *mixes)  // [输入+输出] 待填充的 mix buffer
```

它的工作分八步：

### Step 1：准备阶段

```c
struct ts_info ts = {start_ts_in, end_ts_in};   // 本次 Tick 的时间窗口
deque_push_back(&buffered_timestamps, &ts);       // 推入队尾
deque_peek_front(&buffered_timestamps, &ts);      // 从队首取出——可能不一样！
```

`buffered_timestamps` 是一个双端队列。正常无缓冲时，推入再取出的是同一个。但如果之前有缓冲积累（源时间戳落后），队列里已经插入了更早的时间区间，从队首取出的 `ts` 就被"拉回"到了更早的时间。

### Step 2：构建渲染树

遍历所有 Scene/View 的 channel 以及全局音源（不在任何 Scene 里的纯音频源），构建两个列表：

- **`render_order`**：音频源的有序渲染列表，保证子源在父源之前渲染
- **`root_nodes`**：Top-level 源列表，只有它们直接参与混音（避免子源被重复累加）

### Step 3：渲染每个源

对 `render_order` 中每个 source 调用 `obs_source_audio_render()`。这里区分两种源：

**普通采集源**（麦克风、桌面音频）：从 `audio_input_buf`（deque）取 1024 帧 → 输出到 `audio_output_buf`

**复合源**（Scene、Group）：通过 `audio_submix` 累加所有子源的 `audio_output_buf` → 再走一遍 `obs_source_output_audio` → 这会经过复合源**自己身上挂的 filter 链**。

也就是说子源的 filter 处理单个音源（去噪、压缩），复合源的 filter 处理整体（总线压缩、总限幅），两层独立的 filter 链。

### Step 4：监控去重消音

当用户既做了音频监控，又添加了"音频输出采集"源，且它们指向同一个设备时，会产生回声。`should_silence_monitored_source` 检测这种情况，对重复采集的源调用 `clear_audio_output_buf` 清零——回声被消除了，但监控本身不受影响。

### Step 5：处理时间戳落后的源

```c
if (audio_buffering_maxed(audio)           // 缓冲已满
    && source->audio_ts != 0               // 源曾经产过数据
    && source->audio_ts < ts.start) {      // 源落后于混音窗口
    // 采集源：ignore_audio → 从 deque 弹掉落后帧
    // 渲染型音源（媒体文件等）：audio_pending = true → 跳过本次混音
}
```

为什么渲染型音源不能丢数据？因为媒体源的数据不是在 deque 里的——它是每次 Tick 调用 `audio_render` 回调**当场生成**的。丢音频帧会打乱 A/V 同步（声音和画面对不上），而且外部没法用 `deque_pop_front` 干预内部解码状态。只能标记 pending 等它自己追上来。

### Step 6：缓冲管理

检查是否还有源的时间戳落后。如果缓冲未满，调用 `add_audio_buffering` 把 `ts` 往前拉。注意：**本次加的缓冲要到下一次 Tick 的 peek 才生效**。

### Step 7：混音

只对 `root_nodes` 中不在 pending 状态的源做 `mix_audio`——把 `audio_output_buf` 逐声道累加到 mix buffer。

### Step 8：丢弃已消费数据

遍历所有源，弹出 input buffer 中已被混音覆盖的帧，推进 `source->audio_ts`，释放缓冲空间。

---

## 四、时间戳是怎么玩的

`source->audio_ts` 表示"这个源生产到了哪个时间点"，`ts.start` 表示"混音窗口从哪个时间点开始需要数据"。

```
正常:  source->audio_ts >= ts.start  → 数据够，正常渲染
落后:  source->audio_ts <  ts.start  → 数据不够
  
  缓冲未满 → add_audio_buffering 把 ts 往回拉 → 等源追上
  缓冲已满 → ignore_audio 强行丢数据 / pending 等待
```

这就是 OBS 动态缓冲的核心逻辑——先用延迟换稳定性，延迟到了上限再放弃。

---

## 五、平台采集线程一览

这些线程从操作系统音频 API 读取 PCM 数据，通过 `obs_source_output_audio()` 推入 source 的 `audio_input_buf`（deque），是**生产者**。

| 平台 | 采集线程 | 数据来源 |
|------|---------|---------|
| Windows | `win-wasapi: capture thread` (RTWQ 不可用时) | WASAPI IAudioCaptureClient |
| Windows | RTWQ 系统线程池 (RTWQ 可用时，Win10 1703+) | 同上 |
| Windows | `win-wasapi: reconnect thread` | 设备重连协调 |
| macOS | CoreAudio 内部 I/O 线程 (input_callback) | AudioUnit |
| macOS | `coreaudio: reconnect thread` (pthread) | 设备重连重试 |
| Linux | ALSA listen 线程 (pthread) | snd_pcm_readi |
| Linux | ALSA reopen 线程 (pthread) | 设备重打开 |
| Linux | OSS reader 线程 (pthread) | /dev/dsp |
| Linux | PulseAudio threaded mainloop (PA 内部) | pa_stream_read |
| OpenBSD | sndio 线程 (pthread, detached) | sio_read |

所有采集线程最终都调用同一个函数：`obs_source_output_audio()`。这就是 OBS 音频子系统的"多生产者—单消费者"架构：N 个采集线程 push → 每个 source 一个 deque → 1 个 audio-io 线程 pop、混音、分发。

---

## 六、数据流全景

```
┌──────────────────────────────────────────────────┐
│  采集线程（生产者）                                  │
│                                                    │
│  WASAPI / CoreAudio / ALSA / PulseAudio / OSS     │
│    │                                               │
│    │ obs_source_output_audio(source, &data)       │
│    │   → process_audio (重采样+balance+downmix)   │
│    │   → filter_async_audio (source 自己的 filter) │
│    │   → deque_push_back(audio_input_buf[ch])     │
│    ▼                                               │
│  source->audio_input_buf[0..7] (环形缓冲)          │
└────────────────────┬─────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────┐
│  audio-io 线程（消费者，每 21.33ms 一次 Tick）      │
│                                                    │
│  input_and_output()                                │
│    ├─ 扫描活跃 mix                                 │
│    ├─ 清零 mix buffer                              │
│    ├─ audio_callback()                             │
│    │    ├─ 构建 render_order / root_nodes          │
│    │    ├─ obs_source_audio_render (pop deque)    │
│    │    ├─ 缓冲管理                                │
│    │    ├─ mix_audio (累加到 mix buffer)           │
│    │    └─ discard_audio (释放已消费数据)          │
│    ├─ clamp [-1.0, 1.0]                            │
│    └─ do_audio_output()                            │
│         ├─ receive_audio (编码器)                   │
│         └─ on_audio_playback (监控)                 │
└──────────────────────────────────────────────────┘
```

---

## 七、两个容易混淆的概念

### `obs_core_data` 不只是音频数据

`audio_callback` 第一行：

```c
struct obs_core_data *data = &obs->data;
```

这个变量名 `data` 很抽象。实际上 `obs_core_data` 是 OBS 的**全局对象注册中心**——所有 source、output、encoder、display、service 的链表头都在里面。`audio_callback` 只用到其中两个字段：`first_audio_source`（全局音频源链表）和 `audio_sources_mutex`（保护它的锁）。其他的跟音频一点关系都没有。

### `audio_callback` 不是采集回调

它是**混音渲染**回调，不是采集回调。采集回调是各平台自己的（WASAPI `CaptureThread`、CoreAudio `input_callback` 等），它们负责往 deque 里 push 数据。`audio_callback` 是 audio-io 线程每 Tick 调用一次，负责从 deque 里 pop 数据、混音、分发。

---

## 小结

OBS 音频子系统的设计思路可以概括为：

- **一个核心时钟**（audio-io 线程，固定 1024 帧节奏，永不漂移）
- **多个生产者**（各平台采集线程，用 `obs_source_output_audio` 统一入口推数据）
- **每源一个 deque** 做缓冲解耦
- **6 个 mix** 支持多轨输出
- **动态缓冲**用延迟换稳定性（缓冲未满拉回 ts，缓冲满后才丢数据）
- **Filter 链**在 source 层面独立（单一源和复合源各管各的 filter 层）

理解了这些，OBS 音频的骨架也就清楚了。

---

*上一篇：[obs-studio 源码学习（二）——三大线程梳理]()*  
*下一篇：待续*
