# QEMU 编辑器、clangd 与格式化器笔记
这份笔记记录 QEMU 这种大型 C 项目里，编辑器诊断、clangd、`.editorconfig`、`clang-format` 之间的边界和常见误区。
## 相关笔记
- [C 语言与头文件基础](../c/c-language-and-headers.md)
- [QOM 对象模型基础](../qom/qom-object-model.md)

---

## 为什么 `clangd` 有时会误报

像这类提示：

- `Included header osdep.h is not used directly`
- `Must use 'struct' tag to refer to type 'Object'`

很多时候不是代码真错了，而是：

- `clangd` 把某个头文件当成“独立文件”去分析
- 但这个头文件默认依赖了 QEMU 的常规编译上下文
- 这个上下文里通常已经先有了 `qemu/osdep.h`

换句话说：

- QEMU 某些头文件并不是完全脱离环境也能单独分析得很漂亮
- 单独分析时就容易出现“理解歪了”的噪声诊断

这也是为什么项目里额外加了：

- `.clangd:1`

用来：

- 关掉噪声较大的 `UnusedIncludes`
- 给头文件分析时补一个 `-include qemu/osdep.h`

这只是为了编辑器体验更接近 QEMU 的真实构建方式。

---

## `.editorconfig` 和 `clang-format` 在这个仓库里各管什么

阅读 QEMU 这种 C 项目时，很容易把编辑器规则和代码格式化器混在一起。

在这个仓库里更准确的理解是：

- `.editorconfig`
  - 主要给编辑器用
  - 约束 LF、末尾换行、基础缩进、某些文件类型的 tab/space 习惯
- `clang-format`
  - 是独立的 C/C++ 代码格式化器
  - 它不会原生读取 `.editorconfig`

仓库里的：

- `.editorconfig:15`

更偏向“基础文本约束”和“编辑器行为”。

例如：

- `Makefile*` 用 tab + 8
- `*.{c,h,c.inc,h.inc}` 用 4 空格
- `*.{s,S}` 用 tab + 8

但这不等于：

- 跑 `clang-format` 时就会自动尊重这些规则

### 为什么会出现“宏被 formatter 洗了一遍”

如果仓库没有 `.clang-format`，那么 `clang-format -style=file` 找不到配置时，会退回自己的默认/回退风格。

这就会导致：

- 宏续行重新对齐
- 指针风格变化
- 花括号位置变化
- 长行重新折行

所以：

- `.editorconfig` 防的是“编辑器把文件写脏”
- `clang-format` 做的是“主动重排代码”

这是两套不同层次的规则。

### 这个仓库现在怎么处理

为了让 `clang-format` 尽量别把 QEMU 风格洗得太厉害，仓库根目录现在补了：

- `.clang-format:1`

这份配置至少做了几件事：

- 把缩进设成 4 空格
- 关闭自动重排注释
- 关闭 include 排序
- 尽量减少宏定义体被重写

它不可能完全等同于 QEMU 手写风格，但比直接吃默认回退风格要稳得多。
