# libghostty iOS 兼容性与 Swift 封装分析报告

## 概述

本报告详细分析 Ghostty 项目提供的 libghostty 库，包括其架构、主要 API、以及 iOS 平台的兼容性评估。同时参考 [GitHub Discussion #4087](https://github.com/ghostty-org/ghostty/discussions/4087) 中社区讨论的 iOS 实现方案，评估整体修改难度并提出完整的实现方案。

---

## 1. libghostty 库架构分析

### 1.1 核心设计理念

libghostty 是 Ghostty 的**嵌入式终端库**，设计目标是将 Ghostty 的核心终端模拟功能嵌入到其他应用程序中。与主 Ghostty 应用不同，libghostty：

- **不拥有应用生命周期** - 由宿主应用控制
- **提供终端渲染和事件处理** - 宿主应用负责 UI 决策
- **通过回调机制通信** - 使用 C ABI 接口

### 1.2 核心组件架构

```
┌─────────────────────────────────────────────────────────────┐
│                     Host Application                         │
│                   (Swift/UIKit/SwiftUI)                      │
└─────────────────────────┬───────────────────────────────────┘
                          │ C ABI (ghostty.h)
┌─────────────────────────▼───────────────────────────────────┐
│                     libghostty                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Embedded Runtime (embedded.zig)            ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ ││
│  │  │     App     │  │   Surface   │  │    Inspector    │ ││
│  │  └─────────────┘  └─────────────┘  └─────────────────┘ ││
│  └─────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    Core Terminal                        ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ ││
│  │  │  Terminal   │  │    Font     │  │   Config/PTY    │ ││
│  │  └─────────────┘  └─────────────┘  └─────────────────┘ ││
│  └─────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    Renderer                             ││
│  │  ┌─────────────────────────────────────────────────────┐││
│  │  │              Metal (macOS/iOS)                      │││
│  │  └─────────────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### 1.3 关键源文件

| 文件路径 | 描述 |
|---------|------|
| `include/ghostty.h` | 完整 C API 头文件 (1135 行) |
| `src/main_c.zig` | C API 入口点 |
| `src/apprt/embedded.zig` | 嵌入式运行时实现 (2173 行) |
| `src/App.zig` | 应用控制器 |
| `src/Surface.zig` | 终端表面 (3849 行) |
| `src/renderer/Metal.zig` | Metal 渲染器 |
| `src/build/GhosttyXCFramework.zig` | XCFramework 构建 |

---

## 2. libghostty 主要 API 调用方式

### 2.1 核心类型定义

```c
// 不透明类型
typedef void* ghostty_app_t;       // 应用实例
typedef void* ghostty_config_t;    // 配置
typedef void* ghostty_surface_t;   // 终端表面
typedef void* ghostty_inspector_t; // 调试检查器

// 平台枚举
typedef enum {
    GHOSTTY_PLATFORM_INVALID,
    GHOSTTY_PLATFORM_MACOS,    // = 1
    GHOSTTY_PLATFORM_IOS,      // = 2
} ghostty_platform_e;
```

### 2.2 初始化流程

```c
// 1. 全局初始化
int result = ghostty_init(argc, argv);
if (result != GHOSTTY_SUCCESS) {
    // 处理错误
}

// 2. 创建配置
ghostty_config_t config = ghostty_config_new();
ghostty_config_load_default_files(config);
ghostty_config_finalize(config);

// 3. 设置运行时回调
ghostty_runtime_config_s runtime_cfg = {
    .userdata = app_context,
    .supports_selection_clipboard = true,
    .wakeup_cb = wakeup_callback,
    .action_cb = action_callback,
    .read_clipboard_cb = read_clipboard,
    .confirm_read_clipboard_cb = confirm_clipboard,
    .write_clipboard_cb = write_clipboard,
    .close_surface_cb = close_surface,
};

// 4. 创建应用
ghostty_app_t app = ghostty_app_new(&runtime_cfg, config);
```

### 2.3 Surface 创建与管理

```c
// 创建 Surface 配置
ghostty_surface_config_s surf_config = ghostty_surface_config_new();
surf_config.platform_tag = GHOSTTY_PLATFORM_IOS;    // iOS 平台
surf_config.platform.ios.uiview = (void*)myUIView;  // 传入 UIView
surf_config.scale_factor = [UIScreen mainScreen].scale;
surf_config.font_size = 14.0f;
surf_config.working_directory = "/Users/user";

// 创建 Surface
ghostty_surface_t surface = ghostty_surface_new(app, &surf_config);

// 设置尺寸
ghostty_surface_set_size(surface, width * scale, height * scale);
ghostty_surface_set_content_scale(surface, scale, scale);

// 设置焦点
ghostty_surface_set_focus(surface, true);
```

### 2.4 事件循环与渲染

```c
// 主循环 tick
void on_run_loop() {
    ghostty_app_tick(app);  // 处理队列消息
}

// 渲染
void on_draw() {
    ghostty_surface_draw(surface);
}

// 刷新请求
void on_refresh() {
    ghostty_surface_refresh(surface);
}
```

### 2.5 输入处理

```c
// 键盘输入
ghostty_input_key_s key_event = {
    .action = GHOSTTY_ACTION_PRESS,
    .mods = GHOSTTY_MODS_NONE,
    .keycode = keycode,
    .text = "a",
    .unshifted_codepoint = 'a',
    .composing = false,
};
bool consumed = ghostty_surface_key(surface, key_event);

// 文本输入 (IME)
ghostty_surface_text(surface, "你好", strlen("你好"));

// 鼠标/触摸
ghostty_surface_mouse_button(surface, GHOSTTY_MOUSE_PRESS,
                              GHOSTTY_MOUSE_LEFT, GHOSTTY_MODS_NONE);
ghostty_surface_mouse_pos(surface, x, y, GHOSTTY_MODS_NONE);
ghostty_surface_mouse_scroll(surface, scroll_x, scroll_y, mods);
```

### 2.6 回调实现示例

```c
// 唤醒回调 - 通知主线程处理事件
void wakeup_callback(void* userdata) {
    dispatch_async(dispatch_get_main_queue(), ^{
        AppContext* ctx = (AppContext*)userdata;
        ghostty_app_tick(ctx->app);
    });
}

// Action 回调 - 处理终端请求的操作
bool action_callback(ghostty_app_t app, ghostty_target_s target,
                     ghostty_action_s action) {
    switch (action.tag) {
        case GHOSTTY_ACTION_SET_TITLE:
            // 更新窗口标题
            break;
        case GHOSTTY_ACTION_NEW_SPLIT:
            // 创建新分割
            break;
        case GHOSTTY_ACTION_DESKTOP_NOTIFICATION:
            // 显示通知
            break;
        // ... 30+ 其他 action 类型
    }
    return true;
}
```

---

## 3. 现有 iOS 支持状态

### 3.1 代码库中的 iOS 支持

当前代码库**已包含** iOS 基础支持：

#### 构建系统支持 (`src/build/GhosttyXCFramework.zig:27-54`)
```zig
// iOS 物理设备
const ios = GhosttyLib.initStatic(b, &deps.retarget(b,
    b.resolveTargetQuery(.{
        .cpu_arch = .aarch64,
        .os_tag = .ios,
        .os_version_min = Config.osVersionMin(.ios),
    }),
));

// iOS 模拟器
const ios_sim = GhosttyLib.initStatic(b, &deps.retarget(b,
    b.resolveTargetQuery(.{
        .cpu_arch = .aarch64,
        .os_tag = .ios,
        .abi = .simulator,
    }),
));
```

#### 平台抽象层 (`src/apprt/embedded.zig:342-397`)
```zig
pub const PlatformTag = enum(c_int) {
    macos = 1,
    ios = 2,
};

pub const Platform = union(PlatformTag) {
    macos: MacOS,
    ios: IOS,

    pub const IOS = struct {
        uiview: objc.Object,  // UIView 引用
    };
};
```

#### Metal 渲染器 iOS 适配 (`src/renderer/Metal.zig:78-134`)
```zig
// iOS 特定存储模式
const default_storage_mode: mtl.MTLResourceOptions.StorageMode =
    switch (comptime builtin.os.tag) {
        .ios => .shared,  // iOS 不支持 .managed
        else => if (device.getProperty(bool, "hasUnifiedMemory"))
                    .shared else .managed,
    };

// iOS Layer 设置方式
.ios => {
    const view_layer = objc.Object.fromId(
        info.view.getProperty(?*anyopaque, "layer")
    );
    view_layer.msgSend(void, objc.sel("addSublayer:"), .{layer.layer.value});
},
```

#### Swift 封装 (`macos/Sources/`)
```
macos/Sources/
├── App/iOS/iOSApp.swift              # iOS 应用入口
├── Ghostty/
│   ├── Ghostty.App.swift             # App 封装
│   ├── Surface View/
│   │   ├── SurfaceView_UIKit.swift   # UIKit 版 Surface
│   │   └── SurfaceView.swift         # 通用实现
```

### 3.2 当前 iOS 代码状态

查看 `iOSApp.swift`:
```swift
@main
struct Ghostty_iOSApp: App {
    @StateObject private var ghostty_app: Ghostty.App

    init() {
        if ghostty_init(UInt(CommandLine.argc), CommandLine.unsafeArgv)
           != GHOSTTY_SUCCESS {
            preconditionFailure("Initialize ghostty backend failed")
        }
        _ghostty_app = StateObject(wrappedValue: Ghostty.App())
    }

    var body: some Scene {
        WindowGroup {
            Ghostty.Terminal()
                .environmentObject(ghostty_app)
        }
    }
}
```

iOS 回调实现（`Ghostty.App.swift:260-287`）目前是**空实现**:
```swift
#if os(iOS)
static func wakeup(_ userdata: UnsafeMutableRawPointer?) {}
static func action(_ app: ghostty_app_t, ...) -> Bool { return false }
static func readClipboard(...) {}
static func writeClipboard(...) {}
static func closeSurface(_ userdata: UnsafeMutableRawPointer?, ...) {}
#endif
```

---

## 4. GitHub Discussion #4087 分析

### 4.1 讨论背景

- **请求者**: cschaba 希望将 Ghostty 移植到 iPad
- **维护者回应**: mitchellh 表示 "libghostty 将使他人能够做到这一点"
- **实际实现**: kitknox (2025年11月) 提供了完整实现

### 4.2 kitknox 的实现方案

#### 核心技术栈
- **前端**: Swift/SwiftUI/UIKit
- **目标平台**: iOS, iPadOS, visionOS, macOS
- **底层**: libghostty (GhosttyKit.xcframework)

#### 关键技术挑战与解决方案

| 挑战 | 原因 | 解决方案 |
|-----|------|---------|
| **libxev kevent 不兼容** | iOS 沙盒限制 kqueue 的某些用法 | 修复 libxev 库 |
| **无 PTY 支持** | iOS 不允许 fork/exec | 实现**管道后端** |
| **渲染问题** | 初期只显示光标 | 调试 Metal 层配置 |
| **Zig 标准库问题** | iOS 平台特定代码路径 | 打补丁 |

#### 管道后端架构
```
┌─────────────────────┐     管道      ┌─────────────────────┐
│    libghostty       │◄────────────►│   Swift 后端        │
│  (终端模拟)         │   (读/写)     │  - SSH (Swift-NIO)  │
│                     │              │  - 本地 shell       │
└─────────────────────┘              └─────────────────────┘
```

#### 已实现功能
- SSH 连接 (基于 Swift-NIO-SSH)
- 滚动和选择
- 分割视图 (splits)
- 复制/粘贴
- 标签页管理
- 基础本地 shell

### 4.3 维护者态度

mitchellh 明确表示:
> "管道后端会被接受，设计良好即可"

这表明官方愿意接受为 iOS 添加替代后端的 PR。

---

## 5. 修改难度评估

### 5.1 难度分级

| 组件 | 难度 | 原因 |
|------|------|------|
| **XCFramework 构建** | ★☆☆☆☆ 简单 | 已完成 |
| **C API 适配** | ★☆☆☆☆ 简单 | 已完成，iOS 平台标记存在 |
| **Metal 渲染** | ★★☆☆☆ 较简单 | 已有 iOS 适配代码 |
| **Swift UI 封装** | ★★★☆☆ 中等 | 基础存在，需完善回调 |
| **管道后端** | ★★★★☆ 困难 | 需要全新实现 |
| **libxev 修复** | ★★★★☆ 困难 | 底层事件系统修改 |
| **SSH 集成** | ★★★☆☆ 中等 | 可用现有库 |
| **完整 iOS 应用** | ★★★★★ 复杂 | 需要大量 UI 工作 |

### 5.2 工作量估算

```
┌────────────────────────────────────────────────────────────┐
│                    整体工作量分布                          │
├────────────────────────────────────────────────────────────┤
│ ▓▓▓▓░░░░░░  基础设施 (40% 已完成)                         │
│   - XCFramework ✓                                          │
│   - C API ✓                                                │
│   - Metal 渲染 ✓                                           │
│   - 平台抽象 ✓                                             │
├────────────────────────────────────────────────────────────┤
│ ░░░░░░░░░░  Swift 封装 (需要完善)                          │
│   - 完整回调实现                                            │
│   - 输入处理                                                │
│   - 剪贴板                                                  │
├────────────────────────────────────────────────────────────┤
│ ░░░░░░░░░░  管道后端 (需要新开发)                          │
│   - IO 重定向                                               │
│   - 进程模拟                                                │
├────────────────────────────────────────────────────────────┤
│ ░░░░░░░░░░  SSH/Shell (需要集成)                           │
│   - Swift-NIO-SSH                                          │
│   - 本地命令执行                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 6. 完整实现方案

### 6.1 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                     iOS Application                          │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                  SwiftUI / UIKit                        ││
│  │  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ││
│  │  │  TerminalView │ │   TabBar      │ │  Settings     │ ││
│  │  └───────────────┘ └───────────────┘ └───────────────┘ ││
│  └─────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────┐│
│  │              GhosttySwift (Swift 封装层)                ││
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐││
│  │  │ GhosttyApp  │ │GhosttyView  │ │  InputHandler       │││
│  │  └─────────────┘ └─────────────┘ └─────────────────────┘││
│  │  ┌─────────────────────────────────────────────────────┐││
│  │  │            Callback Handlers                        │││
│  │  │  - ClipboardHandler                                 │││
│  │  │  - ActionHandler                                    │││
│  │  │  - NotificationHandler                              │││
│  │  └─────────────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────┐│
│  │            GhosttyKit.xcframework (libghostty)          ││
│  └─────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────┐│
│  │                   Backend Layer                         ││
│  │  ┌─────────────────────┐ ┌─────────────────────────────┐││
│  │  │    PipeBackend      │ │    SSHBackend               │││
│  │  │  (本地命令)         │ │  (Swift-NIO-SSH)            │││
│  │  └─────────────────────┘ └─────────────────────────────┘││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### 6.2 实现步骤

#### 阶段一: 完善 Swift 封装 (1-2 周)

1. **完善 iOS 回调实现**
```swift
// Ghostty.App.swift - iOS 部分
#if os(iOS)
static func wakeup(_ userdata: UnsafeMutableRawPointer?) {
    let state = Unmanaged<App>.fromOpaque(userdata!).takeUnretainedValue()
    DispatchQueue.main.async { state.appTick() }
}

static func action(_ app: ghostty_app_t, target: ghostty_target_s,
                   action: ghostty_action_s) -> Bool {
    switch action.tag {
    case GHOSTTY_ACTION_SET_TITLE:
        // 更新标题
        return true
    case GHOSTTY_ACTION_DESKTOP_NOTIFICATION:
        // 发送 iOS 通知
        return true
    // ... 实现其他 action
    default:
        return false
    }
}

static func readClipboard(_ userdata: UnsafeMutableRawPointer?,
                          location: ghostty_clipboard_e,
                          state: UnsafeMutableRawPointer?) {
    let surfaceView = surfaceUserdata(from: userdata)
    guard let surface = surfaceView.surface else { return }
    let str = UIPasteboard.general.string ?? ""
    completeClipboardRequest(surface, data: str, state: state)
}

static func writeClipboard(_ userdata: UnsafeMutableRawPointer?,
                           location: ghostty_clipboard_e,
                           content: UnsafePointer<ghostty_clipboard_content_s>?,
                           len: Int, confirm: Bool) {
    guard let content = content, len > 0 else { return }
    let data = String(cString: content.pointee.data!)
    UIPasteboard.general.string = data
}
#endif
```

2. **完善 SurfaceView_UIKit.swift**
```swift
extension Ghostty.SurfaceView {
    // 键盘处理
    override func pressesBegan(_ presses: Set<UIPress>,
                                with event: UIPressesEvent?) {
        for press in presses {
            handleKeyPress(press, action: .press)
        }
    }

    // 触摸处理
    override func touchesBegan(_ touches: Set<UITouch>,
                                with event: UIEvent?) {
        guard let touch = touches.first, let surface = self.surface else { return }
        let location = touch.location(in: self)
        ghostty_surface_mouse_button(surface, GHOSTTY_MOUSE_PRESS,
                                      GHOSTTY_MOUSE_LEFT, GHOSTTY_MODS_NONE)
        ghostty_surface_mouse_pos(surface, location.x, location.y, GHOSTTY_MODS_NONE)
    }
}
```

#### 阶段二: 管道后端实现 (2-3 周)

1. **定义后端协议**
```swift
protocol TerminalBackend {
    func write(_ data: Data)
    func read() -> Data?
    func resize(cols: Int, rows: Int)
    var onData: ((Data) -> Void)? { get set }
    var onExit: ((Int) -> Void)? { get set }
}
```

2. **管道后端实现**
```swift
class PipeBackend: TerminalBackend {
    private var inputPipe: Pipe
    private var outputPipe: Pipe

    var onData: ((Data) -> Void)?
    var onExit: ((Int) -> Void)?

    init() {
        inputPipe = Pipe()
        outputPipe = Pipe()

        // 监听输出
        outputPipe.fileHandleForReading.readabilityHandler = { [weak self] handle in
            let data = handle.availableData
            if !data.isEmpty {
                self?.onData?(data)
            }
        }
    }

    func write(_ data: Data) {
        inputPipe.fileHandleForWriting.write(data)
    }

    func resize(cols: Int, rows: Int) {
        // 发送 resize 信号
    }
}
```

3. **SSH 后端集成**
```swift
import NIOSSHClient

class SSHBackend: TerminalBackend {
    private var channel: SSHChannel?
    private var eventLoop: EventLoop

    var onData: ((Data) -> Void)?
    var onExit: ((Int) -> Void)?

    func connect(host: String, port: Int, username: String,
                 authMethod: SSHAuthMethod) async throws {
        // 使用 Swift-NIO-SSH 建立连接
        let client = try await SSHClient.connect(
            host: host,
            port: port,
            authenticationMethod: authMethod,
            on: eventLoop
        )

        self.channel = try await client.requestPTY(
            term: "xterm-256color"
        )

        // 设置数据回调
        channel?.onData = { [weak self] data in
            self?.onData?(data)
        }
    }

    func write(_ data: Data) {
        channel?.write(data)
    }

    func resize(cols: Int, rows: Int) {
        channel?.resize(cols: cols, rows: rows)
    }
}
```

#### 阶段三: libghostty 管道集成 (2-3 周)

需要修改 Zig 代码以支持管道模式:

```zig
// src/pty/Backend.zig (新文件)
pub const Backend = union(enum) {
    pty: PosixPty,
    pipe: PipeBackend,

    pub const PipeBackend = struct {
        read_fd: std.posix.fd_t,
        write_fd: std.posix.fd_t,

        pub fn init(read_fd: std.posix.fd_t, write_fd: std.posix.fd_t) PipeBackend {
            return .{
                .read_fd = read_fd,
                .write_fd = write_fd,
            };
        }

        pub fn read(self: *PipeBackend, buf: []u8) !usize {
            return std.posix.read(self.read_fd, buf);
        }

        pub fn write(self: *PipeBackend, data: []const u8) !usize {
            return std.posix.write(self.write_fd, data);
        }
    };
};
```

#### 阶段四: iOS 应用 UI (2-4 周)

```swift
// ContentView.swift
struct ContentView: View {
    @StateObject private var terminalManager = TerminalManager()

    var body: some View {
        TabView {
            ForEach(terminalManager.sessions) { session in
                TerminalSessionView(session: session)
                    .tabItem {
                        Label(session.title, systemImage: "terminal")
                    }
            }
        }
        .toolbar {
            ToolbarItem(placement: .primaryAction) {
                Menu {
                    Button("New SSH Session") {
                        terminalManager.showSSHDialog = true
                    }
                    Button("New Local Shell") {
                        terminalManager.createLocalSession()
                    }
                } label: {
                    Image(systemName: "plus")
                }
            }
        }
        .sheet(isPresented: $terminalManager.showSSHDialog) {
            SSHConnectionView()
        }
    }
}

struct TerminalSessionView: View {
    @ObservedObject var session: TerminalSession

    var body: some View {
        GhosttyTerminalView(surface: session.surface)
            .onAppear {
                session.surface.focusDidChange(true)
            }
            .onDisappear {
                session.surface.focusDidChange(false)
            }
            .gesture(
                MagnificationGesture()
                    .onChanged { scale in
                        session.adjustFontSize(by: scale)
                    }
            )
    }
}
```

### 6.3 关键修改点汇总

| 文件 | 修改类型 | 描述 |
|-----|---------|------|
| `src/pty/Backend.zig` | 新增 | 管道后端抽象 |
| `src/apprt/embedded.zig` | 修改 | 添加管道模式支持 |
| `include/ghostty.h` | 修改 | 添加管道相关 API |
| `macos/Sources/Ghostty/Ghostty.App.swift` | 修改 | 完善 iOS 回调 |
| `macos/Sources/Ghostty/Surface View/SurfaceView_UIKit.swift` | 修改 | 完善输入处理 |
| `macos/Sources/App/iOS/` | 新增 | 完整 iOS 应用 |

---

## 7. 风险与建议

### 7.1 主要风险

1. **libxev 兼容性** - 可能需要 fork 并维护修改版本
2. **App Store 审核** - 终端类应用可能面临审核挑战
3. **性能问题** - 管道通信可能有延迟
4. **维护负担** - 需要持续跟进上游变化

### 7.2 建议

1. **渐进式实现** - 先完成 SSH 功能，再考虑本地 shell
2. **与上游协作** - 尽早提交管道后端 PR 获得官方支持
3. **测试覆盖** - 重点测试输入处理和渲染性能
4. **社区合作** - 参考 kitknox 的实现经验

---

## 8. 结论

libghostty 对 iOS 的支持基础设施已经**相当完善**（约 40% 已完成）：
- XCFramework 构建系统 ✓
- C API iOS 平台定义 ✓
- Metal 渲染器 iOS 适配 ✓
- Swift 封装基础框架 ✓

主要剩余工作：
1. **Swift 回调完善** - 中等难度，约 1-2 周
2. **管道后端** - 困难，约 2-3 周，但 kitknox 已验证可行性
3. **SSH 集成** - 中等难度，有现成库可用
4. **iOS 应用 UI** - 工作量取决于功能完整度

**总体评估**: 基于现有代码基础和社区经验，完整的 iOS 终端应用实现是**可行的**，预计需要 **6-10 周**的开发时间。关键成功因素是管道后端的正确实现和与上游的良好协作。
