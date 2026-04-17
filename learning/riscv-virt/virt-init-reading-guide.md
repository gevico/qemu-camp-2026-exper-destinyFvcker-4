# QEMU `virt` 启动执行流程阅读指引

## 目的

这份笔记不是把所有源码细节一次讲完，而是先回答一个更实际的问题：

> 如果我想搞懂 `-M virt` 的启动执行流程，应该先看哪些文件？

可以先沿着下面这条主线读：

1. QEMU 总入口什么时候创建 machine
2. `virt` 机器类型是怎么注册的
3. `virt` 板子本身是怎么初始化的
4. firmware / dtb / kernel 是什么时候装进内存的
5. vCPU 为什么从 `0x1000` 开始执行

参考网页：

- `https://qemu.gevico.online/tutorial/2026/ch1/qemu-init/`

---

## 一句话主线

QEMU 启动后会先在 `system/vl.c` 里创建 machine，对 `-M virt` 来说会选中 `hw/riscv/virt.c` 里的 `virt` 机器类型；随后真正进入 `virt_machine_init()` 搭建整台板子；等 machine 初始化完成后，再在 `virt_machine_done()` 中加载 firmware / kernel / FDT，并调用 `riscv_setup_rom_reset_vec()` 在 MROM 里写入复位跳板；最后 CPU 复位时在 `target/riscv/cpu.c` 中把 `pc` 设为 `resetvec`，默认就是 `0x1000`，于是第一条指令从这里开始执行。

---

## 整个 `qemu-system-*` 二进制的启动入口在哪

如果你问的是：

- “操作系统把 `qemu-system-riscv64` 这个可执行文件跑起来以后，最先进入 QEMU 自己哪段代码？”

那在这份源码里，**system emulator 二进制**的 C 入口是：

- `system/main.c:69`
  - `int main(int argc, char **argv)`

Meson 里也能看到 `qemu-system-*` 可执行文件就是把它编进去的：

- `meson.build:4399`
- `meson.build:4406`

最短主线可以先记成：

```text
system/main.c:main
  -> qemu_init(argc, argv)
  -> qemu_main_loop()
```

再展开一点就是：

```text
main()
  -> qemu_init()                     system/vl.c
     -> qemu_init_subsystems()      system/runstate.c
     -> 解析命令行
     -> qemu_create_machine()       system/vl.c
     -> machine_run_board_init()    hw/core/machine.c
        -> machine_class->init()
           -> virt_machine_init()   hw/riscv/virt.c
  -> qemu_main_loop()               qemu/main-loop.c
```

所以如果你的目标是：

- **“整个 QEMU 二进制从哪开始跑”**

先看：

- `system/main.c`

如果你的目标是：

- **“`-M virt` 这台 RISC-V 板子从哪开始搭起来”**

就继续往下看：

- `system/vl.c`
- `hw/core/machine.c`
- `hw/riscv/virt.c`

---

## 最值得先看的文件

### 1. `system/vl.c`

这是 QEMU system 模式的大总管，先用它定位“什么时候开始创建并初始化板子”。

- `system/vl.c:3766`
  - 调用 `qemu_create_machine(machine_opts_dict)`
- `system/vl.c:2190`
  - `qemu_create_machine()`：选择 machine type，并创建 machine 对象
- `system/vl.c:2715`
  - `machine_run_board_init()`：真正进入板级初始化

你读这个文件时，只需要先抓住两件事：

- `-M virt` 最终会选到一个具体的 machine class
- 后面会调用这个 class 的 `init` 回调

---

### 2. `hw/riscv/virt.c`

这是 `virt` 板子的核心文件，最重要。

- `hw/riscv/virt.c:1923`
  - `virt_machine_class_init()`
  - 在这里把 `mc->init = virt_machine_init`
- `hw/riscv/virt.c:1757`
  - `virt_machine_instance_init()`
  - 创建 `virt` 实例时做基础字段初始化
- `hw/riscv/virt.c:1529`
  - `virt_machine_init()`
  - 真正搭板子：CPU、RAM、中断控制器、UART、virtio、PCIe、ROM、FDT 等都在这里陆续建立
- `hw/riscv/virt.c:1431`
  - `virt_machine_done()`
  - 在 machine 初始化完成后，加载 firmware / kernel / dtb，并准备 reset vector
- `hw/riscv/virt.c:103`
  - `virt` 的 DRAM 基地址是 `0x80000000`

这个文件里可以把函数分成三层理解：

- `virt_machine_class_init()`
  - 定义“这个 machine 类型有什么能力、默认参数、初始化回调”
- `virt_machine_instance_init()`
  - 定义“这个实例刚创建出来时，先把哪些字段设好”
- `virt_machine_init()`
  - 定义“整台 `virt` 机器到底怎么搭起来”

如果你暂时只想抓主线，`virt_machine_init()` 不需要一口气看完。先知道它在做这些事就够了：

- 创建 hart array / CPU
- 建内存映射
- 注册 DRAM 和 MROM
- 建 CLINT / ACLINT / PLIC / AIA
- 建 UART、RTC、flash、virtio-mmio、PCIe、fw_cfg
- 生成或整理 FDT

中断控制器这块如果容易混，可以先看：

- `interrupt-controllers.md`

---

### 3. `hw/riscv/boot.c`

这个文件负责把“能启动的内容”放进内存。

- `hw/riscv/boot.c:136`
  - `riscv_find_and_load_firmware()`
  - 负责决定并加载 firmware
- `hw/riscv/boot.c:157`
  - `riscv_load_firmware()`
  - 实际把 firmware ELF 或 binary 载入内存
- `hw/riscv/boot.c:432`
  - `riscv_setup_rom_reset_vec()`
  - 在 MROM 中写入 reset vector，也就是复位后最先执行的那小段跳板代码
- `hw/riscv/boot.c:500`
  - `riscv_setup_firmware_boot()`
  - 某些启动路径下，把 kernel/initrd/cmdline 通过 `fw_cfg` 交给 firmware

这个文件最关键的理解点是：

- firmware 本体通常不在 `0x1000`
- `0x1000` 只是 reset vector 所在位置
- reset vector 会再跳到真正的 firmware 入口

---

### 4. `target/riscv/cpu.c` 和 `target/riscv/cpu_bits.h`

这两个文件解释“为什么 GDB 一连上先看到 PC 在 `0x1000`”。

- `target/riscv/cpu_bits.h:752`
  - `DEFAULT_RSTVEC = 0x1000`
- `target/riscv/cpu.c:2668`
  - `resetvec` 是 RISC-V CPU 的一个属性，默认值就是 `DEFAULT_RSTVEC`
- `target/riscv/cpu.c:680`
  - `riscv_cpu_reset_hold()` 在 CPU 复位时执行 `env->pc = env->resetvec`
- `target/riscv/cpu.c:88`
  - `cpu_set_exception_base()` 也能在运行时改 `resetvec`

所以逻辑非常直接：

1. CPU 有个 `resetvec`
2. 默认 `resetvec = 0x1000`
3. CPU reset 时 `pc = resetvec`
4. 所以第一条指令从 `0x1000` 开始

---

### 5. `docs/system/riscv/virt.rst`

源码之外，官方文档也值得先看一遍。

- `docs/system/riscv/virt.rst:1`
  - 解释 `virt` 是一个通用虚拟平台，不对应真实硬件
- `docs/system/riscv/virt.rst:47`
  - 介绍 `virt` 的 boot 方式

这个文件适合用来先建立整体直觉，再回头看代码。

---

## 推荐阅读顺序

如果你时间有限，建议按下面顺序读：

1. `docs/system/riscv/virt.rst:47`
2. `system/vl.c:3766`
3. `system/vl.c:2190`
4. `system/vl.c:2715`
5. `hw/riscv/virt.c:1923`
6. `hw/riscv/virt.c:1757`
7. `hw/riscv/virt.c:1529`
8. `hw/riscv/virt.c:1431`
9. `hw/riscv/boot.c:136`
10. `hw/riscv/boot.c:432`
11. `target/riscv/cpu_bits.h:752`
12. `target/riscv/cpu.c:680`

可以把它简化成下面这条跳转链：

`system/vl.c` → `hw/riscv/virt.c` → `hw/riscv/boot.c` → `target/riscv/cpu.c`

---

## 读代码时建议重点追的问题

### 问题 1：QEMU 什么时候选中 `virt`

看：

- `system/vl.c:2190`

你要找的答案是：

- machine type 是怎么被选择的
- `virt` 对应的 `MachineClass` 是怎么实例化的

### 问题 2：`virt` 板子是谁真正搭起来的

看：

- `hw/riscv/virt.c:1929`
- `hw/riscv/virt.c:1529`

你要找的答案是：

- 为什么最终会调用 `virt_machine_init()`
- `virt_machine_init()` 里到底创建了哪些设备和内存区域

### 问题 3：OpenSBI / BIOS 是什么时候加载的

看：

- `hw/riscv/virt.c:1431`
- `hw/riscv/boot.c:136`

你要找的答案是：

- firmware 不是在最开始加载，而是在 machine 初始化完成后处理
- 默认 firmware 会被加载到哪里

### 问题 4：为什么第一条指令在 `0x1000`

看：

- `target/riscv/cpu_bits.h:752`
- `target/riscv/cpu.c:680`
- `hw/riscv/boot.c:432`

你要找的答案是：

- `0x1000` 是默认 reset vector
- 这里放的是跳板代码，不是完整 firmware
- 跳板最终会跳到真正的 firmware 入口（在 `virt` 上通常和 DRAM 基地址相关）

---

## 把整条启动路径串起来

可以先用下面这份“低细节版本”记忆：

1. QEMU 从 `system/vl.c` 进入 system 模式启动流程
2. `qemu_create_machine()` 根据 `-M virt` 创建 `virt` machine 对象
3. `machine_run_board_init()` 调到该 machine 的 `init` 回调
4. `virt_machine_class_init()` 提前把这个回调设成了 `virt_machine_init()`
5. `virt_machine_init()` 把整台 `virt` 板子的 CPU、RAM、IRQ、串口、flash、virtio、PCIe 等设备搭起来
6. machine 初始化完成后，`virt_machine_done()` 开始处理 firmware、kernel、FDT
7. `riscv_find_and_load_firmware()` 把 OpenSBI / BIOS 加载到合适位置
8. `riscv_setup_rom_reset_vec()` 在 MROM 写入复位跳板
9. CPU 复位时，`riscv_cpu_reset_hold()` 执行 `env->pc = env->resetvec`
10. 默认 `resetvec` 是 `0x1000`，因此 CPU 从 `0x1000` 开始执行
11. 跳板再跳到真正的 firmware 入口，后续继续启动

---

## 初学时可以先不用深挖的内容

第一次读时，可以先略过这些细节：

- `virt_machine_init()` 里每个具体设备的 realize 过程
- FDT 每个节点是怎么生成的
- PLIC / AIA / ACLINT 的所有实现细节
- `fw_cfg` 的完整工作机制
- KVM 和 TCG 两套路径的差异（需要时可先看术语表里的 `TCG` / `KVM` 对照）

先把“谁调用谁、谁负责什么、为什么从 `0x1000` 起跑”搞懂，后面再拆细节会轻松很多。

---

## 最后浓缩成一句话

`-M virt` 的启动执行流程，最核心就是四个文件分工：

- `system/vl.c`：创建并启动 machine
- `hw/riscv/virt.c`：搭建 `virt` 板子
- `hw/riscv/boot.c`：加载 firmware 并写 reset vector
- `target/riscv/cpu.c`：CPU 复位后从 `resetvec` 开始跑

如果后面要继续深挖，最值得先啃透的函数只有三个：

- `virt_machine_init()`
- `virt_machine_done()`
- `riscv_setup_rom_reset_vec()`
