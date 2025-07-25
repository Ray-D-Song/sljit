---
sidebar_position: 3
description: SLJIT 的内部工作原理。
---

# 架构

## 低级中间表示

定义一个 LIR 既能提供广泛的优化机会，又能高效地翻译成所有 CPU 上的机器代码，这是本项目最大的挑战。
我们仔细选择了那些在许多（但不一定是所有）架构上都支持的指令形式和特性，并由此创建了一个 LIR。
这些特性也被其余架构以低开销的方式模拟。
例如，SLJIT 支持各种内存寻址模式和设置状态寄存器位。

## 通用 CPU 模型

CPU 具有：
  - 整数寄存器，可以存储 `int32_t`（4字节）或 `intptr_t`（4或8字节）值
  - 浮点寄存器，可以存储单精度（4字节）或双精度（8字节）值
  - 布尔状态标志

某些平台还支持向量寄存器，这些寄存器可能与浮点寄存器重叠。

### 整数寄存器

最重要的规则是：当指令的源操作数是寄存器时，寄存器的数据类型必须与指令期望的数据类型匹配。

例如，以下代码片段是一个有效的指令序列：

```c
sljit_emit_op1(compiler, SLJIT_MOV32,
    SLJIT_R0, 0, SLJIT_MEM1(SLJIT_R1), 0);
// 一个 int32_t 值被加载到 SLJIT_R0 中
sljit_emit_op1(compiler, SLJIT_REV32,
    SLJIT_R0, 0, SLJIT_R0, 0);
// SLJIT_R0 中的 int32_t 值被字节交换
// 结果的类型仍然是 int32_t
```

下一个代码片段是不允许的：

```c
sljit_emit_op1(compiler, SLJIT_MOV,
    SLJIT_R0, 0, SLJIT_MEM1(SLJIT_R1), 0);
// 一个 intptr_t 值被加载到 SLJIT_R0 中
sljit_emit_op1(compiler, SLJIT_REV32,
    SLJIT_R0, 0, SLJIT_R0, 0);
// 指令的结果是未定义的。
// 对于某些指令甚至可能发生崩溃
// （例如在 MIPS-64 上）。
```

但是，无论寄存器的先前值如何，总是允许覆盖寄存器：

```c
sljit_emit_op1(compiler, SLJIT_MOV,
    SLJIT_R0, 0, SLJIT_MEM1(SLJIT_R1), 0);
// 一个 intptr_t 值被加载到 SLJIT_R0 中
sljit_emit_op1(compiler, SLJIT_MOV32,
    SLJIT_R0, 0, SLJIT_MEM1(SLJIT_R2), 0);
// 从现在开始，SLJIT_R0 包含一个 int32_t
// 值。之前的值被丢弃。
```

提供了类型转换指令来将 `int32_t` 值转换为 `intptr_t` 值，反之亦然。
在某些架构中，这些转换是*空操作*（不发出指令）。

#### 内存访问

`SLJIT_MEM1` / `SLJIT_MEM2` 寻址模式的寄存器参数必须包含 `intptr_t` 数据。

#### 有符号/无符号值

大多数操作的执行方式相同，无论值是有符号还是无符号。
这些操作只有一种指令形式（例如 `SLJIT_ADD` / `SLJIT_MUL`）。
结果取决于符号的指令有两种形式（例如整数除法、长乘法）。

### 浮点寄存器

浮点寄存器可以包含单精度或双精度值。
与整数寄存器类似，存储在源寄存器中的值的数据类型必须与指令期望的数据类型匹配。
否则结果是未定义的（甚至可能发生崩溃）。

#### 舍入

与标准 C 类似，浮点计算结果向零舍入。

### 布尔状态标志

条件分支通常依赖于 CPU 状态标志的值。
这些状态标志是布尔值，可以由某些指令设置。

为了更好地与没有标志的 CPU 兼容，SLJIT 只公开两个这样的标志：
  - 如果指定了 `SLJIT_SET_Z` 且结果为零，则设置**零**（或相等）标志。
  - **可变**标志的含义取决于设置它的算术操作以及请求的标志类型。例如，`SLJIT_ADD` 支持设置 `Z`、`CARRY` 和 `OVERFLOW` 标志。但是后两者都将映射到可变标志，因此不能同时请求。

请注意，不同的操作对标志在之后的状态提供不同的保证。
此外，并非所有操作都支持所有标志。
有关更多详细信息，请参阅 [`sljitLir.h`](https://github.com/zherczeg/sljit/blob/master/sljit_src/sljitLir.h)。

## 复杂指令

我们注意到为常见任务引入复杂指令可以提高性能。
例如，如果满足某些条件，比较和分支指令序列可以被优化，但这些条件取决于目标 CPU。
SLJIT 可以进行这些优化，但它需要理解生成代码的"目的"。
静态指令分析有很大的性能开销，所以我们选择了另一种方法：我们为某些非原子任务引入了复杂的指令形式。
SLJIT 可以更有效地优化这些指令，因为编译器知道它们的目的。
这些复杂的指令形式通常可以从其他 SLJIT 指令组装而成，但我们建议使用它们，因为编译器可以在某些 CPU 上优化它们。

## 生成函数

SLJIT 通常用于生成从 C 调用的函数体。
SLJIT 提供两个复杂指令来生成函数入口和返回：`sljit_emit_enter` 和 `sljit_emit_return`。
`sljit_emit_enter` 函数还初始化编译上下文，该上下文指定当前寄存器映射、局部空间大小和其他配置。
`sljit_set_context` 函数也可以设置此上下文而不发出任何机器指令。

此上下文很重要，因为它影响编译器，所以创建编译器后的第一条指令必须是 `sljit_emit_enter` 或 `sljit_set_context`。
可以通过再次调用 `sljit_emit_enter` 或 `sljit_set_context` 来更改上下文。

## 类型

SLJIT 定义了几种类型来表示目标平台上的数据，通常既有有符号变体也有无符号变体：
- 不同大小的整数：`sljit_s8` / `sljit_u8`、`sljit_s16` / `sljit_u16`、`sljit_s32` / `sljit_u32`
- 机器字，能够保存指针：`sljit_sw` / `sljit_uw`、`sljit_sp` / `sljit_up`
- 浮点类型：`f32`、`f64`

建议使用这些类型而不是默认的 C 类型（如 `long`），因为它提高了代码的可读性和可移植性。