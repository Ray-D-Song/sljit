# 您的第一个程序

要使用 SLJIT，只需在您的代码中包含 `sljitLir.h` 头文件并链接到 `sljitLir.c`。
让我们直接进入您的第一个程序：

```c
#include "sljitLir.h"

#include <stdio.h>
#include <stdlib.h>

typedef sljit_sw (SLJIT_FUNC *func3_t)(sljit_sw a, sljit_sw b, sljit_sw c);

static int add3(sljit_sw a, sljit_sw b, sljit_sw c)
{
    void *code;
    sljit_uw len;
    func3_t func;

    /* 创建 SLJIT 编译器 */
    struct sljit_compiler *C = sljit_create_compiler(NULL);

    /* 开始一个上下文（函数序言） */
    sljit_emit_enter(C,
        0,                       /* 选项 */
        SLJIT_ARGS3(W, W, W, W), /* 1 个返回值和 3 个 sljit_sw 类型的参数 */
        1,                       /* 使用 1 个临时寄存器 */
        3,                       /* 使用 3 个保存寄存器 */
        0);                      /* 为函数局部变量分配 0 字节 */

    /* 函数的第一个参数存储在寄存器 SLJIT_S0 中，第二个在 SLJIT_S1 中，依此类推。 */
    /* R0 = first */
    sljit_emit_op1(C, SLJIT_MOV, SLJIT_R0, 0, SLJIT_S0, 0);

    /* R0 = R0 + second */
    sljit_emit_op2(C, SLJIT_ADD, SLJIT_R0, 0, SLJIT_R0, 0, SLJIT_S1, 0);

    /* R0 = R0 + third */
    sljit_emit_op2(C, SLJIT_ADD, SLJIT_R0, 0, SLJIT_R0, 0, SLJIT_S2, 0);

    /* 此语句将 R0 移动到返回寄存器并返回 */
    /* （实际上，R0 本身就是返回寄存器） */
    sljit_emit_return(C, SLJIT_MOV, SLJIT_R0, 0);

    /* 生成机器代码 */
    code = sljit_generate_code(C, 0, NULL);
    len = sljit_get_generated_code_size(C);

    /* 执行代码 */
    func = (func3_t)code;
    printf("func return %ld\n", (long)func(a, b, c));

    /* dump_code(code, len); */

    /* 清理 */
    sljit_free_compiler(C);
    sljit_free_code(code, NULL);

    return 0;
}

int main()
{
    return add3(4, 5, 6);
}
```

使用 SLJIT 生成代码通常从调用 `sljit_emit_enter` 开始，它生成*函数序言*：某些寄存器被保存到栈上，如果请求则为函数局部变量分配栈空间，并建立调用帧。

这是必要的，因为 SLJIT 生成的代码希望与程序的其他部分很好地互操作：它可以像 C 函数指针一样被调用，您可以轻松地从 JIT 编译的代码中[调用 C 函数](04-calling-external-functions.md)。

互操作的规则也被称为*应用程序二进制接口*，简称 ABI。这些通常在不同的架构和操作系统之间有所不同。最常见的是：
- [System V ABI](https://wiki.osdev.org/System_V_ABI) - 被主要的 Unix 操作系统使用（Linux / BSD / AIX / ...）
- [Windows x64 ABI](https://learn.microsoft.com/en-us/cpp/build/x64-software-conventions?view=msvc-170) - 被（您猜对了）Windows 使用

幸运的是，SLJIT 做了所有繁重的工作，因此在大多数情况下您不必担心平台和 ABI 特定的东西。不过，如果您对幕后的工作原理感兴趣，请查看 ABI 文档中的*调用约定*部分。

SLJIT 支持两种类型的寄存器（类似于底层的 ABI）：
- **临时寄存器**（也称为*易失性寄存器*） - 在函数调用中可能不保留其值
- **保存寄存器**（也称为*非易失性寄存器*） - 在函数调用中保留其值

我们在调用 `sljit_emit_enter` 时预先声明函数将使用的临时和保存寄存器的数量。然后通过它们的名称引用这些寄存器（临时寄存器为 `SLJIT_R0`、`SLJIT_R1`、`SLJIT_R2`、...，保存寄存器为 `SLJIT_S0`、`SLJIT_S1`、`SLJIT_S2`、...）。
此外，SLJIT 支持专用的浮点和向量寄存器。它们分别由 `SLJIT_FRN` / `SLJIT_FSN` 和 `SLJIT_VRN` / `SLJIT_VSN` 引用（其中 `N` 是从 `0` 开始的数字）。

由于不同的架构具有不同数量的寄存器，可用的临时和保存寄存器的数量在它们之间有所不同。您可以使用以下宏查询可用寄存器的数量：
- `SLJIT_NUMBER_OF_REGISTERS`（在所有支持的平台上 >= 12）
- `SLJIT_NUMBER_OF_SCRATCH_REGISTERS`
- `SLJIT_NUMBER_OF_SAVED_REGISTERS`（在所有支持的平台上 >= 6）
- `SLJIT_NUMBER_OF_FLOAT_REGISTERS`
- `SLJIT_NUMBER_OF_SAVED_FLOAT_REGISTERS`
- `SLJIT_NUMBER_OF_VECTOR_REGISTERS`
- `SLJIT_NUMBER_OF_SAVED_VECTOR_REGISTERS`

请注意，不同的寄存器集可能重叠，即两个不同的 SLJIT 寄存器可能引用同一个物理寄存器。在同类型的临时和保存寄存器之间以及浮点和向量寄存器之间尤其如此。

函数的签名使用 `SLJIT_ARGS...` 宏之一定义。在上面的示例中，函数声明了一个返回类型和三个类型为 `W` 的参数，即机器字宽度的整数。有关其他支持的类型，请查看 `sljitLir.h` 中的 `SLJIT_ARG_TYPE_...` 宏。参数默认分别在寄存器 `SLJIT_S0` - `SLJIT_S3`（用于整数）和 `SLJIT_FR0` - `SLJIT_FR3`（用于浮点数）中传递。类似地，返回值在 `SLJIT_R0` 或 `SLJIT_FR0` 中传递。

大多数通用操作通过调用 `sljit_emit_opN` 函数之一来执行。这些通常接受一个目标以及 `N` 个源操作数。

| 第一个参数 | 第二个参数 | 语义 | 示例 |
| --- | --- | --- | --- |
| `r` | `0` | 寄存器 `r` 中包含的值，即 `*r` | `SLJIT_R0, 0` |
| `SLJIT_IMM` | 值 `i` | 值 `i` | `SLJIT_IMM, 17` |
| `SLJIT_MEM` / `SLJIT_MEM0` | 地址 `a` | 地址 `a` 处的值 | `SLJIT_MEM0, &my_c_func` |
| `SLJIT_MEM1(r)` | 偏移量 `o` | 地址 `*r + o` 处的值 | `SLJIT_MEM1(SLJIT_R0), 16`<br />访问 `SLJIT_R0` 中的地址偏移 `16` 处的内存 |
| `SLJIT_MEM2(r1, r2)` | 移位 `s` | 地址 `*r1 + (*r2 * (1 << s))` 处的值；<br />`s` 必须在 `[0, 1, 2, 3]` 中 | `SLJIT_MEM2(SLJIT_R0, SLJIT_R1), 2`<br />访问从地址 `SLJIT_R0` 开始的长度为 `2` 字节的项目数组中索引为 `SLJIT_R1` 的项目 |

最后，通过调用 `sljit_emit_return` 将控制权返回给调用者。