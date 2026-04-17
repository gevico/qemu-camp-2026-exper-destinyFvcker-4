# RISC-V `virt` 中断控制器速记

这份笔记专门用来区分 `QEMU RISC-V virt` 机器里容易混在一起的几个中断相关组件：

- `PLIC`
- `AIA`
- `APLIC`
- `IMSIC`
- `CLINT`
- `ACLINT`

先记一个最重要的分类：

> `PLIC` / `AIA` / `APLIC` / `IMSIC` 主要处理外设中断；`CLINT` / `ACLINT` 主要处理 hart 本地的软件中断和定时器中断。

---

## 1. 先分清两条中断线

### 外设中断线

外设中断来自 `UART`、`virtio-mmio`、`PCIe` 设备、网卡、磁盘控制器等外部设备。

大概路径是：

```text
外设 -> 平台级中断控制器 -> CPU hart
```

在传统方案里，平台级中断控制器通常是：

```text
PLIC
```

在新方案里，平台级中断控制器属于：

```text
AIA = APLIC / IMSIC 等组件组成的一套高级中断架构
```

### 本地中断线

本地中断更靠近每个 hart 自己，常见来源是：

- 软件中断
- 定时器中断

大概路径是：

```text
本地软件/定时器事件 -> CLINT 或 ACLINT -> CPU hart
```

所以初学时可以先这样记：

| 类型 | 主要来源 | 常见组件 |
|---|---|---|
| 外设中断 | UART、virtio、PCIe 等设备 | `PLIC`、`AIA`、`APLIC`、`IMSIC` |
| 本地中断 | 软件中断、定时器中断 | `CLINT`、`ACLINT` |

---

## 2. PLIC：传统外部中断控制器

`PLIC` 全称是 `Platform-Level Interrupt Controller`，中文可以叫“平台级中断控制器”。

它负责接收多个外设的中断请求，然后根据优先级、使能状态、目标 hart 等信息，把中断投递给 CPU。

可以把它理解成一个外设中断分发中心：

```text
UART / virtio / PCIe / 其他外设
        |
        v
      PLIC
        |
        v
      hart
```

在 `QEMU virt` 代码里，如果：

```c
s->aia_type == VIRT_AIA_TYPE_NONE
```

就会走传统路线：

```c
s->irqchip[i] = virt_create_plic(...)
```

这也就是题目里“当 `s->aia_type` 为 `VIRT_AIA_TYPE_NONE` 时，每个 socket 创建哪种中断控制器”的答案是 `PLIC` 的原因。

需要注意：

- `PLIC` 管的是外设中断。
- 它不是定时器。
- 它也不是每个 hart 本地的软件中断控制器。
- `CLINT` / `ACLINT` 和它不是同一类东西。

---

## 3. AIA：新一代 RISC-V 中断架构

`AIA` 全称是 `Advanced Interrupt Architecture`，中文可以叫“高级中断架构”。

它不是单个设备名，而是一套新的中断架构方案。  
在 `QEMU virt` 里，`aia` 属性常见取值有：

```text
none
aplic
aplic-imsic
```

对应代码里的枚举大致是：

```c
VIRT_AIA_TYPE_NONE
VIRT_AIA_TYPE_APLIC
VIRT_AIA_TYPE_APLIC_IMSIC
```

可以这样理解：

| `aia` 取值 | 大概含义 |
|---|---|
| `none` | 不启用 AIA，使用传统 `PLIC` |
| `aplic` | 使用 `APLIC` 处理外设中断 |
| `aplic-imsic` | 使用 `APLIC + IMSIC`，走 MSI 风格投递 |

所以如果题目问的是：

```text
s->aia_type 为 VIRT_AIA_TYPE_NONE 时创建什么？
```

答案不是 `AIA`，因为这个值明确表示“不启用 AIA”。

---

## 4. APLIC：AIA 里的平台级控制器

`APLIC` 全称是 `Advanced Platform-Level Interrupt Controller`。

它可以先理解成：

> AIA 体系里的平台级外设中断控制器。

它接收来自外设的有线中断，然后有两种典型工作方式：

### 直接投递模式

当 `aia=aplic` 时，可以粗略理解成：

```text
外设 -> APLIC -> hart
```

这时 `APLIC` 更像是 `PLIC` 的新版本替代方案。

### MSI 投递模式

当 `aia=aplic-imsic` 时，可以粗略理解成：

```text
外设 -> APLIC -> IMSIC -> hart
```

这时 `APLIC` 接收外设中断后，会把中断转换成类似消息的形式，再交给 `IMSIC`。

---

## 5. IMSIC：每个 hart 附近的 MSI 收件箱

`IMSIC` 全称是 `Incoming Message-Signaled Interrupt Controller`。

它可以先理解成：

> 每个 hart 附近的“消息信号中断收件箱”。

`MSI` 是 `Message-Signaled Interrupt`，意思是“消息信号中断”。  
传统中断常给人一种“拉一根中断线”的感觉，而 `MSI` 更像是设备写一条消息到某个中断控制器地址，表示“我要触发某个中断”。

在 `QEMU virt` 里，`IMSIC` 只在下面这种 AIA 模式里出现：

```text
aia=aplic-imsic
```

此时每个 socket 会创建和 hart 相关的 `IMSIC` 区域。代码里还能看到 `M-level IMSIC` 和 `S-level IMSIC` 的区别：

- `M-level IMSIC`：给 `M-mode` 使用
- `S-level IMSIC`：给 `S-mode` 使用

初学时不用一上来钻太深，先记住：

> `APLIC` 接外设中断，`IMSIC` 接收 MSI 消息并靠近 hart。

---

## 6. CLINT：旧式核心本地中断器

`CLINT` 全称是 `Core-Local Interruptor`。

它主要负责和 hart 本地相关的中断，常见包括：

- `MSIP`：machine software interrupt，机器模式软件中断
- `MTIP`：machine timer interrupt，机器模式定时器中断

在设备树里常见兼容字符串是：

```text
sifive,clint0
riscv,clint0
```

所以 `CLINT` 和 `PLIC` 的区别非常重要：

| 组件 | 主要负责 |
|---|---|
| `CLINT` | 本地软件中断、定时器中断 |
| `PLIC` | 外部设备中断 |

在当前 `QEMU virt` 代码里，即使注释里写的是 `Per-socket SiFive CLINT`，实际创建时也会复用 `riscv_aclint_swi_create()` 和 `riscv_aclint_mtimer_create()` 这类 helper。  
这不是说它在概念上变成了 `PLIC`，而是 QEMU 内部实现复用了较新的 ACLINT 相关代码。

---

## 7. ACLINT：CLINT 的较新拆分形式

`ACLINT` 全称是 `Advanced Core Local Interruptor`。

它可以理解成把原来 `CLINT` 里本地软件中断、定时器等功能拆得更清楚：

| ACLINT 组件 | 作用 |
|---|---|
| `MSWI` | machine software interrupt，机器模式软件中断 |
| `SSWI` | supervisor software interrupt，监管者模式软件中断 |
| `MTIMER` | machine timer，机器模式定时器 |

在 `QEMU virt` 里有一个 machine 属性：

```text
aclint=on/off
```

源码描述里写着它是 `TCG only`，也就是只在 TCG 加速模式下可用。

在 `virt_machine_init()` 里，本地中断设备的选择逻辑大概是：

```text
如果允许 ACLINT 且 have_aclint 为 true：
  如果 aia_type == VIRT_AIA_TYPE_APLIC_IMSIC：
    创建 ACLINT MTIMER
  否则：
    创建 ACLINT MSWI + MTIMER + SSWI
否则如果启用 TCG：
  创建 SiFive CLINT 风格的软件中断 + 定时器
```

所以 `ACLINT` / `CLINT` 解决的是“本地中断和定时器”问题，不是“外设中断总控”问题。

---

## 8. `virt_machine_init()` 中的决策树

在 `hw/riscv/virt.c` 里，每个 socket 初始化时大概可以拆成两段看。

第一段创建 hart 和本地中断/定时器：

```text
for each socket:
  创建 RISCV_HART_ARRAY
  创建 CLINT 或 ACLINT 相关本地中断设备
```

第二段创建外部中断控制器：

```text
for each socket:
  if aia_type == VIRT_AIA_TYPE_NONE:
      创建 PLIC
  else:
      创建 AIA 相关设备
```

也就是：

```text
本地中断：CLINT / ACLINT
外设中断：PLIC / AIA
```

这两段代码在同一个 socket 初始化循环里，但它们负责的问题不同。

---

## 9. `QEMU virt` 里常见 MMIO 地址

下面这些地址来自 `hw/riscv/virt.c` 里的 `virt_memmap[]`。

| 组件 | 基地址 | 说明 |
|---|---:|---|
| `VIRT_CLINT` | `0x02000000` | `CLINT` 或 `ACLINT MTIMER/MSWI` 相关区域 |
| `VIRT_ACLINT_SSWI` | `0x02f00000` | `ACLINT SSWI` 区域 |
| `VIRT_PLIC` | `0x0c000000` | 传统 `PLIC` 区域 |
| `VIRT_APLIC_M` | `0x0c000000` | `M-mode APLIC` 区域 |
| `VIRT_APLIC_S` | `0x0d000000` | `S-mode APLIC` 区域 |
| `VIRT_IMSIC_M` | `0x24000000` | `M-level IMSIC` 区域 |
| `VIRT_IMSIC_S` | `0x28000000` | `S-level IMSIC` 区域 |

注意：

- 多 socket 时，代码会在这些基地址上再加 socket 偏移。
- `PLIC` 和 `APLIC_M` 的基地址都从 `0x0c000000` 开始，但它们不会在同一种 `aia_type` 路径里同时作为同一个外部 irqchip 使用。

---

## 10. 和那道选择题对应起来

题目：

> 当 `s->aia_type` 为 `VIRT_AIA_TYPE_NONE` 时，`virt_machine_init()` 会为每个 socket 创建哪种类型的中断控制器？

答案是：

```text
PLIC
```

原因是 `virt_machine_init()` 的外部中断控制器分支就是：

```text
if aia_type == NONE:
    virt_create_plic()
else:
    virt_create_aia()
```

其他选项为什么不对：

| 选项 | 为什么不对 |
|---|---|
| `AIA` | `VIRT_AIA_TYPE_NONE` 明确表示不启用 AIA |
| `ACLINT` | 它是本地中断/定时器相关，不是这个分支里的外部 irqchip |
| `SiFive CLINT` | 它也是本地中断/定时器相关，不是外设中断控制器 |

---

## 11. 阅读源码入口

建议按下面几个函数读：

- `hw/riscv/virt.c`：`virt_machine_init()`
  - 看每个 socket 如何创建 hart、本地中断设备、外部 irqchip。
- `hw/riscv/virt.c`：`virt_create_plic()`
  - 看传统 `PLIC` 如何创建。
- `hw/riscv/virt.c`：`virt_create_aia()`
  - 看 `APLIC` 和 `IMSIC` 如何创建。
- `hw/riscv/virt.c`：`create_fdt_socket_clint()`
  - 看 `CLINT` 风格设备树节点怎么写。
- `hw/riscv/virt.c`：`create_fdt_socket_aclint()`
  - 看 `ACLINT MSWI/SSWI/MTIMER` 设备树节点怎么写。
- `include/hw/riscv/virt.h`
  - 看 `VIRT_AIA_TYPE_NONE`、`VIRT_AIA_TYPE_APLIC`、`VIRT_AIA_TYPE_APLIC_IMSIC` 和相关地址宏。

---

## 12. 最短记忆口诀

可以先背这几句：

```text
PLIC 管传统外设中断。
AIA 是新中断架构，不是单个设备。
APLIC 接外设中断。
IMSIC 接 MSI 消息，靠近 hart。
CLINT 管老式本地软件中断和定时器。
ACLINT 是 CLINT 的新式拆分。
```

再压缩一点：

```text
PLIC/AIA 管外设，CLINT/ACLINT 管本地。
aia=none 走 PLIC，aia=aplic 或 aplic-imsic 走 AIA。
```
