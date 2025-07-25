---
sidebar_position: 1
description: 设置 SLJIT。
---

# 设置

## 先决条件

要编译 SLJIT，您需要一个支持 `C99` 标准的 C/C++ 编译器。

如果您想构建[测试](#构建测试)或[示例](#构建示例)，您还需要 [GNU Make](https://www.gnu.org/software/make/)。如果您在 Windows 上使用 MSVC 工具链（GNU Make 不可用），您可以使用 [CMake](https://cmake.org/) 来构建测试。

## 在您的项目中使用 SLJIT

将 [`sljit_src` 文件夹](https://github.com/zherczeg/sljit/tree/master/sljit_src) 的内容放置在项目源目录中的合适位置。

SLJIT 可以通过以下两种方式之一使用：

### SLJIT 作为库

将 `sljitLir.c` 编译为独立的翻译单元。确保将 `sljit_src` 添加到包含目录列表中，例如在使用 GCC / Clang 编译时指定 `-Ipath/to/sljit`。

要使用 SLJIT，请包含 `sljitLir.h` 并链接到 `sljitLir.o`。

### SLJIT 一体化

如果您想避免将 SLJIT 的接口暴露给其他翻译单元，您也可以将 SLJIT 嵌入为隐藏的实现细节。为此，在包含 `sljitLir.c`（是的，C 文件）之前定义 `SLJIT_CONFIG_STATIC`：

```c title="hidden.c"
#define SLJIT_CONFIG_STATIC 1
#include "sljitLir.c"

// SLJIT 可以在 hidden.c 中使用，但不能在其他翻译单元中使用
```

这种技术也可以用于为多个目标架构生成代码：

```c title="x86-32.c"
#define SLJIT_CONFIG_STATIC 1
#define SLJIT_CONFIG_X86_32 1
#include "sljitLir.c"

// 为 x86 32 位生成代码
```

```c title="x86-64.c"
#define SLJIT_CONFIG_STATIC 1
#define SLJIT_CONFIG_X86_64 1
#include "sljitLir.c"

// 为 x86 64 位生成代码
```

`x86-32.c` 和 `x86-64.c` 都可以链接到同一个二进制文件/库中，而不会因为符号冲突而出现问题。

## 构建测试

### 使用 GNU Make

导航到 SLJIT 仓库的根目录，并使用 GNU Make 构建默认目标：

```bash
make
```

测试可执行文件 `sljit_test` 可以在 `bin` 目录中找到。要运行测试，只需执行它。

### 使用 CMake

> [!NOTE]
> CMake 支持目前仅作为 Windows 系统（GNU Make 不可用）的替代方案。

要在 Windows 上使用 MSVC 和 NMake 构建测试，请执行以下操作：

1. 打开合适的[开发者命令提示符](https://learn.microsoft.com/en-us/visualstudio/ide/reference/command-prompt-powershell?view=vs-2022)并导航到 SLJIT 仓库的根目录
2. 执行以下命令：
    ```bash
    cmake -B bin -G "NMake Makefiles"
    cmake --build bin
    ```

测试可执行文件 `sljit_test.exe` 可以在 `bin` 目录中找到。

## 构建示例

> [!NOTE]
> 目前您无法在 Windows 上使用 MSVC / CMake 构建示例。

使用 GNU Make 构建 `examples` 目标：

```bash
make examples
```

不同的示例可执行文件可以在 `bin` 目录中找到。