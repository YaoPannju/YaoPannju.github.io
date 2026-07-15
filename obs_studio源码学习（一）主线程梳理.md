# OBS Studio 源码学习（一）—— 主线程梳理

> 文件位置：`frontend/obs-main.cpp`
> 核心关注：`main()` → `run_program()` → `AppInit()` → `OBSInit()` → `exec()`

---

## 一、核心结构体关系（基础概念）

在阅读启动流程之前，先理解 OBS 中最核心的几个结构体之间的关系：

```
┌─────────────────────────────────────────────────────────────────────┐
│                         obs (obs_core)                               │
│                         全局唯一实例                                  │
│                                                                      │
│  ┌─ 注册表（"菜单"）─────────────────────────────────────────────┐   │
│  │ source_types[] ─── 所有源类型的 obs_source_info                │   │
│  │   ├─ [0] scene_info     ← obs_init() 里注册的（内置）          │   │
│  │   ├─ [1] group_info     ← obs_init() 里注册的（内置）          │   │
│  │   ├─ [2] audio_line_info ← obs_init() 里注册的（内置）         │   │
│  │   ├─ [3] dshow_input    ← win-dshow 插件 module_load 时注册   │   │
│  │   ├─ [4] game_capture   ← win-capture 插件 module_load 时注册 │   │
│  │   └─ ...                                                      │   │
│  │                                                               │   │
│  │ encoder_types[] ─── 编码器类型的 obs_encoder_info              │   │
│  │ output_types[]  ─── 输出类型的 obs_output_info                │   │
│  │ service_types[] ─── 服务类型的 obs_service_info               │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─ 子系统 ─────────────────────────────────────────────────────┐    │
│  │ video — 视频子系统（渲染线程、帧缓冲）                         │   │
│  │ audio — 音频子系统（混音器、监听器）                           │   │
│  │ data  — 数据管理（源的全局哈希表、链表）                       │   │
│  │ hotkeys — 快捷键管理                                          │   │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  first_module → [obs_module] → [obs_module] → ... (插件链表)       │
```

### 关键概念：「注册」= 往表里加条目

`obs_source_info` 是源类型的"定义单"（包含了 id、create/destroy 函数指针等）。
「注册」就是把这个定义单复制到 `obs->source_types` 数组的尾部。
之后 `obs_source_create("scene", ...)` 就是遍历数组找到匹配的 id，调用对应的 create 函数。

```c
struct obs_source_info {
    const char *id;                    // 唯一 ID，如 "scene"、"dshow_input"
    enum obs_source_type type;         // INPUT / FILTER / TRANSITION / SCENE
    uint32_t output_flags;             // 比特标志：有视频？有音频？异步？

    // ★ 函数指针 — 告诉 OBS "这个源怎么创建/销毁/渲染"
    void *(*create)(obs_data_t *settings, obs_source_t *source);
    void (*destroy)(void *data);
    void (*video_render)(void *data, gs_effect_t *effect);
    // ... 还有 30+ 可选回调
};
```

### 源实例（`obs_source`）

当用户真的"点菜"（创建源），OBS 从注册表中找到对应的 `obs_source_info`，复制为 `obs_source.info`，然后调用 `info.create()` 创建插件私有数据，存入 `obs_source.context.data`。

---

## 二、程序入口 — `main()` 函数

位置：`frontend/obs-main.cpp`

```
main(argc, argv)
│
├── 第一阶段：平台准备（新手扫一眼即可）
│      Linux: 注册信号处理(SIGINT/SIGTERM/SIGHUP/SIGABRT/SIGQUIT)
│              阻塞 SIGPIPE（防止写已关闭的 RTMP socket 时进程被杀死）
│      Windows: 检查 VC++ 运行库版本、安装 DLL 黑名单钩子、
│               设置进程安全缓解策略(DEP/ASLR)、安装 Windows 崩溃处理器、
│               设置 DLL 搜索路径安全模式、提权(调试权限/GPU 优先级)、
│               加载 RTWorkQ.dll（Windows 媒体工作队列）
│
├── 第二阶段：日志 + 命令行解析
│      base_get_log_handler(&def_log_handler)  ← 保存旧的日志处理器
│      obs_set_cmdline_args(argc, argv)        ← 深拷贝命令行参数
│      解析 --multi / --portable / --scene 等参数
│      --help / --version → 直接 exit(0)
│
├── 第三阶段：便携模式/禁用更新/禁用缺失文件检查
│      通过检测标记文件是否存在来判断
│
├── 第四阶段：网络库 + 核心启动
│      curl_global_init(CURL_GLOBAL_ALL)  ← 初始化 libcurl
│      ★ int ret = run_program(logFile, argc, argv)
│
└── 第五阶段：退出清理
       Windows: RTWorkQ 关闭 + 打印被拦截 DLL 列表
       blog("Number of memory leaks: %ld", bnum_allocs())
       base_set_log_handler(nullptr)  清空日志处理器
       如需要重启 → QProcess::startDetached 启动新进程
       return ret
```

### 关键细节

#### 1. `base_get_log_handler(&def_log_handler)` —— 为什么要保存旧日志处理器？

OBS 会把默认日志处理器替换成自己的 `do_log`（写文件 + 更新 UI 面板），但新处理器不负责打印到终端。保存旧的默认处理器，在 `do_log` 内部手动调用它，实现日志**同时写文件 + 打印终端**。

#### 2. `obs_set_cmdline_args(argc, argv)` —— 为什么要深拷贝命令行参数？

`argv` 是操作系统传给 `main()` 的指针，后续 Qt/插件可能修改它。趁早深拷贝一份存到 libobs 内部静态变量 `cmdline_args`，之后浏览器插件（CEF）通过 `obs_get_cmdline_args()` 读取。

#### 3. `curl_global_init(CURL_GLOBAL_ALL)` —— 初始化网络

初始化 SSL/TLS 和 Winsock，OBS 用 libcurl 做 RTMP 推流、更新检查、浏览器源等。在 `main()` 中单线程阶段调用保证安全。

---

## 三、`run_program()` — 应用级启动

```
run_program(logFile, argc, argv)
│
├── 第一阶段：性能分析器 + Qt 配置
│      profiler_start() → 启动性能分析器
│      Qt 高 DPI / 插件路径 / 主题兼容配置
│      设置 QT_NO_SUBTRACTOPAQUESIBLINGS=1 避免性能问题
│
├── 第二阶段：OBSApp 构造
│      OBSApp program(argc, argv, profilerNameStore)
│      内部：加载全局配置、用户配置、语言翻译、主题
│      创建 CrashHandler 对象（哨兵文件检测机制）
│
├── 第三阶段：★ AppInit()
│      MakeUserDirs()          创建用户目录
│      InitGlobalConfig()      加载全局配置
│      InitLocale()            加载语言翻译
│      InitTheme()             加载主题
│      → AppInit 只做目录+配置+语言+主题，与 libobs 核心无关
│
├── 第四阶段：多实例检测
│      CheckIfAlreadyRunning() → 命名互斥体判断
│      已在运行且无 --multi → 弹框"OBS 已经在运行，再开一个？"
│      用户取消 → return 0
│
├── 第五阶段：平台安全检查（goto run 之后）
│      ChromeOS 检测 → 不支持的平台 → 退出
│      create_log_file() → 从此 blog() 写入日志文件
│      checkForUncleanShutdown() → 通过哨兵文件检测上次是否崩溃
│      qInstallMessageHandler → Qt 日志重定向到 OBS 日志系统
│      macOS 权限检查 / Windows Wine 检测
│
├── 第六阶段：★ OBSInit() + exec()
│      │
│      ├── 6.1 平台显示层初始化（Linux X11/Wayland）
│      │     从 Qt 拿到 display，传给 libobs
│      │
│      ├── 6.2 ★ StartupOBS() → obs_startup()
│      │     │
│      │     └── obs_init()
│      │           ├── bzalloc(obs_core)          分配全局对象
│      │           ├── obs_init_data()             初始化数据互斥锁
│      │           ├── obs_init_handlers()         信号/过程调用
│      │           ├── obs_init_hotkeys()          ★ hotkey_thread 启动
│      │           ├── obs_create_main_canvas()    创建主画布
│      │           ├── os_task_queue_create()      异步销毁队列
│      │           ├── obs_register_source(&scene_info)     注册 "scene"
│      │           ├── obs_register_source(&group_info)     注册 "group"
│      │           ├── obs_register_source(&audio_line_info) 注册 "audio_line"
│      │           └── add_default_module_paths()
│      │
│      ├── 6.3 libobs_initialized = true
│      ├── 6.4 obs_set_ui_task_handler(ui_task_handler)
│      │      → libobs 需要主线程执行任务时，通过此回调转发到 Qt 事件循环
│      │
│      ├── 6.5 Browser HWAccel + 打印各种日志
│      │
│      ├── 6.6 new OBSBasic() + mainWindow->OBSInit()
│      │     │
│      │     ├── InitBasicConfig()    基本配置
│      │     ├── ResetAudio()         音频设备
│      │     │     └── obs_reset_audio() → obs_init_audio()
│      │     │           └── audio_output_open()
│      │     │                 └── ★ audio_thread 启动
│      │     │
│      │     ├── ResetVideo()         视频参数
│      │     │     └── obs_reset_video()
│      │     │           ├── obs_init_graphics()  图形后端初始化
│      │     │           └── obs_init_video()
│      │     │                 └── ★ video_thread 启动
│      │     │
│      │     ├── InitOBSCallbacks()   事件回调
│      │     ├── InitHotkeys()        热键
│      │     │
│      │     ├── ★ loadAppModules()
│      │     │     └── obs_load_all_modules2()  加载所有插件
│      │     │           └── 各插件 module_load → obs_register_source/encoder/...
│      │     │               obs->source_types 从 3 条变成几十条
│      │     │
│      │     ├── InitService()        推流服务
│      │     ├── ResetOutputs()       输出通道
│      │     ├── CreateHotkeys()      默认热键
│      │     │
│      │     ├── ★ ActivateSceneCollection()  加载场景和所有源
│      │     │     └── 读取 .json → obs_source_create → 恢复设置和滤镜
│      │     │
│      │     ├── obs_display_add_draw_callback()  每帧调用 RenderMain
│      │     └── show()               显示主窗口
│      │
│      ├── 6.7 连接信号（崩溃清理、热键焦点）
│      └── 6.8 return true
│
│      ├── prof.Stop()                停止启动性能计时
│      ├── ★ ret = program.exec()    Qt 事件循环（阻塞在此）
│      │     [主线程] Qt 事件循环（鼠标/键盘/信号槽）
│      │     [video_thread] 独立渲染线程（60fps）
│      │     [audio_thread] 音频处理线程
│      │     [hotkey_thread] 热键处理线程
│      └── exec() 返回 → 用户关闭了窗口
│
└── 第七阶段：退出 + 重启
       catch 捕获异常
       如需重启 → 保存参数到全局变量 arguments
       return ret
```

---

## 四、AppInit vs OBSInit 对比

| | AppInit() | OBSInit() |
|---|---|---|
| 做什么 | 目录 + 配置 + 语言 + 主题 | libobs 核心 + UI + 插件 + 场景 |
| 与 libobs 有关？ | 无关，纯 Qt 层面 | ★ obs_startup 在这里 |
| 失败怎么办 | throw（被 catch 捕获） | return false（不进入 exec） |

---

## 五、哨兵文件（Sentinel）机制

OBS 使用「哨兵文件」检测上次是否崩溃：

```
正常退出：
  启动 → 创建 .sentinel/run_UUID（空文件）
  退出 → ~CrashHandler() → 删除哨兵文件
  下次启动 → 没找到旧文件 → 正常

崩溃时：
  启动 → 创建 .sentinel/run_UUID
  崩溃 → ~CrashHandler() 没执行 → 文件留在磁盘
  下次启动 → 发现别人的哨兵 → hasUncleanShutdown() = true
```

本质：文件存在本身就是信息，不需要写任何内容。

---

## 六、三大核心线程的启动位置

| 线程 | 启动函数 | 调用链 | 启动时机 |
|------|---------|--------|---------|
| **hotkey_thread** | `obs_init_hotkeys()` | `obs_startup()` → `obs_init()` | OBSApp::OBSInit 中 |
| **audio_thread** | `audio_output_open()` | `ResetAudio()` → `obs_reset_audio()` | OBSBasic::OBSInit 中 |
| **video_thread** | `obs_init_video()` | `ResetVideo()` → `obs_reset_video()` | OBSBasic::OBSInit 中 |

**`obs_startup()` 本身不启动视频/音频线程**，只启动热键线程。视频和音频线程在 `OBSBasic::OBSInit()` 的 `ResetVideo()` / `ResetAudio()` 中才启动。

---

## 七、profilerNameStore 是什么？

`profilerNameStore` = `mutex` + `char*[]`，一个**永不过期的字符串池**。

```c
struct profiler_name_store {
    pthread_mutex_t mutex;       // 线程安全
    DARRAY(char *) names;        // 存字符串指针的动态数组
};
```

profiler 用它来安全地保存性能条目的名字。每当代码写 `profile_start("obs_graphics_thread")` 时，字符串被复制到池子里，返回的指针在整个程序生命周期内都有效，退出时统一释放。

---
