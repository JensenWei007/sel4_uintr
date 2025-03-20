# 多层函数嵌套后再进行系统调用导致在栈帧RSP错误时触发UINTR    ---   已修复
====================

## 问题表现

目前在单核情况下，也就是接收线程需要阻塞后让出调度去执行发送线程，此时用户态中断不会触发，会通过puir位置1来遗留一个中断，在接收线程即将进入用户态时（有点类似于信号的处理时机），会检测其puir位，如果不为0，说明有一个用户态中断被挂起了，需要重新触发一个中断。

这种情况下，需要使用apic给自己发一个中断，代码为 `apic_send_ipi_core(UINTR_NOTIFICATION_VECTOR, cur_cpu);` 。这个中断会在硬件的执行过程中（在陷入内核态触发中断进入内核代码之前），提前检测并拦截此中断号，进入用户态中断处理流程。

但是在sel4中，大概率会发生以下情况导致出现异常：

```
Step    cpu 0 (receiver task)           cpu 0 (sender task)
----    ---------------------           -------------------
1       task is running
2       function call
3       save RSP
4       block and schedule
5                                       task is running
6                                       executes SENDUIPI
7                                       IPI sent
8                                       task is end, schedule
9       waken up, ret to user
11      IPI delivered
12      go to uintr_handler
13      Restore RSP
```

这样会在刚进入用户态且尚未恢复RSP时执行用户态中断处理，导致RSP错误。

如果运行会出现下面的错误：

```
Pagefault from [UINTR0002]: write fault at PC: 0x408471 vaddr: 0xffffffffffffff78, FSR 0x2
Register of root thread in test (may not be the thread that faulted)
Register dump:
rip:	0x408471
rsp:	0xffffffffffffff80
rflags:	0x293
rax:	0xffffffffffffffff
rbx:	0x100112a0
rcx:	0x408471
rdx:	0xffffffffffffffff
rsi:	0x0
rdi:	0x0
rbp:	0x100112b0
r8:	0x0
r9:	0x0
r10:	0x9
r11:	0x293
r12:	0x0
r13:	0x0
r14:	0x0
r15:	0x0
fs_base:	0x4850d0
gs_base:	0x0
	Failure: result == SUCCESS at line 296 of file /home/wjx/sel4/projects/sel4test/apps/sel4test-driver/src/testtypes.c
	Error: result == SUCCESS at line 217 of file /home/wjx/sel4/projects/sel4test/apps/sel4test-driver/src/main.c
```

可以看到，是程序尝试向地址为`0xffffffffffffff78`的地址写入数据，执行操作的部分在qemu中：

```c
void helper_rrnzero(CPUX86State *env){ // 改
    target_ulong temprsp = env->regs[R_ESP];
    if(env->uintr_stackadjust &1){ // adjust[0] = 1
        env->regs[R_ESP] = env->uintr_stackadjust;
    }else{
        env->regs[R_ESP] -= env->uintr_stackadjust;
    }
    env->regs[R_ESP] &= ~0xfLL; /* align stack */
    uint64_t uintr_rrv = 63 - __builtin_clzll(env->uintr_rr);
    target_ulong esp = env->regs[R_ESP];
    PUSHQ(esp, temprsp);
    PUSHQ(esp, env->eflags); // PUSHQ(esp, cpu_compute_eflags(env));
    PUSHQ(esp, env->eip);
    PUSHQ(esp, uintr_rrv); // 64-bit push; upper 58 bits pushed as 0
    env->uintr_rr &= ~(1 << uintr_rrv); // clear rr
    env->regs[R_ESP] = esp;
    env->eflags &= ~(TF_MASK | RF_MASK);
    env->eip = env->uintr_handler;
    switch_uif(env, false);
}
```

此处是qemu压栈后通过函数调用的方式跳转到中断处理函数，此处的写入就是向RSP地址写数据（也就是压栈）。上面qemu代码中`esp`变量的值就是`0xffffffffffffff78`。

正确的预期行为是：

```
Step    cpu 0 (receiver task)           cpu 0 (sender task)
----    ---------------------           -------------------
1       task is running
2       function call
3       save RSP
4       block and schedule
5                                       task is running
6                                       executes SENDUIPI
7                                       IPI sent
8                                       task is end, schedule
9       waken up, ret to user
10      Restore RSP                <=========================
11      IPI delivered
12      go to uintr_handler
```

> 在修复过程中发现，如果你的电脑性能比较好，qemu执行的快，那么大概率你会碰到异常情况，但是如果你的电脑比较卡，那么有一定的概率能执行成功（不会碰到异常）。不改变任何代码和环境，只是重复执行qemu run。

## 问题原因

引起这个问题的原因非常简单但是十分难定位，经过多次定位和比对，上面的错误报告中，`RBX`中的值即为正确的`RSP`值，且在此过程中，是通过系统调用阻塞和返回的，原因位于系统调用部分：

```c
static inline void x64_sys_recv(seL4_Word sys, seL4_Word src, seL4_Word *out_badge, seL4_Word *out_info,
                                seL4_Word *out_mr0, seL4_Word *out_mr1, seL4_Word *out_mr2, seL4_Word *out_mr3,
                                LIBSEL4_UNUSED seL4_Word reply)
{
    register seL4_Word mr0 asm("r10");
    register seL4_Word mr1 asm("r8");
    register seL4_Word mr2 asm("r9");
    register seL4_Word mr3 asm("r15");
    MCS_REPLY_DECL;

    asm volatile(
        "movq   %%rsp, %%rbx    \n"
        "syscall                \n"
        "movq   %%rbx, %%rsp    \n"
        : "=D"(*out_badge),
        "=S"(*out_info),
        "=r"(mr0),
        "=r"(mr1),
        "=r"(mr2),
        "=r"(mr3)
        : "d"(sys),
        "D"(src)
        MCS_REPLY
        : "%rcx", "%rbx", "r11", "memory"
    );
    *out_mr0 = mr0;
    *out_mr1 = mr1;
    *out_mr2 = mr2;
    *out_mr3 = mr3;
}
```

上面过程中，内核执行返回后，下一个地址为执行`movq   %%rbx, %%rsp`。但是在执行完`sysretq`指令后，硬件就会检测中断，此时会检测到用户态中断随后进入处理流程，`movq   %%rbx, %%rsp`的指令被截断了，导致RSP没有恢复。

> 这个写法真的难绷，你为啥不恢复完了所有东西后再返回？linux就是这样做的，抽象。

## 修复方法

修复方法很简单，在返回用户态之前，提前恢复`RSP`的即可：

```c
#if defined(CONFIG_X86_64_UINTR)
            "movq %%rbx, %%rsp\n"
#endif
            // More register but we can ignore and are done restoring
            // enable interrupt disabled by sysenter
            "sysretq\n"
```
