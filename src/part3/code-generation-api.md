# 动态生成与模板配置：addConfigHeader 与 addWriteFiles

在移植成熟的 C/C++ 库或构建复杂系统软件时，我们经常需要处理平台相关的配置文件（如 CMake 生成的 `config.h`）或在构建期动态生成版本描述文件。Zig 为此内置了原生支持。

---

## 1. 配置头文件生成：`b.addConfigHeader`

在传统的 C/C++ 工程中，通常通过 CMake 的 `configure_file(config.h.in config.h)` 根据环境探测结果替换宏定义。Zig 提供了完全对等的原生抽象：`b.addConfigHeader`。

### 示例用法：

```zig
// 1. Declare ConfigHeader step
const config_h = b.addConfigHeader(
    .{
        .style = .{ .cmake = b.path("include/config.h.in") },
        .include_path = "config.h",
    },
    .{
        // Boolean values: generate #define HAVE_UNISTD_H 1 or /* #undef HAVE_UNISTD_H */
        .HAVE_UNISTD_H = target.result.os.tag != .windows,
        .HAVE_PTHREAD = true,

        // Numeric values
        .SIZEOF_SIZE_T = @as(i64, target.result.ptrBitWidth() / 8),

        // String values: generate #define DEFAULT_CHARSET "utf8mb4"
        .DEFAULT_CHARSET = "utf8mb4",
    },
);

// 2. Install into library or pass as include path
lib.installConfigHeader(config_h);
```

### 核心特性：
- **兼容 CMake 风格**：`.cmake` 风格自动解析输入模板中的 `#cmakedefine VAR`、`#cmakedefine01 VAR` 和 `@VAR@` 语法，并替换为对应的 C 宏定义；
- **类型安全**：基于 Zig 的匿名结构体字面量传入键值对，编译器在配置期直接校验类型，杜绝拼写错误。

---

## 2. 动态写入文件：`b.addWriteFiles`

当你需要在构建过程中动态生成某些源代码、临时聚合头文件或版本信息时，可以使用 `b.addWriteFiles`。

### 场景一：生成编译期版本号与构建元数据
```zig
// Dynamic version info generator
const write_files = b.addWriteFiles();

const version_zig = write_files.add("version.zig", b.fmt(
    \\pub const version = "{s}";
    \\pub const git_hash = "{s}";
    \\pub const build_mode = "{s}";
    ,
    .{ "1.2.3", "a1b2c3d", @tagName(optimize) },
));

// Expose as an internal module to the main application
const version_mod = b.createModule(.{
    .root_source_file = version_zig,
});
exe.root_module.addImport("version", version_mod);
```

### 场景二：为 `addTranslateC` 创建聚合头文件
有时一个第三方库有多个分散的头文件（如 `a.h`、`b.h`、`c.h`），而 `addTranslateC` 只能接收一个入口。我们可以动态生成一个聚合入口：

```zig
const bundle_h = b.addWriteFiles().add("bundle.h",
    \\#include <mariadb/ma_pvio.h>
    \\#include <mariadb/ma_tls.h>
    \\#include <mysql.h>
);

const translate_c = b.addTranslateC(.{
    .root_source_file = bundle_h,
    .target = target,
    .optimize = optimize,
});
```

---

## 3. 为什么优先使用这些 API 而不是在 `build()` 中写入文件？

使用 `b.addWriteFiles` 和 `b.addConfigHeader` 生成的文件，其路径均为 `LazyPath`。它们有两大决定性优势：
1. **自动纳入缓存调度**：只有当模板内容或注入的参数值改变时，才会真正触发文件重写；
2. **时序安全**：不会污染源码工作区，统一存放在 `.zig-cache/` 的哈希隔离目录下，保证构建环境的纯净与原子性。
