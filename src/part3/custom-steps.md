# 编写自定义 Step：扩展构建管线

尽管 Zig 标准库提供了丰富的内置 Step，但在复杂工业级项目中，我们有时需要执行定制的任务（例如：调用代码压缩器、生成打包归档文件、校验特定文件指纹等）。

Zig 允许我们通过实现 `std.Build.Step.makeFn` 接口，无缝开发自己的 **Custom Step**。

---

## 1. Step 的实现契约

任何自定义 Step 核心只需满足以下两点：
1. 内存中内嵌一个 `std.Build.Step` 结构体；
2. 提供一个符合签名的执行函数：
   ```zig
   fn make(step: *std.Build.Step, options: std.Build.Step.MakeOptions) anyerror!void
   ```

---

## 2. 完整实战：编写一个归档打包 Step

假设我们需要在构建完成后，自动将 `zig-out/` 目录打包为一个发布用的 `release.tar.gz`。

```zig
const std = @import("std");

pub const PackReleaseStep = struct {
    step: std.Build.Step,
    output_path: []const u8,

    pub fn create(b: *std.Build, output_name: []const u8) *PackReleaseStep {
        const self = b.allocator.create(PackReleaseStep) catch @panic("OOM");
        self.* = .{
            .step = std.Build.Step.init(.{
                .id = .custom,
                .name = "pack-release",
                .owner = b,
                .makeFn = make,
            }),
            .output_path = output_name,
        };
        return self;
    }

    fn make(step: *std.Build.Step, options: std.Build.Step.MakeOptions) anyerror!void {
        _ = options;
        const self: *PackReleaseStep = @fieldParentPtr("step", step);
        const b = step.owner;

        // Perform custom task during the execution phase!
        std.debug.print("Packaging release archive to {s}...\n", .{self.output_path});

        // For example, spawn tar command
        var child = std.process.Child.init(&.{
            "tar", "-czf", self.output_path, "-C", "zig-out", ".",
        }, b.allocator);
        _ = try child.spawnAndWait();
    }
};
```

---

## 3. 在 `build.zig` 中挂载自定义 Step

定义好自定义 Step 后，在普通的 `build(b)` 函数中像使用官方 API 一样挂载它：

```zig
pub fn build(b: *std.Build) void {
    // 1. Define standard application
    const exe = b.addExecutable(.{ ... });
    const install_exe = b.addInstallArtifact(exe, .{});

    // 2. Instantiate custom pack step
    const pack_step = PackReleaseStep.create(b, "dist/app.tar.gz");

    // 3. Declare dependency: packaging must occur AFTER install is finished!
    pack_step.step.dependOn(&install_exe.step);

    // 4. Expose top-level command: "zig build pack"
    const top_pack = b.step("pack", "Package distribution archive");
    top_pack.dependOn(&pack_step.step);
}
```

### 关键收益：
- **同等公民（First-Class Citizen）**：自定义 Step 与官方的 `Step.Compile`、`Step.Run` 共享完全相同的多线程并发调度器与错误传播机制；
- **清晰的生命周期边界**：复杂的外部进程调用与磁盘读写全部收敛在 `make` 执行期，不影响配置期的高速轻量。
