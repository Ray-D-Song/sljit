# 分支

分支允许我们转移控制流。这对于实现条件语句和循环等高级构造很有用。

分支，也称为*跳转*，有两种类型：

- **条件分支：** 只有在满足条件时才执行；否则，在下一条指令处继续执行
- **无条件分支：** 总是执行

跳转可以通过调用 `sljit_emit_cmp`（条件）和 `sljit_emit_jump`（无条件）来发出。这两个函数都返回指向 `struct sljit_jump` 的指针，稍后需要将其连接到 `struct sljit_label`（跳转的目标）。

标签可以通过调用 `sljit_emit_label` 发出，并通过 `sljit_set_label` 连接到跳转。整个机制与 C 中的标签和 goto 非常相似。

为了将条件语句和循环等高级构造映射到 SLJIT，考虑用标签和（无）条件 goto 来思考它们会有所帮助。例如，以下简单函数体：

```c
if ((a & 1) == 0)
    return c;
return b;
```

用标签和 goto 表达隐式控制流：

```
    R0 = a & 1;
    if R0 == 0 then goto ret_c;
    R0 = b;
    goto out;
ret_c:
    R0 = c;
out:
    return R0;
```

这也是高级语言编译器在幕后所做的。现在可以轻松地用 SLJIT 组装结果：

```c
#include "sljitLir.h"

#include <stdio.h>
#include <stdlib.h>

typedef sljit_sw (SLJIT_FUNC *func3_t)(sljit_sw a, sljit_sw b, sljit_sw c);

static int branch(sljit_sw a, sljit_sw b, sljit_sw c)
{
	void *code;
	sljit_uw len;
	func3_t func;

	struct sljit_jump *ret_c;
	struct sljit_jump *out;

	/* 创建 SLJIT 编译器 */
	struct sljit_compiler *C = sljit_create_compiler(NULL);

	/* 3 个参数，1 个临时寄存器，3 个保存寄存器 */
	sljit_emit_enter(C, 0, SLJIT_ARGS3(W, W, W, W), 1, 3, 0);

	/* R0 = a & 1，S0 是参数 a */
	sljit_emit_op2(C, SLJIT_AND, SLJIT_R0, 0, SLJIT_S0, 0, SLJIT_IMM, 1);

	/* 如果 R0 == 0 则跳转到 ret_c，ret_c 在哪里？我们稍后分配它 */
	ret_c = sljit_emit_cmp(C, SLJIT_EQUAL, SLJIT_R0, 0, SLJIT_IMM, 0);

	/* R0 = b，S1 是参数 b */
	sljit_emit_op1(C, SLJIT_MOV, SLJIT_RETURN_REG, 0, SLJIT_S1, 0);

	/* 跳转到 out */
	out = sljit_emit_jump(C, SLJIT_JUMP);

	/* 这里是 'ret_c' 应该跳转的地方，我们发出一个标签并将其设置为 ret_c */
	sljit_set_label(ret_c, sljit_emit_label(C));

	/* R0 = c，S2 是参数 c */
	sljit_emit_op1(C, SLJIT_MOV, SLJIT_RETURN_REG, 0, SLJIT_S2, 0);

	/* 这里是 'out' 应该跳转的地方 */
	sljit_set_label(out, sljit_emit_label(C));

	/* 函数结束 */
	sljit_emit_return(C, SLJIT_MOV, SLJIT_RETURN_REG, 0);

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
	return branch(4, 5, 6);
}
```

*完整的示例源代码可以在[这里](sources/branch.c)找到。*

基于这些基本技术，您可以进一步使用分支来生成循环。因此，给定以下函数体：

```c
i = 0;
ret = 0;
for (i = 0; i < a; ++i) {
    ret += b;
}
return ret;
```

您可以再次使用标签和 goto 使控制流显式：

```
    i = 0;
    ret = 0;
loopstart:
    if i >= a then goto out;
    ret += b
    goto loopstart;
out:
    return ret;
```

然后使用它来用 SLJIT 组装函数：

```c
#include "sljitLir.h"

#include <stdio.h>
#include <stdlib.h>

typedef sljit_sw (SLJIT_FUNC *func2_t)(sljit_sw a, sljit_sw b);

static int loop(sljit_sw a, sljit_sw b)
{
	void *code;
	sljit_uw len;
	func2_t func;

	struct sljit_label *loopstart;
	struct sljit_jump *out;

	/* 创建 SLJIT 编译器 */
	struct sljit_compiler *C = sljit_create_compiler(NULL);

	/* 2 个参数，2 个临时寄存器，2 个保存寄存器 */
	sljit_emit_enter(C, 0, SLJIT_ARGS2(W, W, W), 2, 2, 0);

	/* R0 = 0 */
	sljit_emit_op2(C, SLJIT_XOR, SLJIT_R1, 0, SLJIT_R1, 0, SLJIT_R1, 0);
	/* RET = 0 */
	sljit_emit_op1(C, SLJIT_MOV, SLJIT_RETURN_REG, 0, SLJIT_IMM, 0);
	/* loopstart: */
	loopstart = sljit_emit_label(C);
	/* R1 >= a --> 跳出 */
	out = sljit_emit_cmp(C, SLJIT_GREATER_EQUAL, SLJIT_R1, 0, SLJIT_S0, 0);
	/* RET += b */
	sljit_emit_op2(C, SLJIT_ADD, SLJIT_RETURN_REG, 0, SLJIT_RETURN_REG, 0, SLJIT_S1, 0);
	/* R1 += 1 */
	sljit_emit_op2(C, SLJIT_ADD, SLJIT_R1, 0, SLJIT_R1, 0, SLJIT_IMM, 1);
	/* 跳转 loopstart */
	sljit_set_label(sljit_emit_jump(C, SLJIT_JUMP), loopstart);
	/* out: */
	sljit_set_label(out, sljit_emit_label(C));

	/* 返回 RET */
	sljit_emit_return(C, SLJIT_MOV, SLJIT_RETURN_REG, 0);

	/* 生成机器代码 */
	code = sljit_generate_code(C, 0, NULL);
	len = sljit_get_generated_code_size(C);

	/* 执行代码 */
	func = (func2_t)code;
	printf("func return %ld\n", (long)func(a, b));

	/* dump_code(code, len); */

	/* 清理 */
	sljit_free_compiler(C);
	sljit_free_code(code, NULL);
	return 0;
}

int main()
{
	return loop(4, 5);
}
```

其他条件语句和循环可以用非常相似的方式实现。

*完整的示例源代码可以在[这里](sources/loop.c)找到。*