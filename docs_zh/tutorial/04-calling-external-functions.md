# 调用外部函数

要从 JIT 编译的代码调用外部函数，只需调用 `sljit_emit_icall` 并指定 `SLJIT_CALL`，它代表平台的默认 C 调用约定。还有其他调用约定可用于更高级的用例。

与 `sljit_emit_enter` 类似，`sljit_emit_icall` 需要了解目标函数的签名。因此，要调用一个以 `sljit_sw` 作为唯一参数并返回 `sljit_sw` 的函数，您将使用 `SLJIT_ARGS1(W, W)` 指定其签名。整数参数在寄存器 `R0`、`R1` 和 `R2` 中传递，结果（如果存在）在 `R0` 中返回。

要将 SLJIT 指向您想要调用的函数，您可以使用 `SLJIT_FUNC_ADDR` 将其地址作为立即值传递。

因此，要调用函数 `sljit_sw print_num(sljit_sw a)`，传递 `S2` 中的值，您可以执行以下操作：

```c
/* R0 = S2 */
sljit_emit_op1(C, SLJIT_MOV, SLJIT_R0, 0, SLJIT_S2, 0);
/* print_num(R0) */
sljit_emit_icall(C, SLJIT_CALL, SLJIT_ARGS1(W, W), SLJIT_IMM, SLJIT_FUNC_ADDR(print_num));
```

*完整的示例源代码可以在[这里](sources/func_call.c)找到。*