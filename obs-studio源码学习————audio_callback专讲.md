# obs-studio 源码学习 — audio_callback 专讲

> 本文是对 `audio_callback`（`libobs/obs-audio.c:555`）的逐段深入分析。覆盖构建、渲染、缓冲、混音、清理五个阶段，以及缓冲管理的核心机制。

---

## 一、函数签名与参数来源

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

`out_ts` 是输出参数，可能被缓冲逻辑修正为更早的时间戳，传给下游消费者。

---

## 二、buffered_timestamps 队列机制

这是整个 `audio_callback` 的时钟核心。

```
每 tick 三步：

① deque_push_back(&buffered_timestamps, &ts);   // 当前 ts 存队尾
② deque_peek_front(&buffered_timestamps, &ts);  // 队首取实际用的 ts  ★ ts 可能被覆盖
③ deque_pop_front(&buffered_timestamps);        // tick 结束时弹掉队首
```

正常时（无缓冲）：队列只有 1 条记录。push 什么 peek 就是什么。ts 不变。

缓冲时：`add_audio_buffering` 或 `set_fixed_audio_buffering` 往队首 push 了额外的旧时间区间。peek 取出的就是被拉回的老区间，ts 被覆盖。

队列严格升序：push_back 写晚的，push_front 写早的——队首永远是最老的时间点。

---

## 三、五个阶段

### 阶段 1：构建渲染树（Build）

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

结果：

```
render_order[] = 所有需要本帧渲染音频的源（叶子在前，父源在后）
root_nodes[]   = 需要参与最终混音的顶层源（scene + 被去重升级的重复源）
```

### 阶段 2：渲染（Render）

```c
for (render_order 中每个 source) {
    obs_source_audio_render(source, ...);  // 产出 1024 采样点到输出缓冲

    if (should_silence_monitored(source))  // 监控去重消音
        clear_audio_output_buf(source);
}
```

每个 source 走三条渲染路径之一：

| 源类型 | 函数 | 数据来源 |
|--------|------|----------|
| 有 `audio_render` 回调（scene/媒体）| `custom_audio_render()` | 回调当场生成或累加子源 |
| 有 `audio_mix` 回调（submix） | `audio_submix()` | 累加子源 output_buf |
| 普通采集源（麦克风等） | `process_audio_source_tick()` | peek `audio_input_buf` deque |

### 阶段 3：缓冲管理（Buffer）

```c
calc_min_ts(..., &min_ts);   // 找最慢源的 audio_ts

if (fixed_buffer)
    set_fixed_audio_buffering();          // 预先垫满
else if (min_ts < ts.start)
    add_audio_buffering();                // 按需追加：往队首塞旧区间
```

固定缓冲和动态缓冲的区别只在于**缓冲量**：固定一次性垫满 `max_buffering_ticks`，动态按落后量仅垫需要的 tick 数。底层 push_front 逻辑完全一致。

### 阶段 4：落后源硬兜底

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

- **有 `audio_render` 回调的源**（scene、媒体文件）：数据当场生成，没有 deque 可以 pop。只能标记 pending。
- **无 `audio_render` 的采集源**（麦克风等）：有 deque。`ignore_audio` 计算落后量 → pop 掉落后采样 → 时间戳前移。追上了返回 true 触发 re-render。

### 阶段 5：混音（Mix）

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

### 阶段 6：清理（Cleanup）

```c
discard_audio(audio, source, ...);    // 逐源 pop 已消费的 deque 数据
release_audio_sources(audio);         // 释放 render_order refs
deque_pop_front(&buffered_timestamps); // 弹掉队首
```

`discard_audio` 的关键行为：

```
正常路径：audio_ts = ts.end       ← 推进
落后路径：audio_ts < ts.start → return  ← 不弹，不推进
渲染型源：audio_ts = 0 → return         ← 不管
```

落后的源音频线程不碰它，时间戳原地不动。下次 `calc_min_ts` 里它是检测缓冲触发的关键信号。

---

## 四、缓冲管理的核心机制

### 4.1 `audio_ts`：源数据到哪个时间点

```c
// obs-internal.h:867
uint64_t audio_ts;
```

三个写入源：

| 写位置 | 值 |
|--------|-----|
| `reset_audio_data()` | `os_time`（源刚启动） |
| `custom_audio_render()` | 源自己返回的 ts |
| `discard_audio()` 正常路径 | `ts->end`（弹出后推进） |

采集源没有显式的"设置 audio_ts"——它们通过 `discard_audio` 推进。`discard_audio` 的落后跳过机制保证了当源跟不上处理窗口时，`audio_ts` 停滞不动——这就是 calc_min_ts 发现它落后的信号。

### 4.2 `buffering_wait_ticks`：倒计时

```
改：缓冲触发时 ++（add_audio_buffering 每次 push_front 一个区间时 ++）
改：每 tick 结尾 --（audio_callback 最后几行）
用：mix_audio 前检查 != 0 → 跳过混音
用：返回值 → wait_ticks > 0 时 return false（不输出）
```

缓冲推了几个旧时间区间，就 silent 跑几帧——消耗旧区间、帮慢源清 deque、对齐时间戳。期间混音全部跳过，数据"凭空"被消耗不输出。

### 4.3 为什么要渲染但不要混音

被问到为什么缓冲期间要渲染但不混——直接跳过渲染不是更高效？不行。跳过渲染会导致 deque 停滞，慢源内部的 `audio_ts` 全靠 `discard_audio` 推进，而 `discard_audio` 依赖 `ts` 区间来判断是否对齐。渲染过程中还涉及音频动作（fade in/fade out/音量渐变）的推进，跳过会导致状态混乱。

另外缓冲通常只有 2~3 tick，开销极小。加跳过路径的分支成本可能比直接跑还高。

### 4.4 传入时间 vs 实际使用时间

```
外部时间持续流逝：
Tick N:   start_ts_in = t
Tick N+1: start_ts_in = t+1024
Tick N+2: start_ts_in = t+2048

但 event_callback 内部：
		传递传入的值       	 buffer 覆盖后的实际 ts
Tick N:   t                 	t
Tick N+1: t+1024             	t-1024   ← 缓冲拉回
Tick N+2: t+2048             	t        ← 被覆盖
Tick N+3: t+3072             	t+1024   ← 追上！
```

传入值 push_back 存在队尾排队等。缓冲结束时它自然排到队首。

### 4.5 固定 v.s. 动态

```c
if (audio->fixed_buffer) {
    set_fixed_audio_buffering(...);        // 一次垫满 max_buffering_ticks
} else if (min_ts < ts.start) {
    add_audio_buffering(...);              // 只垫落后源需要的量
}
```

固定缓冲不检查 `min_ts < ts.start`——无脑垫。信任设备时钟的用户不用选。

## 五、一个完整的落后源追踪示例

```
初始：
  快源 deque: [t, t+1024), audio_ts = t+1024
  慢源 deque: [t-m, t-m+333], audio_ts = t-m+333
  buffering_wait_ticks = 0

=== Tick N ===
  ts = [t, t+1024)

  渲染：  慢源 deque < 1024 → pending
          快源 ✓

  calc_min_ts：  慢源 pending → 跳过
                min_ts = t < ts.start(t)？否 → 不缓冲

  混音：  慢源 pending → 跳过
          快源 ✓

  discard：  慢源 audio_ts < t-1 → return（不弹！audio_ts 不变 = t-m+333）
             快源 pop → audio_ts = t+1024

=== Tick N+1 ===
  ts 传入 = [t+1024, t+2048)

  push_back [t+1024, t+2048), peek_front → ts = [t+1024, t+2048)

  渲染：  慢源 deque 够了 → peek → audio_pending = false
          快源 ✓

  calc_min_ts：  慢源 audio_ts = t-m+333 < t+1024 → ★ min_ts = t-m+333
                min_ts < ts.start → ★ add_audio_buffering

  add_audio_buffering：
    落后 = 1024+m-333 帧，ticks = 2
    push_front [t, t+1024), push_front [t-1024, t)
    队列：[[t-1024,t), [t,t+1024), [t+1024,t+2048)]
    *ts = [t-1024, t)，wait_ticks = 2

  渲染 重新执行（忽略，因为这一帧已经 render 过了——等等，不，add_audio_buffering 改了 ts 但没有重新渲染。下一帧才会用新 ts）

  混音：  wait_ticks = 2 > 0 → 跳过

  discard：  ts = [t-1024, t)
             慢源：audio_ts = t-m+333 对齐到 ts.start → pop → audio_ts = t
             快源：audio_ts = t+1024 >> ts.end(t) → 跳过，不弹

  wait_ticks > 0 → return false（不输出，不做任何分发）
  wait_ticks-- → wait_ticks = 1

=== Tick N+2 ===
  ts 传入 = [t+2048, t+3072)

  push_back [t+2048, t+3072), peek_front → ts = [t, t+1024)

  渲染、混音、discard全部基于 ts = [t, t+1024)
  混音跳过，wait_ticks = 0 → return false（不输出）

=== Tick N+3 ===
  ts 传入 = [t+3072, t+4096)

  push_back [t+3072, t+4096), peek_front → ts = [t+1024, t+2048)
  
  现在 wait_ticks = 0 → ★ 混音正常，输出！
  所有源的 audio_ts 对齐到 ts = [t+1024, t+2048)
```

两帧无声后恢复正常——总延迟增加了 ~42ms。用户听不到任何异常。

## 六、关键数据结构速查

| 字段 | 位置 | 含义 |
|------|------|------|
| `obs_core_audio.render_order` | obs-internal.h | 本帧渲染顺序（叶子在前） |
| `obs_core_audio.root_nodes` | obs-internal.h | 混音根节点（顶层源 + 去重升级源） |
| `obs_core_audio.buffered_timestamps` | obs-internal.h:449 | 时间戳 deque，缓冲机制的时钟 |
| `obs_core_audio.buffering_wait_ticks` | obs-internal.h:450 | 缓冲等待倒计时 |
| `obs_core_audio.buffered_ts` | obs-internal.h:448 | 缓冲触发瞬间的锚点时间戳 |
| `obs_source.audio_ts` | obs-internal.h:867 | 源已产数据到哪个时间点 |
| `obs_source.audio_input_buf[ch]` | obs-internal.h:868 | 采集线程推入的 deque |
| `obs_source.audio_output_buf[mix][ch]` | obs-internal.h:871 | 源渲染输出的 1024 float buffer |
| `audio_output_info.input_callback` | audio-io.h | = audio_callback，在 obs.c:1646 绑定 |

---

## 七、obs_source_audio_render — 单源渲染分发

`obs-source.c:5455`

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
| 注册者 | Scene, Transition, Slideshow | **零个** | 麦克风, 桌面音频, 游戏采集 |
| 路径 | 门 1 → return | 门 2 → 往下掉进门 3 | 直接门 3 |
| 做什么 | 全权掌控：遍历子源+对时+自定义混音 | 预留：只累加子源 | peek deque + copy + 上音量 |

**容器源 vs 叶子源：**

- **容器源**：有子 source。Scene 有 scene item → child source，Transition 有 A/B 两个子源。必须实现 `audio_render`，因为 OBS 不知道你怎么排列组合子 source。
- **叶子源**：无子 source。数据从外部来（驱动回调、解码器），通过 `obs_source_output_audio()` 写入 `audio_input_buf`。不需要任何回调，OBS 默认流程全自动。

---

### 7.1 门 1：custom_audio_render

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

---

### 7.2 门 2：audio_submix

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

`audio_submix` 只是把子源的 output_buf 混在一起，当作自己的原始数据推回 deque。剩下的 filter 和 deque 消费全由 OBS 接管。

---

### 7.3 门 3：process_audio_source_tick

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

---

## 八、阶段 D 分发：do_audio_output

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
