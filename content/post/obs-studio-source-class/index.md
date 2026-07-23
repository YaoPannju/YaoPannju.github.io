---
title: "OBS Studio 源码学习（四）：source 类详解"
description: "深入 OBS Studio 最核心的 obs_source 结构体，从 C 语言模拟继承、三层标志位、信号系统、渲染决策树到 filter 链机制，全景拆解 source 类的设计与实现。"
date: 2026-07-21
tags: ["OBS", "源码学习", "C++", "obs-studio"]
categories: ["OBS Studio源码学习"]
image: ""
---

前三篇文章分别梳理了主线程启动流程、图形渲染线程和音频线程。这三条线最终都汇集到同一个核心对象上——**source**。图形线程每帧遍历 source 做渲染，音频线程从 source 取数据做混音，主线程负责创建/销毁/配置 source。可以说，理解了 source 类，就理解了 OBS 整个数据管线的枢纽。

这篇文章是我读 `obs-source.c`（6005 行）、`obs-internal.h` 和相关文件的笔记，从数据结构、生命周期、信号系统、渲染逻辑到底层机制，尽可能完整地拆解 `obs_source` 的设计。

核心源文件：
- `libobs/obs-internal.h` — `obs_source` 结构体完整定义（行 807-1004）
- `libobs/obs-source.c` — 主实现（6005 行）
- `libobs/obs-source.h` — `obs_source_info` 定义 + 公共枚举
- `libobs/obs-source-transition.c` — Transition 源实现
- `libobs/obs-source-deinterlace.c` — 去隔行实现

---

## 一、类属关系：C 语言如何模拟继承

OBS 用纯 C 实现了一套面向对象机制，核心手法很简单：**把基类结构体作为子类的第一个成员**。

### 1.1 内存布局技巧

因为 C 语言保证结构体第一个成员的地址等于结构体本身的地址，所以：

```c
// "向上转型"是安全的
(obs_context_data *)source == &source->context
```

不需要任何偏移计算，一个强制类型转换就能把 `obs_source *` 当 `obs_context_data *` 用。这个技巧在 OBS 里到处都能看到——`obs_output`、`obs_encoder`、`obs_service` 全都用同样的方式继承 `obs_context_data`。

### 1.2 继承关系总览

```
obs_context_data（基类）
  ├── obs_source        ← 本文主角
  │     ├── obs_scene（场景是特殊 source）
  │     └── obs_scene_item（场景项，引用 source）
  ├── obs_output
  ├── obs_encoder
  ├── obs_service
  └── obs_canvas
```

`obs_scene` 本身也是一个 source——它的 `info.type` 是 `OBS_SOURCE_TYPE_SCENE`，只是多了场景特有的方法（添加/移除/排序子项）。这种"场景即源"的设计让场景可以像普通源一样被 filter 处理、被 transition 过渡，非常灵活。

### 1.3 obs_context_data — 基类提供了什么

```c
struct obs_context_data {
    char *name;                       // 对象名
    const char *uuid;                 // 唯一标识
    void *data;                       // 插件私有数据
    obs_data_t *settings;             // 设置（JSON 树）
    signal_handler_t *signals;        // 信号系统
    proc_handler_t *procs;           // 过程调用系统
    enum obs_obj_type type;           // SOURCE/OUTPUT/ENCODER/SERVICE/CANVAS

    struct obs_weak_object *control;  // 引用计数控制器
    obs_destroy_cb destroy;           // 虚析构函数

    DARRAY(obs_hotkey_id) hotkeys;
    obs_data_t *hotkey_data;

    pthread_mutex_t *mutex;           // 全局链表的锁
    struct obs_context_data *next;    // 链表指针
    struct obs_context_data **prev_next;

    UT_hash_handle hh;                // 按名称的哈希表句柄
    UT_hash_handle hh_uuid;           // 按 UUID 的哈希表句柄

    bool private;                     // 是否私有
};
```

基类提供了一套"标准装备"：

| 机制 | 实现方式 |
|------|----------|
| 引用计数 | `control` 指向 `obs_weak_object`，内含原子 `refs` + `weak_refs` |
| 信号系统 | `signals` 是 `signal_handler_t*` |
| 过程调用 | `procs` 是 `proc_handler_t*` |
| 全局查找 | `hh`（按名称）和 `hh_uuid`（按 UUID）两个 uthash 句柄 |
| 对象识别 | `type` 标记 SOURCE/OUTPUT/ENCODER/SERVICE/CANVAS |
| 虚析构 | `destroy` 函数指针 |

### 1.4 构造顺序

source 的构造分三步走：

```
1. obs_context_data_init(&context, ...)
   → 分配信号/过程处理器、生成 UUID、拷贝名称和 settings

2. 子类自己的初始化
   → 分配音频缓冲、初始化 transition 等

3. obs_context_init_control(&context, source, obs_source_destroy)
   → 创建弱引用、设置虚析构函数（指向 obs_source_destroy）
```

第三步的 `obs_source_destroy` 就是 C 语言版的"虚析构函数"——当引用计数归零时，OBS 通过这个函数指针回调，释放 source 占用的所有资源。

---

## 二、obs_source 结构体全景

`obs_source` 定义在 `libobs/obs-internal.h:807-1004`，跨度近 200 行。按功能可以分成几个区域。

### 2.1 身份与类型

```c
struct obs_context_data context;  // 基类
struct obs_source_info info;      // 从注册信息拷贝来的虚函数表
uint32_t flags;                   // 运行时标志（用户可修改）
uint32_t default_flags;           // 默认标志
uint32_t last_obs_ver;            // 创建时的 OBS 版本
bool owns_info_id;                // 是否拥有 info.id 字符串的内存所有权
```

**`owns_info_id`** 是一个容易忽略但重要的细节。正常创建 source 时，`info = *注册表条目`，`info.id` 指向注册表里的**静态字符串常量**，不需要额外管理内存。但如果找不到对应的插件（比如这个 source 来自别的机器、插件没有安装），OBS 会创建一个"空壳 source"，此时需要用 `bstrdup(id)` 自己分配字符串内存，把 `owns_info_id` 设为 `true`，销毁时自己释放。这种"找不到也不崩"的容错设计在 OBS 里很常见。

### 2.2 生命周期与状态

| 字段 | 含义 |
|------|------|
| `show_refs`（原子） | 被多少个视图引用（>0 触发 `show()` 回调） |
| `activate_refs`（原子） | 被多少个主视图引用（>0 触发 `activate()` 回调） |
| `destroying`（原子） | 防止重复销毁 |
| `removed` | 已从全局列表逻辑删除，内存还在（等引用归零） |
| `temp_removed` | 临时隐藏（不在 UI 显示） |
| `active` / `showing` | 缓存的当前状态（用于检测变化） |
| `enabled` | 用户是否启用 |
| `defer_update_count`（原子） | 延迟更新令牌数 |

**`defer_update_count`** 的设计很巧妙。视频源的设置变更（比如改分辨率）不应该在 UI 线程直接调 `info.update()`——那可能和图形线程的渲染产生竞争。OBS 的做法是：UI 线程只是原子地把 `defer_update_count` +1 打个标记，然后下一帧 tick 时图形线程检查这个计数器，安全地执行实际的 update。具体用 CAS 操作：图形线程尝试清零，如果中途 UI 线程又打了新标记，CAS 失败，保留新值，下帧重试。

### 2.3 异步视频管道

异步源（摄像头、采集卡、媒体文件）不在图形线程里产生帧，帧由独立的采集线程/解码线程推送进来。OBS 需要把这种"外部推送→内部消费"的异步关系管理好。

| 字段 | 含义 |
|------|------|
| `async_frames` | 待渲染的帧队列（动态指针数组） |
| `async_cache` | 帧对象缓存池 |
| `cur_async_frame` | 当前帧（CPU 侧数据指针） |
| `async_textures[MAX_AV_PLANES]` | 当前帧的 GPU 纹理 |
| `async_texrender` | YUV→RGB 转换的渲染目标 |
| `async_active` | 是否有可渲染的异步帧（传入 NULL 帧 → false） |
| `async_unbuffered` | 无缓冲模式，永远只要最新一帧（桌面捕获） |
| `async_decoupled` | 解耦模式，音频不跟视频同步 |
| `async_update_texture` | 本帧是否需要重建 GPU 纹理 |
| `async_format` | 视频像素格式 |
| `async_width/height` | 异步视频尺寸 |
| `async_rotation` | 旋转角度 |
| `async_flip` | 是否翻转 |
| `async_preload_frame` | 预加载帧 |

**preload + show 机制**是为媒体源设计的。OBS 可以先用 `preload_video()` 把首帧上传到 GPU 但不标记 `async_active`（此时用户看不到），等播放真正开始时，`show_preloaded_video()` 一键激活——首帧零延迟，不黑屏。这在直播场景切换媒体源时体验很好。

### 2.4 去隔行

隔行视频（interlaced video）需要把奇偶两场合成一帧逐行画面，这就需要保留"前一场"来做参考：

```c
enum obs_deinterlace_mode deinterlace_mode;
bool deinterlace_top_first;                       // 场序
gs_effect_t *deinterlace_effect;                  // 去隔行着色器
struct obs_source_frame *prev_async_frame;        // 前一场（CPU 侧）
gs_texture_t *async_prev_textures[MAX_AV_PLANES]; // 前一场的 GPU 纹理
gs_texrender_t *async_prev_texrender;             // 前一场的转换目标
```

支持的算法从简到繁：DISCARD、RETRO、BLEND、BLEND_2X、LINEAR、LINEAR_2X、YADIF、YADIF_2X。YADIF 是质量最好的实时去隔行算法，但计算量也最大。

### 2.5 音频处理

| 字段 | 含义 |
|------|------|
| `audio_input_buf[MAX_AUDIO_CHANNELS]` | 环形缓冲区（每声道一个 deque） |
| `user_volume` / `volume` | 用户音量 / 实际音量 |
| `user_muted` / `muted` | 用户静音 / 实际静音 |
| `balance` | 声相平衡 (0.0=左, 0.5=中, 1.0=右) |
| `push_to_mute_*` / `push_to_talk_*` | 一键静音/通话 |
| `sync_offset` | 音视频同步偏移（纳秒） |
| `audio_mixers` | 输出到哪些混音器（位掩码） |
| `resampler` | 音频重采样器 |
| `monitoring_type` | 无/仅监听/监听并输出 |

音频部分在第三篇文章里已经详细梳理过了，这里不再展开。需要注意的一点是 `audio_mixers` 是位掩码——一个 source 可以同时输出到多个混音器（比如同时输出到主混音和预览混音）。

### 2.6 Filter 链

```c
struct obs_source *filter_parent;     // 此 filter 附着的父 source
struct obs_source *filter_target;     // 渲染目标（链上的下一个）
DARRAY(struct obs_source *) filters;  // 附着在此 source 上的 filter 列表
gs_texrender_t *filter_texrender;    // filter 链的中间渲染目标
bool rendering_filter;               // 防止递归渲染
bool filter_bypass_active;           // 优化：直接渲染（省一个 GPU pass）
```

filter 链的设计是 OBS 里我最喜欢的一部分——简洁但功能强大。后面第七节会详细拆解。

### 2.7 Transition 专用

```c
obs_source_t *transition_sources[2];   // 过渡的两个源 [from, to]
float transition_manual_val;           // 过渡进度 0.0~1.0
float transition_manual_target;        // 目标进度
float transition_manual_torque;        // 平滑过渡速度
float transition_manual_clamp;         // 最小步长（防止末尾爬行）
gs_texrender_t *transition_texrender[2]; // 两个源的纹理
bool transitioning_video;              // 视频正在过渡
bool transitioning_audio;              // 音频正在过渡
```

`transitioning_video` 和 `transitioning_audio` 是分开的 bool——视频动画可能先于音频交叉淡入淡出完成（或反之），各管各的。两者**都**结束时才调 `obs_transition_stop`。

`transition_manual_clamp` 这个字段很有意思：transition 的 t 值从 0 平滑过渡到 1 时，越接近 1 步长越小，理论上永远不会精确到达 1.0。`clamp` 设一个最小步长阈值——当剩余距离小于这个值时，直接跳到目标，避免"永远差一点"的爬行现象。

---

## 三、三层标志位

OBS 用三层标志位来控制 source 的行为，分层的设计让"类型能力"、"默认值"和"运行时状态"各管各的，互不污染：

| 层次 | 字段 | 谁设置 | 作用 |
|------|------|--------|------|
| 类型能力 | `info.output_flags` | 插件注册时 | 描述这个类型"能做什么"（不变） |
| 默认值 | `default_flags` | 插件/前端 | 新实例的初始值 |
| 运行时 | `flags` | 用户/前端 | 控制这个实例的行为（目前只有 `FORCE_MONO`） |

举个例子：一个麦克风源的 `info.output_flags` 里永远有 `OBS_SOURCE_AUDIO`，这是它的"类型能力"；`default_flags` 可以设为 `OBS_SOURCE_FLAG_FORCE_MONO` 作为默认；用户在 UI 里关闭"强制单声道"后，`flags` 就变成 0。

---

## 四、`private` 标志的影响

source 可以标记为 `private`，这会改变它在全局系统中的"可见性"：

| 影响 | private=true | private=false |
|------|-------------|---------------|
| 加入 `public_sources`（按名称哈希表） | ❌ | ✅ |
| 加入 `sources`（按 UUID 哈希表） | ✅ | ✅ |
| `obs_enum_sources()` 枚举 | ❌ | ✅ |
| `obs_get_source_by_name()` 查找 | ❌ | ✅ |
| 保存到场景 JSON | ❌ | ✅ |
| 发射全局信号 | ❌ | ✅ |
| 发射自己的信号 | ✅ | ✅ |

Filter 永远被强制设 `private = true`——它只附着在 source 上，不应该在全局列表中独立出现，被用户当作独立的源来操作是没有意义的。

注意 private source **仍然在 UUID 哈希表里**，所以可以通过 UUID 精确定位到它。`private` 控制的只是"按名称查找"和"枚举"这两个面向普通用户的入口。

---

## 五、信号系统

### 5.1 两级信号

OBS 的信号分两层——**全局信号**像是系统广播，**对象信号**像是私人通知。两者会同时触发：

```c
static inline void obs_source_dosignal(source, signal_obs, signal_source)
{
    if (signal_obs && !source->context.private)
        signal_handler_signal(obs->signals, signal_obs, &data);   // 全局广播
    if (signal_source)
        signal_handler_signal(source->context.signals, signal_source, &data); // 对象推送
}
```

可以看到，private source 不会触发全局信号——这避免了内部 filter 的销毁事件惊动前端的 UI 更新逻辑。

### 5.2 signal_handler 内部结构

```c
struct signal_handler {
    struct signal_info *first;      // 按信号名分拣的单向链表
    pthread_mutex_t mutex;          // 保护 first 链表
    volatile long refs;

    DARRAY(struct global_callback_info) global_callbacks;  // 统吃：不管什么信号都触发
    pthread_mutex_t global_callbacks_mutex;                 // 保护全局回调
};
```

两种回调模式：
- **精确匹配**：`first` 链表按信号名索引——`"destroy"`→[回调A,B,C]，`"volume"`→[回调D]。触发时只调用对应的链表。
- **全局统吃**：`global_callbacks` 数组里的回调不管什么信号名都触发，回调函数通过参数拿到信号名字符串自行判断。脚本系统和 API 的全局观察者用这个。

### 5.3 使用方式

```c
signal_handler_connect(
    signal_handler_t *handler,  // 1. 挂在谁身上（从 obs_source_get_signal_handler 拿）
    const char *signal,         // 2. 监听哪个信号名
    signal_callback_t callback, // 3. 回调函数 void (void *data, calldata_t *params)
    void *data                  // 4. 回调时原样带回的用户数据（不需要传 NULL）
);
```

### 5.4 proc_handler — 过程调用

信号是 push 模式（source 通知你），过程调用是 **pull / 请求-响应** 模式（你主动问 source）：

```c
// 注册（在插件 create 里）
proc_handler_add(ph, "void current_index(out int idx)", current_slide_proc, ss);

// 调用（前端或其他代码里）
proc_handler_call(ph, "current_index", &cd);
int idx = calldata_int(&cd, "current_index");
```

声明字符串 `"void current_index(out int idx)"` 定义了过程名、参数名、参数方向和类型。OBS 自己解析这个字符串来做参数校验和序列化，比手写样板代码方便。

---

## 六、obs_source_info — 源类型定义表

`obs_source_info`（定义在 `obs-source.h:222-558`）是插件注册源类型时填写的"虚函数表"。OBS 在创建 source 时会把这张表**完整拷贝**到 `obs_source.info` 里，之后所有操作都通过这个表调度。

核心回调分类：

| 类别 | 回调 |
|------|------|
| **必须** | `id`, `type`, `output_flags`, `get_name`, `create`, `destroy`, `get_width`, `get_height` |
| 设置 | `get_defaults`, `get_defaults2`, `get_properties`, `get_properties2`, `update` |
| 生命周期 | `activate`, `deactivate`, `show`, `hide`, `save`, `load` |
| 视频 | `video_tick`, `video_render`, `filter_video` |
| 音频 | `audio_render`, `audio_mix`, `filter_audio` |
| 交互 | `mouse_click`, `mouse_move`, `mouse_wheel`, `focus`, `key_click` |
| 过渡 | `transition_start`, `transition_stop` |
| 媒体 | `media_play_pause`, `media_restart`, `media_stop`, `media_next/previous`, `media_get_duration/time/state` |
| 色彩 | `video_get_color_space` |
| Filter | `filter_add`, `filter_remove` |
| 图标 | `icon_type`, `get_dark_icon`, `get_light_icon` |

这张表很庞大但不是每个插件都要全填——`type` 决定了哪些回调有意义。比如 `OBS_SOURCE_TYPE_INPUT` 的源需要填 `video_render`/`audio_render`，但不需要填 `filter_video`/`filter_audio`；filter 类型正好反过来。

---

## 七、核心函数详解

### 7.1 obs_source_create_internal — 创建流程

这是 source 创建的完整路径：

```
1. bzalloc 分配 struct obs_source
2. get_source_info(id) 查找注册信息
   → 找到：source->info = *info（完整拷贝虚函数表）
   → 找不到：bstrdup(id)，owns_info_id = true（空壳 source）
3. obs_source_init_context（信号/过程处理器/UUID/名称/settings）
4. 调用 get_defaults / get_defaults2
5. 分配音频缓冲（如果是音频源）
6. 初始化 transition（如果是 transition 类型）
7. obs_context_init_control（引用计数 + 虚析构）
8. 注册热键
9. 调用 info.create(settings, source) → context.data
10. source->flags = default_flags
11. 插入全局哈希表（UUID 全部入，名称仅 public 入）
12. 发射 "source_create" 信号
```

第 9 步调 `info.create` 时，传进去的 `source` 对象已经有了完整的 settings、信号系统和 UUID——插件可以在 `create` 里立刻注册信号监听者、读取初始设置，不需要等"二次初始化"。`create` 的返回值存入 `context.data`，之后每次回调都会把这个 data 传回给插件。

### 7.2 obs_source_video_tick — 每帧状态更新

这是图形线程每帧对每个活跃 source 调用的核心函数。`float seconds` 是距上一帧的秒数，用于时间驱动的逻辑（transition 进度、媒体播放位置等）。

```c
void obs_source_video_tick(obs_source_t *source, float seconds)
{
    // 1. 空指针防御
    if (!obs_source_valid(source, "obs_source_video_tick")) return;

    // 2. Transition 专属 tick（推进过渡 t 值）
    if (source->info.type == OBS_SOURCE_TYPE_TRANSITION)
        obs_transition_tick(source, seconds);

    // 3. 异步源：从帧队列挑帧、去隔行、filter 链异步过滤
    if ((source->info.output_flags & OBS_SOURCE_ASYNC) != 0)
        async_tick(source);

    // 4. 媒体控制命令：播放/暂停/停止/跳转
    if ((source->info.output_flags & OBS_SOURCE_CONTROLLABLE_MEDIA) != 0)
        process_media_actions(source);

    // 5. 延迟更新：UI 线程打的标记，在图形线程安全执行 info.update()
    if (os_atomic_load_long(&source->defer_update_count) > 0)
        obs_source_deferred_update(source);

    // 6. 清空 filter 中间纹理
    if (source->filter_texrender)
        gs_texrender_reset(source->filter_texrender);

    // 7. show/hide 状态跳变检测（仅 show_refs 0↔非0 时执行）
    now_showing = !!source->show_refs;
    if (now_showing != source->showing) { show/hide_source + filter 同步; }

    // 8. activate/deactivate 状态跳变检测（仅 activate_refs 0↔非0 时执行）
    now_active = !!source->activate_refs;
    if (now_active != source->active) { activate/deactivate_source + filter 同步; }

    // 9. 插件自己的 video_tick
    if (source->context.data && source->info.video_tick)
        source->info.video_tick(source->context.data, seconds);

    // 10. 重置帧内标志（保证同帧多次渲染不重复上传 GPU）
    source->async_rendered = false;
    source->deinterlace_rendered = false;
}
```

这里有几点值得注意：

- **步骤顺序是有意安排的**：先 transition tick（可能改变 source 引用），再异步 tick（挑选当前帧），然后才执行插件的 video_tick。这个顺序保证了插件看到的数据状态是一致的。
- **步骤 7 和 8 做的是"边沿检测"**：只有当 `show_refs` 从 0 变正数或从正数变 0 时，才会触发实际的 show/hide 回调。不是每帧都调用。
- **步骤 10** 重置 `async_rendered` 和 `deinterlace_rendered`，这两个标志位保证同帧内即使多次调用 `render_video`，异步帧也只会上传一次 GPU，去隔行也只执行一次。

### 7.3 render_video — 渲染决策树

```c
static inline void render_video(obs_source_t *source)
{
    // 分支 1：没视频能力的非 filter 源 → 跳过
    if (不是filter && 没视频) { skip_video_filter 或 return; }

    // 分支 2：异步输入源 → 上传帧到 GPU（每帧只做一次）
    if (是输入源 && 是异步 && 不在filter渲染中) {
        去隔行 → deinterlace_update_async_video;
        obs_source_update_async_video;   // YUV→RGB 转换
    }

    // 分支 3：没数据或没启用 → 跳过
    if (!context.data || !enabled) { skip_video_filter 或 return; }

    // 分支 4：有 filter → 走 filter 链渲染
    if (filters.num && !rendering_filter)
        obs_source_render_filters(source);

    // 分支 5：有 video_render → 同步源主渲染
    else if (video_render) obs_source_main_render(source);

    // 分支 6：是 filter 有 target → 渲染 target
    else if (filter_target) obs_source_video_render(filter_target);

    // 分支 7：去隔行 → 去隔行渲染
    else if (deinterlacing_enabled) deinterlace_render(source);

    // 分支 8：兜底 → 异步视频普通渲染
    else obs_source_render_async_video(source);
}
```

这是一个典型的**优先级决策树**：filter 链 > 同步渲染 > filter 递归 > 去隔行 > 异步渲染。每个 source 只会走一条路径，不会有"既走 filter 链又单独渲染"的情况。

---

## 八、Filter 链机制

这是 source 系统里最精巧的设计之一。

### 8.1 数据结构

```
Input Source "摄像头"
  │  filters: [F3, F2, F1]      ← 数组头=最后加的=离屏幕最近
  │  filter_parent: NULL
  │
  ├─ F1: filter_parent=摄像头, filter_target=摄像头
  ├─ F2: filter_parent=摄像头, filter_target=F1
  └─ F3: filter_parent=摄像头, filter_target=F2
```

- 新 filter 永远 `da_insert(filters, 0)` 插到数组**头部**
- `filter_target` 的规则：`!filters.num ? source : filters.array[0]`

这意味着：
- 第一个被加的 filter（F1）的 target 指向原始 source
- 最后加的 filter（F3）在数组最前面，它的 target 指向 F2

### 8.2 渲染调用栈

filter 链的渲染是**嵌套递归**的：

```
render_video(F3)
  ├─ process_filter_begin(F3)      ← 把 F2 渲染到中间纹理
  │   └─ render_video(F2)
  │       ├─ process_filter_begin(F2) ← 把 F1 渲染到中间纹理
  │       │   └─ render_video(F1)
  │       │       ├─ process_filter_begin(F1) ← 把原始 source 渲染到纹理
  │       │       └─ process_filter_end(F1)   ← 施加 F1 效果 ✅
  │       └─ process_filter_end(F2)           ← 施加 F2 效果 ✅
  └─ process_filter_end(F3)                   ← 施加 F3 效果 ✅
```

**结论：先加的 filter 靠近原始画面，后加的靠近屏幕。最后加的 filter 第一个被调用，但因为 begin/end 包裹递归，它的效果实际是最后施加的。** 这就是"从上到下"的 UI 顺序对应的底层逻辑。

### 8.3 Bypass 优化

六个条件**全部**满足时，filter 可以省一个 GPU pass（不创建中间纹理，直接渲染）：

1. target == parent（filter 目标就是父源本身）
2. allow_direct == OBS_ALLOW_DIRECT_RENDERING
3. 父源不用自定义绘制
4. 父源不是异步视频
5. filter 与 parent 的 SRGB 标志一致
6. 色彩空间一致

不满足任何一条都要老老实实渲染到中间纹理再处理，多一个 GPU pass 的开销。

### 8.4 递归防护

`rendering_filter` 标志位防止一种边界情况：filter 的 `filter_video` 回调里不小心又去渲染了同一个 source，导致无限递归。这个标志位在进入 filter 链时设为 true，退出时恢复。

---

## 九、View 与 Scene 与 Source 的关系

这是三个容易搞混的概念：

| | Source | Scene | View |
|---|---|---|---|
| 本质 | 内容生产者 | 特殊 source（容器） | 频道映射器 |
| 有 obs_context_data | ✅ | ✅（通过包裹的 source） | ❌ |
| 有 UUID/信号/引用计数 | ✅ | ✅ | ❌ |
| 作用 | 生产/处理音视频 | 管理子 source 的层次排列 | 选频道→source 的映射 |

用一句话概括：
- **Source** 是**内容从哪来**（摄像头、图片、文本）
- **Scene** 是**这堆源怎么排列**（谁在上谁在下、位置和大小）
- **View** 是**这个频道现在播哪个 scene**（选台器）

View 不是 source，就是三字段的轻量结构：`{ channels[6], type, mutex }`。工作室模式的预览窗口和主输出本质上就是同一个 scene 放到不同的 View 上渲染——View 只是一个"指向 scene 的指针"。

---

## 十、帧队列与赶帧机制

### 10.1 普通缓冲模式

异步帧按时间戳排队，tick 时根据当前系统时间挑最合适的帧：

```
系统时钟 8.33ms，帧队列 [t=0, t=4.16, t=8.33]
→ 8.33 > 0 → 丢 t=0      （太老了）
→ 8.33 > 4.16 → 丢 t=4.16 （也太老了）
→ 8.33 > 8.33（false）  → 渲染 t=8.33 ✅
```

"过期"的定义是帧的时间戳小于当前系统时间。丢掉过期帧后，队列里最老的那帧就是应该渲染的帧。如果队列全过期了，就挑时间戳最大的那个（总比没有好）。

### 10.2 无缓冲模式

桌面捕获和游戏捕获用无缓冲模式——永远只保留最新一帧，前面的全扔。因为这些源帧率高、对延迟敏感，缓冲会引入额外延迟，用户操作不同步的感觉很明显。

### 10.3 帧还没到时

没有合适的帧 → `cur_async_frame = NULL` → GPU 纹理不更新 → **重复显示上一帧**。如果从来没有过帧，整个渲染跳过，画面保持透明/黑色。

---

## 十一、obs_source_duplicate — 复制粘贴

`obs_source_duplicate` 不是简单的指针复制，是**深拷贝**：

1. **复制粘贴源**（右键→复制→粘贴副本）：深拷贝，独立的 settings 和 filter 链
2. **场景复制**：已在目标场景里的源只加引用，不在的独立复制
3. **复制 filter**：复制源时身上的 filter 链逐个深拷贝
4. **DO_NOT_DUPLICATE 标志**：硬件设备源（如采集卡）不支持拷贝，直接加引用（一个物理设备不能被两个 source 同时独占）

---

## 十二、obs_source_get_id — 类型识别

返回源的类型字符串（如 `"dshow_input"`、`"ffmpeg_source"`），前端用这个信息做很多事：

- 根据类型显示不同图标（摄像头图标 vs 图片图标）
- 根据类型显示不同 UI 控件（Freetype 文本源有字体选择器）
- 判断能否添加某种 filter
- 复制粘贴时保留过渡类型
- 判断是否是 group 做特殊处理

本质上 `id` 是**类型元数据的 key**——前端拿这个 key 去查注册表、查配置模板、查 UI 布局。

---

## 十三、obs_source_output_video — 帧提交流程全景

把帧从采集线程推进异步队列，到最终显示在屏幕上，这个全链路涉及多个阶段：

```
obs_source_output_video(source, frame)
  → cache_video：把帧数据缓存到 async_cache 池
  → da_push_back(async_frames)：帧指针入队
  → async_active = true

下一帧 tick：
  async_tick()
    → get_closest_frame(sys_time)：按系统时间挑帧
    → filter_frame()：送 filter 链做异步视频过滤
    → set_async_texture_size()：检查尺寸

同帧 render：
  obs_source_update_async_video()
    → update_async_textures()
      → upload_raw_frame()：上传原始数据为 GPU 纹理
      → gs_texrender + 转换 shader：YUV→RGB
    → async_rendered = true
  obs_source_render_async_video()
    → gs_draw_sprite：画到屏幕
```

关键节奏是：**采集线程 push → 图形线程 tick 挑选 → 图形线程 render 上传+绘制**。三个阶段在不同的时间点执行，通过 `async_frames` 队列和 `async_rendered` 标志位协调。

---

## 十四、关键文件索引

| 文件 | 用途 |
|------|------|
| `libobs/obs-source.h` | `obs_source_info` 定义 + 公共枚举 |
| `libobs/obs-source.c` | 主实现（6005 行） |
| `libobs/obs-source-transition.c` | Transition 源实现（1072 行） |
| `libobs/obs-source-deinterlace.c` | 去隔行实现（480 行） |
| `libobs/obs-internal.h` | `obs_source` 结构体完整定义 |
| `libobs/obs.h` | 公共 API 声明 |
| `libobs/obs-scene.c` | Scene/Group 源实现 |
| `libobs/obs-video.c` | 视频渲染主循环 |
| `libobs/obs-audio.c` | 音频混音管线 |
| `libobs/obs-view.c` | View 抽象实现 |
| `libobs/obs-display.c` | Display（窗口）实现 |
| `libobs/callback/signal.c` | Signal handler 实现 |
| `libobs/callback/proc.c` | Proc handler 实现 |

---

## 十五、总结

OBS 的 source 系统是一个设计得很好的**插件化内容管线**。回顾全篇，几个核心设计决策贯穿始终：

1. **C 语言对象系统**：结构体嵌套 + 函数指针表，不用 C++ 也能做到虚函数分派，而且内存布局完全可控
2. **注册-查找-创建模式**：类型定义（`obs_source_info`）和实例（`obs_source`）分离，插件只需填一张表就能接入整个管线
3. **两级信号 + 过程调用**：push（信号）和 pull（proc）互补，覆盖了通知和查询两种交互模式
4. **Filter 链递归渲染**：用最简单的数组 + 嵌套调用实现了可组合的效果管线，bypass 优化又保证了性能
5. **延迟更新**：UI 线程打标记、图形线程执行，用最小的开销解决了跨线程同步问题
6. **帧队列 + 赶帧**：把异步采集和同步渲染解耦，缓冲模式在不同场景下（普通 vs 无缓冲）表现不同

理解了 source 类，OBS 的其他子系统——视频渲染、音频混音、场景管理、输出编码——就都有了支点。因为它们都在围绕 source 工作：生产数据、处理数据、传输数据。

下一篇文章计划梳理 OBS 的场景（scene）和场景项（scene_item）系统，看看"场景即源"这个设计是如何具体实现的。
