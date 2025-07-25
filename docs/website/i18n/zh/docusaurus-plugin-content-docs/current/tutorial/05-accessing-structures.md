# 访问结构体

虽然 SLJIT 不直接支持记录类型（结构体和类），但使用它们仍然非常容易。假设您在 `S0` 中有一个指向如下定义的 `struct point` 的地址：

```c
struct point_st {
	sljit_sw x;
	sljit_s32 y;
	sljit_s16 z;
	sljit_s8 d;
};
```

要将成员 `y` 移动到 `R0`，您可以使用 `SLJIT_MEM1` 寻址模式，它允许我们指定偏移量。要获得 `y` 在 `point_st` 中的偏移量，您可以使用方便的 `SLJIT_OFFSETOF` 宏，如下所示：

```c
sljit_emit_op1(C, SLJIT_MOV_S32, SLJIT_R0, 0, SLJIT_MEM1(SLJIT_S0), SLJIT_OFFSETOF(struct point_st, y));
```

并且始终记住使用正确类型的 `SLJIT_MOV` 变体！

*完整的示例源代码可以在[这里](sources/struct_access.c)找到。*