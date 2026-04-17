# 学习笔记索引

这个目录用于沉淀阅读 QEMU、RISC-V `virt`、QOM、工具链配置时反复会用到的结论。

## 推荐阅读顺序

1. [RISC-V `virt` 启动执行流程阅读指引](riscv-virt/virt-init-reading-guide.md)
2. [QEMU / RISC-V 术语速查](riscv-virt/glossary.md)
3. [RISC-V `virt` 中断控制器速记](riscv-virt/interrupt-controllers.md)
4. [QEMU 源码阅读里的 C 语言与头文件基础](c/c-language-and-headers.md)
5. [QEMU QOM 对象模型基础](qom/qom-object-model.md)
6. [QEMU 编辑器、clangd 与格式化器笔记](tooling/editor-clang-format.md)
7. [CXL Type-2 加速器仿真方向笔记](cxl/cxl-type2-gpu-sim-direction.md)
8. [作业方向比较：`CXL Type-2 GPU` vs `QEMU + Wine-CE`](project-direction-comparison.md)
9. [围绕 QEMU 能做什么：主线方向地图（2026-04）](qemu-project-map.md)
10. [QEMU 社区里现在可以怎么贡献（截至 2026-04-19）](qemu-community-contribution.md)

## 分类目录

- [C 语言基础](c/README.md)
  - C 头文件习惯、`typedef`、`struct Foo`、宏、线程基础类型
- [QOM 对象模型](qom/README.md)
  - `Object` 布局、运行时类型检查、类型注册、属性系统
- [RISC-V `virt`](riscv-virt/README.md)
  - 启动流程、术语、中断控制器、设备树和源码阅读入口
- [工具链与编辑器](tooling/README.md)
  - `clangd`、`clang-format`、`.editorconfig`、Rust + Meson 分析环境
- [CXL / 加速器仿真](cxl/README.md)
  - `CXL Type-2`、一致性、设备模型、GPU 仿真研究方向
- [作业方向比较](project-direction-comparison.md)
  - 用学习收益、工程边界、风险和论文味比较两个选题
- [QEMU 后续项目方向](qemu-project-map.md)
  - 从板级平台、设备模型、外置后端、QMP、TCG plugin 到云虚拟化特性的一页路线图
- [QEMU 社区贡献入口](qemu-community-contribution.md)
  - 从 `Submitting a Patch`、`Trivial Patches`、`Bite Sized` 到 `RISC-V` issue 的入门路线
