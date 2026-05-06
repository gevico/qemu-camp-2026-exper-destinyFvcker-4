# C 语言基础

这里收的是“读 QEMU C 源码时总会被绊一下”的底层问题，不直接讲某个设备，而是先把语言和基础库层补齐。

## 本目录节点

- [QEMU 源码阅读里的 C 语言与头文件基础](c-language-and-headers.md)
  - 包括头文件习惯、`struct`/`typedef`、`qemu/osdep.h`、`GLib`、常见源码阅读障碍

## 图谱关系

- 上游入口：[learning/README.md](../README.md)
- 平行主题：[QOM 对象模型](../qom/README.md)
- 平行主题：[工具链与编辑器](../tooling/README.md)
- 常见下游：[RISC-V `virt`](../riscv-virt/README.md)

## 什么时候先来这里

- 你看到某个 C 语法、宏或头文件组织方式就开始发懵
- 你分不清 `GLib`、`GObject`、QEMU 自己的代码边界
- 你不是没看懂设备逻辑，而是先被 C 代码表面形式卡住了
