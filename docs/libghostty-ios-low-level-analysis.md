# libghostty iOS 底层技术分析与改造方案

## 1. 核心问题概述

在 iOS 上运行 libghostty 存在四个底层技术障碍：

| 问题 | 根源位置 | 严重程度 |
|-----|---------|---------|
| **PTY 不可用** | `src/pty.zig` | 致命 - iOS 沙盒禁止 fork/exec |
| **libxev EVFILT_PROC** | libxev 库 | 致命 - iOS 沙盒禁止进程监视 |
| **libxev Async (Mach Ports)** | libxev 库 | 严重 - iOS Mach 端口受限 |
| **termio 架构耦合** | `src/termio/` | 中等 - 与 PTY 紧密耦合 |

---

## 2. 问题一：PTY 系统分析

### 2.1 当前 PTY 架构

**文件**: `src/pty.zig`

```zig
// 第 18-22 行：平台分派
pub const Pty = switch (builtin.os.tag) {
    .windows => WindowsPty,
    .ios => NullPty,        // 临时存根！
    else => PosixPty,
};
```

**NullPty 存根** (第 43-81 行):
```zig
// TODO: This should be removed. This is only temporary until we have
// a termio that doesn't use a pty.
const NullPty = struct {
    pub fn open(size: winsize) OpenError!Pty {
        return .{ .master = 0, .slave = 0 };  // 返回无效 FD
    }
    // 其他方法均为空操作...
};
```

### 2.2 PosixPty 核心依赖

**依赖的系统调用** (iOS 全部不可用):

| 系统调用 | 用途 | iOS 状态 |
|---------|------|---------|
| `openpty()` | 创建伪终端对 | ❌ 沙盒禁止 |
| `fork()` | 创建子进程 | ❌ 沙盒禁止 |
| `exec()` | 执行程序 | ❌ 沙盒禁止 |
| `setsid()` | 创建会话 | ❌ 沙盒禁止 |
| `TIOCSCTTY` | 设置控制终端 | ❌ 沙盒禁止 |

### 2.3 iOS PTY 限制根本原因

iOS 沙盒架构基于以下安全模型：
1. **无 fork** - 应用不能创建子进程
2. **无 exec** - 不能执行外部二进制
3. **无 PTY** - 无伪终端设备（`/dev/pts/` 不存在）
4. **无 shell** - `/bin/sh` 等不存在

---

## 3. 问题二：libxev 事件循环分析

### 3.1 当前 libxev 集成

**文件**: `src/global.zig:17`
```zig
pub const xev = @import("xev").Dynamic;
```

**初始化** (第 122-126 行):
```zig
if (comptime xev.dynamic) xev.detect() catch |err| {
    std.log.warn("failed to detect xev backend, falling back to " ++
        "most compatible backend err={}", .{err});
};
```

### 3.2 libxev 后端支持

| 平台 | 后端 | iOS 状态 |
|------|------|---------|
| macOS | kqueue | ✅ 可用 |
| iOS | kqueue | ⚠️ 有限制 |
| Linux | epoll/io_uring | N/A |

### 3.3 iOS kqueue 限制

iOS 的 kqueue 受以下限制:

1. **EVFILT_PROC 不可用** - 不能监视进程事件
2. **部分 kevent 过滤器受限** - 某些过滤器在沙盒中被禁用
3. **文件描述符限制** - iOS 对 FD 有更严格的限制

**libxev 中的相关代码** (需要在 libxev 库中修改):

```zig
// 在 libxev 中，Process 监视使用 EVFILT_PROC
// iOS 沙盒禁止这个过滤器
pub const Process = struct {
    pub fn init(pid: posix.pid_t) !Process {
        // kevent with EVFILT_PROC - 在 iOS 上会失败
    }
};
```

### 3.4 termio 中的 xev 使用点

**文件**: `src/termio/Thread.zig`
```zig
// 第 48, 55, 59, 64, 67, 71 行
loop: xev.Loop              // 主事件循环
stop: xev.Async             // 停止信号
scroll: xev.Timer           // 滚动定时器 (15ms)
coalesce: xev.Timer         // 调整大小合并 (25ms)
draw_now: xev.Async         // 立即绘制
sync_reset: xev.Timer       // 同步重置 (1000ms)
```

**文件**: `src/termio/Exec.zig`
```zig
// 第 105-108 行：进程监视 - iOS 上不可用
var process: ?xev.Process = if (self.subprocess.process) |v|
    switch (v) {
        .fork_exec => |cmd| try xev.Process.init(cmd.pid),
        .flatpak => null,
    }
else return error.ProcessNotStarted;

// 第 128 行：PTY 写入流
var stream = xev.Stream.initFd(pty_fds.write);

// 第 134 行：termios 定时器
var termios_timer = try xev.Timer.init();
```

---

## 4. 问题三：termio 架构耦合

### 4.1 当前 Backend 架构

**文件**: `src/termio/backend.zig`
```zig
/// 后端类型 - 当前只有一种
pub const Kind = enum { exec };

/// 后端配置联合体
pub const Config = union(Kind) {
    exec: termio.Exec.Config,
};

/// 后端实现联合体
pub const Backend = union(Kind) {
    exec: termio.Exec,

    pub fn threadEnter(...) !void {
        switch (self.*) {
            .exec => |*exec| try exec.threadEnter(...),
        }
    }

    pub fn queueWrite(...) !void {
        switch (self.*) {
            .exec => |*exec| try exec.queueWrite(...),
        }
    }
    // ... 其他方法
};
```

### 4.2 Exec 后端数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                        当前架构 (POSIX)                         │
└─────────────────────────────────────────────────────────────────┘

用户输入                                      PTY 输出
   │                                            ▲
   ▼                                            │
┌──────────────┐   queueWrite()   ┌─────────────┴─────────────┐
│  Surface     │ ───────────────► │      termio.Exec         │
│  (输入处理)  │                  │   ┌─────────────────────┐ │
└──────────────┘                  │   │    Subprocess       │ │
                                  │   │  ┌───────────────┐  │ │
                                  │   │  │      PTY      │  │ │
                                  │   │  │ master ◄─► slave│ │ │
                                  │   │  └───────┬───────┘  │ │
                                  │   │          │ fork()   │ │
                                  │   │          ▼          │ │
                                  │   │  ┌───────────────┐  │ │
                                  │   │  │    Shell      │  │ │
                                  │   │  │  (/bin/zsh)   │  │ │
                                  │   │  └───────────────┘  │ │
                                  │   └─────────────────────┘ │
                                  │                           │
                                  │   ┌─────────────────────┐ │
                                  │   │   ReadThread        │ │
                                  │   │ (poll + read PTY)   │ │
                                  │   └──────────┬──────────┘ │
                                  └──────────────┼────────────┘
                                                 │
                                                 ▼
                                  ┌─────────────────────────────┐
                                  │     Termio.processOutput()  │
                                  │     (解析 VT 转义序列)       │
                                  └──────────────┬──────────────┘
                                                 │
                                                 ▼
                                  ┌─────────────────────────────┐
                                  │    Terminal State Update    │
                                  │    (更新屏幕网格)           │
                                  └─────────────────────────────┘
```

### 4.3 需要解耦的核心接口

```zig
// 当前 Exec 的关键方法签名
pub fn threadEnter(self: *Exec, alloc: Allocator, io: *termio.Termio,
                   td: *termio.Termio.ThreadData) !void;
pub fn threadExit(self: *Exec, td: *termio.Termio.ThreadData) void;
pub fn queueWrite(self: *Exec, alloc: Allocator,
                  td: *termio.Termio.ThreadData,
                  data: []const u8, linefeed: bool) !void;
pub fn resize(self: *Exec, grid_size: renderer.GridSize,
              screen_size: renderer.ScreenSize) !void;
```

---

## 5. 问题四：libxev 异步机制深度分析

这是 Discussion #4087 中 kitknox 提到的核心问题：

> "libghostty was already in a state that it compiled on iOS, it did not actually work on iOS.
> The kevent queues in `libxev` didn't work on iOS without fixes."
> "These were all async crashes"

### 5.1 libxev kqueue 后端架构

libxev 在 Darwin 平台（macOS/iOS）使用 kqueue 作为事件循环后端：

```
┌─────────────────────────────────────────────────────────────────┐
│                     libxev kqueue 后端                           │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   xev.Async     │  │   xev.Process   │  │   xev.Timer     │ │
│  │ (EVFILT_MACHPORT│  │ (EVFILT_PROC)   │  │ (Timer Heap)    │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘ │
│           │                    │                    │           │
│           └────────────────────┼────────────────────┘           │
│                                ▼                                 │
│                    ┌───────────────────────┐                    │
│                    │      kqueue()         │                    │
│                    │   (kevent syscall)    │                    │
│                    └───────────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 iOS 沙盒对 kqueue 过滤器的限制

| kqueue 过滤器 | 用途 | macOS | iOS | 问题原因 |
|--------------|------|-------|-----|---------|
| `EVFILT_READ` | 文件/Socket 读 | ✅ | ✅ | - |
| `EVFILT_WRITE` | 文件/Socket 写 | ✅ | ✅ | - |
| `EVFILT_TIMER` | 定时器 | ✅ | ✅ | - |
| `EVFILT_PROC` | 进程监视 | ✅ | ❌ | **沙盒禁止监视其他进程** |
| `EVFILT_MACHPORT` | Mach 端口 | ✅ | ⚠️ | **部分 Mach API 受限** |
| `EVFILT_USER` | 用户事件 | ✅ | ⚠️ | **需要测试** |

### 5.3 xev.Async 实现分析

libxev 的 `Async` 在 Darwin 上使用 **Mach Ports**:

```zig
// libxev kqueue.zig 中的 Async 实现
// 使用 EVFILT_MACHPORT 和 Mach 消息传递

pub const Async = struct {
    mach_port: mach_port_t,  // Mach 端口

    pub fn init() !Async {
        // 创建 Mach 端口
        const port = mach_port_allocate(...);
        return .{ .mach_port = port };
    }

    pub fn notify(self: *Async) !void {
        // 发送 Mach 消息唤醒等待者
        mach_msg_send(...);
    }

    pub fn wait(self: *Async, loop: *Loop, ...) void {
        // 使用 EVFILT_MACHPORT 监听端口
        kevent(..., EVFILT_MACHPORT, ...);
    }
};
```

**iOS Mach 端口限制**:
- iOS 沙盒对 Mach 端口有严格的 IPC 限制
- 某些 Mach API 在沙盒应用中可能失败
- 端口分配和消息发送可能需要特殊权限

### 5.4 xev.Process 实现分析

```zig
// libxev kqueue.zig 中的 Process 实现
pub const Process = struct {
    pid: pid_t,

    pub fn init(pid: pid_t) !Process {
        return .{ .pid = pid };
    }

    pub fn wait(self: *Process, loop: *Loop, ...) void {
        // 使用 EVFILT_PROC 监视进程
        kevent(loop.fd, &[_]Kevent{
            .{
                .ident = @intCast(self.pid),
                .filter = EVFILT_PROC,
                .flags = EV_ADD | EV_ONESHOT,
                .fflags = NOTE_EXIT | NOTE_EXITSTATUS,
                ...
            }
        }, ...);
    }
};
```

**iOS 上 EVFILT_PROC 完全不可用**:
- iOS 沙盒禁止应用监视其他进程
- 即使是自己 fork 的子进程也无法监视（因为 iOS 禁止 fork）
- 这是 Ghostty 在 iOS 上的致命问题之一

### 5.5 Ghostty 中 xev.Async 的所有使用位置

| 文件 | 用途 | 类型 | iOS 风险 |
|------|------|------|---------|
| `src/termio/Thread.zig:55` | 停止信号 | `stop: xev.Async` | ⚠️ 高 |
| `src/termio/Thread.zig:67` | 立即绘制 | `draw_now: xev.Async` | ⚠️ 高 |
| `src/termio/mailbox.zig:32` | 唤醒 | `wakeup: xev.Async` | ⚠️ 高 |
| `src/renderer/Thread.zig:47` | 唤醒 | `wakeup: xev.Async` | ⚠️ 高 |
| `src/renderer/Thread.zig:51` | 停止 | `stop: xev.Async` | ⚠️ 高 |
| `src/renderer/Thread.zig:67` | 立即绘制 | `draw_now: xev.Async` | ⚠️ 高 |
| `src/terminal/search/Thread.zig:55` | 唤醒 | `wakeup: xev.Async` | ⚠️ 高 |
| `src/terminal/search/Thread.zig:59` | 停止 | `stop: xev.Async` | ⚠️ 高 |
| `src/termio/Termio.zig:47` | 渲染器唤醒 | `renderer_wakeup: xev.Async` | ⚠️ 高 |
| `src/os/cf_release_thread.zig:40` | 唤醒 | `wakeup: xev.Async` | ⚠️ 高 |

### 5.6 xev.Process 在 Ghostty 中的使用

```zig
// src/termio/Exec.zig:105-115
var process: ?xev.Process = if (self.subprocess.process) |v|
    switch (v) {
        .fork_exec => |cmd| try xev.Process.init(
            cmd.pid orelse return error.ProcessNoPid,
        ),
        .flatpak => null,
    }
else return error.ProcessNotStarted;
```

**这段代码在 iOS 上会导致崩溃**，因为:
1. iOS 不允许 fork，所以 `subprocess.process` 永远是 null
2. 即使有 PID，`xev.Process.init` 也会失败

### 5.7 kitknox 的修复方案推测

根据讨论中的描述，kitknox 可能采取了以下修复:

**方案 A: 替换 Mach 端口为管道**
```zig
// 使用 pipe() + poll() 替代 Mach 端口
pub const Async = struct {
    pipe_fds: [2]fd_t,

    pub fn init() !Async {
        const fds = try posix.pipe();
        return .{ .pipe_fds = fds };
    }

    pub fn notify(self: *Async) !void {
        _ = try posix.write(self.pipe_fds[1], &[_]u8{1});
    }

    pub fn wait(self: *Async, loop: *Loop) void {
        // 使用 EVFILT_READ 监听管道
        kevent(..., EVFILT_READ, self.pipe_fds[0], ...);
    }
};
```

**方案 B: 使用 dispatch_source (Apple 推荐)**
```zig
// 使用 Grand Central Dispatch
pub const Async = struct {
    source: dispatch_source_t,

    pub fn init() !Async {
        const queue = dispatch_get_global_queue(...);
        const source = dispatch_source_create(
            DISPATCH_SOURCE_TYPE_DATA_ADD, 0, 0, queue
        );
        return .{ .source = source };
    }

    pub fn notify(self: *Async) !void {
        dispatch_source_merge_data(self.source, 1);
    }
};
```

### 5.8 iOS 异步问题的完整修复清单

| 问题 | 当前实现 | 修复方案 | 优先级 |
|------|---------|---------|--------|
| `xev.Async` 崩溃 | Mach 端口 | 使用 pipe + EVFILT_READ | P0 |
| `xev.Process` 不可用 | EVFILT_PROC | 管道后端不需要进程监视 | P0 |
| 事件循环初始化失败 | kqueue 创建 | 应该可用，需测试 | P1 |
| Timer 问题 | Timer Heap | 应该可用，需测试 | P2 |

### 5.9 libxev iOS 修复方案

#### 方案一：Fork libxev 添加 iOS 支持 (推荐)

创建 libxev 的 fork 版本，针对 iOS 修改 kqueue 后端：

**修改文件**: `src/backend/kqueue.zig`

```zig
// 1. 修改 Async 实现，使用管道替代 Mach 端口
pub const Async = struct {
    // iOS: 使用管道
    // macOS: 保持 Mach 端口
    backend: if (builtin.os.tag == .ios) PipeAsync else MachPortAsync,

    const PipeAsync = struct {
        read_fd: posix.fd_t,
        write_fd: posix.fd_t,

        pub fn init() !PipeAsync {
            const fds = try posix.pipe();
            // 设置非阻塞
            _ = try posix.fcntl(fds[0], .F_SETFL, posix.O.NONBLOCK);
            return .{ .read_fd = fds[0], .write_fd = fds[1] };
        }

        pub fn deinit(self: *PipeAsync) void {
            posix.close(self.read_fd);
            posix.close(self.write_fd);
        }

        pub fn notify(self: *PipeAsync) !void {
            _ = posix.write(self.write_fd, &[_]u8{1}) catch |err| switch (err) {
                error.WouldBlock => {}, // 忽略，已经有通知待处理
                else => return err,
            };
        }

        pub fn wait(self: *PipeAsync, loop: *Loop, c: *Completion) void {
            // 使用 EVFILT_READ 替代 EVFILT_MACHPORT
            c.* = .{
                .op = .{ .async_wait = .{ .async = self } },
                .kevent = .{
                    .ident = @intCast(self.read_fd),
                    .filter = std.c.EVFILT.READ,
                    .flags = std.c.EV.ADD | std.c.EV.ENABLE | std.c.EV.ONESHOT,
                    .fflags = 0,
                    .data = 0,
                    .udata = @intFromPtr(c),
                },
            };
            loop.add(c);
        }

        pub fn drain(self: *PipeAsync) void {
            // 排空管道中的所有通知字节
            var buf: [64]u8 = undefined;
            while (true) {
                _ = posix.read(self.read_fd, &buf) catch break;
            }
        }
    };
};
```

#### 方案二：条件编译跳过问题组件

在 Ghostty 中添加 iOS 条件编译：

```zig
// src/termio/Exec.zig
pub fn threadEnter(...) !void {
    // ...

    // iOS: 不使用进程监视
    var process: ?xev.Process = if (comptime builtin.os.tag == .ios)
        null
    else if (self.subprocess.process) |v|
        switch (v) {
            .fork_exec => |cmd| try xev.Process.init(cmd.pid),
            .flatpak => null,
        }
    else
        return error.ProcessNotStarted;

    // ...
}
```

#### 方案三：使用 Zig 标准库替代

对于某些场景，可以使用 Zig 标准库的线程原语替代 xev.Async：

```zig
const std = @import("std");

pub const SimpleAsync = struct {
    mutex: std.Thread.Mutex = .{},
    cond: std.Thread.Condition = .{},
    signaled: bool = false,

    pub fn notify(self: *SimpleAsync) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        self.signaled = true;
        self.cond.signal();
    }

    pub fn wait(self: *SimpleAsync) void {
        self.mutex.lock();
        defer self.mutex.unlock();
        while (!self.signaled) {
            self.cond.wait(&self.mutex);
        }
        self.signaled = false;
    }
};
```

**注意**: 这种方案不能与 xev.Loop 集成，需要独立的等待线程。

### 5.10 libxev 修复的工作量评估

| 任务 | 复杂度 | 时间估计 |
|------|--------|---------|
| Fork libxev 并设置构建 | 低 | 0.5 天 |
| 实现 PipeAsync 替代 | 中 | 1-2 天 |
| 禁用/存根 Process | 低 | 0.5 天 |
| 测试基本事件循环 | 中 | 1 天 |
| 集成到 Ghostty | 中 | 1 天 |
| **总计** | | **4-5 天** |

---

## 6. 改造方案：管道后端 (Pipe Backend)

### 6.1 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                     改造后架构 (iOS)                            │
└─────────────────────────────────────────────────────────────────┘

用户输入                                      终端输出
   │                                            ▲
   ▼                                            │
┌──────────────┐   queueWrite()   ┌─────────────┴─────────────┐
│  Surface     │ ───────────────► │      termio.Pipe         │
│  (输入处理)  │                  │   (新增后端)              │
└──────────────┘                  │                           │
                                  │   ┌─────────────────────┐ │
                                  │   │    PipeBackend      │ │
                                  │   │  ┌───────────────┐  │ │
                                  │   │  │  write_fd     │──┼─┼──► 到宿主应用
                                  │   │  │  read_fd      │◄─┼─┼─── 从宿主应用
                                  │   │  └───────────────┘  │ │
                                  │   └─────────────────────┘ │
                                  │                           │
                                  │   ┌─────────────────────┐ │
                                  │   │   ReadThread        │ │
                                  │   │ (poll + read pipe)  │ │
                                  │   └──────────┬──────────┘ │
                                  └──────────────┼────────────┘
                                                 │
                                                 ▼
                                  ┌─────────────────────────────┐
                                  │     Termio.processOutput()  │
                                  │     (解析 VT 转义序列)       │
                                  └──────────────┬──────────────┘
                                                 │
                                                 ▼
                                  ┌─────────────────────────────┐
                                  │    Terminal State Update    │
                                  └─────────────────────────────┘


┌─────────────────────────────────────────────────────────────────┐
│                     宿主应用 (Swift/iOS)                        │
└─────────────────────────────────────────────────────────────────┘

                          ┌─────────────────────┐
                          │   I/O Provider      │
                          │  ┌───────────────┐  │
 从 libghostty ──────────►│  │  SSH Backend  │  │───► 远程服务器
                          │  │(Swift-NIO-SSH)│  │
                          │  └───────────────┘  │
                          │         或          │
                          │  ┌───────────────┐  │
                          │  │  Local Shell  │  │───► 内置命令
                          │  │  (受限)       │  │
                          │  └───────────────┘  │
                          └─────────────────────┘
```

### 6.2 新增文件: `src/termio/Pipe.zig`

```zig
//! Pipe 后端实现，用于 iOS 等无 PTY 环境
//! 通过管道与宿主应用通信，宿主负责实际的 I/O
const Pipe = @This();

const std = @import("std");
const builtin = @import("builtin");
const Allocator = std.mem.Allocator;
const posix = std.posix;
const xev = @import("../global.zig").xev;
const termio = @import("../termio.zig");
const renderer = @import("../renderer.zig");
const terminal = @import("../terminal/main.zig");

const log = std.log.scoped(.io_pipe);

/// 管道配置 - 由宿主应用提供 FD
pub const Config = struct {
    /// 读取端 FD (从宿主应用接收数据)
    read_fd: posix.fd_t,
    /// 写入端 FD (向宿主应用发送数据)
    write_fd: posix.fd_t,
    /// 终端初始尺寸
    initial_size: struct {
        cols: u16 = 80,
        rows: u16 = 24,
        width_px: u16 = 800,
        height_px: u16 = 600,
    } = .{},
};

/// 管道状态
read_fd: posix.fd_t,
write_fd: posix.fd_t,
size: renderer.Size,

/// 线程数据
pub const ThreadData = struct {
    write_stream: xev.Stream,
    read_thread: std.Thread,
    quit_pipe: [2]posix.fd_t,

    // 写入池
    write_req_pool: WriteReqPool,
    write_buf_pool: WriteBufPool,
    write_queue: xev.WriteQueue,

    const WriteReqPool = @import("../datastruct/main.zig")
        .SegmentedPool(xev.WriteRequest, 32);
    const WriteBufPool = @import("../datastruct/main.zig")
        .SegmentedPool([64]u8, 32);

    pub fn deinit(self: *ThreadData, alloc: Allocator) void {
        // 发送退出信号
        _ = posix.write(self.quit_pipe[1], &[_]u8{0}) catch {};

        // 等待读线程结束
        self.read_thread.join();

        // 关闭管道
        posix.close(self.quit_pipe[0]);
        posix.close(self.quit_pipe[1]);

        // 释放写入流
        self.write_stream.deinit();

        // 释放池
        self.write_req_pool.deinit(alloc);
        self.write_buf_pool.deinit(alloc);
    }
};

pub fn init(alloc: Allocator, cfg: Config) !Pipe {
    _ = alloc;

    return .{
        .read_fd = cfg.read_fd,
        .write_fd = cfg.write_fd,
        .size = .{
            .grid = .{
                .columns = cfg.initial_size.cols,
                .rows = cfg.initial_size.rows,
            },
            .screen = .{
                .width = cfg.initial_size.width_px,
                .height = cfg.initial_size.height_px,
            },
        },
    };
}

pub fn deinit(self: *Pipe) void {
    // 管道 FD 由宿主应用管理，这里不关闭
    self.* = undefined;
}

pub fn initTerminal(self: *Pipe, term: *terminal.Terminal) void {
    term.resize(.{
        .cols = self.size.grid.columns,
        .rows = self.size.grid.rows,
        .width_px = self.size.screen.width,
        .height_px = self.size.screen.height,
    });
}

pub fn threadEnter(
    self: *Pipe,
    alloc: Allocator,
    io: *termio.Termio,
    td: *termio.Termio.ThreadData,
) !void {
    // 创建退出管道
    const quit_pipe = try posix.pipe();
    errdefer {
        posix.close(quit_pipe[0]);
        posix.close(quit_pipe[1]);
    }

    // 创建写入流
    var stream = xev.Stream.initFd(self.write_fd);
    errdefer stream.deinit();

    // 启动读取线程
    const read_thread = try std.Thread.spawn(
        .{},
        readThreadMain,
        .{ self.read_fd, io, quit_pipe[0] },
    );
    read_thread.setName("io-pipe-reader") catch {};

    // 初始化线程数据
    td.backend = .{ .pipe = .{
        .write_stream = stream,
        .read_thread = read_thread,
        .quit_pipe = quit_pipe,
        .write_req_pool = ThreadData.WriteReqPool.init(alloc),
        .write_buf_pool = ThreadData.WriteBufPool.init(alloc),
        .write_queue = .{},
    }};
}

pub fn threadExit(self: *Pipe, td: *termio.Termio.ThreadData) void {
    _ = self;
    td.backend.pipe.deinit(td.alloc);
}

pub fn queueWrite(
    self: *Pipe,
    alloc: Allocator,
    td: *termio.Termio.ThreadData,
    data: []const u8,
    linefeed: bool,
) !void {
    _ = self;

    const pipe_td = &td.backend.pipe;

    // 分块写入 (64 字节块)
    var remaining = data;
    while (remaining.len > 0) {
        const chunk_len = @min(remaining.len, 64);
        const chunk = remaining[0..chunk_len];
        remaining = remaining[chunk_len..];

        // 从池获取缓冲区
        const buf = try pipe_td.write_buf_pool.get(alloc);
        @memcpy(buf[0..chunk_len], chunk);

        // 处理 linefeed 转换
        if (linefeed and chunk_len > 0 and chunk[chunk_len - 1] == '\r') {
            // 需要添加 '\n'，这里简化处理
        }

        // 获取写入请求
        const req = try pipe_td.write_req_pool.get(alloc);

        // 队列写入
        pipe_td.write_stream.queueWrite(
            td.loop,
            &pipe_td.write_queue,
            req,
            .{ .slice = buf[0..chunk_len] },
            ThreadData,
            pipe_td,
            onWriteComplete,
        );
    }
}

fn onWriteComplete(
    pipe_td: ?*ThreadData,
    _: *xev.Loop,
    _: *xev.Completion,
    req: *xev.WriteRequest,
    buf: xev.WriteBuffer,
    r: xev.WriteError!usize,
) xev.CallbackAction {
    const td = pipe_td orelse return .disarm;

    // 归还资源到池
    td.write_req_pool.put(req);
    if (buf == .slice) {
        // 归还缓冲区
    }

    // 检查错误
    _ = r catch |err| {
        log.warn("pipe write error: {}", .{err});
        return .disarm;
    };

    return .disarm;
}

pub fn resize(
    self: *Pipe,
    grid_size: renderer.GridSize,
    screen_size: renderer.ScreenSize,
) !void {
    self.size = .{
        .grid = grid_size,
        .screen = screen_size,
    };

    // 通知宿主应用尺寸变化
    // 可以通过特殊的控制消息或回调实现
}

pub fn focusGained(self: *Pipe, td: *termio.Termio.ThreadData, focused: bool) !void {
    _ = self;
    _ = td;
    _ = focused;
}

/// 读取线程主函数
fn readThreadMain(
    read_fd: posix.fd_t,
    io: *termio.Termio,
    quit_fd: posix.fd_t,
) void {
    // 设置非阻塞
    const flags = posix.fcntl(read_fd, posix.F.GETFL, 0) catch return;
    _ = posix.fcntl(read_fd, posix.F.SETFL, flags | posix.O.NONBLOCK) catch return;

    var buf: [4096]u8 = undefined;

    while (true) {
        // 使用 poll 监视读取 FD 和退出 FD
        var fds = [_]posix.pollfd{
            .{ .fd = read_fd, .events = posix.POLL.IN, .revents = 0 },
            .{ .fd = quit_fd, .events = posix.POLL.IN, .revents = 0 },
        };

        const poll_result = posix.poll(&fds, -1) catch break;
        if (poll_result == 0) continue;

        // 检查退出信号
        if (fds[1].revents & posix.POLL.IN != 0) break;

        // 检查读取事件
        if (fds[0].revents & posix.POLL.IN != 0) {
            // 紧密循环读取以最大化性能
            while (true) {
                const n = posix.read(read_fd, &buf) catch |err| switch (err) {
                    error.WouldBlock => break,
                    else => return,
                };

                if (n == 0) break;

                // 处理输出
                io.processOutput(buf[0..n]);
            }
        }

        // 检查挂断
        if (fds[0].revents & posix.POLL.HUP != 0) break;
    }
}
```

### 6.3 修改 `src/termio/backend.zig`

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;
const builtin = @import("builtin");
const renderer = @import("../renderer.zig");
const terminal = @import("../terminal/main.zig");
const termio = @import("../termio.zig");

/// 后端类型
pub const Kind = enum {
    exec,
    pipe,  // 新增
};

/// 配置联合体
pub const Config = union(Kind) {
    exec: termio.Exec.Config,
    pipe: termio.Pipe.Config,  // 新增
};

/// 后端实现联合体
pub const Backend = union(Kind) {
    exec: termio.Exec,
    pipe: termio.Pipe,  // 新增

    pub fn deinit(self: *Backend) void {
        switch (self.*) {
            .exec => |*exec| exec.deinit(),
            .pipe => |*pipe| pipe.deinit(),  // 新增
        }
    }

    pub fn initTerminal(self: *Backend, t: *terminal.Terminal) void {
        switch (self.*) {
            .exec => |*exec| exec.initTerminal(t),
            .pipe => |*pipe| pipe.initTerminal(t),  // 新增
        }
    }

    pub fn threadEnter(
        self: *Backend,
        alloc: Allocator,
        io: *termio.Termio,
        td: *termio.Termio.ThreadData,
    ) !void {
        switch (self.*) {
            .exec => |*exec| try exec.threadEnter(alloc, io, td),
            .pipe => |*pipe| try pipe.threadEnter(alloc, io, td),  // 新增
        }
    }

    pub fn threadExit(self: *Backend, td: *termio.Termio.ThreadData) void {
        switch (self.*) {
            .exec => |*exec| exec.threadExit(td),
            .pipe => |*pipe| pipe.threadExit(td),  // 新增
        }
    }

    pub fn queueWrite(
        self: *Backend,
        alloc: Allocator,
        td: *termio.Termio.ThreadData,
        data: []const u8,
        linefeed: bool,
    ) !void {
        switch (self.*) {
            .exec => |*exec| try exec.queueWrite(alloc, td, data, linefeed),
            .pipe => |*pipe| try pipe.queueWrite(alloc, td, data, linefeed),  // 新增
        }
    }

    pub fn resize(
        self: *Backend,
        grid_size: renderer.GridSize,
        screen_size: renderer.ScreenSize,
    ) !void {
        switch (self.*) {
            .exec => |*exec| try exec.resize(grid_size, screen_size),
            .pipe => |*pipe| try pipe.resize(grid_size, screen_size),  // 新增
        }
    }

    // 其他方法类似处理...
};

/// 线程数据联合体
pub const ThreadData = union(Kind) {
    exec: termio.Exec.ThreadData,
    pipe: termio.Pipe.ThreadData,  // 新增

    pub fn deinit(self: *ThreadData, alloc: Allocator) void {
        switch (self.*) {
            .exec => |*exec| exec.deinit(alloc),
            .pipe => |*pipe| pipe.deinit(alloc),  // 新增
        }
    }
};
```

### 6.4 修改 `src/pty.zig` - 添加 PipePty

```zig
pub const Pty = switch (builtin.os.tag) {
    .windows => WindowsPty,
    .ios => PipePty,  // 修改：使用 PipePty 替代 NullPty
    else => PosixPty,
};

/// 管道模式 PTY - 用于 iOS 等无真实 PTY 的环境
/// 不创建真实的伪终端，而是使用宿主提供的管道
pub const PipePty = struct {
    pub const Error = OpenError || SetSizeError;
    pub const Fd = posix.fd_t;

    /// 读取端 (从宿主接收)
    read_fd: Fd,
    /// 写入端 (向宿主发送)
    write_fd: Fd,
    /// 当前尺寸
    size: winsize,

    pub const OpenError = error{InvalidPipeFds};

    /// 使用宿主提供的管道 FD 初始化
    pub fn initWithPipe(
        read_fd: Fd,
        write_fd: Fd,
        size: winsize,
    ) OpenError!PipePty {
        if (read_fd < 0 or write_fd < 0) {
            return error.InvalidPipeFds;
        }
        return .{
            .read_fd = read_fd,
            .write_fd = write_fd,
            .size = size,
        };
    }

    /// 兼容原有接口，但在管道模式下不应被调用
    pub fn open(size: winsize) OpenError!PipePty {
        // iOS 上不能直接 open PTY，必须由宿主提供管道
        _ = size;
        return error.InvalidPipeFds;
    }

    pub fn deinit(self: *PipePty) void {
        // 管道由宿主管理，这里不关闭
        self.* = undefined;
    }

    pub const SetSizeError = error{};

    pub fn setSize(self: *PipePty, size: winsize) SetSizeError!void {
        self.size = size;
        // 通知宿主尺寸变化（通过回调或特殊消息）
    }

    pub fn getSize(self: PipePty) winsize {
        return self.size;
    }

    pub const GetModeError = error{};

    pub fn getMode(self: PipePty) GetModeError!Mode {
        _ = self;
        // 管道模式下返回默认值
        return .{ .canonical = false, .echo = true };
    }

    pub const ChildPreExecError = error{NotSupported};

    pub fn childPreExec(self: PipePty) ChildPreExecError!void {
        _ = self;
        // iOS 上没有子进程
        return error.NotSupported;
    }
};
```

---

## 7. C API 扩展

### 7.1 新增 API 函数 (`include/ghostty.h`)

```c
// 管道后端配置
typedef struct {
    int read_fd;   // 读取端 FD (从宿主接收数据)
    int write_fd;  // 写入端 FD (向宿主发送数据)
    uint16_t cols;
    uint16_t rows;
    uint16_t width_px;
    uint16_t height_px;
} ghostty_pipe_config_s;

// 使用管道后端创建 Surface (iOS 专用)
ghostty_surface_t ghostty_surface_new_with_pipe(
    ghostty_app_t app,
    const ghostty_surface_config_s* config,
    const ghostty_pipe_config_s* pipe_config
);

// 通知管道后端尺寸变化
void ghostty_surface_pipe_resize(
    ghostty_surface_t surface,
    uint16_t cols,
    uint16_t rows,
    uint16_t width_px,
    uint16_t height_px
);

// 向管道后端写入数据 (模拟终端输出)
void ghostty_surface_pipe_write(
    ghostty_surface_t surface,
    const char* data,
    size_t len
);
```

### 7.2 实现 (`src/apprt/embedded.zig`)

```zig
/// 使用管道后端创建 Surface
export fn ghostty_surface_new_with_pipe(
    app: *App,
    config: *const Surface.Options,
    pipe_config: *const PipeConfig,
) ?*Surface {
    // 创建带管道后端的 Surface
    const backend_config = termio.Backend.Config{
        .pipe = .{
            .read_fd = pipe_config.read_fd,
            .write_fd = pipe_config.write_fd,
            .initial_size = .{
                .cols = pipe_config.cols,
                .rows = pipe_config.rows,
                .width_px = pipe_config.width_px,
                .height_px = pipe_config.height_px,
            },
        },
    };

    return app.newSurfaceWithBackend(config.*, backend_config) catch |err| {
        log.err("failed to create pipe surface err={}", .{err});
        return null;
    };
}

const PipeConfig = extern struct {
    read_fd: c_int,
    write_fd: c_int,
    cols: u16,
    rows: u16,
    width_px: u16,
    height_px: u16,
};
```

---

## 8. libxev iOS 修复方案

### 8.1 问题分析

libxev 在 iOS 上的主要问题是 `xev.Process` 使用 `EVFILT_PROC`，这在 iOS 沙盒中被禁用。

### 8.2 修复策略

**方案 A: 禁用 Process 监视 (推荐)**

在管道后端中，我们不需要进程监视（因为没有子进程），所以可以完全跳过这部分：

```zig
// src/termio/Pipe.zig
// 不使用 xev.Process，因为管道模式没有子进程

pub fn threadEnter(...) !void {
    // 无需 process watcher
    // 只需要 Timer 和 Stream，这些在 iOS 上可用
}
```

**方案 B: 修改 libxev (如果需要)**

如果需要在 iOS 上使用 libxev 的其他功能，可能需要 fork libxev 并：

```zig
// 在 libxev 中添加平台检查
pub const Process = if (builtin.os.tag == .ios)
    struct {
        // iOS 存根实现
        pub fn init(pid: posix.pid_t) !Process { return error.NotSupported; }
    }
else
    struct {
        // 原有实现
    };
```

### 8.3 安全的 xev 组件

以下 xev 组件在 iOS 上应该可以正常工作：

| 组件 | iOS 状态 | 用途 |
|------|---------|------|
| `xev.Loop` | ✅ 可用 | 事件循环 |
| `xev.Timer` | ✅ 可用 | 定时器 |
| `xev.Async` | ⚠️ 需测试 | 线程间信号 |
| `xev.Stream` | ✅ 可用 | 文件描述符 I/O |
| `xev.Process` | ❌ 不可用 | 进程监视 |

---

## 9. 宿主应用集成

### 9.1 Swift 集成示例

```swift
import GhosttyKit

class PipeTerminalBackend {
    private var readPipe: [Int32] = [-1, -1]
    private var writePipe: [Int32] = [-1, -1]
    private var sshChannel: SSHChannel?

    init() throws {
        // 创建管道
        // readPipe: libghostty 读取 (来自 SSH)
        // writePipe: libghostty 写入 (发送到 SSH)
        var pipeFds = [Int32](repeating: 0, count: 2)
        guard pipe(&pipeFds) == 0 else {
            throw PipeError.creationFailed
        }
        readPipe[0] = pipeFds[0]  // libghostty 读取端
        readPipe[1] = pipeFds[1]  // 我们写入端

        guard pipe(&pipeFds) == 0 else {
            throw PipeError.creationFailed
        }
        writePipe[0] = pipeFds[0]  // 我们读取端
        writePipe[1] = pipeFds[1]  // libghostty 写入端
    }

    func createSurface(app: ghostty_app_t, view: UIView) -> ghostty_surface_t? {
        var surfaceConfig = ghostty_surface_config_new()
        surfaceConfig.platform_tag = GHOSTTY_PLATFORM_IOS
        surfaceConfig.platform.ios.uiview = Unmanaged.passUnretained(view).toOpaque()

        var pipeConfig = ghostty_pipe_config_s(
            read_fd: readPipe[0],    // libghostty 从这里读取
            write_fd: writePipe[1],  // libghostty 向这里写入
            cols: 80,
            rows: 24,
            width_px: 800,
            height_px: 600
        )

        return ghostty_surface_new_with_pipe(app, &surfaceConfig, &pipeConfig)
    }

    // 从 SSH 接收数据，转发到 libghostty
    func onSSHData(_ data: Data) {
        data.withUnsafeBytes { bytes in
            write(readPipe[1], bytes.baseAddress, bytes.count)
        }
    }

    // 从 libghostty 读取数据，发送到 SSH
    func startReadLoop() {
        DispatchQueue.global().async { [weak self] in
            guard let self = self else { return }
            var buffer = [UInt8](repeating: 0, count: 4096)

            while true {
                let n = read(self.writePipe[0], &buffer, buffer.count)
                if n <= 0 { break }

                let data = Data(bytes: buffer, count: n)
                self.sshChannel?.write(data)
            }
        }
    }
}
```

---

## 10. 实施步骤

### 阶段 1: 基础管道后端 (1 周)

1. 创建 `src/termio/Pipe.zig`
2. 修改 `src/termio/backend.zig` 添加 `pipe` 变体
3. 更新 `src/pty.zig` 的 `PipePty`
4. 基础单元测试

### 阶段 2: C API 扩展 (3-4 天)

1. 更新 `include/ghostty.h`
2. 实现 `src/apprt/embedded.zig` 中的新函数
3. 添加 iOS 构建配置

### 阶段 3: libxev 兼容 (3-4 天)

1. 测试 xev 组件在 iOS 上的行为
2. 如需要，提交 libxev PR 或维护 fork
3. 更新 `build.zig.zon` 依赖

### 阶段 4: 集成测试 (1 周)

1. 创建测试 iOS 应用
2. 集成 SSH 后端测试
3. 性能优化

---

## 11. 关键文件变更清单

| 文件 | 操作 | 变更内容 |
|------|------|---------|
| `src/termio/Pipe.zig` | **新增** | 管道后端完整实现 |
| `src/termio/backend.zig` | 修改 | 添加 `pipe` 变体 |
| `src/termio.zig` | 修改 | 导出 `Pipe` 模块 |
| `src/pty.zig` | 修改 | `PipePty` 替代 `NullPty` |
| `include/ghostty.h` | 修改 | 添加管道相关 API |
| `src/apprt/embedded.zig` | 修改 | 实现新的 C API |
| `build.zig` | 修改 | iOS 构建配置 |

---

## 12. SwiftTerm iOS 架构参考

[SwiftTerm](https://github.com/migueldeicaza/SwiftTerm) 是一个成熟的 Swift 终端模拟器实现，其 iOS 架构对 libghostty iOS 改造具有重要参考价值。

### 12.1 SwiftTerm 架构概述

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SwiftTerm 架构                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Terminal (核心引擎)                            │   │
│  │  • VT100/Xterm 转义序列解析                                       │   │
│  │  • 双缓冲区管理 (normalBuffer + altBuffer)                        │   │
│  │  • UTF-8 处理 (ReadingBuffer with putback)                       │   │
│  │  • TerminalDelegate 回调接口                                      │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                 TerminalView (iOS/UIKit)                         │   │
│  │  • 继承 UIScrollView                                              │   │
│  │  • 实现 TerminalDelegate                                          │   │
│  │  • CADisplayLink 渲染同步                                         │   │
│  │  • UITextInput 键盘支持                                           │   │
│  │  • 手势识别 (tap, pan, long-press)                                │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              TerminalViewDelegate (宿主应用)                      │   │
│  │  • send(source:data:) - 发送用户输入                              │   │
│  │  • sizeChanged() - 终端尺寸变化                                   │   │
│  │  • hostCurrentDirectory() - 当前目录                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 12.2 关键设计模式

#### 12.2.1 双向委托模式 (Dual Delegation)

SwiftTerm 使用两层委托实现解耦：

| 委托层 | 接口 | 方向 | 用途 |
|--------|------|------|------|
| **Terminal → View** | `TerminalDelegate` | 内 → 外 | 终端状态变化通知 |
| **View → Host** | `TerminalViewDelegate` | 视图 → 宿主 | 用户交互转发 |

```swift
// TerminalDelegate - 终端引擎回调
protocol TerminalDelegate {
    func send(source: Terminal, data: ArraySlice<UInt8>)  // 终端生成的数据
    func showCursor(source: Terminal)
    func hideCursor(source: Terminal)
    func scrolled(source: Terminal, yDisp: Int)
    func linefeed(source: Terminal)
    func cursorStyleChanged(source: Terminal, newStyle: CursorStyle)
    func setTerminalTitle(source: Terminal, title: String)
    func colorChanged(source: Terminal, idx: Int, color: Color)
}

// TerminalViewDelegate - 视图层回调
protocol TerminalViewDelegate {
    func send(source: TerminalView, data: ArraySlice<UInt8>)  // 用户输入
    func sizeChanged(source: TerminalView, newCols: Int, newRows: Int)
    func requestOpenLink(source: TerminalView, link: String, params: [String:String])
    func clipboardCopy(source: TerminalView, content: Data)
}
```

#### 12.2.2 数据流分离

```
输入流 (用户 → 远程):
┌──────────┐   UITextInput    ┌──────────────┐   delegate.send()   ┌──────────┐
│ Keyboard │ ───────────────► │ TerminalView │ ──────────────────► │ SSH/Pipe │
└──────────┘                  └──────────────┘                     └──────────┘

输出流 (远程 → 显示):
┌──────────┐   terminal.feed()  ┌──────────┐   TerminalDelegate   ┌──────────────┐
│ SSH/Pipe │ ─────────────────► │ Terminal │ ───────────────────► │ TerminalView │
└──────────┘                    └──────────┘                      └──────────────┘
```

#### 12.2.3 iOS SSH 实现 (UIKitSshTerminalView)

SwiftTerm 的 iOS SSH 示例展示了关键的异步 I/O 模式：

```swift
class SshTerminalView: TerminalView, TerminalViewDelegate {
    var shell: SSHShell?
    let sshQueue = DispatchQueue(label: "SSH Queue")

    func connect() {
        sshQueue.async {
            self.shell = try? SSHShell(host: "example.com", port: 22)
            self.shell?.connect()
                .authenticate(...)
                .open { [weak self] data in
                    // 接收 SSH 数据，分块处理
                    self?.processSSHData(data)
                }
        }
    }

    private func processSSHData(_ data: Data) {
        // 分块避免阻塞 UI
        let chunkSize = 1024
        var offset = 0
        while offset < data.count {
            let chunk = data[offset..<min(offset + chunkSize, data.count)]
            DispatchQueue.main.sync {
                // 必须在主线程 feed 终端
                self.getTerminal().feed(buffer: Array(chunk))
            }
            offset += chunkSize
        }
    }

    // TerminalViewDelegate: 用户输入
    func send(source: TerminalView, data: ArraySlice<UInt8>) {
        sshQueue.async {
            self.shell?.write(Data(data))
        }
    }

    // 终端尺寸变化
    func sizeChanged(source: TerminalView, newCols: Int, newRows: Int) {
        sshQueue.async {
            self.shell?.setTerminalSize(width: newCols, height: newRows)
        }
    }
}
```

### 12.3 对 libghostty iOS 改造的启示

#### 12.3.1 架构映射

| SwiftTerm 组件 | libghostty 对应 | 改造建议 |
|----------------|-----------------|----------|
| `Terminal` | `terminal.Terminal` | 保持不变，核心引擎已解耦 |
| `TerminalView` | Metal 渲染视图 | 使用现有渲染器 |
| `TerminalDelegate` | 回调机制 | 已存在，无需修改 |
| `TerminalViewDelegate` | C API 回调 | 需扩展 |
| SSH 后端 | **需新增** | 管道后端 (`termio.Pipe`) |

#### 12.3.2 推荐的 C API 回调结构

基于 SwiftTerm 的委托模式，libghostty 应添加以下回调：

```c
// ghostty.h - iOS 回调结构
typedef struct {
    // 用户输入回调 (对应 TerminalViewDelegate.send)
    void (*on_user_input)(void* userdata, const uint8_t* data, size_t len);

    // 终端输出回调 (对应 SSH data handler)
    void (*on_terminal_output)(void* userdata, const uint8_t* data, size_t len);

    // 尺寸变化回调
    void (*on_size_change)(void* userdata, uint16_t cols, uint16_t rows,
                           uint16_t width_px, uint16_t height_px);

    // 标题变化回调
    void (*on_title_change)(void* userdata, const char* title);

    // 剪贴板回调
    void (*on_clipboard_copy)(void* userdata, const uint8_t* data, size_t len);

    // 链接打开请求
    void (*on_link_open)(void* userdata, const char* url);

    void* userdata;
} ghostty_ios_callbacks_s;

// 创建带回调的 Surface
ghostty_surface_t ghostty_surface_new_ios(
    ghostty_app_t app,
    const ghostty_surface_config_s* config,
    const ghostty_ios_callbacks_s* callbacks
);

// 向终端输入数据 (从 SSH 接收的数据)
void ghostty_surface_feed_input(
    ghostty_surface_t surface,
    const uint8_t* data,
    size_t len
);
```

#### 12.3.3 Swift 封装示例

```swift
import GhosttyKit

/// libghostty iOS 包装器，采用 SwiftTerm 风格的委托模式
class GhosttyTerminal {
    private var surface: ghostty_surface_t?
    private var app: ghostty_app_t?

    weak var delegate: GhosttyTerminalDelegate?

    // 用于 SSH/管道数据的队列
    private let ioQueue = DispatchQueue(label: "io.ghostty.ios")

    init(config: GhosttyConfig) throws {
        // 创建 App
        var appConfig = ghostty_config_new()
        // 配置...
        app = ghostty_app_new(&appConfig)

        // 设置回调
        var callbacks = ghostty_ios_callbacks_s()
        callbacks.userdata = Unmanaged.passUnretained(self).toOpaque()
        callbacks.on_terminal_output = { userdata, data, len in
            let this = Unmanaged<GhosttyTerminal>.fromOpaque(userdata!).takeUnretainedValue()
            let buffer = Data(bytes: data!, count: len)
            DispatchQueue.main.async {
                this.delegate?.terminal(this, didReceiveOutput: buffer)
            }
        }
        callbacks.on_size_change = { userdata, cols, rows, width, height in
            let this = Unmanaged<GhosttyTerminal>.fromOpaque(userdata!).takeUnretainedValue()
            this.delegate?.terminal(this, didChangeSize: (cols, rows))
        }
        // 其他回调...

        // 创建 Surface
        var surfaceConfig = ghostty_surface_config_new()
        // 配置...
        surface = ghostty_surface_new_ios(app, &surfaceConfig, &callbacks)
    }

    /// 向终端 feed 数据 (来自 SSH)
    /// 采用 SwiftTerm 的分块处理模式
    func feed(data: Data) {
        ioQueue.async { [weak self] in
            guard let self = self, let surface = self.surface else { return }

            // 分块处理大数据，避免阻塞
            let chunkSize = 4096
            var offset = 0
            while offset < data.count {
                let end = min(offset + chunkSize, data.count)
                data[offset..<end].withUnsafeBytes { bytes in
                    ghostty_surface_feed_input(surface, bytes.baseAddress, bytes.count)
                }
                offset = end
            }
        }
    }

    /// 发送用户输入
    func send(data: Data) {
        guard let surface = surface else { return }
        data.withUnsafeBytes { bytes in
            ghostty_surface_key_input(surface, bytes.baseAddress, bytes.count)
        }
    }
}

protocol GhosttyTerminalDelegate: AnyObject {
    func terminal(_ terminal: GhosttyTerminal, didReceiveOutput data: Data)
    func terminal(_ terminal: GhosttyTerminal, didChangeSize size: (cols: UInt16, rows: UInt16))
}
```

### 12.4 SwiftTerm 与 libghostty 对比

| 特性 | SwiftTerm | libghostty (改造后) |
|------|-----------|---------------------|
| **语言** | 纯 Swift | Zig + C API + Swift 包装 |
| **终端模拟** | 自实现 VT100/Xterm | 完整 Ghostty 引擎 |
| **渲染** | CoreText + NSAttributedString | Metal GPU 加速 |
| **字体** | 系统字体 | 自定义字体渲染 |
| **Unicode** | 基础支持 | 完整 Unicode 15.1 |
| **性能** | 良好 | 优秀 (Metal) |
| **功能完整性** | 基础终端 | 完整桌面级终端 |
| **iOS 原生支持** | ✅ 原生 | ⚠️ 需改造 |

### 12.5 从 SwiftTerm 学到的关键教训

1. **解耦 I/O 和渲染**
   - SwiftTerm 将终端引擎与 I/O 后端完全分离
   - libghostty 的管道后端需要遵循同样原则

2. **异步 I/O 必须使用专用队列**
   - SSH 操作在 `DispatchQueue` 上执行
   - 终端更新通过 `DispatchQueue.main.sync` 同步到主线程
   - libghostty 的 `xev.Loop` 需要与 GCD 协调

3. **分块处理大数据**
   - SwiftTerm 以 1KB 块处理 SSH 数据
   - 避免大量数据阻塞 UI
   - libghostty 管道后端应采用类似策略

4. **委托模式简化集成**
   - 清晰的回调接口比直接管道 FD 更易使用
   - libghostty C API 应提供回调选项

---

## 13. 与上游协作建议

根据 GitHub Discussion #4087 中 mitchellh 的表态：

> "管道后端会被接受，设计良好即可"

建议：

1. **早期沟通** - 在开始大规模开发前提交 RFC issue
2. **渐进式 PR** - 分多个小 PR 提交:
   - PR 1: `termio/backend.zig` 抽象改进
   - PR 2: `Pipe` 后端实现
   - PR 3: C API 扩展
3. **保持兼容** - 不破坏现有 POSIX/Windows 功能
4. **文档完善** - 添加 iOS 集成指南

---

## 14. 总结与工作量评估

### 14.1 核心改造任务

| 任务 | 复杂度 | 工作量 | 优先级 |
|------|--------|--------|--------|
| libxev iOS 修复 (PipeAsync) | 高 | 4-5 天 | P0 |
| termio.Pipe 后端实现 | 中 | 5-7 天 | P0 |
| C API 扩展 | 低 | 2-3 天 | P1 |
| Swift 封装层 | 中 | 3-4 天 | P1 |
| iOS 集成测试 | 中 | 3-4 天 | P1 |
| **总计** | | **3-4 周** | |

### 14.2 参考资源

- [SwiftTerm](https://github.com/migueldeicaza/SwiftTerm) - iOS 终端实现参考
- [SwiftTermApp](https://github.com/migueldeicaza/SwiftTermApp) - 完整 iOS SSH 应用
- [GitHub Discussion #4087](https://github.com/ghostty-org/ghostty/discussions/4087) - iOS 兼容性讨论
- [libxev](https://github.com/Cloudef/libxev) - 事件循环库
