# 标准选项与顶层入口：b.standardTargetOptions 与 b.step

在编写 `build.zig` 时，第一步通常是处理来自命令行的构建参数，并向用户暴露可以执行的构建目标。

---

## 1. 目标平台与优化级别

Zig 提供了开箱即用的标准选项解析器：

```zig
pub fn build(b: *std.Build) void {
    // 1. Standard target options: allows -Dtarget=...
    const target = b.standardTargetOptions(.{});

    // 2. Standard optimize option: allows -Doptimize=Debug/ReleaseFast/ReleaseSafe/ReleaseSmall
    const optimize = b.standardOptimizeOption(.{});
}
```

### 1.1 `b.standardTargetOptions`
- **默认行为**：若命令行未指定 `-Dtarget`，默认使用当前主机的 Native 架构与操作系统；
- **交叉编译支持**：当用户传入 `-Dtarget=x86_64-windows` 或 `-Dtarget=aarch64-linux-musl` 时，Zig 会自动解析并设置目标架构、OS 以及 ABI。

### 1.2 `b.standardOptimizeOption`
- **默认级别**：默认为 `Debug` 模式（包含全面的运行时安全检查，关闭重度优化）；
- **四种标准模式**：
  - `Debug`：快速编译，全量安全断言，无优化；
  - `ReleaseSafe`：开启中重度优化，同时保留数组越界、整数溢出等运行时安全校验；
  - `ReleaseFast`：极限速度优化（相当于 `-O3`），关闭运行时安全校验；
  - `ReleaseSmall`：极限体积优化（相当于 `-Os` / `-Oz`），优先减少产物体积。

---

## 2. 自定义命令行参数：`b.option`

除了内置的标准选项，工程往往需要自定义构建开关（例如“是否启用 SSL”、“是否开启某项特性”）：

```zig
pub fn build(b: *std.Build) void {
    // Parse boolean flag: zig build -Denable-tls=true
    const enable_tls = b.option(bool, "enable-tls", "Enable TLS support") orelse false;

    // Parse string option: zig build -Dapi-endpoint=https://api.example.com
    const endpoint = b.option([]const u8, "api-endpoint", "Custom backend API endpoint");

    // Parse enum option
    const Backend = enum { sqlite, postgres, mysql };
    const backend = b.option(Backend, "backend", "Database backend driver") orelse .sqlite;
}
```

### 核心特性：
- **强类型解析**：`b.option` 接受 Zig 的任意原始类型（`bool`、`usize`、`[]const u8`）或 `enum`。若传入不受支持的值，Zig 命令行会在终端直接输出清晰的类型不匹配错误；
- **自解释与帮助文档**：所有通过 `b.option` 定义的参数，都会自动汇总在 `zig build --help` 的输出列表中，具备极佳的自说明性。

---

## 3. 注册顶层命令入口：`b.step` 与 `b.default_step`

用户在终端执行 `zig build <step_name>` 时，调度引擎根据注册的顶层 Step 决定触发哪条依赖链路。

```zig
pub fn build(b: *std.Build) void {
    // 1. Create a top-level step named "run"
    const run_step = b.step("run", "Run the application");

    // 2. Create executable and run step
    const exe = b.addExecutable(.{ ... });
    const run_cmd = b.addRunArtifact(exe);

    // 3. Bind dependency: running "zig build run" triggers execution
    run_step.dependOn(&run_cmd.step);

    // 4. Default step: executed when running just "zig build"
    // By default, b.default_step is already wired to install all registered artifacts.
}
```

### 顶层 Step 机制：
1. **`b.step(name, description)`**：在构建图中挂载一个命名的 `Step.Id.top_level` 节点。用户只需执行 `zig build <name>` 即可调用；
2. **`b.default_step`**：当用户仅仅敲击 `zig build` 时执行的默认目标。默认情况下，它对应 `install` 步骤；
3. **参数转发支持**：结合 `run_cmd.addArgs(b.args orelse &.{})`，可以将 `zig build run -- --port 8080` 中 `--` 后面的参数完整透传给被调用的应用程序。
