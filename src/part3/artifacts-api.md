# 产物构建：Executable、Library 与 Test

在构建管线中，所有的终态二进制产物（可执行文件、库文件和测试运行器）均由 `*std.Build.Step.Compile` 承载。

---

## 1. 构建可执行程序：`b.addExecutable`

```zig
const exe = b.addExecutable(.{
    .name = "my_app",
    .root_module = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    }),
});

// Install artifact into zig-out/bin/
b.installArtifact(exe);
```

### 关键配置项与属性：
- **`root_module`**：挂载主程序的编译上下文；
- **产物安装**：通过 `b.installArtifact(exe)` 将生成物安装到 `zig-out/bin/my_app`（在 Windows 上自动追加 `.exe` 后缀）。

---

## 2. 构建库文件：`b.addLibrary`

在 Zig 中，静态库（`.a` / `.lib`）与动态库（`.so` / `.dylib` / `.dll`）统一通过 `b.addLibrary` 进行声明，由 `.linkage` 枚举区分：

```zig
// 1. Static Library
const static_lib = b.addLibrary(.{
    .name = "my_lib",
    .linkage = .static,
    .root_module = my_module,
});
b.installArtifact(static_lib);

// 2. Dynamic / Shared Library
const shared_lib = b.addLibrary(.{
    .name = "my_lib",
    .linkage = .dynamic,
    .version = .{ .major = 1, .minor = 2, .patch = 0 },
    .root_module = my_module,
});
b.installArtifact(shared_lib);
```

### 动态库版本控制：
当构建动态共享库时，通过设置 `.version`（`std.SemanticVersion`），Zig 会自动为目标操作系统生成合规的符号链接链条（例如在 Linux 上输出 `libmy_lib.so.1.2.0` 并创建 `libmy_lib.so.1` 与 `libmy_lib.so` 的软链接）。

---

## 3. 运行与单元测试：`b.addTest` 与 `b.addRunArtifact`

Zig 的单元测试与语言核心深度集成。在构建脚本中，测试任务被建模为两步组合：
1. 编译单元测试可执行文件（`addTest`）；
2. 调度并执行该测试二进制进程（`addRunArtifact`）。

```zig
// 1. Declare test compile step
const unit_tests = b.addTest(.{
    .root_module = b.createModule(.{
        .root_source_file = b.path("src/root.zig"),
        .target = target,
        .optimize = optimize,
    }),
});

// 2. Declare test execution step
const run_unit_tests = b.addRunArtifact(unit_tests);

// 3. Bind to "zig build test" command
const test_step = b.step("test", "Run all unit tests");
test_step.dependOn(&run_unit_tests.step);
```

### 为什么将“测试编译”与“测试运行”拆开？
- **支持交叉编译下的测试隔离**：如果通过 `-Dtarget=aarch64-linux-gnu` 在 x86_64 宿主机上编译，测试程序可以在不需要 QEMU 的情况下完成编译阶段验证；
- **环境隔离与输出断言**：`run_unit_tests` 允许构建脚本定制工作目录、环境变量，甚至断言进程的标准输出或退出码：
  ```zig
  run_unit_tests.expectExitCode(0);
  run_unit_tests.setEnvironmentVariable("LOG_LEVEL", "DEBUG");
  ```
