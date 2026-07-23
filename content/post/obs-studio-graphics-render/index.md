---
title: "OBS Studio 源码学习（二）：图形渲染线程梳理"
description: "从 obs_graphics_thread 入口到 video_sleep 帧定时，完整梳理 OBS Studio 图形渲染线程的主循环、GPU 管线、三缓冲机制和按需输出策略。"
date: 2026-07-16
tags: ["OBS", "源码学习", "C++", "obs-studio"]
categories: ["OBS Studio源码学习"]
image: ""
---

上一篇文章把主线程从 `main()` 到 `exec()` 的启动链路走了一遍，当时在最后留了个尾巴——"下一篇计划梳理 video_thread 的渲染管线"。这篇就来填坑。

OBS Studio 的图形渲染线程是整个视频系统的核心引擎，负责每一帧的定时推进、场景渲染、格式转换、编码输出和显示刷新。这篇文章是我读 `obs-video.c`、`obs-source.c` 和相关文件的笔记，从源码层面把图形渲染线程的工作流程完整走一遍，同时补充了源级渲染决策、filter 链、异步帧处理、去隔行等关键子系统的细节。

本文基于 OBS Studio 主分支源码，核心文件为：
- `libobs/obs-video.c` — 图形线程主循环
- `libobs/obs-source.c` — 源的渲染逻辑、filter 链、异步上传
- `libobs/obs-source-deinterlace.c` — 去隔行实现
- `libobs/obs-view.c` — 视图（view）的定义和渲染
- `libobs/obs.c` — 全局初始化和公共 API
- `libobs/graphics/graphics.c` — 图形子系统抽象层

---

## 一、先搞清楚几个概念的关系

在深入线程循环之前，得先把 Mix、View、Channel、Canvas 这几个核心概念的关系理清楚。刚开始看代码的时候我就在这里绕了很久——名字都差不多，很容易搞混。

### 1.1 全局结构

`obs` 全局单例的大致结构长这样：

```
obs (全局单例)
 ├─ obs_core_video    图形子系统
 │   ├─ graphics      封装的 GPU 设备（D3D11/OpenGL）
 │   ├─ shader        内置效果（缩放、颜色转换、去隔行...）
 │   ├─ video_thread  图形渲染线程
 │   ├─ video_time    当前视频时间戳
 │   └─ mixes[]       所有视频混合器（mix）的数组
 │
 ├─ obs_core_data     数据层
 │   ├─ main_canvas   主画布
 │   ├─ sources       源列表
 │   ├─ draw_callbacks   预渲染钩子
 │   └─ rendered_callbacks 渲染完成钩子
 │
 └─ obs_core_audio    音频子系统
```

### 1.2 Mix、View、Channel、Canvas 的关系

这是理解 OBS 渲染体系最重要的一组概念，值得多花点篇幅讲清楚。

**mix（obs_core_video_mix）** —— 渲染管线实例
- 持有一套完整的渲染→输出流水线：渲染纹理、缩放纹理、转换纹理、staging surface、编码器队列、`video_t` 输出对象
- 通过 `mix->view` 知道要渲染什么内容
- 一个 mix = 一个独立的视频输出流

**view（obs_view）** —— 场景内容载体
- 有 64 个 channel（`MAX_CHANNELS=64`），每个 channel 放一个 source
- `obs_view_render()` 时从 channel[0] 到 channel[63] 按顺序渲染（画家算法：先画的在底层，后画的在上面）

```c
struct obs_view {
    pthread_mutex_t channels_mutex;
    obs_source_t *channels[MAX_CHANNELS];  // 64 个槽位
    enum view_type type;
};
```

**channel（通道）** —— 渲染层
- channel 0 是主内容，通常是场景源或过渡源
- channel 编号越大层级越高，后面的可以遮挡前面的
- 前端切换场景本质上就是：`obs_set_output_source(0, new_transition)`

**canvas（obs_canvas）** —— 画布，封装了 view + mix + 场景管理：
```c
struct obs_canvas {
    struct obs_view view;              // 内嵌 View
    struct obs_core_video_mix *mix;    // 内嵌 Mix
    struct obs_video_info ovi;         // 视频参数
    struct obs_source *sources;        // 属于此画布的场景/组（哈希表）
};
```

Canvas 相比纯 Mix 多的价值是 `sources` 哈希表——管理哪些 Scene 属于这个画布。纯 Mix 只有渲染链路，没有场景管理。

**关系总结：**
```
canvas ─── view ─── channels[0..63] ─── source（场景/过渡/输入源）
   │
   └─────── mix ─── render_texture, output_texture, video_t, 编码器...
```
- Canvas : View : Mix = 1 : 1 : 1（但 View 可以被多个 Display 指向）
- 无 Canvas 的 Mix 可以独立存在（通过 `obs_view_add2` 创建）
- 一个 Mix 可挂多个 Encoder（推流 x264 + 录制 NVENC + 录制 AV1）
- 同一 View 指向的多个 Mix 共享 `render_texture`（`can_reuse_mix_texture`）
- Channel 机制是早期设计残留：现在 OBS 默认只使用 ch[0] 放 Scene，ch[1]~ch[63] 通常为 NULL

### 1.3 工作室模式的工作原理

```
Main View                         Preview View
  ch[0] → Scene "直播中"            ch[0] → Scene "编辑中"

两个 View 独立渲染：
  Main View    → render_main_texture  → 推流/录制
  Preview View → 另一个 render target  → 预览窗口显示
```

`obs_view_set_source(view, 0, new_scene)` → ch[0] 指针换向 → 下一帧渲染的就是新场景。过渡时 ch[0] 指向 transition source（内部混合旧场景+新场景）。

---

## 二、图形线程入口：`obs_graphics_thread`

搞清楚概念之后，直接看图形线程的入口函数。

```c
// libobs/obs-video.c:1161
void *obs_graphics_thread(void *param)
{
    is_graphics_thread = true;  // 标记为图形线程，供其他模块通过 TLS 判断

    const uint64_t interval = obs->video.video_frame_interval_ns;  // 帧间隔（如 60fps → 16.67ms）
    obs->video.video_time = os_gettime_ns();  // 起始时间

    os_set_thread_name("libobs: graphics thread");  // 系统级线程名

    // 上下文结构体，在每帧循环中传递
    struct obs_graphics_context context;
    context.interval = interval;
    context.last_time = 0;
    // ...

    while (obs_graphics_thread_loop(&context))
        ;  // 每一帧都调用 obs_graphics_thread_loop，返回 false 时退出

    return NULL;
}
```

退出条件：当所有 mix 的 `video_output` 都停止时，`obs_graphics_thread_loop` 返回 false，线程退出。

---

## 三、核心循环：`obs_graphics_thread_loop` —— 每一帧都在做什么

这是整篇文章最核心的部分。一帧的执行顺序如下：

```c
bool obs_graphics_thread_loop(struct obs_graphics_context *context)
{
    uint64_t frame_start = os_gettime_ns();

    // ─── 阶段1: 活跃状态检测 ───
    update_active_states();

    // ─── 阶段2: 帧开始信号 ───
    gs_enter_context(obs->video.graphics);
    gs_begin_frame();
    gs_leave_context();

    // ─── 阶段3: Tick（时间推进）────
    context->last_time = tick_sources(obs->video.video_time, context->last_time);

    // ─── 阶段4: 渲染+输出所有 mix ──
    output_frames();

    // ─── 阶段5: 渲染显示窗口 ───────
    render_displays();

    // ─── 阶段6: 延迟任务执行 ───────
    execute_graphics_tasks();

    // ─── 阶段7: 性能统计 ────────
    frame_time_ns = os_gettime_ns() - frame_start;
    // 计算实际 FPS, 平均帧时间...

    // ─── 阶段8: 睡眠到下一帧 ────
    video_sleep(&obs->video, &obs->video.video_time, context->interval);

    return !stop_requested();
}
```

八个阶段，下面逐个展开。

---

## 四、阶段 1：`update_active_states()` —— 检测谁在看

### 4.1 目的

检测每个 mix 的"活跃状态"是否发生了变化。如果消费者（编码器/raw 回调）从无到有，则清零对应帧状态，确保管线从干净状态开始。

### 4.2 实现

```c
static inline void update_active_state(struct obs_core_video_mix *video)
{
    // 读取原子变量中的当前活跃计数
    bool raw_active = os_atomic_load_long(&video->raw_active) > 0;    // CPU 消费者
    const bool gpu_active = os_atomic_load_long(&video->gpu_encoder_active) > 0;  // GPU 编码器
    const bool active = raw_active || gpu_active;

    // 检测 0→1 跳变，清零帧状态
    if (!was_active && active)
        clear_base_frame_data(video);     // texture_rendered=false, cur_texture=0, 清空 vframe_info_buffer
    if (!raw_was_active && raw_active)
        clear_raw_frame_data(video);      // textures_copied[] 全置 false
    if (!gpu_was_active && gpu_active)
        clear_gpu_frame_data(video);      // 清空 vframe_info_buffer_gpu

    // 保存快照供本帧后续使用
    video->gpu_was_active = gpu_active;
    video->raw_was_active = raw_active;
    video->was_active = active;
}
```

### 4.3 `raw_active` 和 `gpu_encoder_active` 的含义

| 计数器 | 谁 +1 | 含义 |
|---|---|---|
| `raw_active` | `start_raw_video()` — 软件编码器启动、输出模块注册 | 有人需要 CPU 端原始像素数据 |
| `gpu_encoder_active` | `start_gpu_encode()` — NVENC/AMF 等硬件编码器启动 | 有人需要 GPU 端纹理数据 |

这里解释一下 **"raw"的含义**：原始像素数据（`uint8_t*`），已经从 GPU 显存的纹理对象搬到了 CPU 可访问的内存中。对应管线中最昂贵的操作——GPU→CPU 数据搬运。

### 4.4 为什么用"快照"（`raw_was_active`）？

帧开始时读一次原子变量转为 bool 快照，整帧以内都用快照值。因为原子变量随时可能被其他线程修改（编码器随时可能启动/停止），如果帧中间直接读原子变量，可能出现决策不一致。用快照保证一帧内"这条分支走还是不走"是稳定的。

---

## 五、阶段 2：帧开始信号 —— enter → begin → leave

```c
gs_enter_context(obs->video.graphics);
gs_begin_frame();
gs_leave_context();
```

这个三段式一开始看会觉得奇怪——为什么 enter 了又马上 leave？拆开看就清楚了。

### 5.1 `gs_enter_context` / `gs_leave_context`

**`thread_graphics`** 是一个线程局部变量，指向当前线程正在使用的图形设备上下文：

```c
static THREAD_LOCAL graphics_t *thread_graphics = NULL;
```

整个 `gs_*` 图形 API 系列（`gs_draw`、`gs_begin_scene`、`gs_set_render_target`...）的每一个函数，都以 `gs_valid()` 检查 `thread_graphics` 是否为空。为空则静默返回，不加锁不走 GPU：

```c
static inline bool gs_valid(const char *f) {
    if (!thread_graphics) {
        blog(LOG_DEBUG, "%s: called while not in a graphics context", f);
        return false;
    }
    return true;
}
```

`gs_enter_context` 的核心逻辑：

```c
void gs_enter_context(graphics_t *graphics)
{
    bool is_current = (thread_graphics == graphics);

    // 如果当前线程持有另一个 context → 全部清空（在 OBS 中几乎不可能发生）
    if (thread_graphics && !is_current)
        while (thread_graphics) gs_leave_context();

    // 第一次获取 → 加锁并绑定
    if (!is_current) {
        pthread_mutex_lock(&graphics->mutex);     // ← 互斥锁
        thread_graphics = graphics;
    }

    os_atomic_inc_long(&graphics->ref);  // 引用计数 +1（支持嵌套）
}

void gs_leave_context(void)
{
    if (ref 归零) {
        pthread_mutex_unlock(&thread_graphics->mutex);
        thread_graphics = NULL;
    }
}
```

引用计数支持嵌套：一个已经持有 context 的函数，内部调用了另一个也 `enter_context` 的函数，内层的 enter 跳过锁只累加 ref，内层的 leave 不会错误解锁。

**真实嵌套例子**：`obs_source_show_preloaded_video()` 先 `obs_enter_graphics()`，内部调用 `set_async_texture_size()` 又自己 `gs_enter_context()`，形成嵌套。

### 5.2 `gs_begin_frame()`

D3D11 后端下重置桌面复制（Desktop Duplication）的脏标记，让新一帧能正确检测哪些显示器有新画面。OpenGL 后端为空实现。

### 5.3 为什么要 enter→begin→leave 三段式？

`gs_begin_frame()` 需要 context 已持有才能执行。但紧接的 `tick_sources()` 是纯 CPU 操作，不应占用图形上下文锁。所以在发完帧开始信号后立即释放，等真正渲染时再重新获取。

---

## 六、阶段 3：`tick_sources()` —— 推进所有源的逻辑时钟

### 6.1 整体流程

```c
static uint64_t tick_sources(uint64_t cur_time, uint64_t last_time)
{
    // ① 计算帧间隔（秒）
    delta_time = cur_time - last_time;
    seconds = (float)((double)delta_time / 1000000000.0);

    // ② 调用全局 tick 回调
    for (tick_callbacks)
        callback->tick(callback->param, seconds);

    // ③ 从 UUID 哈希表收集所有未被 removed 的 source
    da_clear(data->sources_to_tick);
    source = data->sources;
    while (source) {
        if (!obs_source_removed(source))
            da_push_back(data->sources_to_tick, &source);
        source = source->context.hh_uuid.next;
    }

    // ④ 对每个 source 调用 obs_source_video_tick
    for (sources_to_tick)
        obs_source_video_tick(s, seconds);

    return cur_time;  // 返回当前时间，下一帧作为 last_time 计算 deltaTime
}
```

关键点：
- 遍历**所有** UUID 哈希表中的 source，不限 active——被 hide 的 source 也会 tick
- 已 removed 的 source 跳过
- `seconds` 是距上一帧的实际秒数（60fps 时约为 0.0167）

### 6.2 `obs_source_video_tick` —— 每个源每帧执行什么

这是整个 tick 体系中最核心的函数，每个 source 每帧执行一次，有 10 个步骤：

```c
void obs_source_video_tick(obs_source_t *source, float seconds)
{
    // 1. 空指针防御
    if (!obs_source_valid(source, "obs_source_video_tick")) return;

    // 2. Transition 专属 tick（推进过渡 t 值）
    if (source->info.type == OBS_SOURCE_TYPE_TRANSITION)
        obs_transition_tick(source, seconds);

    // 3. 异步源：从帧队列挑帧、去隔行、filter 链过滤
    if ((source->info.output_flags & OBS_SOURCE_ASYNC) != 0)
        async_tick(source);

    // 4. 媒体控制命令队列：播放/暂停/停止/跳转
    if ((source->info.output_flags & OBS_SOURCE_CONTROLLABLE_MEDIA) != 0)
        process_media_actions(source);

    // 5. 延迟更新：UI 线程打了标记，在图形线程安全执行 info.update()
    if (os_atomic_load_long(&source->defer_update_count) > 0)
        obs_source_deferred_update(source);

    // 6. 清空 filter 中间纹理
    if (source->filter_texrender)
        gs_texrender_reset(source->filter_texrender);

    // 7. show/hide 状态跳变检测（仅 show_refs 0↔非0 时执行）
    now_showing = !!source->show_refs;
    if (now_showing != source->showing) {
        show_source(source) 或 hide_source(source);
        同步对所有 filter 做 show/hide;
        source->showing = now_showing;
    }

    // 8. activate/deactivate 状态跳变检测（仅 activate_refs 0↔非0 时执行）
    now_active = !!source->activate_refs;
    if (now_active != source->active) {
        activate_source(source) 或 deactivate_source(source);
        同步对所有 filter 做 activate/deactivate;
        source->active = now_active;
    }

    // 9. 插件自己的 video_tick（每帧永远执行）
    if (source->context.data && source->info.video_tick)
        source->info.video_tick(source->context.data, seconds);

    // 10. 重置帧内标志（同帧多视图渲染时防止重复上传 GPU）
    source->async_rendered = false;
    source->deinterlace_rendered = false;
}
```

### 6.3 show 和 activate 的区别

这两个概念名字很像，刚看代码时容易搞混：

| 操作 | 计数器 | 触发条件 | 回调 |
|------|--------|---------|------|
| show | `show_refs` | 任何 View（主 + 预览）引用时 +1 | `info.show()` / `info.hide()` |
| activate | `activate_refs` | 仅 MAIN_VIEW 引用时 +1 | `info.activate()` / `info.deactivate()` |

```c
void obs_source_activate(obs_source_t *source, enum view_type type)
{
    os_atomic_inc_long(&source->show_refs);     // 任何 view 都 +1
    obs_source_enum_active_tree(source, show_tree, NULL);

    if (type == MAIN_VIEW) {                    // 只有主视图才
        os_atomic_inc_long(&source->activate_refs);
        obs_source_enum_active_tree(source, activate_tree, NULL);
    }
}
```

简单记：**show = "在任何一个视图里可见"，activate = "在主视图里活跃"**。预览窗口里的源只 show 不 activate。

### 6.4 帧内去重标志

```c
source->async_rendered = false;
source->deinterlace_rendered = false;
```

这两个标志确保同一帧内多个视图调用 `render_video` 时，异步纹理上传和去隔行只执行一次。第一个视图触发上传，后续视图看到标志已置 true，跳过上传直接使用已有纹理。

---

## 七、阶段 4：`output_frames()` —— 渲染+输出所有 mix

这是整帧里最重头的阶段，也是管线最长的部分。

### 7.1 外层遍历

```c
static inline void output_frames(void)
{
    pthread_mutex_lock(&obs->video.mixes_mutex);
    for (size_t i = 0; i < obs->video.mixes.num; i++) {
        struct obs_core_video_mix *mix = obs->video.mixes.array[i];
        if (mix->view)
            output_frame(mix);           // 正常渲染
        else
            obs_free_video_mix(mix);     // view 被置空 → 销毁回收
    }
    pthread_mutex_unlock(&obs->video.mixes_mutex);
}
```

### 7.2 `output_frame(mix)` — 单个 mix 的完整 GPU 管线

```c
static inline void output_frame(struct obs_core_video_mix *video)
{
    const bool raw_active = video->raw_was_active;   // 有 CPU 消费者？
    const bool gpu_active = video->gpu_was_active;   // 有 GPU 编码器？

    int cur_texture  = video->cur_texture;            // 当前写入的三缓冲索引
    int prev_texture = (cur_texture == 0) ? NUM_TEXTURES - 1 : cur_texture - 1;

    // ========== GPU 阶段 ==========
    gs_enter_context(obs->video.graphics);
    render_video(video, raw_active, gpu_active, cur_texture);

    if (raw_active)
        download_frame(video, prev_texture, &frame);   // GPU→CPU 映射
    gs_flush();
    gs_leave_context();

    // ========== CPU 阶段 ==========
    if (raw_active && frame_ready) {
        deque_pop_front(&video->vframe_info_buffer, &vframe_info, ...);
        frame.timestamp = vframe_info.timestamp;
        output_video_data(video, &frame, vframe_info.count);
    }

    // 推进三缓冲索引
    if (++video->cur_texture == NUM_TEXTURES)
        video->cur_texture = 0;
}
```

注意 `download_frame` 读的是 `prev_texture` 而非 `cur_texture`——这是三缓冲机制的核心：GPU 往当前槽位写入，CPU 从上一槽位读出，读写分离，不阻塞。

### 7.3 `render_video()` — GPU 端渲染流水线

```c
static inline void render_video(...)
{
    gs_begin_scene();

    // ① 渲染场景到 render_texture（基础分辨率），永远执行
    render_main_texture(video);

    if (raw_active || gpu_active) {
        // ② 缩放到 output_texture（输出分辨率）
        gs_texture_t *output_texture = render_output_texture(video);

        // ③ GPU 颜色空间转换 RGBA→YUV（可选）
        if (video->gpu_conversion)
            render_convert_texture(video, convert_textures, output_texture);

        // ④ 喂给 GPU 硬件编码器（可选）
        if (gpu_active)
            output_gpu_encoders(video, raw_active);

        // ⑤ 拷贝到 staging surface，准备 CPU 回读（可选）
        if (raw_active)
            stage_output_texture(video, cur_texture, convert_textures,
                                 output_texture, copy_surfaces, channel_count);
    }

    gs_set_render_target(NULL, NULL);
    gs_enable_blending(true);
    gs_end_scene();
}
```

没有消费者（`raw_active==false && gpu_active==false`）时，仅执行 ① 渲染场景，后续缩放/转换/拷贝全部跳过，节省 GPU/CPU 资源。但场景仍然会被渲染（在 `render_texture` 中），因为后续可能有别的 mix 复用。

#### 7.3.1 `render_main_texture()` — 渲染场景

```c
static inline void render_main_texture(struct obs_core_video_mix *video)
{
    // 设置画布并清空
    gs_set_render_target(video->render_texture, NULL);
    gs_clear(GS_CLEAR_COLOR, ...);

    // Pre-draw 钩子（插件可在此叠加自定义绘制）
    for (draw_callbacks)
        callback->draw(callback->param, base_width, base_height);

    // 尝试复用前一个 mix 的渲染结果
    if (can_reuse_mix_texture(video, &reuse_idx))
        draw_mix_texture(reuse_idx);          // 直接贴图，零开销！
    else
        obs_view_render(video->view);         // ★ 真正遍历 channels 渲染

    // Post-draw 钩子
    for (rendered_callbacks)
        callback->rendered(callback->param);
}
```

**`can_reuse_mix_texture`** 是一个关键优化。当多个 mix 共享同一个 view、同基础分辨率、同颜色空间时，排在后面的 mix 可以直接贴第一个 mix 渲染好的纹理，不需要重画一遍场景树。

#### 7.3.2 `render_video` —— 源的渲染决策树

`render_main_texture` 里调用的 `obs_view_render` → `obs_source_video_render` → `render_video`，这是单个 source 的渲染决策入口。每个 source 走到这里时，根据自身类型和属性走不同的渲染分支：

```c
static inline void render_video(obs_source_t *source)
{
    // 分支 1：不是 filter 且没有视频能力 → 跳过
    if (source->info.type != OBS_SOURCE_TYPE_FILTER
        && (source->info.output_flags & OBS_SOURCE_VIDEO) == 0) {
        if (source->filter_parent)
            obs_source_skip_video_filter(source);
        return;
    }

    // 分支 2：异步输入源 → 上传帧到 GPU（每帧只做一次）
    if (source->info.type == OBS_SOURCE_TYPE_INPUT
        && (source->info.output_flags & OBS_SOURCE_ASYNC) != 0
        && !source->rendering_filter) {

        if (deinterlacing_enabled(source))
            deinterlace_update_async_video(source);
        obs_source_update_async_video(source);
    }

    // 分支 3：没数据或没启用 → 跳过
    if (!source->context.data || !source->enabled) {
        if (source->filter_parent)
            obs_source_skip_video_filter(source);
        return;
    }

    // 分支 4：有 filter → 走 filter 链
    if (source->filters.num && !source->rendering_filter)
        obs_source_render_filters(source);

    // 分支 5：有 video_render → 同步源主渲染
    else if (source->info.video_render)
        obs_source_main_render(source);

    // 分支 6：是 filter 有 target → 递归渲染 target
    else if (source->filter_target)
        obs_source_video_render(source->filter_target);

    // 分支 7：需要去隔行 → 去隔行渲染
    else if (deinterlacing_enabled(source))
        deinterlace_render(source);

    // 分支 8：兜底 → 异步视频普通渲染
    else
        obs_source_render_async_video(source);
}
```

这个决策树按优先级：先判断有没有渲染能力 → 异步源上传帧 → 检查数据就绪 → filter 链优先 → 同步渲染次之 → 去隔行 → 最后兜底异步渲染。

`obs_source_main_render` 内部还会根据是否有自定义绘制和色彩空间是否匹配，决定用默认 effect 还是调用 `source_render`（含色彩空间感知转换）：

```c
static inline void obs_source_main_render(obs_source_t *source)
{
    uint32_t flags = source->info.output_flags;
    bool custom_draw = (flags & OBS_SOURCE_CUSTOM_DRAW) != 0;
    bool default_effect = !source->filter_parent
                       && source->filters.num == 0
                       && !custom_draw;

    if (default_effect)
        obs_source_default_render(source);    // 用 OBS 默认 effect
    else
        source_render(source, custom_draw ? NULL : gs_get_effect());
}
```

`source_render` 还会做色彩空间感知——如果 source 的色彩空间和当前画布不匹配，先渲染到中间纹理（source 原始色彩空间），再通过转换 shader（Draw/DrawMultiply/DrawTonemap）画到主画布。

#### 7.3.3 异步视频上传（YUV→RGB）

异步视频源（摄像头、采集卡、媒体源等）的数据是 YUV 格式，需要上传到 GPU 并转换为 RGB 纹理才能渲染：

```c
static void obs_source_update_async_video(obs_source_t *source)
{
    if (source->async_rendered) return;     // 同帧只做一次
    source->async_rendered = true;

    struct obs_source_frame *frame = source->cur_async_frame;
    if (!frame) return;

    set_async_texture_size(source, frame);

    // 不需要 GPU 转换 → 直接 upload 纹理
    if (!source->async_gpu_conversion) {
        update_async_texture(source, frame, source->async_textures[0], NULL);
        return;
    }

    // 需要 GPU 转换 → upload 原始 YUV → shader 转换
    update_async_textures(source, frame,
        source->async_textures, source->async_texrender);
}
```

YUV→RGB 转换流程：`upload_raw_frame`（原始 YUV 上 GPU）→ 根据格式选转换技术（NV12→"NV12_Reverse"、I420→"I420_Reverse"、P010 等）→ 设 shader 参数（色彩矩阵、范围、HDR）→ `gs_texrender_begin` → `gs_draw` → `gs_texrender_end` → 得到 RGBA 纹理，可直接绘制。

#### 7.3.4 Filter 链渲染机制

filter 是 OBS 中非常重要的概念——色键、缩放、LUT、锐化等都是 filter。理解它的渲染链对看懂源码很有帮助。

**数据结构：** filter 存在 source 的 `filters` 数组中，新 filter 总插在头部（`da_insert(filters, 0, &filter)`）。每个 filter 的 `filter_target` 指向链中上一个 filter（或原始 source），`filter_parent` 指向挂载的父 source。

```
Input Source "摄像头"
  │  filters: [F3, F2, F1]            ← 数组，新 filter 总插在头部
  │
  ├─ F1: filter_parent=摄像头  filter_target=摄像头
  ├─ F2: filter_parent=摄像头  filter_target=F1
  └─ F3: filter_parent=摄像头  filter_target=F2
```

**渲染入口：** 从数组头部开始：

```c
static inline void obs_source_render_filters(obs_source_t *source)
{
    first_filter = obs_source_get_ref(source->filters.array[0]);
    source->rendering_filter = true;       // 防递归标记
    obs_source_video_render(first_filter);
    source->rendering_filter = false;
    obs_source_release(first_filter);
}
```

**递归调用栈——后加的 filter 效果最后施加：**

```
render_video(F3)                            ← 最晚加的 filter，第一个被调
  ├─ process_filter_begin(F3)               ← 把 F2 渲染到中间纹理
  │   └─ obs_source_video_render(F2)
  │       └─ render_video(F2)
  │           ├─ process_filter_begin(F2)   ← 把 F1 渲染到中间纹理
  │           │   └─ obs_source_video_render(F1)
  │           │       └─ render_video(F1)
  │           │           ├─ process_filter_begin(F1)
  │           │           │   └─ 渲染原始 source 画面到纹理
  │           │           └─ process_filter_end(F1)    ← F1 效果施加 ✅
  │           └─ process_filter_end(F2)                ← F2 效果施加 ✅
  └─ process_filter_end(F3)                            ← F3 效果施加 ✅
```

**begin/end 内部：** `process_filter_begin` 把 target 渲染到中间纹理，`process_filter_end` 取中间纹理用 filter 自己的 effect 画出来。还有一个 **Bypass 优化**——当 6 个条件全满足（target 就是父源、允许直接渲染、父源不用自定义绘制、非异步视频、色彩空间兼容），跳过中间纹理，省一个 GPU pass。

**Filter 兼容性检查：** 纯音频 filter 去掉 ASYNC 标志后，source 的能力必须覆盖 filter 的需求：

```c
static bool filter_compatible(obs_source_t *source, obs_source_t *filter)
{
    uint32_t s_caps = source->info.output_flags & (OBS_SOURCE_ASYNC_VIDEO | OBS_SOURCE_AUDIO);
    uint32_t f_caps = filter->info.output_flags & (OBS_SOURCE_ASYNC_VIDEO | OBS_SOURCE_AUDIO);

    if ((f_caps & OBS_SOURCE_AUDIO) != 0 && (f_caps & OBS_SOURCE_VIDEO) == 0)
        f_caps &= ~OBS_SOURCE_ASYNC;

    return (s_caps & f_caps) == f_caps;
}
```

#### 7.3.5 去隔行子系统

隔行视频（1080i）每帧只有一半扫描线，直接显示有梳齿纹。去隔行需要**前一场 + 当前场**两帧合成一帧完整的逐行画面。

关键数据：

```c
enum obs_deinterlace_mode deinterlace_mode;
bool deinterlace_top_first;
gs_effect_t *deinterlace_effect;
struct obs_source_frame *prev_async_frame;        // 前一场（CPU 侧）
gs_texture_t *async_prev_textures[MAX_AV_PLANES]; // 前一场 GPU 纹理
gs_texrender_t *async_prev_texrender;
```

去隔行模式从低开销到高画质：

| 模式 | 效果 |
|------|------|
| DISCARD | 丢弃一半行，1x 输出，最低开销 |
| RETRO | 丢弃一半行，2x 输出（复古风格） |
| BLEND | 两场混合，1x 输出 |
| BLEND_2X | 两场混合，2x 输出 |
| LINEAR | 线性插值，1x 输出 |
| LINEAR_2X | 线性插值，2x 输出 |
| YADIF | YADIF 算法，画面质量最高 |
| YADIF_2X | YADIF 2x 输出 |

#### 7.3.6 异步帧缓冲与赶帧

异步源（采集卡、摄像头）数据到达速率不一定和渲染帧率一致。OBS 在 `ready_async_frame()` 中有两种缓冲模式：

**普通缓冲模式**——按时间戳追赶，丢掉过期的老帧：

```
系统时间 8.33ms，帧队列 [t=0, t=4.16, t=8.33]
→ 8.33 > 0    → 丢 t=0
→ 8.33 > 4.16 → 丢 t=4.16
→ 8.33 > 8.33 → false → 渲染 t=8.33
```

**无缓冲模式（unbuffered）**——永远只保留最新一帧，桌面捕获、游戏捕获默认使用，保证最低延迟。

**帧没到时：** 队列为空或帧时间戳在未来 → `cur_async_frame = NULL` → 不更新 GPU 纹理 → **重复显示上一帧**，画面定格但不会黑屏。

#### 7.3.7 `render_output_texture()` — 缩放

```c
static inline gs_texture_t *render_output_texture(struct obs_core_video_mix *mix)
{
    // 输出分辨率==基础分辨率 → 跳过，直接返回 render_texture
    if (width == base_width && height == base_height) return texture;

    // 选缩放 shader（bicubic/lanczos/area/bilinear）
    gs_effect_t *effect = get_scale_effect(mix, width, height);

    // GPU 上单次 pass 全屏四边形缩放
    gs_draw_sprite(texture, 0, width, height);
    return target;
}
```

#### 7.3.8 `render_convert_texture()` — 颜色空间转换（可选）

将 RGBA 格式的 `output_texture` 通过 GPU shader 转换为 YUV 平面（NV12/I420/P010 等），写入 `convert_textures[]` 数组：

```c
render_convert_plane(effect, convert_textures[0], "NV12_Y");   // Y 平面（亮度）
render_convert_plane(effect, convert_textures[1], "NV12_UV");  // UV 平面（色度）
```

#### 7.3.9 `output_gpu_encoders()` — 喂给硬件编码器（可选）

将 `convert_textures_encode[]` 中的 GPU 纹理直接传给 NVENC/AMF，**零拷贝**，编码器直接从显存读取。

#### 7.3.10 `stage_output_texture()` — 拷贝到 staging surface（可选）

```c
// 将（可能已转换的）纹理从显存拷贝到 staging surface
gs_stage_texture(copy_surfaces[0], convert_textures[0]);  // Y plane
gs_stage_texture(copy_surfaces[1], convert_textures[1]);  // UV plane
video->textures_copied[cur_texture] = true;  // "已发起拷贝"
```

### 7.4 `download_frame()` — GPU→CPU 映射

```c
static inline bool download_frame(struct obs_core_video_mix *video,
                                   int prev_texture, struct video_data *frame)
{
    if (!video->textures_copied[prev_texture])  // 上一帧的 staging 没完成？
        return false;                            // 不等，直接跳过

    for (int channel = 0; channel < NUM_CHANNELS; ++channel) {
        gs_stagesurface_map(surface,
            &frame->data[channel],       // 指向 Y/U/V 平面的像素数据指针
            &frame->linesize[channel]);  // 每行的字节跨度
        video->mapped_surfaces[channel] = surface;
    }
    return true;
}
```

读的是 `prev_texture`（上一帧的结果），不是当前帧。如果 GPU 还没完成异步拷贝，函数返回 false，`output_frame` 里跳过本次输出——宁丢一帧，不让图形线程阻塞等待。

### 7.5 `output_video_data()` — 推入输出

```c
static inline void output_video_data(struct obs_core_video_mix *video,
                                      struct video_data *input_frame, int count)
{
    // 从 video_t 锁定一帧缓冲
    video_output_lock_frame(video->video, &output_frame, count, timestamp);

    // 把 GPU 回读的数据（YUV planes）memcpy 到输出缓冲
    set_gpu_converted_data(&output_frame, input_frame, info);

    // 解锁 → 数据就绪 → 自动分发给所有注册的编码器和 raw callbacks
    video_output_unlock_frame(video->video);
}
```

### 7.6 完整纹理流转图

整个管线的纹理流转可以画成下面这张图，从头到尾每一步的角色都很清楚：

```
场景树 (view→channels→sources→递归渲染)
        │ obs_view_render()
        ▼
 ┌──────────────────┐
 │  render_texture  │  基础分辨率, RGBA, GPU 显存
 └──────┬───────────┘
        │ render_output_texture()
        ▼
 ┌──────────────────┐
 │  output_texture  │  输出分辨率, RGBA, GPU 显存
 └──────┬───────────┘
        │ render_convert_texture() [可选]
   ┌────┴────┐
   ▼         ▼
┌─────┐  ┌──────┐
│ Y   │  │ UV   │   NV12 planes, GPU 显存
└──┬──┘  └──┬───┘
   │        │
   ├────────────────────┐
   │ stage_output_      │ output_gpu_encoders()
   │ texture()          │
   ▼                    ▼
┌──────────────┐   ┌──────────┐
│staging surface│   │NVENC/AMF │  零拷贝
└──────┬───────┘   └──────────┘
       │ download_frame()
       ▼
┌──────────────┐
│video_data    │   CPU 内存, uint8_t*
│  .data[0]=Y  │
│  .data[1]=UV │
└──────┬───────┘
       │ output_video_data()
       ▼
┌──────────────┐
│  video_t     │   分发给软件编码器/raw callbacks/录制/推流
└──────────────┘
```

### 7.7 三缓冲机制

`NUM_TEXTURES=3`：三个槽位构成环形缓冲区。

```
cur_texture = 0 → 渲染到 copy_surfaces[0][*]
cur_texture = 1 → 渲染到 copy_surfaces[1][*]
cur_texture = 2 → 渲染到 copy_surfaces[2][*]
cur_texture = 0 → 循环...

download_frame 读 prev_texture：
  cur=0 → prev=2（读上一帧的 staging 结果）
  cur=1 → prev=0
  cur=2 → prev=1
```

一个在渲染，一个在回读，一个在输出——GPU 和 CPU 流水线不阻塞。

---

## 八、阶段 5：`render_displays()` — 渲染到屏幕上

```c
static inline void render_displays(void)
{
    gs_enter_context(obs->video.graphics);
    for (每个 display)
        render_display(display);     // 渲染到 swap chain → 显示到屏幕
    gs_leave_context();
}
```

display 通过 swap chain 将渲染结果显示在程序窗口的预览/节目区域。这一步独立于 `output_frames()`，有自己独立的 enter/leave context 配对。

---

## 九、阶段 6：`execute_graphics_tasks()` — 帮别的线程干 GPU 活

```c
static void execute_graphics_tasks(void)
{
    while (video->tasks.size) {
        struct obs_task_info info;
        deque_pop_front(&video->tasks, &info, sizeof(info));
        info.task(info.param);   // 在图形线程中执行回调
    }
}
```

其他线程（如 UI 线程、销毁线程）想要操作 GPU 资源时，不是自己去 `gs_enter_context`（那样需要和图形线程竞争锁），而是通过 `obs_queue_task(OBS_TASK_GRAPHICS, callback, param)` 把任务投递到 `obs->video.tasks` 队列。图形线程在每帧末尾统一执行。不过也有例外——源销毁线程（`destruction_task_thread`）在某些情况下确实会直接 `gs_enter_context` 来销毁 GPU 资源（如纹理），通过 mutex 和图形线程互斥。

---

## 十、阶段 7 & 8：帧结束与睡眠

```c
// 统计帧时间、计算实际 FPS
frame_time_ns = os_gettime_ns() - frame_start;
// ...

// 睡眠到下一帧的时间点
video_sleep(&obs->video, &obs->video.video_time, context->interval);
```

### `video_sleep()` — 帧定时器

```c
static inline void video_sleep(...)
{
    if (os_sleepto_ns(t)) {
        count = 1;  // 准时
    } else {
        // 超时了！计算错过了多少帧
        count = (clamped_diff / interval_ns);
    }

    video->total_frames += count;
    video->lagged_frames += count - 1;  // 丢了多少帧

    // 预先将时间戳信息入队到每个 mix 的 vframe_info_buffer
    vframe_info.timestamp = cur_time;
    vframe_info.count = count;
    for (每个 mix) {
        if (raw_active)
            deque_push_back(&mix->vframe_info_buffer, &vframe_info, ...);
        if (gpu_active)
            deque_push_back(&mix->vframe_info_buffer_gpu, &vframe_info, ...);
    }
}
```

时间戳在帧末尾入队，下一帧的 `output_frame()` 中出队——确保时间戳和帧数据保持顺序一致。

---

## 十一、完整帧时间线

把前面八个阶段串起来，一帧的完整时间线长这样：

```
帧开始
 │
 ├─ update_active_states()             ← 检测活跃状态变化
 │
 ├─ gs_enter_context → gs_begin_frame → gs_leave_context   ← 帧信号
 │
 ├─ tick_sources(video_time, last_time)  ← 计算 deltaTime，推进所有源
 │
 ├─ output_frames()                     ← GPU 渲染所有 mix
 │   └─ for each mix:
 │       ├─ [enter context]
 │       ├─ render_video() → 渲染→缩放→转换→编码→staging
 │       ├─ download_frame() → GPU→CPU 映射
 │       ├─ [leave context]
 │       └─ output_video_data() → 推入 video_t 分发给编码器
 │
 ├─ render_displays()                   ← 渲染到预览/节目窗口
 │
 ├─ execute_graphics_tasks()            ← 执行别的线程投递的 GPU 任务
 │
 ├─ 计算 frame_time, 统计 FPS
 │
 └─ video_sleep()                       ← 睡眠到下一帧，提前入队时间戳
 │
 ▼ 下一帧开始
```

---

## 十二、几个值得记住的设计要点

看完整条管线之后，有几个设计决策我觉得特别巧妙，单独列出来。

### 12.1 按需渲染

管线的后半段（缩放、颜色转换、编码、staging、回读）全是有消费者才执行：
- `raw_active > 0`：走 CPU 回读路径
- `gpu_encoder_active > 0`：走 GPU 编码路径
- 两者都为零：节省 GPU/CPU 资源

### 12.2 纹理复用

`can_reuse_mix_texture` 让多个 mix（共享 view）避免重复渲染场景树，直接贴图，零开销。

### 12.3 三缓冲异步读写

GPU 写入 `cur_texture` 槽位，CPU 从 `prev_texture` 槽位读出，GPU 异步拷贝中不阻塞图形线程。

### 12.4 图形上下文引用计数

`gs_enter_context`/`gs_leave_context` 的嵌套支持让函数可以无脑地 `enter → 干活 → leave`，不关心调用者是否已持有 context。

### 12.5 线程安全

- `thread_graphics`（TLS）+ `graphics_subsystem.mutex`：同一时刻只有一个线程操作 GPU 状态
- `video->mixes_mutex`：保护 mix 数组的并发访问
- `vframe_info_buffer` 队列：图形线程在帧末尾入队时间戳，帧中间出队绑定帧数据

---

## 小结

读 `obs-video.c` 的这几天，最大的感受是 OBS 的图形渲染线程就是一个典型的**游戏引擎式帧循环**。如果你之前接触过 Unity 或 Unreal 的主循环，看这段代码会有很强的既视感：

1. **DeltaTime** 驱动所有源的逻辑更新（tick）
2. **画家算法**按 channel 顺序渲染场景树（递归渲染）
3. **按需输出**——有消费者才走缩放/转换/回读/编码
4. **三缓冲**保证 GPU-CPU 流水线不阻塞
5. **纹理复用**优化多输出场景

整个管线的设计思路其实很朴素：**每一帧就是一次完整的"推进→画→输出→显示→睡"循环**。代码虽然长，但八个阶段的边界非常清晰，顺着 `obs_graphics_thread_loop` 一路读下来就不会迷路。

核心文件索引，方便以后回看：

| 文件 | 内容 |
|---|---|
| `libobs/obs-video.c` | 图形线程主循环、render_video、tick_sources、output_frame、render_main_texture |
| `libobs/obs-source.c` | obs_source_video_tick、render_video 决策树、filter 链渲染、异步上传、去隔行 |
| `libobs/obs-source-transition.c` | Transition 过渡逻辑 |
| `libobs/obs-source-deinterlace.c` | 去隔行实现 |
| `libobs/obs-view.c` | View 抽象（channel 映射） |
| `libobs/obs-display.c` | Display 窗口渲染 |
| `libobs/obs.c` | 全局初始化、start/stop_raw_video、obs_queue_task |
| `libobs/graphics/graphics.c` | gs_enter_context/gs_begin_frame 等图形子系统 API |
| `libobs/graphics/texture-render.c` | gs_texrender（离屏渲染目标） |
| `libobs/obs-internal.h` | obs_core_video_mix、obs_view、obs_core、obs_source 等全部内部结构体定义 |
| `libobs/obs-scene.c` | Scene 和 SceneItem 实现 |

---

学习过程中参考了 OBS Studio 官方源码仓库，感谢开源社区。如有理解偏差，欢迎指正交流。
