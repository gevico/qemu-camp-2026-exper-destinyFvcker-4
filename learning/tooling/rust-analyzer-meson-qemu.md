# QEMU Rust + Meson + rust-analyzer 排错笔记

## 问题现象

在基于 Meson 构建的 QEMU Rust 工程里，编辑器里出现类似报错：

- `unresolved import hwcore_sys`
- `#[cfg(MESON)]` 看起来没有生效

但实际使用 Meson 构建时，代码本身可能是能正常编译的。

## 这次问题的关键结论

### 1. Meson 构建时，`cfg(MESON)` 其实是打开的

Meson 调用 `rustc` 时会显式传入：

```text
--cfg MESON
```

因此像下面这种代码，在 **Meson 真正编译** 时会走上面的分支：

```rust
#[cfg(MESON)]
include!("bindings.inc.rs");

#[cfg(not(MESON))]
include!(concat!(env!("OUT_DIR"), "/bindings.inc.rs"));
```

也就是说：

- **Meson 编译**：走 `include!("bindings.inc.rs")`
- **Cargo / rust-analyzer 分析**：通常走 `OUT_DIR` 分支

### 2. `MESON_BUILD_ROOT` 不是 `cfg(MESON)`

这两个东西完全不是一回事：

- `MESON_BUILD_ROOT`：环境变量
- `MESON`：Rust 编译条件（`cfg`）

给 VS Code / rust-analyzer 配了：

```json
"rust-analyzer.cargo.extraEnv": {
  "MESON_BUILD_ROOT": "${workspaceFolder}/build-rust"
}
```

只代表 Cargo 侧的 `build.rs` 能找到 Meson 生成物，**不代表** rust-analyzer 自动拥有 `cfg(MESON)`。

## 为什么 rust-analyzer 会误报

rust-analyzer 默认主要依赖 Cargo 项目图来分析。

但这个仓库里真正的 Rust 构建关系，很多是 **Meson 决定的**，而不是 Cargo 单独决定的。  
如果 rust-analyzer 只看 Cargo，就可能出现：

- 条件编译分支判断和真实构建不一致
- crate 依赖关系和 Meson 不一致
- 编辑器报错，但 Meson 真编译没问题

## 这次的正确做法

让 rust-analyzer 直接读取 Meson 生成的项目描述：

```json
"rust-analyzer.linkedProjects": [
  "${workspaceFolder}/build-rust/rust-project.json"
]
```

同时保留：

```json
"rust-analyzer.cargo.extraEnv": {
  "MESON_BUILD_ROOT": "${workspaceFolder}/build-rust"
}
```

最终 `.vscode/settings.json` 类似这样：

```json
{
  "clangd.arguments": ["--compile-commands-dir=${workspaceFolder}/build-rust"],
  "rust-analyzer.linkedProjects": [
    "${workspaceFolder}/build-rust/rust-project.json"
  ],
  "rust-analyzer.cargo.extraEnv": {
    "MESON_BUILD_ROOT": "${workspaceFolder}/build-rust"
  }
}
```

## 为什么这个配置有效

`build-rust/rust-project.json` 是 Meson 生成的 Rust 项目图，里面包含：

- 每个 crate 的真实依赖关系
- `cfg = ["MESON"]`
- Meson 生成后的源码入口和结构化文件路径

所以 rust-analyzer 读取它之后，就能和 Meson 的实际构建行为对齐。

## 关于 `bindings.inc.rs` 的两条路径

这个仓库专门兼容了两种场景：

### A. Meson 直接编译

Meson 已经把生成好的 `bindings.inc.rs` 放在构建目录里，并通过 `--cfg MESON` 让代码直接：

```rust
include!("bindings.inc.rs");
```

### B. Cargo / rust-analyzer 分析

Cargo 不知道 Meson 的整个构建图，所以仓库用了 `build.rs`：

- 读取 `MESON_BUILD_ROOT`
- 在构建目录中找到 Meson 生成的 `bindings.inc.rs`
- 再把它链接到 Cargo 的 `OUT_DIR`

这样非 Meson 分支也能工作：

```rust
include!(concat!(env!("OUT_DIR"), "/bindings.inc.rs"));
```

## 为什么 `trace_object_dynamic_cast_assert(...)` 会跳到 `build-rust`

这个现象很容易让人误以为：

- “是不是 `qom/object.c` 里的 trace 调用已经和 Rust 实现绑在一起了？”

这次可以先记住结论：

- **`qom/object.c` 里的那次调用，本质上仍然走的是 C 侧生成的 trace helper**
- **IDE 跳到 `build-rust`，主要是因为当前分析/构建根就在 `build-rust`，而 trace 相关文件本来就是构建期生成物**
- **同名 `.rs` 文件确实存在，但那是给 Rust 代码复用同一份 trace 事件描述生成的包装，不等于这次 C 调用“链接进 Rust 实现”**

### 1. 事件定义写在源码树里

以 QOM 为例，事件定义源头在：

- `qom/trace-events`

其中有：

```text
object_dynamic_cast_assert(const char *type, const char *target, const char *file, int line, const char *func) "%s->%s (%s:%d:%s)"
```

也就是说：

- 这里先声明“有一个叫 `object_dynamic_cast_assert` 的 trace 事件”
- 参数列表和输出格式都写在这里

### 2. 构建时 `tracetool` 会生成 C 侧 helper

QEMU 文档里明确说：

- 每个子目录的 `trace-events` 都会在构建时被 `tracetool` 处理
- 自动生成 `<builddir>/trace/trace-<subdir>.h` 和 `<builddir>/trace/trace-<subdir>.c`

对 `qom/` 来说，对应就是：

- `<builddir>/trace/trace-qom.h`
- `<builddir>/trace/trace-qom.c`

在本仓库当前构建结果里，你可以直接看到：

- `build-rust/trace/trace-qom.h`
- `build-rust/trace/trace-qom.c`

其中 `build-rust/trace/trace-qom.h` 里真正定义了：

- `static inline void trace_object_dynamic_cast_assert(...)`

而 `build-rust/trace/trace-qom.c` 里定义了：

- 事件状态变量
- `TraceEvent` 结构
- `trace_qom_register_events()`

所以从 C 代码角度，这条调用链是：

```text
qom/object.c
  -> #include "trace.h"
  -> qom/trace.h
  -> #include "trace/trace-qom.h"
  -> 生成出来的 C inline helper: trace_object_dynamic_cast_assert(...)
```

### 3. 为什么会看到 `build-rust/trace/trace-qom.rs`

这是因为同一套 `tracetool` 现在还会额外生成 Rust 版本的 trace 包装：

- 生成脚本：`scripts/tracetool/format/rs.py`
- Rust 侧入口：`rust/trace/src/lib.rs`

`rust/trace/src/lib.rs` 里会：

```rust
include!(concat!("@MESON_BUILD_ROOT@/trace/trace-", $name, ".rs"));
```

这表示：

- Rust 代码也会去包含 Meson 构建目录里的 `trace-*.rs`
- 所以在 `build-rust/trace/` 下面，除了 C 的 `trace-qom.h/.c`，也会看到 Rust 的 `trace-qom.rs`

但要分清：

- **`trace-qom.h/.c` 是给 C 编译单元用的**
- **`trace-qom.rs` 是给 Rust crate 用的**
- 它们共享同一个事件源头：`qom/trace-events`

### 4. 所以这次是不是“和 Rust 链接在一起了”

更精确的回答是：

- **不是你这句 C 调用被 Rust 实现接管了**
- **而是这个仓库开启了 Rust 支持后，Meson 在 `build-rust` 这个构建根里同时生成了 C 和 Rust 两套 trace 包装**

如果只看 `qom/object.c` 这一句：

```c
trace_object_dynamic_cast_assert(...);
```

它在 C 侧实际对应的是：

- `qom/trace.h` -> `trace/trace-qom.h`

并不是“先跳去 Rust 再回来”。

### 5. 为什么 IDE 容易跳到 `build-rust`

常见原因有两个：

1. 你的 `clangd` / `compile_commands.json` 本来就指向：
   - `build-rust/compile_commands.json`
2. 生成函数名在 `build-rust/trace/` 下同时存在：
   - `trace-qom.h`
   - `trace-qom.rs`

所以 IDE 往往会优先带你去：

- **当前构建目录下的生成文件**

这通常说明的是：

- **“这是构建生成物”**

而不必然说明：

- **“这段 C 逻辑已经跨语言调用 Rust 了”**

## QEMU 里的 tracing 是不是一种日志系统

可以先回答：

- **是，它可以理解成一种“可开关、结构化、低侵入的日志 / 事件追踪系统”**
- 但它比普通 `printf` / `qemu_log` 更像是：
  - 先在源码里埋好 **tracepoint（追踪点）**
  - 再通过运行时配置决定要不要启用这些事件
  - 最后由不同 backend 决定输出到哪里、怎么输出

以 QOM 这条为例：

```c
trace_object_dynamic_cast_assert(type, target, file, line, func);
```

它不是普通手写日志函数，而是由：

```text
qom/trace-events
```

里的事件描述生成出来的。

可以把 QEMU tracing 和普通日志粗略区分成：

| 对比项 | 普通日志 | QEMU tracing |
| - | - | - |
| 写法 | 直接调用日志函数 | 先写 `trace-events`，再调用生成的 `trace_xxx(...)` |
| 开关 | 通常按日志级别 | 可以按事件名精确启用 / 禁用 |
| 结构 | 经常是自由文本 | 每个事件有固定参数列表和格式 |
| 生成 | 一般手写函数 | `tracetool` 根据事件描述生成 C/Rust 包装 |
| 用途 | 报错、调试、人读信息 | 观察内部行为、性能路径、设备事件、调试执行流 |

所以最短记法：

- **log 更像“我现在打印一句话”**
- **trace 更像“我在这里埋一个可控观测点”**

在当前生成的 `trace-qom.h` 里，具体 backend 走的是：

```c
qemu_log("object_dynamic_cast_assert ...");
```

这说明：

- 这次 trace 事件最后确实可以落到 QEMU 的日志输出里
- 但上层机制仍然叫 tracing，因为它先经过了：
  - `trace-events` 声明
  - `tracetool` 生成
  - 事件开关检查
  - backend 输出

所以可以更准确地说：

- **tracing 是 QEMU 的事件追踪框架**
- **日志输出只是 tracing backend 可能采用的一种落地方式**

## 一条经验总结

在 **Meson 主导 Rust 构建** 的项目里：

- 不要默认认为 rust-analyzer 看到的就是“真实构建状态”
- 也不要把环境变量和 Rust `cfg` 混为一谈
- 最稳妥的办法，是让 rust-analyzer 直接读取 Meson 生成的 `rust-project.json`

## 后续排错 checklist

以后再遇到类似问题，可以按这个顺序查：

1. 先确认真实构建工具是谁：`cargo` 还是 `meson`
2. 看实际 `rustc` 命令里有没有 `--cfg MESON`
3. 看 rust-analyzer 是否连接到了 `build-rust/rust-project.json`
4. 看 `.vscode/settings.json` 是否设置了 `MESON_BUILD_ROOT`
5. 修改配置后执行：
   - `Rust Analyzer: Reload Workspace`
   - 必要时 `Developer: Reload Window`

## 本次收获

这次问题的本质不是代码错了，而是：

- **Meson 的真实构建上下文**
- **rust-analyzer 的分析上下文**

两者没有对齐。

一旦把 rust-analyzer 接到 Meson 的 `rust-project.json`，问题就恢复正常了。
