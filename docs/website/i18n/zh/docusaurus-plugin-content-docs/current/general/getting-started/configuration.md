---
sidebar_position: 2
description: 配置 SLJIT。
---

# 配置

SLJIT 的行为可以通过 `sljitConfig.h` 中描述的一系列定义来控制。

| 定义 | 默认启用 | 描述 |
| --- | --- | --- |
| `SLJIT_DEBUG` | ✅ | 启用断言。在发布模式下应该禁用。 |
| `SLJIT_VERBOSE` | ✅ | 当启用此宏时，可以使用 `sljit_compiler_verbose` 函数来转储 SLJIT 指令。否则此函数不可用。在发布模式下应该禁用。 |
| `SLJIT_SINGLE_THREADED` | ❌ | 单线程程序可以定义此标志，这消除了 `pthread` 依赖。 |