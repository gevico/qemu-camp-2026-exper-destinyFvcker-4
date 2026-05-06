# QOM 对象模型

这里现在不再用一篇超长笔记硬塞所有内容，而是改成：

- 一页总览
- 多页子主题

这样你追某条线时，不用在一个 5000 多行文件里来回找。

## 推荐阅读顺序

1. [QEMU QOM 对象模型总览](qom-object-model.md)
2. [QOM 四个核心对象与两条链](qom-core-objects.md)
3. [QOM 类型注册与模块初始化](qom-type-registration.md)
4. [QOM 对象布局与转型宏](qom-casts-and-layout.md)
5. [QOM 的 `TypeImpl`、类初始化与对象创建](qom-typeimpl-and-object-creation.md)
6. [QOM 的 interface、property 与对象树](qom-interfaces-properties-and-composition.md)

## 本目录节点

- [QEMU QOM 对象模型总览](qom-object-model.md)
  - 总入口，先看整体关系图和阅读路径
- [QOM 四个核心对象与两条链](qom-core-objects.md)
  - 讲 `TypeInfo`、`TypeImpl`、`ObjectClass`、`Object`，以及实例链 / 类链
- [QOM 类型注册与模块初始化](qom-type-registration.md)
  - 讲 `type_init(...)`、`module_call_init(...)`、`type_register_static(...)`、根类型自举
- [QOM 对象布局与转型宏](qom-casts-and-layout.md)
  - 讲“首字段嵌入”、`OBJECT_CHECK(...)`、`OBJECT_DECLARE_TYPE(...)`
- [QOM 的 `TypeImpl`、类初始化与对象创建](qom-typeimpl-and-object-creation.md)
  - 讲 `type_initialize()`、`class_init` / `instance_init`、对象创建主线
- [QOM 的 interface、property 与对象树](qom-interfaces-properties-and-composition.md)
  - 讲 interface 三层结构、property 两层结构、`Object.parent`、`opaque`

## 按问题跳转

- 如果你在追 `TypeInfo` / `type_init(...)` / `MODULE_INIT_QOM`
  - 看 [QOM 类型注册与模块初始化](qom-type-registration.md)
- 如果你在追 `OBJECT_CHECK(...)` / `DEVICE(obj)` / `OBJECT_DECLARE_TYPE(...)`
  - 看 [QOM 对象布局与转型宏](qom-casts-and-layout.md)
- 如果你在追 `TypeImpl` / `type_initialize()` / `class_init` / `instance_init`
  - 看 [QOM 的 `TypeImpl`、类初始化与对象创建](qom-typeimpl-and-object-creation.md)
- 如果你在追 interface / property / 对象树
  - 看 [QOM 的 interface、property 与对象树](qom-interfaces-properties-and-composition.md)

## 图谱关系

- 上游基础：[C 语言基础](../c/README.md)
- 下游应用：[RISC-V `virt`](../riscv-virt/README.md)
- 工具辅助：[工具链与编辑器](../tooling/README.md)
- 总入口：[learning/README.md](../README.md)
