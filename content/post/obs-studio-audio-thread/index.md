---
title: "OBS Studio 源码学习（三）：音频线程梳理"
description: "从 audio_output_open 到 input_and_output，完整梳理 OBS 音频子系统：核心线程主循环、音频树构建、obs_source_output_audio 全流程、动态缓冲机制、以及多平台采集线程的数据流全景。"
date: 2026-07-16
tags: ["OBS", "源码学习", "C++", "obs-studio"]
categories: ["OBS Studio源码学习"]
image: ""
---

前两篇分别梳理了主线程启动流程和图形渲染管线，这篇来看音频子系统。这篇文章是我在啃音频相关源码时的学习笔记，以初学者的第一视角记录了整个学习过程。全篇围绕"数据从哪来、经过谁、到哪去"这条主线，把采集线程、音频树构建、混音渲染、缓冲管理等细节逐一拆开。建议配合前两篇一起看。

---

## 一、核心线程的启动和主循环

### 1.1 启动链路

```
obs_reset_audio2()                                   [libobs/obs.c:1617]
  └─ obs_init_audio()                                [libobs/obs.c:919]
       ├─ 初始化 monitoring_mutex、task_mutex
       ├─ 将 set_audio_thread 任务推入 audio->tasks 队列
       ├─ 注册 "deduplication_changed" 信号
       └─ audio_output_open(&audio->audio, &ai)      [libobs/media-io/audio-io.c:353]
            ├─ bzalloc 分配 struct audio_output
            ├─ 从 info 拷贝配置（采样率/格式/声道）
            ├─ 计算 block_size（planar 下 = sizeof(float) = 4）
            ├─ 计算 channels（如立体声 = 2）
            ├─ 计算 planes（planar 下 = channels，交错下 = 1）
            ├─ 保存 input_callback = audio_callback    ← 核心回调
            ├─ pthread_mutex_init_recursive(&input_mutex)
            ├─ os_event_init(&stop_event, MANUAL)      ← 停止信号
            └─ pthread_create(&out->thread, NULL, audio_thread, out)  ← 🔴 线程诞生
```

几个初始化细节值得注意：
- `block_size` 是**一个采样点的单声道字节数**，OBS 内部用 `AUDIO_FORMAT_FLOAT_PLANAR`（声道分离的 float），所以 `block_size = sizeof(float) = 4`
- `planes` 在 planar 格式下等于声道数，意思是"声道数 = 独立缓冲区数"
- `set_audio_thread` 这个 task 推入队列后，会在第一次 `audio_callback` 末尾的 `execute_audio_tasks()` 里被执行，把 `THREAD_LOCAL is_audio_thread = true` 设好，之后 `obs_queue_task(OBS_TASK_AUDIO, ...)` 就能判断自己是否已经在音频线程上

### 1.2 主循环

```c
static void *audio_thread(void *param)
{
    struct audio_output *audio = param;
    size_t rate = audio->info.samples_per_sec;     // 比如 48000
    uint64_t samples = 0;
    uint64_t start_time = os_gettime_ns();         // 🔴 只取一次！所有后续时间从此推导
    uint64_t prev_time = start_time;

#ifdef _WIN32
    AvSetMmThreadCharacteristics(L"Audio", &unused);  // 设置 MMCSS Audio 优先级
#endif
    os_set_thread_name("audio-io: audio thread");

    while (os_event_try(audio->stop_event) == EAGAIN) {
        samples += AUDIO_OUTPUT_FRAMES;            // 每次 +1024
        uint64_t audio_time = start_time + audio_frames_to_ns(rate, samples);

        os_sleepto_ns_fast(audio_time);            // 精确睡到下一个帧边界

        profile_start(audio_thread_name);
        input_and_output(audio, audio_time, prev_time);
        prev_time = audio_time;
        profile_end(audio_thread_name);
        profile_reenable_thread();
    }
    return NULL;
}
```

**时间计算：**

```c
// audio-io.h:195-198
static inline uint64_t audio_frames_to_ns(size_t sample_rate, uint64_t frames)
{
    return util_mul_div64(frames, 1000000000ULL, sample_rate);
    //     = frames × 1,000,000,000 ÷ sample_rate
}
```

推算一下：

| 迭代 | samples | 计算 | audio_time - start_time | 
|------|---------|------|------------------------|
| 1 | 1024 | 1024 × 10⁹ ÷ 48000 | 21,333,333 ns (21.33 ms) |
| 2 | 2048 | 2048 × 10⁹ ÷ 48000 | 42,666,666 ns (42.67 ms) |
| 3 | 3072 | 3072 × 10⁹ ÷ 48000 | 64,000,000 ns (64.00 ms) |

一次 Tick 的时间预算：48kHz 下 21.33ms（每秒约 47 次），44.1kHz 下 23.22ms（每秒约 43 次）。

**不会累积漂移的原因**：`audio_time = start_time + 累积帧数换算的绝对时间`。`start_time` 只取了**一次**，所以即使某次 `input_and_output` 多花了 5ms，下一次的 `audio_time` 目标值完全不受影响——线程会少睡 5ms 来追赶。

### 1.3 `input_and_output` 逐行解析

```c
static void input_and_output(struct audio_output *audio,
                              uint64_t audio_time,    // 本次 Tick 结束时间戳
                              uint64_t prev_time)     // 上次 Tick 结束 = 本次开始
{
    size_t bytes = AUDIO_OUTPUT_FRAMES * audio->block_size;
    //              1024              × 4 = 4096 (单声道 planar float 下)
    struct audio_output_data data[MAX_AUDIO_MIXES];  // data[0..5]
    uint32_t active_mixes = 0;                       // 位掩码
    uint64_t new_ts = 0;
    bool success;

    memset(data, 0, sizeof(data));
```

`block_size = sizeof(float) × (交错? channels : 1)`。OBS 内部分配 planar 格式，每个声道有独立 buffer，所以 `block_size = 4`，`bytes = 4096`（1024 个 float = 4096 字节，正好是一个声道一个 Tick 的数据量）。后续代码里 `bytes` 用来做 memset 长度、clamp 长度等，都是按**单声道平面**算的。

```c
    // ① 扫描哪些 mix 有 consumer
    pthread_mutex_lock(&audio->input_mutex);
    for (size_t i = 0; i < MAX_AUDIO_MIXES; i++) {
        if (audio->mixes[i].inputs.num)            // mix[i] 的 inputs 列表非空
            active_mixes |= (1 << i);
    }
    pthread_mutex_unlock(&audio->input_mutex);
```

`audio->mixes[i].inputs` 是一个动态数组，元素是 `struct audio_input`——每个元素就是一个 consumer（编码器或监控）。这些 consumer 通过 `audio_output_connect()` 挂载上来。上锁是因为 `connect/disconnect` 可能在其他线程被调用。

```c
    // ② 清零 mix buffer，把 data 指针指向它们
    for (size_t mix_idx = 0; mix_idx < MAX_AUDIO_MIXES; mix_idx++) {
        struct audio_mix *mix = &audio->mixes[mix_idx];
        memset(mix->buffer, 0, sizeof(mix->buffer));
        // sizeof = 8声道 × 1024帧 × 4B = 32KB per mix

        for (size_t i = 0; i < audio->planes; i++)
            data[mix_idx].data[i] = mix->buffer[i];
        // data[mix].data[ch] 指向 mix->buffer[ch]，后面 audio_callback
        // 往 data[].data[] 写就是在操作 mix buffer
    }
```

`data` 是个输出参数——它告诉 `audio_callback` "往哪写"。调用前 buffer 全是 0，调用后就是混好的 1024 帧数据。data 指针只需赋值一次，后面一直有效，因为地址没变，变的是地址里的内容。

```c
    // ③ 调用 audio_callback —— 核心
    success = audio->input_cb(audio->input_param,
                               prev_time, audio_time, &new_ts, active_mixes, data);
    if (!success)
        return;   // buffering 未就绪，跳过后续 clamp 和分发

    // ④ 钳位
    clamp_audio_output(audio, bytes);

    // ⑤ 分发
    for (size_t i = 0; i < MAX_AUDIO_MIXES; i++)
        do_audio_output(audio, i, new_ts, AUDIO_OUTPUT_FRAMES);
}
```

`clamp_audio_output` 做的事：
- 先把 `mix->buffer` **整份复制**到 `mix->buffer_unclamped`（保留原始值）
- 再对 `mix->buffer` 做钳位：`val > 1.0 → 1.0`、`val < -1.0 → -1.0`、`isNaN → 0`
- consumer 可以通过 `allow_clipping` 标志选择用 raw 版还是 clamped 版

`do_audio_output` 做的事：
- 遍历该 mix 的所有 consumer（倒序遍历，安全删除）
- 根据 `allow_clipping` 选择 `buffer_unclamped` 还是 `buffer`
- 如果 consumer 要求的格式/采样率/声道跟 OBS 内部不一样，调 `audio_resampler_resample`
- 调用 `input->callback(param, mix_idx, &data)`，即 `receive_audio`（编码器）或 `on_audio_playback`（监控）

---

## 二、音频树是怎么构建的

这是整个音频系统里最容易搞混的地方。`audio_callback` 每 Tick 都要重建两个列表：`render_order` 和 `root_nodes`。

### 2.1 推入函数

**`push_audio_tree(parent, source, audio)`**（obs-audio.c:30-43）：

```c
static void push_audio_tree(obs_source_t *parent, obs_source_t *source, void *p)
{
    struct obs_core_audio *audio = p;

    if (da_find(audio->render_order, &source, 0) == DARRAY_INVALID) {
        obs_source_t *s = obs_source_get_ref(source);  // ref 计数 +1
        if (s) {
            da_push_back(audio->render_order, &s);
            s->audio_is_duplicated = false;
        }
    }
}
```

逻辑很简单：如果在 `render_order` 里没找到这个 source，就 ref 它并追加到列表末尾。ref 是为了防止 source 在使用期间被释放。

**`push_audio_tree2(parent, source, audio)`**（obs-audio.c:59-83）：

```c
static void push_audio_tree2(obs_source_t *parent, obs_source_t *source, void *p)
{
    if (obs_source_removed(source)) return;

    struct obs_core_audio *audio = p;
    size_t idx = da_find(audio->render_order, &source, 0);

    if (idx == DARRAY_INVALID) {
        // 第一次见到 → 加入 render_order
        obs_source_t *s = obs_source_get_ref(source);
        if (s) {
            da_push_back(audio->render_order, &s);
            s->audio_is_duplicated = false;
        }
    } else {
        // 已经见过了 → 这个 source 在音频树里出现了多次！
        obs_source_t *s = audio->render_order.array[idx];
        if (is_individual_audio_source(s) && !s->audio_is_duplicated) {
            da_push_back(audio->root_nodes, &source);  // 直接作为 root node 混音
            s->audio_is_duplicated = true;              // 标记，避免重复处理
        }
    }
}
```

`push_audio_tree2` 多了重复检测：如果同一个 source 在多条音频路径里出现了（比如同一麦克风源被放在两个 Scene 的不同 Group 里），只把它作为一个独立的 root_node 混入 mix，而不是在每个子路径上都混一次。这就是 `audio_is_duplicated` 标记的作用。

### 2.2 构建过程

在 `audio_callback` 里：

```c
da_resize(audio->render_order, 0);    // 元素数归零，保留内存
da_resize(audio->root_nodes, 0);

// A. 遍历所有 video mix（场景画布）
pthread_mutex_lock(&obs->video.mixes_mutex);
for (size_t j = 0; j < obs->video.mixes.num; j++) {
    struct obs_view *view = obs->video.mixes.array[j]->view;
    if (!view) continue;

    pthread_mutex_lock(&view->channels_mutex);

    // 遍历这个 view 的每个 channel（最多 32 个）
    for (uint32_t i = 0; i < MAX_CHANNELS; i++) {
        obs_source_t *source = view->channels[i];
        if (!source || !obs_source_active(source) || obs_source_removed(source))
            continue;

        // 如果这个 video mix 需要混音频 → 该 source 加入 root_nodes
        if (obs->video.mixes.array[j]->mix_audio)
            da_push_back(audio->root_nodes, &source);

        // 递归遍历子树 → push_audio_tree2 处理重复源
        obs_source_enum_active_tree(source, push_audio_tree2, audio);

        // 加到渲染列表
        push_audio_tree(NULL, source, audio);
    }
    pthread_mutex_unlock(&view->channels_mutex);
}
pthread_mutex_unlock(&obs->video.mixes_mutex);

// B. 遍历全局音频源（不在任何 Scene 里的纯音频源）
pthread_mutex_lock(&data->audio_sources_mutex);
source = data->first_audio_source;
while (source) {
    if (!obs_source_removed(source))
        push_audio_tree(NULL, source, audio);
    source = (struct obs_source *)source->next_audio_source;
}
pthread_mutex_unlock(&data->audio_sources_mutex);
```

### 2.3 举例说明

假设有这样的场景结构：

```
Scene "游戏画面":
  ├── 游戏画面捕获  (视频+音频)
  ├── 麦克风        (纯音频)
  └── Group "语音":
        ├── Discord  (纯音频)
        └── 麦克风   (同一个麦克风源！重复了)
```

遍历后得到：

```
                          render_order                        root_nodes
                          ────────────                        ──────────
obs_source_enum_active_tree(游戏画面, push_audio_tree2):
  【没有子源】

push_audio_tree(NULL, 游戏画面):
  加入 render_order          [0] 游戏画面                       [0] 游戏画面

obs_source_enum_active_tree(麦克风, push_audio_tree2):
  【没有子源】

push_audio_tree(NULL, 麦克风):
  加入 render_order          [1] 麦克风                         [1] 麦克风

obs_source_enum_active_tree(Group, push_audio_tree2):
  → 递归进入 Group 的子源:
    push_audio_tree2(Group, Discord): 第一次见 → 加入
    push_audio_tree2(Group, 麦克风):  已见过！  → 麦克风标记为 duplicated
                                                  → 推入 root_nodes

push_audio_tree(NULL, Group):
  加入 render_order          [2] Discord                        [2] Group
                             [3] 麦克风(重复标记)                [3] 麦克风(duplicated)
                             [4] Group
```

渲染时按 `render_order` 顺序，子源（Discord）先于父源（Group）渲染：

```
render_order[0]: 游戏画面  → obs_source_audio_render → output_buf 就绪
render_order[1]: 麦克风    → output_buf 就绪
render_order[2]: Discord   → output_buf 就绪
render_order[3]: 麦克风    → 重复标记，跳过（或由 root_nodes 直接混音）
render_order[4]: Group     → audio_submix 累加 Discord 的 output_buf
```

混音时只遍历 `root_nodes`：

```
root_nodes[0]: 游戏画面  → mix_audio 累加到 mix buffer
root_nodes[1]: 麦克风    → mix_audio 累加
root_nodes[2]: Group     → mix_audio 累加（Group 里已包含 Discord）
root_nodes[3]: 麦克风    → 重复源，audio_submix 自己处理
```

这样既保证了子源先渲染（父源才能累加），又避免了重复源被加两次。

---

## 三、从采集到入队：`obs_source_output_audio` 全流程

所有平台采集线程最终都调这个函数把数据推入 OBS 音频管线。

```c
void obs_source_output_audio(obs_source_t *source, const struct obs_source_audio *audio_in)
{
    // ① 拷贝并规范化：把超过声道数的 data[i] 设为 NULL
    //   （因为有些 filter 插件不检查声道数而是检查指针是否非零）
    struct obs_source_audio audio = *audio_in;
    size_t channels = get_audio_planes(audio.format, audio.speakers);
    for (size_t i = channels; i < MAX_AUDIO_CHANNELS; i++)
        audio.data[i] = NULL;

    // ② 格式处理：重采样、平衡、下混
    process_audio(source, &audio);

    // ③ 过 filter 链
    pthread_mutex_lock(&source->filter_mutex);
    output = filter_async_audio(source, &source->audio_data);

    if (output) {
        struct audio_data data;
        for (int i = 0; i < MAX_AV_PLANES; i++)
            data.data[i] = output->data[i];
        data.frames = output->frames;
        data.timestamp = output->timestamp;

        // ④ 入队
        pthread_mutex_lock(&source->audio_mutex);
        source_output_audio_data(source, &data);
        pthread_mutex_unlock(&source->audio_mutex);
    }
    pthread_mutex_unlock(&source->filter_mutex);
}
```

### 3.1 `process_audio` —— 格式统一

```c
static void process_audio(obs_source_t *source, const struct obs_source_audio *audio)
{
    // 如果源格式变了（比如设备换了采样率）→ 重建 resampler
    if (格式变了)
        reset_resampler(source, audio);

    if (有 resampler)
        audio_resampler_resample(source->resampler, output, &frames, &offset,
                                  audio->data, audio->frames);
        // 把源的格式（比如 44100Hz/16bit）转成 OBS 内部格式（48000Hz/float planar）
    else
        copy_audio_data(source, audio->data, audio->frames, audio->timestamp);
        // 格式一致，直接拷贝到 source->audio_data

    // 应用左右声道平衡
    process_audio_balancing(source, frames, source->balance);

    // 如果全局输出是单声道且源是立体声 → 下混
    downmix_to_mono_planar(source, frames);
}
```

### 3.2 `filter_async_audio` —— filter 链

```c
static struct obs_audio_data *filter_async_audio(obs_source_t *source,
                                                  struct obs_audio_data *in)
{
    // 倒序遍历 filter 数组
    for (size_t i = source->filters.num; i > 0; i--) {
        struct obs_source *filter = source->filters.array[i - 1];

        if (!filter->enabled)
            continue;

        if (filter->context.data && filter->info.filter_audio) {
            in = filter->info.filter_audio(filter->context.data, in);
            //  ↑ 链式调用：上一个 filter 的输出 → 下一个的输入
            if (!in)
                return NULL;   // filter 返回 NULL = 丢弃这帧
        }
    }
    return in;
}
```

这是一个典型的责任链模式。常见音频 filter：噪声明明抑制（RNNoise/Speex）、压限器、增益、均衡器（EQ）、VST 插件。

### 3.3 `source_output_audio_data` —— 入 deque

把处理完的数据推到 source 的 `audio_input_buf[ch]`（每声道一个 deque），同时更新 `source->audio_ts`：

```c
// 简化逻辑
for (size_t ch = 0; ch < channels; ch++)
    deque_push_back(&source->audio_input_buf[ch], data->data[ch], size);

source->audio_ts = data->timestamp + audio_frames_to_ns(rate, data->frames);
// 源的时间戳推进到 "这批数据的结束时间"
```

---

## 四、从 deque 取出渲染：`obs_source_audio_render` 全流程

audio-io 线程在 `audio_callback` 的 Step 3 中调这个函数。

```c
void obs_source_audio_render(obs_source_t *source, uint32_t mixers,
                              size_t channels, size_t sample_rate, size_t size)
{
    if (!source->audio_output_buf[0][0]) {
        source->audio_pending = true;
        return;
    }

    // 路径 1：自定义渲染（罕见）
    if (source->info.audio_render) {
        custom_audio_render(source, mixers, channels, sample_rate);
        return;
    }

    // 路径 2：复合源（Scene/Group），先累加子源
    if (source->info.audio_mix) {
        audio_submix(source, channels, sample_rate);
    }

    // 路径 3：标准处理（所有源都要走）
    if (!source->audio_ts) {
        source->audio_pending = true;
        return;
    }

    process_audio_source_tick(source, mixers, channels, sample_rate, size);
}
```

### 4.1 路径 1：`custom_audio_render` —— 渲染型音源（媒体文件等）

```c
static void custom_audio_render(obs_source_t *source, uint32_t mixers,
                                 size_t channels, size_t sample_rate)
{
    struct obs_source_audio_mix audio_data;

    // 把 output_buf 地址传给源
    for (mix...) for (ch...)
        audio_data.output[mix].data[ch] = source->audio_output_buf[mix][ch];

    // 清零活跃 mix
    if (active_mixes)
        memset(output_buf[mix][0], 0, 1024 * sizeof(float) * channels);

    // 调用源的 audio_render → 当场生成 1024 帧，回填 ts
    success = source->info.audio_render(source->context.data,
                                         &ts, &audio_data, mixers,
                                         channels, sample_rate);
    source->audio_ts = success ? ts : 0;
    source->audio_pending = !success;

    // 清零未使用的 mix
    for (mix...) if ((mixers & bit) == 0 || (audio_mixers & bit) == 0)
        memset(output_buf[mix][0], 0, ...);

    apply_audio_volume(source, mixers, channels, sample_rate);
}
```

关键：数据不是从 deque 取出来的——是**当场生成**的。源的 `audio_ts` 完全由源自己决定（解码器播放到哪了）。这就是为什么后面处理落后时，渲染型源不能 `ignore_audio`（没法 pop deque），只能标记 pending。

### 4.2 路径 2：`audio_submix` —— 复合源累加子源

```c
static void audio_submix(obs_source_t *source, size_t channels, size_t sample_rate)
{
    struct audio_output_data audio_data;
    struct obs_source_audio audio = {0};

    // 把 mix_buf 的地址暴露给 audio_mix 回调
    for (ch...) audio_data.data[ch] = source->audio_mix_buf[ch];

    // 清零
    memset(mix_buf[0], 0, 1024 * sizeof(float) * channels);

    // 调用源的 audio_mix 回调
    // Scene: 遍历所有子源，累加它们的 audio_output_buf
    // Group: 同 Scene
    // Transition: 按过渡进度混合两个子源
    success = source->info.audio_mix(source->context.data,
                                      &ts, &audio_data, channels, sample_rate);

    // 把累加结果组装成 obs_source_audio
    for (ch...) audio.data[ch] = (const uint8_t *)audio_data.data[ch];
    audio.samples_per_sec = sample_rate;
    audio.frames = 1024;
    audio.format = AUDIO_FORMAT_FLOAT_PLANAR;
    audio.timestamp = ts;

    // 🔴 关键：再次调用 obs_source_output_audio！
    // 这会过这个复合源自己的 filter 链，然后 push 到它自己的 deque
    obs_source_output_audio(source, &audio);
}
```

这一步的设计很精妙：复合源把子源的 output 累加后，当作"自己的原始音频"，再走一遍 `obs_source_output_audio` → `process_audio` → `filter_async_audio` → push deque。然后路径 3 的 `process_audio_source_tick` 再从 deque 取出来。这意味着复合源挂的 filter（如总线压缩）会作用于子源的混音结果。

### 4.3 路径 3：`process_audio_source_tick` —— 从 deque 取数据输出

```c
static void process_audio_source_tick(obs_source_t *source, uint32_t mixers,
                                       size_t channels, size_t sample_rate,
                                       size_t size)  // size = 4096
{
    pthread_mutex_lock(&source->audio_buf_mutex);

    // 检查第一声道有没有攒够 1024 帧
    if (source->audio_input_buf[0].size < size) {
        source->audio_pending = true;
        pthread_mutex_unlock(&source->audio_buf_mutex);
        return;   // 数据不够 → 标记 pending → 跳过
    }

    // peek（只读不弹）：从 deque 取 4096 字节 = 1024 个 float
    for (size_t ch = 0; ch < channels; ch++)
        deque_peek_front(&source->audio_input_buf[ch],
                         source->audio_output_buf[0][ch], size);
    //                          ↑ 先统一输出到 mix[0]
    pthread_mutex_unlock(&source->audio_buf_mutex);

    // 复制到其他 mix，或清零不用的
    for (size_t mix = 1; mix < MAX_AUDIO_MIXES; mix++) {
        if ((source->audio_mixers & bit) == 0 || (mixers & bit) == 0) {
            // 源不输出到这个 mix，或这个 mix 不活跃 → 清零
            memset(source->audio_output_buf[mix][0], 0, size * channels);
            // size * channels = 4096 × 2 = 8192（清掉所有声道）
        } else {
            for (ch...) memcpy(output_buf[mix][ch], output_buf[0][ch], size);
            // 逐声道拷贝 4096 字节
        }
    }

    // mix[0] 自身如果未使用也清零
    if ((audio_mixers & 1) == 0 || (mixers & 1) == 0)
        memset(output_buf[0][0], 0, size * channels);

    // 应用音量
    apply_audio_volume(source, mixers, channels, sample_rate);
    source->audio_pending = false;
}
```

这里 `size = 4096`（1024×4）是**单声道的字节数**。因为 OBS 用 planar 格式，每个声道的 buffer 都是独立的，所以检查只需要检查 `audio_input_buf[0].size >= 4096`——左声道有 1024 个 float 了，就认为其他声道也够了。

---

## 五、缓冲管理：时间戳怎么玩

### 5.1 核心概念

```
source->audio_ts  =  源"已经生产到哪个时间点"（现实墙上时间的纳秒）
ts.start          =  混音窗口"需要从哪个时间点开始的数据"
ts.end            =  混音窗口"需要到哪个时间点"
```

正常情况：`source->audio_ts >= ts.start`，源的数据覆盖了 ts 区间，直接渲染。

落后情况：`source->audio_ts < ts.start`，源的数据产得不够快。

### 5.2 动态缓冲：`add_audio_buffering`

```c
static void add_audio_buffering(struct obs_core_audio *audio, size_t sample_rate,
                                 struct ts_info *ts, uint64_t min_ts,
                                 const char *buffering_name)
{
    if (audio_buffering_maxed(audio))
        return;   // 缓冲已到上限，不再增加

    if (!audio->buffering_wait_ticks)
        audio->buffered_ts = ts->start;   // 记录基准点

    // 计算落后的帧数 → 需要几个 Tick 的缓冲
    offset = ts->start - min_ts;                  // 落后了多少纳秒
    frames = ns_to_audio_frames(sample_rate, offset);  // 折合多少帧
    ticks  = (frames + 1023) / 1024;             // 向上取整，几个 Tick

    // 如果超限，截断
    if (audio->total_buffering_ticks + ticks >= max) {
        ticks -= (total + ticks - max);
        audio->total_buffering_ticks = max;
    } else {
        audio->total_buffering_ticks += ticks;
    }

    // 往 buffered_timestamps 队首插入更早的 ts 区间
    new_ts.start = buffered_ts - audio_frames_to_ns(sample_rate,
                         buffering_wait_ticks * 1024);
    while (ticks--) {
        ++buffering_wait_ticks;
        new_ts.end = new_ts.start;
        new_ts.start = buffered_ts - audio_frames_to_ns(sample_rate,
                                  buffering_wait_ticks * 1024);
        deque_push_front(&audio->buffered_timestamps, &new_ts, sizeof(new_ts));
    }
    *ts = new_ts;  // 把当前 ts 也替换成更早的
}
```

举个具体例子。假设 48kHz，一个源落后了 10ms（约 480 帧）：

```
当前 ts = [12:00:00.100, 12:00:00.121)
落后源 min_ts = 12:00:00.090  ← 源的数据才产到 0.090

offset = 0.100 - 0.090 = 10ms = 10,000,000 ns
frames = 10,000,000 × 48000 ÷ 1,000,000,000 = 480
ticks  = (480 + 1023) / 1024 = 1   ← 需要 1 个 Tick 的缓冲

buffered_timestamps 变化前:    变化后（队首插入）:
┌───────────────────┐        ┌───────────────────┬───────────────────┐
│ [0.100, 0.121)    │   →    │ [0.079, 0.100)    │ [0.100, 0.121)    │
└───────────────────┘        └───────────────────┴───────────────────┘
                                  队首（下次 peek 取出）

ts 也被替换成 [0.079, 0.100)

下次 Tick:
  peek_front 取出 [0.079, 0.100)
  source->audio_ts = 0.090 → 0.090 >= 0.079 → 数据够了！正常渲染
```

`buffered_timestamps` 是一个双端队列：
- 正常时每次 Tick 往**队尾** push 当前 ts，peek **队首**取出的就是当前 ts
- 需要缓冲时往**队首** push 更早的 ts，peek 出来就变成了过去的 ts
- 等 `buffering_wait_ticks` 耗尽后，队列恢复为"push 一个 pop 一个"的正常模式

### 5.3 缓冲满了怎么办：`ignore_audio`

条件触发：

```c
if (audio_buffering_maxed(audio)       // total_buffering_ticks == max
    && source->audio_ts != 0           // 曾经产过数据
    && source->audio_ts < ts.start) {  // 确实落后了
```

对于**采集源**：

```c
static bool ignore_audio(obs_source_t *source, size_t channels,
                          size_t sample_rate, uint64_t start_ts)
{
    // 一批数据都没有 → 直接返回
    if (!source->audio_input_buf[0].size) return false;

    // 计算要丢弃多少帧
    size_t drop = (start_ts - source->audio_ts - 1) * sample_rate / 1e9 + 1;
    if (drop > deque_里的) drop = deque_里的;  // 最多全丢

    // 逐声道 pop
    for (ch...) deque_pop_front(&source->audio_input_buf[ch], NULL,
                                  drop * sizeof(float));

    // 推进时间戳
    source->audio_ts += drop * 1e9 / sample_rate;

    // 追上了？
    if (source->audio_ts >= start_ts)
        return true;   // → 重新渲染（rerender）
    return false;      // → 还是不够，标记 pending
}
```

对于**渲染型音源**（媒体文件）：

```c
// 没法 pop deque，因为数据不是在 deque 里的
// 只能标记 pending，跳过本次混音
source->audio_pending = true;
```

为什么不能丢？两个原因：
1. A/V 同步：媒体的音频帧和视频帧是配对好的，丢了音频声音和画面对不上
2. 内部状态：媒体源的数据是调用 `audio_render` 回调当场生成的，外部没法用 `deque_pop_front` 方式干预

渲染型源"追上来"的方式是：每次 Tick 都调 `audio_render`，即使数据被标记 pending 丢弃了，源内部的解码位置还是往前推进了。如果解码速度 ≥ 实时，终归能追上（audio_ts 会逐渐靠近 ts.start）。

但如果解码就是跟不上（CPU 太忙），就会一直 pending，用户听到的就是音频卡顿/断断续续。直到 CPU 负载降低或用户重启源。

### 5.4 `discard_audio` —— 消费已用数据

缓冲管理和丢弃落后数据之后，`audio_callback` 的 Step 8 遍历所有源，把 input buffer 里已经被混音覆盖的部分弹掉：

```c
static void discard_audio(struct obs_core_audio *audio, obs_source_t *source,
                           size_t channels, size_t sample_rate, struct ts_info *ts)
{
    // 渲染型音源 → 不管，它们没有 deque
    if (source->info.audio_render) {
        source->audio_ts = 0;
        return;
    }

    // ts 区间后的数据才保留，前面的全弹出
    if (source->audio_ts < ts->start) {
        // 源落后了，但缓冲还有，先不弹
        return;
    }

    // 计算需要保留的起始偏移
    size_t start_point = (source->audio_ts == ts->start)
        ? 0 : convert_time_to_frames(sample_rate, source->audio_ts - ts->start);

    size_t discard_size = (1024 - start_point) * sizeof(float);

    // 逐声道弹出已消费帧
    for (ch...) deque_pop_front(&source->audio_input_buf[ch], NULL, discard_size);

    source->audio_ts = ts->end;  // 推进到 ts 窗口末尾
}
```

---

## 六、监控去重消音

当用户既启用了音频监控（在耳机里听自己的声音），又添加了"音频输出采集"源（采集桌面声音），且两者指向同一设备时，会产生回声。

```c
static bool should_silence_monitored_source(obs_source_t *source,
                                              struct obs_core_audio *audio)
{
    obs_source_t *dup_src = audio->monitoring_duplicating_source;

    if (!dup_src || !active)        return false;  // 没有重复采集源
    if (dup_src->monitoring_type == MONITOR_ONLY)  return false;

    // 输出采集源没静音且推子不为零
    bool unmuted = !dup_src->muted && volume > 0;

    if (unmuted) {
        // 这个 source 同时送往了"输出"和"监控"
        if (source->monitoring_type == MONITOR_AND_OUTPUT
            && source != dup_src) {
            return true;  // 需要消音
        }
    }
    return false;
}
```

消音动作：

```c
clear_audio_output_buf(source, audio);
// → memset(source->audio_output_buf[mix][ch], 0, 1024 * sizeof(float))
// → 输出采集源"听不到"这个 source 了，避免回声
// → 但监控本身不受影响（监控走的是 audio_capture_callback，不经过 output_buf）
```

---

## 七、平台采集线程细节

### 7.1 Windows WASAPI

两个线程（RTWQ 不可用时）：

**采集线程** `WASAPISource::CaptureThread`：

```cpp
// 创建（构造函数中）
captureThread = CreateThread(nullptr, 0, WASAPISource::CaptureThread,
                              this, 0, nullptr);
os_set_thread_name("win-wasapi: capture thread");
AvSetMmThreadCharacteristics(L"Audio", &unused);  // MMCSS Audio 优先级

// 主循环等待 4 个信号
HANDLE sigs[] = {exitSignal, stopSignal, receiveSignal, restartSignal};
//  - exitSignal:   析构时触发
//  - stopSignal:   场景切换/停用时触发
//  - receiveSignal: WASAPI 通知有音频数据
//  - restartSignal: 用户换了设备

// 收到 receiveSignal 时：
ProcessCaptureData():
    while (capture->GetNextPacketSize(&captureSize)) {
        capture->GetBuffer(&buffer, &frames, &flags, &pos, &ts);
        obs_source_audio data = {};
        data.data[0] = buffer;
        data.frames = frames;
        data.timestamp = useDeviceTiming ? ts * 100 : os_gettime_ns();
        obs_source_output_audio(source, &data);
        capture->ReleaseBuffer(frames);
    }
```

**重连线程** `WASAPISource::ReconnectThread`：

```cpp
// 创建（Activate 时）
reconnectThread = CreateThread(nullptr, 0, WASAPISource::ReconnectThread,
                                this, 0, nullptr);
os_set_thread_name("win-wasapi: reconnect thread");

// 主循环
while (!exit) {
    WaitForMultipleObjects({reconnectExitSignal, reconnectSignal});
    if (收到 reconnectSignal && reconnectDuration > 0)
        Wait(冷却时间);
    source->Start();  // 触发重新初始化
}
```

RTWQ（Real-Time Work Queue）可用时（Win10 1703+），采集线程不创建，改用 `RtwqPutWaitingWorkItem` 把工作提交到系统工作队列。

### 7.2 macOS CoreAudio

CoreAudio 使用 `AudioUnit` 的 I/O 回调，**不创建独立的采集线程**——音频数据在 CoreAudio 内部的实时 I/O 线程上收到：

```c
static OSStatus input_callback(void *data, ...) {
    AudioUnitRender(ca->unit, ..., ca->buf_list);   // 从设备取数据
    for (i..) audio.data[i] = ca->buf_list->mBuffers[i].mData;
    audio.timestamp = AudioConvertHostTimeToNanos(ts_data->mHostTime);
    obs_source_output_audio(ca->source, &audio);     // 推入管线
}
```

重连线程：

```c
// 设备断开/格式变化时 → coreaudio_begin_reconnect
static void *reconnect_thread(void *param) {
    while (os_event_timedwait(exit_event, retry_time) == ETIMEDOUT)
        if (coreaudio_init(ca)) break;   // 重试成功
}
```

### 7.3 Linux PulseAudio

PulseAudio 使用 `pa_threaded_mainloop`——PulseAudio 库内部创建一个事件循环线程。OBS 不创建 pthread，而是注册回调：

```c
// pulse_init() 时
pulse_mainloop = pa_threaded_mainloop_new();
pa_threaded_mainloop_start(pulse_mainloop);   // ← PA 内部线程启动

// 注册读取回调
pa_stream_set_read_callback(stream, pulse_stream_read, data);

// PA 线程在有数据时调用此回调
static void pulse_stream_read(pa_stream *p, size_t nbytes, void *userdata) {
    pa_stream_peek(data->stream, &frames, &bytes);
    obs_source_output_audio(data->source, &out);  // 直接推入
    pa_stream_drop(data->stream);
}
```

---

## 八、完整数据流（从采集到编码/监控）

把前面所有环节串起来，整个音频数据的流动全景如下：

```
┌─────────────────────────────────────────────────────────┐
│ 采集线程（各平台实现）                                     │
│                                                         │
│ WASAPI CaptureThread / CoreAudio input_callback         │
│ / PulseAudio stream_read / ALSA listen / OSS reader    │
│   │                                                     │
│   │ obs_source_output_audio(source, &raw_audio)        │
│   │   ├── process_audio(source, &audio)                 │
│   │   │     ├── reset_resampler (如果格式变了)           │
│   │   │     ├── audio_resampler_resample (格式转换)      │
│   │   │     ├── process_audio_balancing (左右平衡)      │
│   │   │     └── downmix_to_mono_planar (强制单声道)     │
│   │   ├── filter_async_audio(source, &audio)            │
│   │   │     └── for each filter:                        │
│   │   │           filter->info.filter_audio(in) → out   │
│   │   └── source_output_audio_data(source, &data)       │
│   │         └── deque_push_back(audio_input_buf[ch],    │
│   │                               data, frames*4)       │
│   ▼                                                     │
│ source->audio_input_buf[0..7]  (每声道一个 deque)       │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│ audio-io 线程 (每 Tick = 1024 帧 = ~21.33ms)             │
│                                                         │
│ input_and_output(audio, audio_time, prev_time)          │
│   │                                                     │
│   ├─ 扫描活跃 mix (哪些有 consumer)                      │
│   ├─ 清零 mix->buffer (32KB × 6)                        │
│   │                                                     │
│   ├─ audio_callback(param, prev_time, audio_time, ...) │
│   │   │                                                 │
│   │   ├─ buffered_timestamps: push 当前 ts, peek 队首   │
│   │   │   (有缓冲时 peek 出来的是更早的 ts)              │
│   │   │                                                 │
│   │   ├─ 构建渲染树                                      │
│   │   │   ├─ 遍历 scene/view channel → 递归枚举子源     │
│   │   │   ├─ push_audio_tree2 (处理重复源)              │
│   │   │   ├─ 遍历全局音源链表                            │
│   │   │   → render_order (有序) + root_nodes (顶层)     │
│   │   │                                                 │
│   │   ├─ 渲染每个 source (按 render_order 顺序)          │
│   │   │   ├─ obs_source_audio_render:                   │
│   │   │   │   ├─ 渲染型音源 → custom_audio_render       │
│   │   │   │   │     └─ source->info.audio_render()      │
│   │   │   │   │        当场生成 1024 帧                  │
│   │   │   │   ├─ 复合源 → audio_submix                  │
│   │   │   │   │     └─ 累加子源 output_buf              │
│   │   │   │   │     └─ obs_source_output_audio(self)   │
│   │   │   │   │        过自己的 filter → push deque     │
│   │   │   │   └─ → process_audio_source_tick            │
│   │   │   │         deque_peek_front → output_buf       │
│   │   │   │         memcpy 到各 mix → apply_volume      │
│   │   │   │                                             │
│   │   │   ├─ 监控去重消音 (should_silence)              │
│   │   │   └─ 落后检测 (buffering_maxed && audio_ts落后) │
│   │   │       ├─ 采集源 → ignore_audio (pop deque)      │
│   │   │       └─ 渲染源 → pending (跳过混音)            │
│   │   │                                                 │
│   │   ├─ 缓冲管理                                       │
│   │   │   ├─ calc_min_ts (找最落后的源)                 │
│   │   │   └─ add_audio_buffering (往队首插旧 ts)        │
│   │   │                                                 │
│   │   ├─ mix_audio (root_nodes → 累加到 mix buffer)     │
│   │   ├─ discard_audio (弹出已消费数据)                  │
│   │   └─ execute_audio_tasks (OBS_TASK_AUDIO 任务)     │
│   │                                                     │
│   ├─ clamp_audio_output ([-1.0, 1.0])                  │
│   └─ do_audio_output (逐 consumer 分发)                 │
│       ├─ 选择 clamped / unclamped buffer                │
│       ├─ 需要时重采样                                   │
│       └─ input->callback(param, mix_idx, &data)        │
│           ├─ receive_audio(encoder) → buffer_audio      │
│           │     → deque_push_back → do_encode           │
│           └─ on_audio_playback(monitor) → 平台播放      │
└─────────────────────────────────────────────────────────┘
```

---

## 九、一些容易踩的命名坑

- **`obs_core_data`** 不是"音频数据"——它是全局对象注册中心，all sources/outputs/encoders/displays 的链表头都在里面。`audio_callback` 只用了 `first_audio_source` 和 `audio_sources_mutex`
- **`audio_callback`** 不是采集回调——采集回调是各平台的 WASAPI/CoreAudio/PulseAudio 回调。`audio_callback` 是混音引擎的入口
- **`input_callback`** 的命名是从 `audio_output` 模块视角看的——"你给我输入数据"，所以叫 input。对 OBS 整体来说它是混音渲染核心
- **`audio_size = 1024 × sizeof(float)`** 是**单声道的字节数**，因为 OBS 内部用 planar 格式，每个声道独立 buffer
- **`block_size`** 也是单声道单采样点字节数，planar float 下 = 4

---

## 小结

读音频子系统的这几天，最大的感受是和视频管线在思路上高度对称——都有独立的核心时钟线程、都有按需消费的机制、都用 deque 做缓冲解耦。OBS 音频子系统的设计可以用几个词概括：

- **一个时钟**：audio-io 线程，固定 1024 帧节奏，从 `start_time` 绝对推导，永不漂移
- **N 个生产者**：各平台采集线程，通过 `obs_source_output_audio` 统一入口推 deque
- **每源一个 deque**：环形双端队列解耦生产者和消费者
- **两级缓冲**：动态缓冲用延迟换稳定性（缓冲未满拉回 ts → 缓冲满后才丢数据 / pending）
- **两层 filter**：子源 filter（单路处理）+ 复合源 filter（总线处理），各管各的
- **6 个 mix**：多轨输出的基础，每个 source 的 `audio_mixers` 决定去哪些轨道

理解了这些，再去看 WASAPI/PulseAudio/CoreAudio 的具体实现、音频编码器、监控输出，就都有清晰的上下文了。

---

学习过程中参考了 OBS Studio 官方源码仓库，感谢开源社区。如有理解偏差，欢迎指正交流。
