---
title: "OBS Studio 源码学习（三）：音频线程深度梳理"
description: "从 audio_output_open 到 input_and_output，再到 audio_callback 内部六阶段的逐帧拆解：覆盖核心线程主循环、buffered_timestamps 时钟队列、音频树构建、三路径单源渲染、缓冲管理完整机制、采集入队全流程、以及多平台采集线程的数据流全景。"
date: 2026-07-23
tags: ["OBS", "源码学习", "C++", "obs-studio", "音频"]
categories: ["OBS Studio源码学习"]
image: ""
---

前两篇分别梳理了主线程启动流程和图形渲染管线，这篇来看音频子系统。文章以"数据从哪来、经过谁、到哪去"为主线，先建立整体框架（线程启动、主循环、四阶段处理模型），再逐层深入 `audio_callback` 内部的六个阶段、缓冲管理的完整机制、以及各路数据如何汇入和流出。全篇综合了两轮源码阅读的笔记，建议配合前两篇一起看。

---

## 一、核心线程的启动与主循环

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

- `block_size` 是**一个采样点的单声道字节数**。OBS 内部用 `AUDIO_FORMAT_FLOAT_PLANAR`（声道分离的 float），所以 `block_size = sizeof(float) = 4`
- `planes` 在 planar 格式下等于声道数——"声道数 = 独立缓冲区数"
- `set_audio_thread` 这个 task 推入队列后，在第一次 `audio_callback` 末尾的 `execute_audio_tasks()` 里被执行，把 `THREAD_LOCAL is_audio_thread = true` 设好。之后 `obs_queue_task(OBS_TASK_AUDIO, ...)` 就能判断自己是否已经在音频线程上

### 1.2 主循环：永不漂移的时钟

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

**不会累积漂移的原因**：`audio_time = start_time + 累积帧数换算的绝对时间`。`start_time` 只取了**一次**，所以即使某次 `input_and_output` 多花了 5ms，下一次的 `audio_time` 目标值完全不受影响——线程会少睡 5ms 来追赶。这是音频时钟设计的精髓：用墙上钟的绝对时间做目标，而非"上次结束 + 固定间隔"的相对时间。

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

参数传递的细节值得注意：`prev_time` 对应 `start_ts_in`（窗口起始），`audio_time` 对应 `end_ts_in`（窗口结束）。两者由音频线程的墙上钟绝对推导，`end_ts_in - start_ts_in` 永远等于 1024 采样点对应时长（48kHz 下 21.33ms）。`new_ts` 是输出参数——可能被缓冲逻辑修正为更早的时间戳，传给下游消费者。

### 1.4 四阶段处理框架概览

`input_and_output` 内部做的事情可以清晰地分成 A/B/C/D 四个阶段，这个框架贯穿整个音频线程：

| 阶段 | 名称 | 核心动作 | 位置 |
|------|------|---------|------|
| **A** | 渲染（Render） | 构建渲染树 → 逐源产出 `output_buf` → 累加到 mix buffer → 清理已消费数据 | `audio_callback()` |
| **B** | 缓冲管理（Buffering） | `calc_min_ts()` 找最早时间戳 → 如有滞后源，回退处理窗口 → 上限到了就丢帧 | `audio_callback()` 内部 |
| **C** | 钳制（Clamp） | 混音值钳到 [-1.0, 1.0]，NaN→0，同时保留 unclamped 副本 | `clamp_audio_output()` |
| **D** | 分发（Output） | 混音完成的 mix buffer 广播给所有订阅者（编码器/raw output/插件） | `do_audio_output()` |

阶段 A 涵盖音频树构建、逐源渲染、混音和清理，是 `audio_callback` 的核心——下一节我们将把它拆成六个子阶段逐一分析。阶段 B 也在 `audio_callback` 内部完成。阶段 C 和 D 是 `audio_callback` 返回后由 `input_and_output` 调用的。

这里先快速过一下阶段 C 和 D 的要点（第八节还会展开）：

**阶段 C 的 NaN 处理：**

```c
float val = *mix_data;
val = (val == val) ? val : 0.0f;     // NaN → 0（NaN 不等于自己，这个特性很巧妙）
val = (val > 1.0f)  ? 1.0f : val;
val = (val < -1.0f) ? -1.0f : val;
```

**阶段 D 的三类消费者：**

| 消费者类型 | 注册位置 | 回调 | 用途 |
|-----------|----------|------|------|
| 音频编码器 | `obs-encoder.c:375` | `receive_audio()` | AAC/Opus 编码 → 推流/录制 |
| 原始音频输出 | `obs-output.c:2453` | `default_raw_audio_callback()` | FFmpeg muxer、重放缓冲 |
| 外部插件/用户 | `obs.c:3165`（公共 API） | 自定义 | 第三方拿最终混音数据 |

本质是一个**发布-订阅**模式：前面 A/B/C 阶段把数据准备好，阶段 D 通知所有订阅者来取。

---

## 二、audio_callback 深度解析：参数、队列与六阶段

`audio_callback`（`libobs/obs-audio.c:555`）是整个音频子系统的绝对心脏。每 Tick 被调用一次，参数由 `input_and_output` 传入，内部完成构建、渲染、缓冲、混音、清理五个阶段——加上"落后源硬兜底"这个异常路径，共六个阶段。本节先把函数签名和时钟核心 `buffered_timestamps` 讲清楚，再逐一拆解六个阶段。

### 2.1 函数签名与参数来源

```c
bool audio_callback(void *param,
                    uint64_t start_ts_in,   // 窗口起始（墙上钟算出的"理想"）
                    uint64_t end_ts_in,     // 窗口结束
                    uint64_t *out_ts,       // 输出：修正后的窗口起始
                    uint32_t mixers,        // 活跃 mix 位掩码
                    struct audio_output_data *mixes)  // 往这写 = 往 mix buffer 写
```

参数来自 `input_and_output` [audio-io.c:193]：

```c
// audio_thread 主循环中
samples += 1024;
audio_time = start_time + audio_frames_to_ns(rate, samples);
// prev_time = 上一帧的 audio_time

input_and_output(audio, audio_time, prev_time);
    → audio->input_cb(param, prev_time, audio_time, &new_ts, active_mixes, data);
```

`start_ts_in` 和 `end_ts_in` 由音频线程的墙上钟绝对推导，永不漂移。`end_ts_in - start_ts_in` 永远等于 1024 采样点对应时长（48kHz 下 21.33ms）。

`out_ts` 是输出参数，可能被缓冲逻辑修正为更早的时间戳，传给下游消费者。如果没有缓冲介入，`*out_ts = start_ts_in`。

### 2.2 `buffered_timestamps`：整个回调的时钟核心

这是理解 `audio_callback` 最关键的数据结构。类型是 `deque`（双端队列），存储在 `obs_core_audio.buffered_timestamps` [obs-internal.h:449]。

每 Tick 三步操作：

```
① deque_push_back(&buffered_timestamps, &ts);   // 当前 ts 存队尾
② deque_peek_front(&buffered_timestamps, &ts);  // 队首取实际用的 ts  ★ ts 可能被覆盖
③ deque_pop_front(&buffered_timestamps);        // tick 结束时弹掉队首
```

**正常时（无缓冲）**：队列只有 1 条记录。push 什么 peek 就是什么。ts 不变。

**缓冲时**：`add_audio_buffering` 或 `set_fixed_audio_buffering` 往**队首** push 了额外的旧时间区间。peek 取出的就是被拉回的老区间，ts 被覆盖。

队列严格升序：`push_back` 写晚的，`push_front` 写早的——队首永远是最老的时间点。这个设计非常简洁：正常时队列自平衡（push 一个 pop 一个），缓冲时只需要往队首塞旧区间，缓冲结束自动恢复。

下面六个阶段全部围绕这个队列展开。先记住这个三步走，后面每个阶段都会涉及。

### 2.3 阶段一：构建渲染树（Build）

每帧的第一件事，是找出"本帧有哪些源需要处理音频"，产出两个列表：

```c
da_resize(render_order, 0);
da_resize(root_nodes, 0);

// 路径 A：从画面反推
for (每个 video mix)
    for (每个 view->channels[i])
        source = view->channels[i];

        if (mix->mix_audio)
            root_nodes ← source;           // 顶层源标记为混音根

        obs_source_enum_active_tree(source, push_audio_tree2);
        // 深搜子树 → 去重检测 → 重复源升级为 root_nodes

        push_audio_tree(NULL, source);     // 加入 render_order

// 路径 B：兜底纯音频源
for (first_audio_source 链表)
    push_audio_tree(NULL, source);         // 已在 render_order 里的跳过
```

**路径 A** 遍历所有 video mix 的 view channel——"画面上有哪些源在活动？它们的音频也拉进来"。这里容易混淆的是 view channel 和音频声道的区别：view channel 是源槽位（一个 view 有 64 个输入口），音频声道是物理声道数（立体声=2）。两者完全正交。

真正让音频"分流到不同输出"的，不是 view channel，而是每个 source 上的 `audio_mixers` 位掩码——控制这个 source 的音频数据拷贝到第几个 audio mix。

**路径 B** 是兜底：有些纯音频源没有被挂在任何 Scene 里（比如"音频输出采集"），但它们注册到了 `first_audio_source` 全局链表中，需要在这里捞起来。已经在 render_order 里的会跳过。

结果：

```
render_order[] = 所有需要本帧渲染音频的源（叶子在前，父源在后）
root_nodes[]   = 需要参与最终混音的顶层源（scene + 被去重升级的重复源）
```

去重逻辑在第三节详述。

### 2.4 阶段二：逐源渲染（Render）

```c
for (render_order 中每个 source) {
    obs_source_audio_render(source, ...);  // 产出 1024 采样点到输出缓冲

    if (should_silence_monitored(source))  // 监控去重消音
        clear_audio_output_buf(source);
}
```

每个 source 走三条渲染路径之一（第四节详述）：

| 源类型 | 函数 | 数据来源 |
|--------|------|----------|
| 有 `audio_render` 回调（scene/媒体）| `custom_audio_render()` | 回调当场生成或累加子源 |
| 有 `audio_mix` 回调（submix） | `audio_submix()` | 累加子源 output_buf |
| 普通采集源（麦克风等） | `process_audio_source_tick()` | peek `audio_input_buf` deque |

叶子源先于父源渲染——因为 Scene/Group 需要子源的 output_buf 已经就绪才能累加。

### 2.5 阶段三：缓冲管理判断（Buffer）

```c
calc_min_ts(..., &min_ts);   // 找最慢源的 audio_ts

if (fixed_buffer)
    set_fixed_audio_buffering();          // 预先垫满
else if (min_ts < ts.start)
    add_audio_buffering();                // 按需追加：往队首塞旧区间
```

`calc_min_ts` 遍历所有源（跳过 pending 源），找最小的 `audio_ts`。如果最慢的源的 `audio_ts` 比当前处理窗口的 `ts.start` 还小，说明有源数据没跟上——触发缓冲。

固定缓冲和动态缓冲的区别只在于**缓冲量**：固定一次性垫满 `max_buffering_ticks`，动态按落后量仅垫需要的 tick 数。底层 push_front 逻辑完全一致。缓冲的具体计算逻辑在第六节展开。

### 2.6 阶段四：落后源硬兜底

当缓冲量已经达到上限（`audio_buffering_maxed`），但仍有源落后时，进入硬兜底：

```c
if (audio_buffering_maxed(audio) && source->audio_ts != 0 && source->audio_ts < ts.start) {
    if (source->info.audio_render) {
        source->audio_pending = true;     // 渲染型源：只能等
    } else {
        ignore_audio(source, ...);        // 采集源：pop deque 丢数据，追平
        if (rerender)
            obs_source_audio_render(src, ...);  // 追上了重新渲染
    }
}
```

关键区别：

- **有 `audio_render` 回调的源**（scene、媒体文件）：数据当场生成，没有 deque 可以 pop。只能标记 pending，等待源自己追赶。媒体源的音频帧和视频帧是配对好的，丢音频帧会破坏 A/V 同步。
- **无 `audio_render` 的采集源**（麦克风等）：有 deque。`ignore_audio` 计算落后量 → pop 掉落后采样 → 时间戳前移。追上了返回 true 触发 re-render。

### 2.7 阶段五：混音（Mix）

```c
if (!audio->buffering_wait_ticks) {          // ★ 只有缓冲等待结束时才混
    for (root_nodes 中每个 source) {
        if (source->audio_pending) continue;  // 数据不够，跳过

        mix_audio(mixes, source, ...);
        // mixes[mix].data[ch][i] += source->output_buf[mix][ch][i]
    }
}
```

纯浮点累加。`mix_audio` 内部按时间戳偏移 `start_point` 对齐，不会把过期数据混入。

**关键门控**：`buffering_wait_ticks` 不为零时整个混音阶段跳过——此时缓冲正在"消耗"旧的时间区间，让落后源有时间追赶。数据的生产和消费仍在进行，只是不输出。

### 2.8 阶段六：清理（Cleanup）

```c
discard_audio(audio, source, ...);    // 逐源 pop 已消费的 deque 数据
release_audio_sources(audio);         // 释放 render_order refs
deque_pop_front(&buffered_timestamps); // 弹掉队首 ← 与 2.2 节的第③步呼应
```

`discard_audio` 的关键行为（按 `audio_ts` 与 `ts` 区间的关系分三种情况）：

```
正常路径：audio_ts >= ts.end    → pop 已消费帧，推进 audio_ts = ts.end
落后路径：audio_ts < ts.start   → return（不弹！不推进！）
渲染型源：audio_ts = 0          → return（不管，它们没有 deque）
```

落后的源音频线程不碰它，时间戳原地不动。下次 `calc_min_ts` 里它是检测缓冲触发的关键信号。这个"不弹落后源"的设计非常关键——它让 `audio_ts` 成为一个天然的"进度指针"，缓冲管理完全靠读这个指针来判断谁落后了。

---

## 三、音频树构建详解

这是整个音频系统里最容易搞混的地方。`audio_callback` 每 Tick 都要重建两个列表：`render_order` 和 `root_nodes`。

### 3.1 推入函数

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

**"独立音频源"的判断标准**（只有满足全部三个条件才升级为 root_node）：

```c
type == OBS_SOURCE_TYPE_INPUT
&& (output_flags & OBS_SOURCE_AUDIO)
&& !(output_flags & OBS_SOURCE_COMPOSITE)
```

即：是输入源、有音频能力、不是复合源（非 scene/group/transition）。复合源（如 Scene）自己出现在多个地方时不会升级——它们的子源已经由各自的 `push_audio_tree2` 处理过了。

### 3.2 构建过程

在 `audio_callback` 里（对应 2.3 节的阶段一）：

```c
da_resize(audio->render_order, 0);    // 元素数归零，保留内存
da_resize(audio->root_nodes, 0);

// A. 遍历所有 video mix（场景画布）
pthread_mutex_lock(&obs->video.mixes_mutex);
for (size_t j = 0; j < obs->video.mixes.num; j++) {
    struct obs_view *view = obs->video.mixes.array[j]->view;
    if (!view) continue;

    pthread_mutex_lock(&view->channels_mutex);

    // 遍历这个 view 的每个 channel（最多 64 个）
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

### 3.3 实例推演

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

## 四、单源渲染三路径：`obs_source_audio_render`

`obs-source.c:5455`。这是阶段二（2.4 节）中每个源被调用的入口，根据源类型分发到三条路径。

### 4.1 入口分发逻辑

```c
void obs_source_audio_render(source, mixers, channels, sample_rate, size)
{
    // 第零步：output_buf 都没分配？
    if (!source->audio_output_buf[0][0]) {
        source->audio_pending = true;      // 标记无效
        return;
    }

    // 门 1：自定义渲染（scene、媒体文件等）
    if (source->info.audio_render) {
        custom_audio_render(source, ...);  // 全权委托，OBS 不插手
        return;
    }

    // 门 2：子混音（submix 型 composite source，当前无人使用）
    if (source->info.audio_mix) {
        audio_submix(source, ...);
    }

    // 门 3：标准流程（普通采集源：麦克风等）
    if (!source->audio_ts) {
        source->audio_pending = true;
        return;
    }
    process_audio_source_tick(source, ...);
}
```

三条路径的真相：

| | `audio_render` | `audio_mix` | 无回调（叶子源） |
|---|---|---|---|
| 注册者 | Scene, Transition, Slideshow | **零个**（预留） | 麦克风, 桌面音频, 游戏采集 |
| 路径 | 门 1 → return | 门 2 → 往下掉进门 3 | 直接门 3 |
| 做什么 | 全权掌控：遍历子源+对时+自定义混音 | 预留：只累加子源 | peek deque + copy + 上音量 |

**容器源 vs 叶子源：**

- **容器源**：有子 source。Scene 有 scene item → child source，Transition 有 A/B 两个子源。必须实现 `audio_render`，因为 OBS 不知道你怎么排列组合子 source。
- **叶子源**：无子 source。数据从外部来（驱动回调、解码器），通过 `obs_source_output_audio()` 写入 `audio_input_buf`。不需要任何回调，OBS 默认流程全自动。

### 4.2 门一：`custom_audio_render` — 渲染型源

`obs-source.c:5338-5373`

```c
static void custom_audio_render(obs_source_t *source,
                                 uint32_t mixers,        // 活跃 mix 位掩码
                                 size_t channels,        // 声道数
                                 size_t sample_rate)     // 采样率
{
    struct obs_source_audio_mix audio_data;  // 打包 buffer 指针传给回调
    bool success;
    uint64_t ts;                             // 回调返回的时间戳

    // ── ① 绑指针 + 清活跃 mix ──
    for (size_t mix = 0; mix < 6; mix++) {
        // 各声道 output_buf 的地址填进 audio_data
        for (size_t ch = 0; ch < channels; ch++)
            audio_data.output[mix].data[ch] = source->audio_output_buf[mix][ch];

        // 这个 mix 既被源勾选、又被全局设为活跃 → 先清零
        // 原因：scene 类回调用累加模式（+=），必须从 0 开始
        if ((source->audio_mixers & mixers & (1 << mix)) != 0) {
            memset(source->audio_output_buf[mix][0], 0,
                   sizeof(float) * 1024 * channels);
        }
    }

    // ── ② ★ 调源自己的回调 ──
    // 回调在 audio_data.output[mix].data[ch] 上直接写采样值
    success = source->info.audio_render(source->context.data,
                                         &ts, &audio_data,
                                         mixers, channels, sample_rate);

    // ── ③ 更新状态 ──
    source->audio_ts      = success ? ts : 0;
    source->audio_pending = !success;
    if (!success || !source->audio_ts || !mixers)
        return;

    // ── ④ 清理未使用 mix ──
    // 全局活跃但源没勾选的 mix → 可能有旧残留 → 清零
    for (size_t mix = 0; mix < 6; mix++) {
        uint32_t bit = 1 << mix;
        if ((mixers & bit) == 0) continue;
        if ((source->audio_mixers & bit) == 0)
            memset(source->audio_output_buf[mix][0], 0,
                   sizeof(float) * 1024 * channels);
    }

    // ── ⑤ 应用音量 ──
    apply_audio_volume(source, mixers, channels, sample_rate);
}
```

注意两处清零的区别：
- 第①步清的：源勾选的活跃 mix（为回调累加做准备）
- 第④步清的：源没勾选但全局活跃的 mix（清残留防杂音）

关键：数据不是从 deque 取出来的——是**当场生成**的。源的 `audio_ts` 完全由源自己决定（解码器播放到哪了）。这就是为什么后面处理落后时，渲染型源不能 `ignore_audio`（没法 pop deque），只能标记 pending。

### 4.3 门二：`audio_submix` — 复合源累加

`obs-source.c:5375-5403`

```c
static void audio_submix(obs_source_t *source,
                          size_t channels, size_t sample_rate)
{
    struct audio_output_data audio_data;
    struct obs_source_audio audio = {0};
    bool success;
    uint64_t ts;

    // ── ① 绑指针：把 mix_buf 地址填进 audio_data ──
    for (size_t ch = 0; ch < channels; ch++)
        audio_data.data[ch] = source->audio_mix_buf[ch];

    // ── ② 清零（回调用累加模式）──
    memset(source->audio_mix_buf[0], 0,
           sizeof(float) * 1024 * channels);

    // ── ③ 调 audio_mix 回调，遍历子源累加 ──
    success = source->info.audio_mix(source->context.data,
                                      &ts, &audio_data,
                                      channels, sample_rate);
    if (!success) return;

    // ── ④ 包装成 obs_source_audio ──
    for (size_t i = 0; i < channels; i++)
        audio.data[i] = (const uint8_t *)audio_data.data[i];
    audio.samples_per_sec = (uint32_t)sample_rate;
    audio.frames = 1024;
    audio.format = AUDIO_FORMAT_FLOAT_PLANAR;
    audio.speakers = (enum speaker_layout)channels;
    audio.timestamp = ts;

    // ── ⑤ ★ 推回自己的管线 ──
    // → filter_async_audio → source_output_audio_data → push deque
    obs_source_output_audio(source, &audio);

    // 不 return，回到 obs_source_audio_render 往下掉进门 3
}
```

这一步的设计很精妙：复合源把子源的 output 累加后，当作"自己的原始音频"，再走一遍 `obs_source_output_audio` → `process_audio` → `filter_async_audio` → push deque。然后门三的 `process_audio_source_tick` 再从 deque 取出来。这意味着复合源挂的 filter（如总线压缩）会作用于子源的混音结果。

### 4.4 门三：`process_audio_source_tick` — 标准采集源

`obs-source.c:5405-5453`

```c
static inline void process_audio_source_tick(
    obs_source_t *source,
    uint32_t mixers, size_t channels,
    size_t sample_rate, size_t size)       // size = 4096 = 1024 × sizeof(float)
{
    bool is_submix = source->info.output_flags & OBS_SOURCE_SUBMIX;

    // ── ① 检查 deque 够不够 1024 帧 ──
    pthread_mutex_lock(&source->audio_buf_mutex);
    if (source->audio_input_buf[0].size < size) {
        source->audio_pending = true;
        pthread_mutex_unlock(&source->audio_buf_mutex);
        return;
    }

    // ── ② peek（非 pop）从 deque 取数据到 output_buf[0] ──
    for (size_t ch = 0; ch < channels; ch++)
        deque_peek_front(&source->audio_input_buf[ch],
                         source->audio_output_buf[0][ch], size);
    pthread_mutex_unlock(&source->audio_buf_mutex);

    // ── ③ 从 mix 0 复制到其他活跃 mix ──
    for (size_t mix = 1; mix < 6; mix++) {
        uint32_t bit = 1 << mix;

        // submix 源（audio_line）特殊处理：只走一轮 mix=1
        if (is_submix) {
            if (mix > 1) break;
            mixers = 1;      // 强制只认 mix 0
            bit = 1;         // 权限位也改成 mix 0 的
        }

        // 不活跃或源没勾选 → 清零
        if ((source->audio_mixers & bit) == 0 || (mixers & bit) == 0) {
            memset(source->audio_output_buf[mix][0], 0, size * channels);
            continue;
        }

        // 逐声道 memcpy 4096 字节
        for (size_t ch = 0; ch < channels; ch++)
            memcpy(source->audio_output_buf[mix][ch],
                   source->audio_output_buf[0][ch], size);
    }

    // ── ④ submix 源直接返回（volume 在 filter 链中已处理）──
    if (is_submix) {
        source->audio_pending = false;
        return;
    }

    // ── ⑤ 清理 mix 0（如果未使用）+ 上音量 ──
    if ((source->audio_mixers & 1) == 0 || (mixers & 1) == 0)
        memset(source->audio_output_buf[0][0], 0, size * channels);

    apply_audio_volume(source, mixers, channels, sample_rate);
    source->audio_pending = false;
}
```

`OBS_SOURCE_SUBMIX` 唯一注册者是内部源 `audio_line`（"Audio line, internal use only"），用于监听系统内部路由。它只输出到 mix 0。

这里 `size = 4096`（1024×4）是**单声道的字节数**。因为 OBS 用 planar 格式，每个声道的 buffer 都是独立的，所以检查只需要检查 `audio_input_buf[0].size >= 4096`——左声道有 1024 个 float 了，就认为其他声道也够了。

---

## 五、数据入队：`obs_source_output_audio` 全流程

所有平台采集线程最终都调这个函数把数据推入 OBS 音频管线。与第四节（消费端）形成完整的生产者-消费者闭环。

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

### 5.1 `process_audio` —— 格式统一

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

### 5.2 `filter_async_audio` —— Filter 责任链

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

这是一个典型的责任链模式。常见音频 filter：噪声抑制（RNNoise/Speex）、压限器、增益、均衡器（EQ）、VST 插件。

### 5.3 `source_output_audio_data` —— 入 deque

把处理完的数据推到 source 的 `audio_input_buf[ch]`（每声道一个 deque），同时更新 `source->audio_ts`：

```c
// 简化逻辑
for (size_t ch = 0; ch < channels; ch++)
    deque_push_back(&source->audio_input_buf[ch], data->data[ch], size);

source->audio_ts = data->timestamp + audio_frames_to_ns(rate, data->frames);
// 源的时间戳推进到 "这批数据的结束时间"
```

这里有三个写入 `audio_ts` 的位置（全貌）：

| 写位置 | 值 | 说明 |
|--------|-----|------|
| `reset_audio_data()` | `os_time` | 源刚启动时的初始时间戳 |
| `custom_audio_render()` | 源自己返回的 ts | 渲染型源自主管理时间（解码器位置） |
| `source_output_audio_data()` | timestamp + frames 时长 | 采集源推入数据时推进（本节路径） |

`discard_audio` 也会推进 `audio_ts`（消费端），但那个是在音频线程侧，后面 6.8 节详述。

---

## 六、缓冲管理深度解析

缓冲管理是 OBS 音频子系统中最精妙的部分。它的本质是**用延迟换稳定性**——发现某个源数据还没到，就把混音窗口往回拉，给慢源一点时间追赶。本节从核心概念出发，逐步深入到动态缓冲、固定缓冲、硬兜底三层机制。

### 6.1 核心概念

三个核心量的关系：

```
source->audio_ts  =  源"已经生产到哪个时间点"（现实墙上时间的纳秒）
ts.start          =  混音窗口"需要从哪个时间点开始的数据"
ts.end            =  混音窗口"需要到哪个时间点"
```

正常情况：`source->audio_ts >= ts.end`，源的数据覆盖了 ts 区间，直接渲染。

落后情况：`source->audio_ts < ts.start`，源的数据产得不够快，够不着处理窗口的起点。

```
USB 麦克风:  ████████  ← audio_ts = T₁ = 100ns
桌面音频:          ████████  ← audio_ts = T₂ = 105ns（比麦克风晚了 5ms）

当前处理窗口: [100, 121)
               ↑ 麦克风有数据 ✓    桌面音频还没到 ✗
```

`audio_ts` 有三个写入源（见 5.3 节），而它的读出场景就是这里——`calc_min_ts` 遍历所有源的 `audio_ts`，找最小值来判断谁落后了。

### 6.2 `buffering_wait_ticks`：缓冲倒计时

```
改：缓冲触发时 ++（add_audio_buffering 每次 push_front 一个区间时 ++）
改：每 tick 结尾 --（audio_callback 最后几行）
用：mix_audio 前检查 != 0 → 跳过混音
用：返回值 → wait_ticks > 0 时 return false（不输出）
```

缓冲推了几个旧时间区间，就 silent 跑几帧——消耗旧区间、帮慢源清 deque、对齐时间戳。期间混音全部跳过，数据"凭空"被消耗不输出。

### 6.3 为什么要渲染但不要混音

缓冲期间 `audio_callback` 照常执行渲染（阶段二），照常执行清理（阶段六），但跳过混音（阶段五，因为 `buffering_wait_ticks != 0`）。直接跳过渲染不是更高效？

不行。跳过渲染会导致 deque 停滞——慢源内部的 `audio_ts` 全靠 `discard_audio` 推进，而 `discard_audio` 依赖 `ts` 区间来判断是否对齐。渲染过程中还涉及音频动作（fade in/fade out/音量渐变）的推进，跳过会导致状态混乱。

另外缓冲通常只有 2~3 tick，开销极小。加跳过路径的分支成本可能比直接跑还高。

### 6.4 传入时间 vs 实际使用时间

这是理解缓冲机制最关键的一张图：

```
外部时间持续流逝：
Tick N:   start_ts_in = t
Tick N+1: start_ts_in = t+1024
Tick N+2: start_ts_in = t+2048

但 audio_callback 内部：
        传入的值        buffer 覆盖后的实际 ts
Tick N:   t                  t
Tick N+1: t+1024             t-1024   ← 缓冲拉回
Tick N+2: t+2048             t        ← 被覆盖
Tick N+3: t+3072             t+1024   ← 追上！
```

传入值 push_back 存在队尾排队等。缓冲结束时它自然排到队首。这就是 2.2 节 `buffered_timestamps` 三步走的威力——外部时间马不停蹄往前走，内部处理窗口被队列"拖住"，等慢源追上来。

### 6.5 动态缓冲：`add_audio_buffering`

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

`buffered_timestamps` 双端队列的精髓：

- 正常时每次 Tick 往**队尾** push 当前 ts，peek **队首**取出的就是当前 ts
- 需要缓冲时往**队首** push 更早的 ts，peek 出来就变成了过去的 ts
- 等 `buffering_wait_ticks` 耗尽后，队列恢复为"push 一个 pop 一个"的正常模式

### 6.6 固定缓冲：`set_fixed_audio_buffering`

```c
if (audio->fixed_buffer) {
    set_fixed_audio_buffering(...);        // 一次垫满 max_buffering_ticks
} else if (min_ts < ts.start) {
    add_audio_buffering(...);              // 只垫落后源需要的量
}
```

固定缓冲不检查 `min_ts < ts.start`——无脑垫满预设值。信任设备时钟的用户不用选动态缓冲，预先垫好固定的缓冲量，启动后延迟恒定可预测。

两者的底层实现完全一致——都是往 `buffered_timestamps` 队首 push 旧时间区间，区别仅在于触发条件和垫多少。

### 6.7 缓冲满后：`ignore_audio` 硬兜底

当 `total_buffering_ticks == max_buffering_ticks`（默认 45 帧 ≈ 960ms @ 48kHz），缓冲已经到上限了，但仍有源落后时，触发硬兜底：

```c
if (audio_buffering_maxed(audio)       // total_buffering_ticks == max
    && source->audio_ts != 0           // 曾经产过数据
    && source->audio_ts < ts.start) {  // 确实落后了
```

对于**采集源**（有 deque）：

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

为什么不能丢渲染型源的数据？两个原因：

1. **A/V 同步**：媒体的音频帧和视频帧是配对好的，丢了音频声音和画面对不上
2. **内部状态**：媒体源的数据是调用 `audio_render` 回调当场生成的，外部没法用 `deque_pop_front` 方式干预

渲染型源"追上来"的方式是：每次 Tick 都调 `audio_render`，即使数据被标记 pending 丢弃了，源内部的解码位置还是往前推进了。如果解码速度 ≥ 实时，终归能追上（audio_ts 会逐渐靠近 ts.start）。但如果解码就是跟不上（CPU 太忙），就会一直 pending，用户听到的就是音频卡顿/断断续续。

### 6.8 `discard_audio` —— 消费已用数据

`audio_callback` 阶段六（2.8 节）对每个源调用，把 input buffer 里已经被混音覆盖的部分弹掉：

```c
static void discard_audio(struct obs_core_audio *audio, obs_source_t *source,
                           size_t channels, size_t sample_rate, struct ts_info *ts)
{
    // 渲染型音源 → 不管，它们没有 deque
    if (source->info.audio_render) {
        source->audio_ts = 0;
        return;
    }

    // ts 区间前的数据才保留，前面的全弹出
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

三种情况的决策树：

```
正常路径：audio_ts >= ts.end    → pop 已消费帧，推进 audio_ts = ts.end
落后路径：audio_ts < ts.start   → return（不弹！让 audio_ts 停滞作为"落后信号"）
渲染型源：有 audio_render 回调  → audio_ts = 0，return（不管）
```

落后的源音频线程不碰它，时间戳原地不动。下次 `calc_min_ts` 里它仍是检测缓冲触发的关键信号——这是一个自愈的闭环。

---

## 七、完整追踪示例：一个落后源的两帧缓冲之旅

本节用一个虚构但精确的例子，把前面所有环节串起来。假设有快源和慢源两个源，48kHz，慢源落后了几毫秒。

```
初始状态：
  快源 deque: [t, t+1024), audio_ts = t+1024
  慢源 deque: [t-m, t-m+333], audio_ts = t-m+333
  buffering_wait_ticks = 0
```

### Tick N

```
ts = [t, t+1024)

Step 1 — 构建渲染树：快源、慢源都入 render_order

Step 2 — 渲染：
  慢源 deque size < 1024 → audio_pending = true
  快源 deque size >= 1024 → peek 1024 frames → audio_pending = false

Step 3 — 缓冲判断：
  calc_min_ts：慢源 pending → 跳过（不参与 min_ts 计算）
              min_ts = t（来自快源）
              min_ts < ts.start(t)？否 → 不触发缓冲

Step 4 — 硬兜底：未触发（缓冲未 maxed）

Step 5 — 混音：
  慢源 pending → 跳过
  快源 ✓ → mix_audio 累加

Step 6 — discard：
  慢源 audio_ts = t-m+333 < ts.start(t) → return（不弹！audio_ts 不变 = t-m+333）
  快源 pop → audio_ts = t+1024
```

关键：慢源的 `audio_ts` 没有被推进！它仍然是 `t-m+333`。这为下一帧触发缓冲埋下了伏笔。

### Tick N+1

```
ts 传入 = [t+1024, t+2048)

push_back [t+1024, t+2048), peek_front → ts = [t+1024, t+2048)

Step 2 — 渲染：
  慢源 deque 攒够 1024 了 → peek → audio_pending = false
  快源 ✓

Step 3 — 缓冲判断：
  calc_min_ts：慢源 audio_ts = t-m+333 < ts.start(t+1024) → ★ min_ts = t-m+333
              min_ts < ts.start → ★ 触发 add_audio_buffering

  add_audio_buffering：
    落后 = 1024+m-333 帧，ticks = 2
    push_front [t, t+1024), push_front [t-1024, t)
    队列：[[t-1024,t), [t,t+1024), [t+1024,t+2048)]
    *ts = [t-1024, t)，wait_ticks = 2

Step 5 — 混音：
  buffering_wait_ticks = 2 > 0 → ★ 跳过全部混音

Step 6 — discard：
  ts = [t-1024, t)
  慢源：audio_ts = t-m+333 对齐到 ts.start(t-1024) → pop → audio_ts = t
  快源：audio_ts = t+1024 >> ts.end(t) → 跳过，不弹

buffering_wait_ticks-- → wait_ticks = 1
return false（不输出，不做任何分发）
```

### Tick N+2

```
ts 传入 = [t+2048, t+3072)

push_back [t+2048, t+3072), peek_front → ts = [t, t+1024)

渲染、discard 全部基于 ts = [t, t+1024)
混音跳过（wait_ticks = 1 > 0）

buffering_wait_ticks-- → wait_ticks = 0
return false（不输出）
```

### Tick N+3

```
ts 传入 = [t+3072, t+4096)

push_back [t+3072, t+4096), peek_front → ts = [t+1024, t+2048)

现在 wait_ticks = 0 → ★ 混音正常执行，输出！
所有源的 audio_ts 对齐到 ts = [t+1024, t+2048)
```

两帧无声后恢复正常——总延迟增加了约 42ms。用户听不到任何异常。缓冲机制用 42ms 的额外延迟换来了两个源的重新对齐，整个过程对用户透明。

---

## 八、钳制与分发

阶段 C（钳制）和阶段 D（分发）在 `audio_callback` 返回后由 `input_and_output` 调用。虽然代码不如回调内部复杂，但细节值得展开。

### 8.1 `clamp_audio_output`：钳位与 NaN 处理

```c
// 先把 mix->buffer 整份复制到 mix->buffer_unclamped（保留原始值）
// 再对 mix->buffer 做钳位：
float val = *mix_data;
val = (val == val) ? val : 0.0f;     // NaN → 0（NaN 不等于自己，很巧妙）
val = (val > 1.0f)  ? 1.0f : val;
val = (val < -1.0f) ? -1.0f : val;
```

consumer 可以通过 `allow_clipping` 标志选择用 raw 版（`buffer_unclamped`）还是 clamped 版（`buffer`）。大多数编码器用 clamped 版防止爆音，专业用户可能用 raw 版保留动态范围做后期处理。

### 8.2 `do_audio_output`：广播给所有消费者

`audio-io.c:107-130`

```c
static inline void do_audio_output(
    struct audio_output *audio,
    size_t mix_idx,           // 当前音轨编号（0~5）
    uint64_t timestamp,       // 经 audio_callback 修正后的 new_ts
    uint32_t frames)          // 固定 1024
{
    struct audio_mix *mix = &audio->mixes[mix_idx];
    struct audio_data data;

    pthread_mutex_lock(&audio->input_mutex);

    // ★ 倒序遍历该音轨的所有消费者
    for (size_t i = mix->inputs.num; i > 0; i--) {
        struct audio_input *input = mix->inputs.array + (i - 1);

        // ① 选钳位版还是未钳位版 buffer
        float (*buf)[1024] = input->conversion.allow_clipping
                             ? mix->buffer_unclamped : mix->buffer;

        // ② 把 mix buffer 的平面指针绑进 data
        for (size_t p = 0; p < audio->planes; p++)
            data.data[p] = (uint8_t *)buf[p];

        data.frames    = frames;
        data.timestamp = timestamp;

        // ③ 格式不匹配 → 重采样；成功 → 调消费者回调
        if (resample_audio_output(input, &data))
            input->callback(input->param, mix_idx, &data);
    }

    pthread_mutex_unlock(&audio->input_mutex);
}
```

倒序原因：消费者回调中可能调 `audio_output_disconnect` 从数组里删除自己。倒着跑保证剩余未处理元素的索引不变。

`input->callback` 的可能值：

| 回调 | 注册者 | 用途 |
|------|--------|------|
| `receive_audio()` | 编码器（obs-encoder.c:375） | AAC/Opus 编码 |
| `default_raw_audio_callback()` | 输出（obs-output.c:2453） | FFmpeg、重放缓冲 |
| 任意函数 | `obs_add_raw_audio_callback` API | 第三方插件 |

---

## 九、监控去重消音

当用户既启用了音频监控（在耳机里听自己的声音），又添加了"音频输出采集"源（采集桌面声音），且两者指向同一设备时，会产生回声。OBS 有一套去重消音机制：

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

## 十、平台采集线程细节

### 10.1 Windows WASAPI

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

### 10.2 macOS CoreAudio

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

### 10.3 Linux PulseAudio

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

三种平台的采集方式虽不同，但**最终汇入 OBS 管线的入口完全一致**——都是 `obs_source_output_audio()`。这就是 OBS 跨平台音频架构的设计精髓：平台差异在采集层隔离，核心管线完全统一。

---

## 十一、完整数据流全景

把前面所有环节串起来，整个音频数据的流动全景如下。这张图同时包含了**音频线程侧**（上）和**采集源输入侧**（下），两个方向的数据流一目了然：

```
audio_thread()  [audio-io.c:205]  ← 独立线程，"audio-io: audio thread"
  │                                  每 1024 帧跑一次，创建于 audio_output_open()
  │
  ├─ ① input_and_output()  [audio-io.c:231]  ← 一帧音频的全部处理入口
  │     │
  │     ├─ 阶段 A · 渲染 ──────────────────────────────────────
  │     │   清空 mix buffer → 构建渲染树 → 逐源渲染 → 混音 → 清理
  │     │
  │     │   audio->input_cb() 即 audio_callback()  [obs-audio.c:555]
  │     │     │
  │     │     ├─ ① push_back ts 到 buffered_timestamps 队尾
  │     │     │
  │     │     ├─ ② 构建渲染树
  │     │     │   ├─ 遍历 video mixes → view channels → source 树
  │     │     │   │   ├─ push_audio_tree2() — 深搜，查重升级
  │     │     │   │   └─ push_audio_tree()  — 顶层源加入
  │     │     │   └─ 遍历 first_audio_source 链表 — 兜底
  │     │     │
  │     │     ├─ ③ 渲染每个源（按 render_order）
  │     │     │   └─ obs_source_audio_render()
  │     │     │       ├─ scene 类 → custom_audio_render() → audio_render 回调
  │     │     │       ├─ submix 类 → audio_submix() → audio_mix 回调
  │     │     │       └─ 普通源  → process_audio_source_tick()
  │     │     │                     peek input_buf → output_buf → apply_volume
  │     │     │
  │     │     ├─ ④ calc_min_ts() → 缓冲判断
  │     │     │   ├─ fixed_buffer → set_fixed_audio_buffering()
  │     │     │   └─ min_ts < ts.start → add_audio_buffering()
  │     │     │         → deque_push_front(buffered_timestamps, 旧区间)
  │     │     │         → buffering_wait_ticks++
  │     │     │
  │     │     ├─ ⑤ peek_front(buffered_timestamps) → 获取实际使用的 ts
  │     │     │
  │     │     ├─ ⑥ 落后源硬兜底（缓冲 maxed 时）
  │     │     │   ├─ 采集源 → ignore_audio() → pop deque 丢帧
  │     │     │   └─ 渲染型源 → audio_pending = true
  │     │     │
  │     │     ├─ ⑦ 混音（按 root_nodes，wait_ticks == 0 时才执行）
  │     │     │   └─ mix_audio(): mixes[mix].data[ch][i] += source.output_buf[mix][ch][i]
  │     │     │
  │     │     └─ ⑧ 清理
  │     │         ├─ discard_audio(): deque_pop_front(audio_input_buf)
  │     │         ├─ release_audio_sources()
  │     │         └─ deque_pop_front(buffered_timestamps)
  │     │
  │     ├─ 阶段 C · 钳制 ─────────────────────────────────────
  │     │   clamp_audio_output() — 钳到 [-1.0, 1.0]，保留 unclamped 副本
  │     │
  │     └─ 阶段 D · 分发 ─────────────────────────────────────
  │         do_audio_output() — 广播给所有消费者
  │           ├─ receive_audio()           — 编码路径
  │           ├─ default_raw_audio_callback() — 原始 PCM 路径
  │           └─ obs_add_raw_audio_callback() — 第三方插件
  │
  └─ os_sleepto_ns_fast(audio_time)  ← 精确睡眠到下一帧


  音频数据输入路径（异步，由各采集源线程调用）

  obs_source_output_audio()  [obs-source.c:4029]
    ├─ process_audio()           — 重采样 / 平衡 / 强制单声道
    ├─ filter_async_audio()      — filter 链（责任链模式）
    └─ source_output_audio_data()
          ├─ deque_push_back(audio_input_buf)  — 排队等待音频线程消费
          └─ source_signal_audio_data()
                ├─ on_audio_playback()          — 音频监听输出
                └─ volmeter_source_data_received() — VU 表
```

上面是音频线程侧（消费端），下面是采集源侧（生产端）。生产者只管往 deque 里塞数据，消费者按 1024 帧的节奏取出、渲染、混音、分发。两端通过 `audio_input_buf` 解耦。生产者和消费者跑在不同线程上，这也是为什么需要缓冲管理——两端的时间戳不可能天然对齐。

---

## 十二、关键数据结构速查

贯穿全文的核心数据结构汇总：

| 字段 | 位置 | 含义 |
|------|------|------|
| `obs_core_audio.render_order` | obs-internal.h | 本帧渲染顺序（叶子在前，父源在后） |
| `obs_core_audio.root_nodes` | obs-internal.h | 混音根节点（顶层源 + 去重升级源） |
| `obs_core_audio.buffered_timestamps` | obs-internal.h:449 | 时间戳 deque，缓冲机制的时钟核心 |
| `obs_core_audio.buffering_wait_ticks` | obs-internal.h:450 | 缓冲等待倒计时（>0 时跳过混音） |
| `obs_core_audio.buffered_ts` | obs-internal.h:448 | 缓冲触发瞬间的锚点时间戳 |
| `obs_core_audio.total_buffering_ticks` | obs-internal.h | 累计缓冲量（达到 max 后触发硬兜底） |
| `obs_source.audio_ts` | obs-internal.h:867 | 源已产数据到哪个时间点（缓冲判断的关键信号） |
| `obs_source.audio_input_buf[ch]` | obs-internal.h:868 | 采集线程推入的 deque（生产者-消费者缓冲区） |
| `obs_source.audio_output_buf[mix][ch]` | obs-internal.h:871 | 源渲染输出的 1024 float buffer（单次 Tick 的工作区） |
| `obs_source.audio_mix_buf[ch]` | obs-internal.h | submix 源的累加缓冲区 |
| `obs_source.audio_pending` | obs-internal.h | 标记本帧该源数据不足，跳过混音 |
| `obs_source.audio_is_duplicated` | obs-internal.h | 标记该源在音频树中重复出现 |
| `audio_output_info.input_callback` | audio-io.h | = audio_callback，在 obs.c:1646 绑定 |
| `audio_mix.buffer[planes][1024]` | audio-io.h | mix 的混音输出 buffer（每 Tick 清空重填） |
| `audio_mix.buffer_unclamped` | audio-io.h | 未钳位的副本（供有需要的消费者使用） |

---

## 十三、容易踩的命名坑

- **`obs_core_data`** 不是"音频数据"——它是全局对象注册中心，all sources/outputs/encoders/displays 的链表头都在里面。`audio_callback` 只用了 `first_audio_source` 和 `audio_sources_mutex`
- **`audio_callback`** 不是采集回调——采集回调是各平台的 WASAPI/CoreAudio/PulseAudio 回调。`audio_callback` 是混音引擎的入口，参数由 `input_and_output` 传入
- **`input_callback`** 的命名是从 `audio_output` 模块视角看的——"你给我输入数据"，所以叫 input。对 OBS 整体来说它是混音渲染核心
- **`audio_size = 1024 × sizeof(float)`** 是**单声道的字节数**，因为 OBS 内部用 planar 格式，每个声道独立 buffer
- **`block_size`** 也是单声道单采样点字节数，planar float 下 = 4
- **view channel ≠ 音频声道**：view channel 是源槽位（最多 64 个），音频声道是物理声道数。两者在变量名上容易混淆

---

## 小结

读音频子系统的这段时间，最大的感受是和视频管线在思路上高度对称——都有独立的核心时钟线程、都有按需消费的机制、都用 deque 做缓冲解耦。把 OBS 音频子系统的设计用几个词概括：

- **一个时钟**：audio-io 线程，固定 1024 帧节奏，从 `start_time` 绝对推导，永不漂移
- **一个核心回调**：`audio_callback`，六个阶段覆盖构建→渲染→缓冲→兜底→混音→清理
- **一个队列**：`buffered_timestamps`，用双端队列的 push_front/push_back 实现处理窗口的时间旅行
- **N 个生产者**：各平台采集线程，通过 `obs_source_output_audio` 统一入口推 deque
- **每源一个 deque**：环形双端队列解耦生产者和消费者
- **两级缓冲**：动态缓冲用延迟换稳定性 → 缓冲满后才丢数据 / pending
- **两层 filter**：子源 filter（单路处理）+ 复合源 filter（总线处理），各管各的
- **三路径渲染**：`audio_render`（容器源自主）→ `audio_mix`（submix 预留）→ 默认 peek deque
- **6 个 mix**：多轨输出的基础，每个 source 的 `audio_mixers` 决定去哪些轨道

`audio_callback` 是当之无愧的心脏——它每 21.33ms 跳动一次，把散落在各线程的音频数据收集起来，对齐时间戳、处理缓冲、混音叠加，最后分发给编码器和监控。理解了 `buffered_timestamps` 的三步走和六个阶段的协作方式，整个 OBS 音频子系统就尽在掌握。

---

学习过程中参考了 OBS Studio 官方源码仓库，感谢开源社区。如有理解偏差，欢迎指正交流。
