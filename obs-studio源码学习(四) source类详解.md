# obs-studio 源码学习 (四) source 类详解

## 一、类属关系

### 1.1 C 语言模拟继承

OBS 用 C 语言实现面向对象：把基类 `obs_context_data` 作为**第一个成员**嵌入子类结构体。因为第一个成员的地址等于结构体本身的地址，所以 `(obs_context_data *)source == &source->context`，可以安全地"向上转型"。

```
obs_context_data（基类）
  ├── obs_source        ← 核心
  │     ├── obs_scene（场景是特殊 source）
  │     └── obs_scene_item（场景项，引用 source）
  ├── obs_output
  ├── obs_encoder
  ├── obs_service
  └── obs_canvas
```

### 1.2 obs_context_data — 基类

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

提供的通用能力：

| 机制 | 实现方式 |
|------|----------|
| 引用计数 | `control` 指向 `obs_weak_object`，内含原子 `refs` + `weak_refs` |
| 信号系统 | `signals` 是 `signal_handler_t*` |
| 过程调用 | `procs` 是 `proc_handler_t*` |
| 全局查找 | `hh`（按名称）和 `hh_uuid`（按 UUID）两个 uthash 句柄 |
| 对象识别 | `type` 标记 SOURCE/OUTPUT/ENCODER/SERVICE/CANVAS |
| 名称管理 | `name` + `rename_cache` |
| 虚析构 | `destroy` 函数指针 |

### 1.3 构造顺序

```
1. obs_context_data_init(&context, ...)  → 分配信号/过程处理器、生成 UUID、拷贝名称和 settings
2. 子类自己的初始化（分配音频缓冲、初始化 transition 等）
3. obs_context_init_control(&context, source, obs_source_destroy) → 创建弱引用、设置虚析构函数
```

---

## 二、obs_source 结构体全景

> 完整定义见 `libobs/obs-internal.h:807-1004`

### 2.1 身份与类型

```c
struct obs_context_data context;  // 基类
struct obs_source_info info;      // 从注册信息拷贝来的虚函数表
uint32_t flags;                   // 运行时标志（用户可修改）
uint32_t default_flags;           // 默认标志
uint32_t last_obs_ver;            // 创建时的 OBS 版本
bool owns_info_id;                // 是否拥有 info.id 字符串的内存所有权
```

**`owns_info_id`**：正常创建时 `info = *注册表条目`，`info.id` 指向注册表里的静态字符串。找不到插件时空壳 source 用 `bstrdup(id)` 自己分配内存，`owns_info_id = true`，销毁时自己释放。

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

**`defer_update_count`**：视频源设置变更时不直接在 UI 线程调 `info.update()`，而是 +1 打标记，下一帧 tick 在图形线程执行。CAS 操作清零，如果中途又被打标记则 CAS 失败保留新值，下帧重试。

### 2.3 异步视频管道

| 字段 | 含义 |
|------|------|
| `async_frames` | 待渲染的帧队列（动态指针数组） |
| `async_cache` | 帧对象缓存池 |
| `cur_async_frame` | 当前帧（CPU 侧数据指针） |
| `async_textures[MAX_AV_PLANES]` | 当前帧的 GPU 纹理 |
| `async_texrender` | YUV→RGB 转换的渲染目标 |
| `async_active` | 是否有可渲染的异步帧（传入 NULL 帧 → false） |
| `async_unbuffered` | 无缓冲模式，永远只要最新一帧（桌面捕获） |
| `async_decoupled` | 解耦模式，音频不跟视频同步（避免网络摄像头音频卡顿） |
| `async_update_texture` | 本帧是否需要重建 GPU 纹理 |
| `async_format` | 视频像素格式 |
| `async_width/height` | 异步视频尺寸 |
| `async_rotation` | 旋转角度 |
| `async_flip` | 是否翻转 |
| `async_preload_frame` | 预加载帧（用于 `show_preloaded_video`） |

**preload + show 机制**：对应 OBS 的媒体源。先用 `preload_video()` 把首帧上传到 GPU 但不标记 `async_active`；播放开始时 `show_preloaded_video()` 一键激活，首帧零延迟不黑屏。

### 2.4 去隔行

```c
enum obs_deinterlace_mode deinterlace_mode;
bool deinterlace_top_first;                       // 场序
gs_effect_t *deinterlace_effect;                  // 去隔行着色器
struct obs_source_frame *prev_async_frame;        // 前一场（CPU 侧）
gs_texture_t *async_prev_textures[MAX_AV_PLANES]; // 前一场的 GPU 纹理
gs_texrender_t *async_prev_texrender;             // 前一场的转换目标
```

去隔行需要**当前场 + 前一场**两帧合成一帧逐行画面。支持的算法：DISCARD、RETRO、BLEND、BLEND_2X、LINEAR、LINEAR_2X、YADIF、YADIF_2X。

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

### 2.6 Filter 链

```c
struct obs_source *filter_parent;     // 此 filter 附着的父 source
struct obs_source *filter_target;     // 渲染目标（链上的下一个）
DARRAY(struct obs_source *) filters;  // 附着在此 source 上的 filter 列表
gs_texrender_t *filter_texrender;    // filter 链的中间渲染目标
bool rendering_filter;               // 防止递归渲染
bool filter_bypass_active;           // 优化：直接渲染（省一个 GPU pass）
```

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

`transitioning_video` 和 `transitioning_audio` 是分开的 bool：视频动画可能先于音频交叉淡入淡出完成（或反之），各管各的。两者都结束时才调 `obs_transition_stop`。

---

## 三、三层标志位

| 层次 | 字段 | 谁设置 | 作用 |
|------|------|--------|------|
| 类型能力 | `info.output_flags` | 插件注册时 | 描述这个类型"能做什么"（不变） |
| 默认值 | `default_flags` | 插件/前端 | 新实例的初始值 |
| 运行时 | `flags` | 用户/前端 | 控制这个实例的行为（目前只有 `FORCE_MONO`） |

---

## 四、`private` 标志的影响

| 影响 | private=true | private=false |
|------|-------------|---------------|
| 加入 `public_sources`（按名称哈希表） | ❌ | ✅ |
| 加入 `sources`（按 UUID 哈希表） | ✅ | ✅ |
| `obs_enum_sources()` 枚举 | ❌ | ✅ |
| `obs_get_source_by_name()` 查找 | ❌ | ✅ |
| 保存到场景 JSON | ❌ | ✅ |
| 发射全局信号 | ❌ | ✅ |
| 发射自己的信号 | ✅ | ✅ |

Filter 永远被强制设 `private = true`，因为它只附着在 source 上，不应在全局列表中出现。

---

## 五、Signal 系统

### 5.1 两级信号

```c
// 全局信号（obs->signals）  →  广播频道，对所有监听者
// 对象信号（source->context.signals） → 私人专线，只发给自己

static inline void obs_source_dosignal(source, signal_obs, signal_source)
{
    if (signal_obs && !source->context.private)
        signal_handler_signal(obs->signals, signal_obs, &data);   // 广播
    if (signal_source)
        signal_handler_signal(source->context.signals, signal_source, &data); // 私下
}
```

### 5.2 signal_handler 结构

```c
struct signal_handler {
    struct signal_info *first;      // 按信号名分拣的单向链表
    pthread_mutex_t mutex;          // 保护 first 链表
    volatile long refs;

    DARRAY(struct global_callback_info) global_callbacks;  // 统吃：不管什么信号都触发
    pthread_mutex_t global_callbacks_mutex;                 // 保护 global_callbacks
};
```

`first` 链表：精确匹配信号名（`"destroy"`→[回调A,B,C], `"volume"`→[回调D]）
`global_callbacks`：全信号触发（回调里收到信号名字符串参数），用于脚本/API 全局观察

### 5.3 signal_handler_connect 参数

```c
signal_handler_connect(
    signal_handler_t *handler,  // 1. 挂在谁身上（从 obs_source_get_signal_handler 拿）
    const char *signal,         // 2. 监听哪个信号名
    signal_callback_t callback, // 3. 回调函数 void (void *data, calldata_t *params)
    void *data                  // 4. 回调时原样带回的用户数据（不需要传 NULL）
);
```

### 5.4 proc_handler — 过程调用

与信号的 push 不同，proc 是 **pull / 请求-响应** 模式。信号是 source 通知你，过程是你主动问 source 某件事。

```c
proc_handler_add(ph, "void current_index(out int idx)", current_slide_proc, ss);
// → 注册

proc_handler_call(ph, "current_index", &cd);
int idx = calldata_int(&cd, "current_index");
// → 调用并获取返回值
```

---

## 六、obs_source_info — 源类型定义表

`obs_source.h:222-558`。插件注册的虚函数表。核心回调分类：

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

---

## 七、核心函数详解

### 7.1 obs_source_create_internal — 创建流程

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

### 7.2 obs_source_video_tick — 每帧状态更新

被图形线程每帧对每个活跃 source 调用。`float seconds` 是距上一帧的秒数。

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

### 7.4 Filter 链机制

#### 数据结构

```
Input Source "摄像头"
  │  filters: [F3, F2, F1]      ← 数组头=最后加的=离屏幕最近
  │  filter_parent: NULL
  │
  ├─ F1: filter_parent=摄像头, filter_target=摄像头
  ├─ F2: filter_parent=摄像头, filter_target=F1
  └─ F3: filter_parent=摄像头, filter_target=F2
```

- 新 filter 永远 `da_insert(filters, 0)` 插到数组头部
- `filter_target = !filters.num ? source : filters.array[0]`

#### 渲染调用栈（嵌套递归）

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

**结论：先加的 filter 靠近原始画面，后加的靠近屏幕。最后加的 filter 第一个被调用，但因为 begin/end 包裹递归，它的效果实际是最后施加的。**

#### Bypass 优化

六个条件全满足时省一个 GPU pass：
1. target == parent（filter 目标就是父源本身）
2. allow_direct == OBS_ALLOW_DIRECT_RENDERING
3. 父源不用自定义绘制
4. 父源不是异步视频
5. filter 与 parent 的 SRGB 标志一致
6. 色彩空间一致

---

## 八、View 与 Scene 的关系

| | Source | Scene | View |
|---|---|---|---|
| 本质 | 内容生产者 | 特殊 source（容器） | 频道映射器 |
| 有 obs_context_data | ✅ | ✅（通过包裹的 source） | ❌ |
| 有 UUID/信号/引用计数 | ✅ | ✅ | ❌ |
| 作用 | 生产/处理音视频 | 管理子 source 的层次排列 | 选频道→source 的映射 |

- **Source** 是**内容从哪来**
- **Scene** 是**这堆源怎么排列**（谁在上谁在下）
- **View** 是**这个频道现在播哪个 scene**（选台器）

View 不是 source，就是三字段的轻量结构：`{ channels[6], type, mutex }`。工作室模式的预览窗口和主输出是同一个 scene 放到不同的 View 上渲染。

---

## 九、帧队列与赶帧机制

### 普通缓冲模式
按时间戳排队，丢了过期帧追赶：
```
系统时钟 8.33ms，帧队列 [t=0, t=4.16, t=8.33]
→ 8.33 > 0 → 丢 t=0
→ 8.33 > 4.16 → 丢 t=4.16
→ 8.33 > 8.33（false）→ 渲染 t=8.33
```

### 无缓冲模式（桌面/游戏捕获）
永远只保留最新一帧，前面的全扔。

### 帧还没到时
没有合适的帧 → `cur_async_frame = NULL` → GPU 纹理不更新 → **重复显示上一帧**（如果从未有帧则跳过渲染）。

---

## 十、obs_source_duplicate 使用场景

1. **复制粘贴源**（右键→复制→粘贴副本）：深拷贝，独立 settings 和 filter 链
2. **场景复制**：已在目标场景里的源只加引用，不在的独立复制
3. **复制 filter**：复制源时身上的 filter 链逐个深拷贝
4. **DO_NOT_DUPLICATE 标志**：硬件设备源不支持拷贝，直接加引用

---

## 十一、obs_source_get_id 使用场景

返回源的类型字符串（如 `"dshow_input"`、`"ffmpeg_source"`），前端用于：

- 根据类型显示不同图标
- 根据类型显示不同 UI 控件（Freetype 文本源有字体选择器）
- 判断能否添加某种 filter
- 复制粘贴时保留过渡类型
- 判断是否是 group 做特殊处理

---

## 十二、obs_source_output_video 帧提交流程

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

---

## 十三、关键文件索引

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
