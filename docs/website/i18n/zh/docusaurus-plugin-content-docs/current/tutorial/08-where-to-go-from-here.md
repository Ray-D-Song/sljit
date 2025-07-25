# 后续学习指南

您现在应该对如何使用 SLJIT 有了基本了解。如果事情仍然不清楚或您想了解有关某个主题的更多信息，请查看 [`sljitLir.h`](https://github.com/zherczeg/sljit/blob/master/sljit_src/sljitLir.h)。

此外，您可能已经注意到在本教程的源代码中注释掉的 `dump_code` 调用。如果您对 SLJIT 实际生成的汇编代码感兴趣，您可以提供自己的例程来执行此操作（假设您使用的是 Linux `x86-32` 或 `x86-64`）：

```c
static void dump_code(void *code, sljit_uw len)
{
    FILE *fp = fopen("/tmp/sljit_dump", "wb");
    if (!fp)
        return;
    fwrite(code, len, 1, fp);
    fclose(fp);
#if defined(SLJIT_CONFIG_X86_64)
    system("objdump -b binary -m l1om -D /tmp/sljit_dump");
#elif defined(SLJIT_CONFIG_X86_32)
    system("objdump -b binary -m i386 -D /tmp/sljit_dump");
#endif
}
```

最后，要查看 SLJIT 在更复杂场景中的使用，请查看这个 [Brainfuck](https://en.wikipedia.org/wiki/Brainfuck) [编译器](sources/brainfuck.c)。