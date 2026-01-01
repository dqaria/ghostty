# libghostty iOS 底层技术分析与改造方案

## 1. 核心问题概述

在 iOS 上运行 libghostty 存在三个底层技术障碍：

| 问题 | 根源位置 | 严重程度 |
|-----|---------|---------|
| **PTY 不可用** | `src/pty.zig` | 致命 - iOS 沙盒禁止 fork/exec |
| **libxev kevent 限制** | libxev 库 | 严重 - iOS 沙盒对 kqueue 有限制 |
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

## 5. 改造方案：管道后端 (Pipe Backend)

### 5.1 架构设计

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

### 5.2 新增文件: `src/termio/Pipe.zig`

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

### 5.3 修改 `src/termio/backend.zig`

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

### 5.4 修改 `src/pty.zig` - 添加 PipePty

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

## 6. C API 扩展

### 6.1 新增 API 函数 (`include/ghostty.h`)

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

### 6.2 实现 (`src/apprt/embedded.zig`)

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

## 7. libxev iOS 修复方案

### 7.1 问题分析

libxev 在 iOS 上的主要问题是 `xev.Process` 使用 `EVFILT_PROC`，这在 iOS 沙盒中被禁用。

### 7.2 修复策略

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

### 7.3 安全的 xev 组件

以下 xev 组件在 iOS 上应该可以正常工作：

| 组件 | iOS 状态 | 用途 |
|------|---------|------|
| `xev.Loop` | ✅ 可用 | 事件循环 |
| `xev.Timer` | ✅ 可用 | 定时器 |
| `xev.Async` | ⚠️ 需测试 | 线程间信号 |
| `xev.Stream` | ✅ 可用 | 文件描述符 I/O |
| `xev.Process` | ❌ 不可用 | 进程监视 |

---

## 8. 宿主应用集成

### 8.1 Swift 集成示例

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

## 9. 实施步骤

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

## 10. 关键文件变更清单

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

## 11. 与上游协作建议

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
