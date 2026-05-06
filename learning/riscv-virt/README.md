# RISC-V `virt`

这里是当前学习主线的核心目录，主要围绕 QEMU `virt` 机器、RISC-V 启动链、设备树和中断控制器来组织。

## 本目录节点

- [QEMU `virt` 启动执行流程阅读指引](virt-init-reading-guide.md)
  - 从 `system/vl.c` 到 `hw/riscv/virt.c`，给出源码阅读主线
- [QEMU / RISC-V 术语速查](glossary.md)
  - 把启动链、虚拟化、兼容层、设备树、中断这些高频词先对齐
- [RISC-V `virt` 中断控制器速记](interrupt-controllers.md)
  - 重点区分 `PLIC`、`AIA`、`APLIC`、`IMSIC` 和不同中断投递路径

## 图谱关系

- 上游模型：[QOM 对象模型](../qom/README.md)
- 上游基础：[C 语言基础](../c/README.md)
- 工具辅助：[工具链与编辑器](../tooling/README.md)
- 延伸路线：[方向与路线](../directions/README.md)
- 总入口：[learning/README.md](../README.md)

## 推荐阅读顺序

1. 先看 [QEMU `virt` 启动执行流程阅读指引](virt-init-reading-guide.md)，建立主线。
2. 再看 [QEMU / RISC-V 术语速查](glossary.md)，把术语补齐。
3. 遇到中断相关源码时，再跳到 [RISC-V `virt` 中断控制器速记](interrupt-controllers.md)。
