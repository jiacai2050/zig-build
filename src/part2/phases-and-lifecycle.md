# 两阶段生命周期：配置期与执行期

理解 Zig 构建系统最关键的一道思维分水岭，是**严格区分“配置期”（Configuration Phase）与“执行期”（Execution Phase）**。

初学者编写 `build.zig` 时遇到的绝大多数疑惑（例如“为什么动态生成的文件在 `build()` 读不到”、“为什么我的文件写入逻辑总是报错”），本质上都是因为混淆了这两个截然不同的生命周期阶段。

---

## 1. 生命周期的宏观演进

当你敲下 `zig build` 之后，整个构建流程经历以下两个阶段：

```mermaid
graph TD
    subgraph Phase1 ["阶段一：配置期 (Configuration / Graph Evaluation)"]
        B_Code["执行 build.zig 中的 build(b) 入口"]
        B_Graph["在内存中实例化 Step 节点"]
        B_Option["解析命令行选项 (-Dtarget, -Doptimize 等)"]
        B_Edge["建立节点间依赖边 (dependOn / LazyPath)"]
        B_Code --> B_Option
        B_Option --> B_Graph
        B_Graph --> B_Edge
    end

    subgraph Phase2 ["阶段二：执行期 (Execution / Graph Execution)"]
        E_Topo["拓扑遍历截取目标子图 (如 install / run)"]
        E_Pool["工作线程池并发拉取就绪任务"]
        E_Cache{"Cache.Manifest<br/>内容哈希比对"}
        E_Skip["命中缓存：跳过执行 (Cache Hit)"]
        E_Worker["未命中：调用 Step.make() 生成文件/调用编译器"]
        E_Topo --> E_Pool
        E_Pool --> E_Cache
        E_Cache -- "是" --> E_Skip
        E_Cache -- "否" --> E_Worker
    end

    Phase1 -- "构建图定型，移交调度引擎" --> Phase2

    classDef default fill:#f8f9fa,stroke:#495057;
    style Phase1 fill:#fff0e6,stroke:#ff9900,stroke-width:2px;
    style Phase2 fill:#e6ffe6,stroke:#009900,stroke-width:2px;
    style B_Code fill:#f8f9fa,stroke:#495057,stroke-width:2px;
    style B_Option fill:#f8f9fa,stroke:#495057,stroke-width:2px;
    style B_Graph fill:#cce5ff,stroke:#0066cc,stroke-width:2px;
    style B_Edge fill:#cce5ff,stroke:#0066cc,stroke-width:2px;
    style E_Topo fill:#cce5ff,stroke:#0066cc,stroke-width:2px;
    style E_Pool fill:#e6ffe6,stroke:#009900,stroke-width:2px;
    style E_Cache fill:#fff3cd,stroke:#ffc107,stroke-width:2px;
    style E_Skip fill:#d1e7dd,stroke:#198754,stroke-width:2px;
    style E_Worker fill:#cce5ff,stroke:#0066cc,stroke-width:2px;
```

---

## 2. 阶段一：配置期（Graph Evaluation）

在配置期，由构建运行器调用你在 `build.zig` 中定义的唯一公开函数：

```zig
pub fn build(b: *std.Build) void {
    // Configuration logic executed here
}
```

### 核心特征与限制：
1. **纯内存图构建**：
   在 `build(b)` 函数体内，无论是调用 `b.addExecutable(...)`、`b.addLibrary(...)` 还是 `b.addConfigHeader(...)`，**都不会触发实际的编译操作，也不会向磁盘输出任何生成文件**。
   这些 API 的唯一行为，是在 `b.allocator` 堆内存中分配一个个 `std.Build.Step` 对象，记录配置参数，并将它们连成图结构。
2. **极速运行**：
   因为不涉及任何耗时的磁盘 I/O 和编译器调用，配置阶段通常在几毫秒至十几毫秒内即可极速完成。
3. **禁止假设生成物存在**：
   **绝对不要**在 `build()` 函数内部使用普通的同步文件系统 API 读取由前序步骤生成的文件：
   ```zig
   // ❌ 常见新手错误：在配置期试图读取执行期才生成的文件
   const config_h = b.addConfigHeader(...);
   // 错误！此时磁盘上根本还没有生成 config.h 文件，此处会抛出 FileNotFound 异常！
   const file = try std.fs.cwd().openFile("config.h", .{});
   ```

---

## 3. 阶段二：执行期（Graph Execution）

当 `build(b)` 函数返回后，内存中的计算图（DAG）结构已经完全定型。此时，控制权移交给构建引擎的调度器。

### 核心流程：
1. **确定目标子图（Target Subgraph）**：
   命令行参数指定了要执行的顶层任务（如 `zig build` 默认执行 `b.default_step`，即 `install`；`zig build test` 执行 `test` Step）。
   引擎从目标节点出发，进行反向深度优先遍历，只截取完成该目标必须依赖的前置 Step 子图，并完成**拓扑排序（Topological Sort）**。
2. **多线程并发调度**：
   调度器启动与 CPU 核心数相匹配的工作线程池。没有前置依赖、或前置依赖已全部就绪的任务节点，会被推入待执行任务队列。
3. **哈希缓存判定（Cache Lookup）**：
   每个 Step 在真正干活之前，会根据自身的输入参数、引用的文件内容生成 Manifest Hash。如果 `.zig-cache/` 中已有该哈希的记录且输出完整，则直接命中（Cache Hit），毫秒级跳过。
4. **触发 `Step.make()`**：
   若未命中缓存，线程池才会调用该 Step 绑定的 `makeFn` 函数，真正派生编译器进程、写入文件或执行测试。

---

## 4. 总结与开发心智模型

| 维度 | 配置期（Configuration Phase） | 执行期（Execution Phase） |
| :--- | :--- | :--- |
| **入口** | `pub fn build(b: *std.Build) void` | `step.makeFn(step, options)` |
| **主要工作** | 声明构建图、解析参数、连接数据依赖 | 检查缓存、执行真实编译、落盘产物 |
| **并发特征** | 单线程主流程 | 多线程任务池高度并发 |
| **I/O 操作** | 只允许读取工程中现存的只读静态配置 | 读写 `.zig-cache` 与 `zig-out`，生成动态代码 |
| **时间开销** | 几毫秒至几十毫秒 | 取决于编译代码量与缓存命中情况 |

将这一心智模型牢记于心后，我们便能自然地理解后续章节中介绍的 `Step`、`LazyPath` 和动态生成 API。
