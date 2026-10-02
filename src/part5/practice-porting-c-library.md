# 实战三：复杂第三方 C 库的完整移植实践（以 MariaDB Connector 为例）

移植一个“Hello World”级别的 C 文件非常简单，但在真实工业界，主流 C 库（如 MariaDB Connector/C、SQLite、OpenSSL、cURL）往往包含数十万行源码、复杂的 CMake 宏探测、数十个条件编译源文件以及跨平台的系统 API 差异。

本章以作者开源的 [zig-mariadb-connector](https://github.com/jiacai2050/zig-mariadb-connector) 为实际工程蓝本，系统梳理移植大型成熟 C 库的标准工业范式。

---

## 1. 移植面临的核心挑战

大型 C 库的构建脚本通常包含以下四个核心难题：

```mermaid
graph TD
    subgraph S_Challenge ["复杂 C 库移植四大拦路虎"]
        C_Cfg["1. 平台检测配置头<br/>(config.h.in / 宏探测)"]
        C_Src["2. 条件编译源文件裁剪<br/>(POSIX vs Windows 平台分支)"]
        C_Tls["3. 第三方系统库链接<br/>(OpenSSL / Schannel / 平台依赖)"]
        C_Exp["4. 头文件树与产物规范导出<br/>(供下游纯 Zig / C 顺畅消费)"]
    end

    classDef default fill:#f8f9fa,stroke:#495057;
    style S_Challenge fill:#fff0e6,stroke:#ff9900,stroke-width:2px;
    style C_Cfg fill:#cce5ff,stroke:#0066cc,stroke-width:2px;
    style C_Src fill:#cce5ff,stroke:#0066cc,stroke-width:2px;
    style C_Tls fill:#cce5ff,stroke:#0066cc,stroke-width:2px;
    style C_Exp fill:#e6ffe6,stroke:#009900,stroke-width:2px;
```

---

## 2. 移植核心步骤拆解

### 第一步：零侵入管理 Upstream 源码
推荐保持上游代码仓库的纯净，通过 `b.dependency("upstream", ...)` 或特定子目录引入，不修改上游任何源码文件。

### 第二步：使用 `addConfigHeader` 替代 CMake 探测
MariaDB Connector 依赖 `include/config.h.in`。在 Zig 构建中，我们通过 `b.addConfigHeader` 并结合目标平台信息直接计算宏值：

```zig
const target_is_windows = target.result.os.tag == .windows;

const config_h = b.addConfigHeader(
    .{
        .style = .{ .cmake = upstream.path("include/config.h.in") },
        .include_path = "config.h",
    },
    .{
        .HAVE_STDDEF_H = true,
        .HAVE_STDINT_H = true,
        .HAVE_SYS_TYPES_H = true,
        .HAVE_UNISTD_H = !target_is_windows,
        .HAVE_PTHREAD = !target_is_windows,
        .SIZEOF_SIZE_T = @as(i64, target.result.ptrBitWidth() / 8),
        .DEFAULT_CHARSET = "utf8mb4",
        .MARIADB_PORT = 3306,
        .MARIADB_UNIX_ADDR = "/tmp/mysql.sock",
    },
);
```

### 第三步：按目标操作系统分拣源文件
根据操作系统分支，精准组织需要参与编译的 `.c` 源文件列表：

```zig
const common_sources = &.{
    "libmariadb/ma_client_plugin.c",
    "libmariadb/mariadb_charset.c",
    "libmariadb/mariadb_lib.c",
    "libmariadb/ma_time.c",
    "libmariadb/ma_default.c",
    "libmariadb/ma_errmsg.c",
};

const posix_sources = &.{
    "libmariadb/secure/openssl.c",
};

const windows_sources = &.{
    "libmariadb/secure/schannel.c",
};

// Add to module based on OS
c_module.addCSourceFiles(.{
    .root = upstream.path(""),
    .files = common_sources,
    .flags = c_flags,
});

if (target_is_windows) {
    c_module.addCSourceFiles(.{
        .root = upstream.path(""),
        .files = windows_sources,
        .flags = c_flags,
    });
} else {
    c_module.addCSourceFiles(.{
        .root = upstream.path(""),
        .files = posix_sources,
        .flags = c_flags,
    });
}
```

### 第四步：静态库构建与公共头文件树导出
上游导出的静态库必须同时提供完整的头文件树：

```zig
const lib = b.addLibrary(.{
    .name = "mariadb",
    .linkage = .static,
    .root_module = c_module,
});

// 1. Install static headers from upstream include directory
lib.installHeadersDirectory(upstream.path("include"), "mariadb", .{});

// 2. Install dynamically rendered configuration header
lib.installConfigHeader(config_h);

// 3. Expose library artifact
b.installArtifact(lib);
```

### 第五步：编写集成测试套件验证交付
在工程下建立独立的 `test/` 子工程，通过 `b.dependency("zig-mariadb-connector", ...)` 引入刚才构建的库产物，执行真实的 MySQL/MariaDB 握手与句柄初始化测试，确保跨平台符号与 ABI 完全兼容。

---

## 3. 经验总结

移植复杂 C 库的核心在于**充分信任并利用 Zig 构建系统的声明式表达力**。通过将庞杂的 CMakeLists.txt 转化为结构清晰的 `build.zig`，不仅构建速度获得数倍提升，更能使该 C 库瞬间获得“零依赖跨平台交叉编译”的超级能力！
