# 访问数组

访问数组与访问结构体类似。如果您处理的是最多 8 字节项目的数组，您可以利用 SLJIT 的 `SLJIT_MEM2` 寻址模式的移位。例如，假设您在 `S0` 中有一个 `sljit_sw` 数组的基地址，并想要访问第 `S2` 个项目：

```c
sljit_sw s0[];
r0 = s0[s2];
```

您可以简单地使用 `SLJIT_WORD_SHIFT` 来正确地考虑每个项目的大小：

```c
sljit_emit_op1(C, SLJIT_MOV, SLJIT_R0, 0, SLJIT_MEM2(SLJIT_S0, SLJIT_S2), SLJIT_WORD_SHIFT);
```

如果项目的大小不能用 `SLJIT_MEM2` 的移位表示，您仍然可以回退到手动计算偏移量并将其传递给 `SLJIT_MEM1`。这也可以与结构体访问结合。假设您想要组装以下内容：

```c
struct point_st {
	sljit_sw x;
	sljit_s32 y;
	sljit_s16 z;
	sljit_s8 d;
};

point_st s0[];
r0 = s0[s2].x;
```

由于 `point_st` 大于 8 字节，您需要事先将数组索引转换为字节偏移量并将其存储在临时变量中：

```c
/* 计算数组索引：R2 = S2 * sizeof(point_st) */
sljit_emit_op2(C, SLJIT_MUL, SLJIT_R1, 0, SLJIT_S2, 0, SLJIT_IMM, sizeof(struct point_st));
/* 访问数组项目：R0 = S0[R2] */
sljit_emit_op1(C, SLJIT_MOV, SLJIT_R0, 0, SLJIT_MEM1(SLJIT_S0), SLJIT_R2);
```

上面的示例依赖于 `x` 是 `point_st` 的第一个成员，因此与周围的结构体共享其地址这一事实。如果您想要访问不同的成员，您需要进一步考虑其偏移量。

*完整的示例源代码可以在[这里](sources/array_access.c)找到。*