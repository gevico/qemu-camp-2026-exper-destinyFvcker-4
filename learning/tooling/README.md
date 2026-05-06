# 工具链与编辑器

这里收的是“源码本身没错，但环境、索引、构建链把你绊住了”的问题，重点是把学习环境调顺。

## 本目录节点

- [QEMU 编辑器、clangd 与格式化器笔记](editor-clang-format.md)
  - 处理 `clangd` 误报、`.clangd` 规则、`.editorconfig` 和 `clang-format` 的边界
- [QEMU 里的 `configure` 和 `make`](qemu-configure-make.md)
  - 讲清配置阶段和构建阶段的区别，以及这个仓库的 `build/` 组织
- [QEMU Rust + Meson + rust-analyzer 排错笔记](rust-analyzer-meson-qemu.md)
  - 聚焦 Rust bindings、`cfg(MESON)`、Meson 图和编辑器分析环境错位

## 图谱关系

- 服务对象：[RISC-V `virt`](../riscv-virt/README.md)
- 服务对象：[QOM 对象模型](../qom/README.md)
- 上游基础：[C 语言基础](../c/README.md)
- 延伸路线：[方向与路线](../directions/README.md)
- 总入口：[learning/README.md](../README.md)

## 什么时候先来这里

- 你怀疑问题不是代码逻辑，而是 `clangd`、`rust-analyzer`、Meson 或构建目录选错了
- 你在问“为什么这里直接 `make` 不行”“为什么编辑器和真实构建结果不一样”
