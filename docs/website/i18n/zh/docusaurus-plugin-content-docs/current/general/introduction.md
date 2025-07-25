---
sidebar_position: 1
description: SLJIT 简介。
---

# 简介

## 概述

SLJIT 是一个无栈、低级且平台无关的 JIT 编译器。
也许*平台无关汇编器*能更好地描述它。
其核心设计原则是不试图比开发者更聪明。
这一原则通过提供对生成的机器代码的控制来实现，就像在其他汇编语言中一样。
然而，与其他汇编语言不同，SLJIT 利用了一种平台无关的*低级中间表示*（LIR），这大大提高了可移植性。

SLJIT 试图在性能和可维护性之间取得良好的平衡。
LIR 代码可以编译到许多 CPU 架构，生成代码的性能非常接近用本机汇编语言编写的代码。
虽然 SLJIT 不支持自动寄存器分配等高级功能，但它可以作为其他 JIT 编译器库的代码生成后端。
在 SLJIT 之上开发这些中间库所需的时间要少得多，因为它们只需要支持单一后端。

如果您想直接开始并看到 SLJIT 的实际应用，请查看[教程](../tutorial/01-overview.md)。


## 特性

- 支持多种[目标架构](#支持的平台)
- 支持大量操作
    - 自修改代码
    - 尾调用
    - 快速调用（非 ABI 兼容）
    - 字节序反转（字节序转换）
    - 非对齐内存访问
    - SIMD[^1]
    - 原子操作[^1]
- 允许直接访问寄存器（整数和浮点数）
- 支持为函数局部变量分配栈空间
- 支持[一体化](getting-started/setup.md#sljit-一体化)编译
    - 允许完全隐藏 SLJIT 的 API，不对外使用
- 允许将编译器序列化到字节缓冲区
    - 对提前（AOT）编译很有用
    - 反序列化后可以恢复代码生成（部分 AOT 编译）

## 支持的平台

| 平台 | 64位 | 32位 |
| --- | --- | --- |
| `x86` | ✅ | ✅ |
| `ARM` | ✅ | ✅[^2] |
| `RISC-V` | ✅ | ✅ |
| `s390x` | ✅ | ❌ |
| `PowerPC` | ✅ | ✅ |
| `LoongArch` | ✅ | ❌ |
| `MIPS` | ✅[^3] | ✅[^3] |

[^1]: 仅在特定平台上支持
[^2]: ARM-v6、ARM-v7 和 Thumb2 指令集
[^3]: III、R1