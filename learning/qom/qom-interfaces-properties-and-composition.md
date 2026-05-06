# QOM 的 interface、property 与对象树

这页专门处理那些容易和主线缠在一起的旁支结构：

- interface
- property
- QOM composition tree
- `opaque`

## interface 先看三层

最容易混的，是这三个名字其实不在同一层：

| 名字 | 层次 | 它是什么 |
| --- | --- | --- |
| `TypeInfo.interfaces` | 注册输入层 | 源码里声明“我实现了哪些接口” |
| `TypeImpl.interfaces[]` | 运行时中间层 | QOM 内部保存下来的接口名字清单 |
| `ObjectClass.interfaces` | 运行时结果层 | 类对象上真正挂好的接口类链表 |

最短记法：

- `TypeInfo.interfaces`
  - 输入
- `TypeImpl.interfaces[]`
  - 待解析清单
- `ObjectClass.interfaces`
  - 最终运行时结果

## `TypeInfo.interfaces` 为什么看着像一个指针

字段声明是：

```c
const InterfaceInfo *interfaces;
```

但它的语义不是：

- “这里只有一个接口”

而是：

- “这里指向一个静态接口数组的首元素”

典型写法是：

```c
.interfaces = (const InterfaceInfo[]) {
    { INTERFACE_CONVENTIONAL_PCI_DEVICE },
    { },
},
```

也就是：

- 外部注册层用“数组首指针 + 结尾哨兵 `{ }`”表达一串接口声明

## `TypeImpl.interfaces[]` 又是什么

`type_new()` 会把上面的输入拷进内部结构：

```c
for (i = 0; info->interfaces && info->interfaces[i].type; i++) {
    ti->interfaces[i].typename = g_strdup(info->interfaces[i].type);
}
ti->num_interfaces = i;
```

所以：

- `TypeImpl.interfaces[]`
  - 是 QOM 内部自己的接口声明数组
- `num_interfaces`
  - 记录实际个数

它更像：

- “待处理的接口名字清单”

## `InterfaceImpl` 到底在干什么

它长这样：

```c
struct InterfaceImpl {
    const char *typename;
};
```

看起来字段很少，但职责其实很清楚：

- 记录“这个类型声明要实现哪个接口名字”

所以它不是：

- 具体接口方法表
- 接口对象实例
- 最终挂到类对象上的接口 class

## `ObjectClass.interfaces` 才是最终运行时结果

等 `type_initialize()` 真正处理接口时，会把：

- 父类继承来的接口结果
- 当前类型自己声明的新接口

一起汇总到：

- `ti->class->interfaces`

也就是：

- `ObjectClass.interfaces`

这时候每个链表节点里放的不是 `TypeImpl *`，也不是 `InterfaceImpl *`，而是：

- `InterfaceClass *`

更准确说：

- 某个“具体类型 + 某个接口”的接口类对象头

## interface 的函数指针到底放在哪里

不放在裸的 `InterfaceClass` 里。

`InterfaceClass` 只是公共头：

```c
struct InterfaceClass {
    ObjectClass parent_class;
    Type interface_type;
};
```

真正的函数指针放在：

- 具体接口自己的 class 结构体

例如：

```c
struct UserCreatableClass {
    InterfaceClass parent_class;

    void (*complete)(UserCreatable *uc, Error **errp);
    bool (*can_be_deleted)(UserCreatable *uc);
};
```

所以要分清：

- `InterfaceClass`
  - 接口 class 公共头
- `UserCreatableClass` / `MemoryDeviceClass`
  - 具体接口 class，函数指针在这里

## 一句话梳理 interface 三层

1. `InterfaceInfo`
   - 源码里写的接口说明
2. `InterfaceImpl`
   - QOM 内部保存的接口名字条目
3. `InterfaceClass`
   - 最后真正挂到 `ObjectClass.interfaces` 上的运行时实体

## property 有两层

| 位置 | 含义 |
| --- | --- |
| `class->properties` | 类级属性表 |
| `obj->properties` | 实例级属性表 |

可以粗略记成：

- 类属性
  - 更像“这个类型默认提供什么属性入口”
- 实例属性
  - 更像“这个具体对象当前有哪些属性值 / 子项”

## `class->properties = g_hash_table_new_full(...)` 是在干嘛

它的核心意思是：

- 给这个类对象新建一张“属性名 -> `ObjectProperty *`”的哈希表

最值得记住的是：

- key 按字符串内容比较
- value 的释放规则是 `object_property_free`

也就是说：

- 这不是随便开一张表
- 它同时也规定了类属性对象怎么清理

## `Object.parent` 什么时候会有值

刚 `object_new(...)` 出来时，通常可以先记：

- `class`
  - 已经准备好了
- `parent`
  - 往往还是 `NULL`

只有对象真正挂进 QOM composition tree 之后，例如：

- `object_property_add_child(...)`

`parent` 才会指向某个实际父对象。

所以：

- `parent`
  - 是“对象树关系”
- 不是“继承关系”

## `edu` 这类对象要怎么看 `class` 和 `parent`

如果手上是 `EduState`：

- `class`
  - 指向 `"edu"` 这个运行时类型对应的共享类对象
- `parent`
  - 指向 QOM 树里的父对象

它不是：

- `PCIDevice`
- `DeviceState`

最短记法：

- `class`
  - 说明“我是什么类型”
- `parent`
  - 说明“我被谁收在对象树里”

## `opaque` 是什么，为什么老和 QOM 一起出现

`opaque` 直译是：

- 不透明的

在 QEMU 里更准确的意思是：

- 框架层不解释它的具体类型
- 只把它当作上下文指针保存
- 真正懂它类型的是注册回调的人

以 `MemoryRegion` 为例：

```c
struct MemoryRegionOps {
    uint64_t (*read)(void *opaque, hwaddr addr, unsigned size);
    void (*write)(void *opaque, hwaddr addr, uint64_t data, unsigned size);
};
```

典型心智模型是：

```text
框架保存 opaque
  -> 发生 MMIO 访问
  -> 回调收到同一个 opaque
  -> 设备自己再把它转回 MyDeviceState *
```

所以它的核心不是“神秘指针”，而是：

- **框架不解释的上下文指针**

## 为什么这和 QOM 容易混在一起

因为 `opaque` 很多时候指向的恰好是：

- 某个 QOM 设备对象
- 或某个设备状态结构体

这时会出现两层关系同时存在：

1. 对象还挂在 QOM 对象树里
2. 同一个地址又被某个回调框架保存为 `void *opaque`

所以你要分清：

- `Object.parent`
  - 是对象树关系
- `opaque`
  - 是回调上下文关系

它们经常指向同一类对象，但语义不是一回事。

## 一句话收束

1. `TypeInfo.interfaces` 是输入，`TypeImpl.interfaces[]` 是中间清单，`ObjectClass.interfaces` 是最终结果。
2. interface 的函数指针放在具体接口 class 结构体里，不在裸 `InterfaceClass` 里。
3. property 分类级和实例级两层。
4. `Object.parent` 表示对象树关系，不表示继承。
5. `opaque` 是框架不解释的上下文指针，常常指向某个 QOM 设备对象或设备状态。
