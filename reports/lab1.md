# Lab 1 Report

## U 态下执行 S 态特权指令/寄存器实验现象

在正确进入 U 态（用户态）后，用户程序如果尝试执行 S 态（特权态）指令或访问 S 态寄存器，会触发异常，导致程序出错。可以通过运行三个 bad 测例（ch2b_bad_address.rs、ch2b_bad_instructions.rs、ch2b_bad_register.rs）进行验证。

**测试过程与现象描述：**

1. **ch2b_bad_address.rs**  
	该程序尝试访问无效地址（如 0x0），会触发 Store/Page Fault。内核输出类似如下信息，并杀死该应用：
	```
	[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003a4, kernel killed it.
	```

2. **ch2b_bad_instructions.rs**  
	该程序在 U 态下直接执行 SRET（特权指令），会触发 IllegalInstruction 异常。内核输出如下，并杀死该应用：
	```
	[kernel] IllegalInstruction in application, kernel killed it.
	```

3. **ch2b_bad_register.rs**  
	该程序尝试读取 S 态寄存器 sstatus，同样会触发 IllegalInstruction 异常。内核输出如下，并杀死该应用：
	```
	[kernel] IllegalInstruction in application, kernel killed it.
	```

**SBI 及版本说明：**  
本实验环境使用的是 rustsbi 0.2.0-alpha.2 版本（见启动信息）：
```
[rustsbi] RustSBI version 0.3.0-alpha.2, adapting to RISC-V SBI v1.0.0
[rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.2
```

**结论：**  
在 U 态下，执行 S 态特权指令或访问 S 态寄存器会被内核捕获并终止应用，符合 RISC-V 权限隔离机制。


## trap.S 理解

### __alltraps 和 __restore 的作用
__alltraps：作为所有 trap（中断/异常）进入 S 态的入口，保存用户上下文（寄存器等），并调用 trap_handler 处理异常。
__restore：用于从 S 态恢复用户上下文，最终通过 sret 指令返回 U 态。

### L40：刚进入 __restore 时，sp 代表了什么值？__restore 的两种使用情景？
刚进入 __restore 时，sp 指向内核栈顶（即 TrapContext 保存区的最低地址），此时 sp 仍在内核栈。
两种使用情景：
1. 从 trap_handler 返回用户态时恢复上下文。
2. 任务切换时恢复新任务的上下文。

### L43-L48：特殊处理的寄存器及其意义
```
ld t0, 32*8(sp)
ld t1, 33*8(sp)
ld t2, 2*8(sp)
csrw sstatus, t0
csrw sepc, t1
csrw sscratch, t2
```
这几行分别恢复 sstatus（特权状态寄存器）、sepc（异常返回地址）、sscratch（用户栈指针）。
- sstatus：决定返回后 CPU 的特权级、全局中断等，必须恢复。
- sepc：trap 返回后从哪里继续执行用户代码。
- sscratch：保存用户栈指针，便于下次 trap 时切换。

### L50-L56：为何跳过了 x2 和 x4？
x2（sp）和 x4（tp）在上下文切换时有特殊用途：
- sp（x2）在 trap 期间已被切换到内核栈，用户栈指针已通过 sscratch 保存/恢复。
- tp（x4）通常用于线程指针，用户程序未用到，故跳过。

### L60：该指令之后，sp 和 sscratch 的意义
```
csrrw sp, sscratch, sp
```
执行后：
- sp 恢复为用户栈指针（即 sscratch 的值），
- sscratch 恢复为内核栈指针（即 sp 的值），为下次 trap 做准备。

### __restore 中发生状态切换在哪一条指令？为何该指令执行后会进入用户态？
状态切换发生在最后一条：
```
sret
```
sret 指令会根据 sstatus 的 SPP 位决定返回 S/U 态，恢复 sepc 指定的 PC，正式切换回用户态。

### L13：该指令之后，sp 和 sscratch 的意义
```
csrrw sp, sscratch, sp
```
此时：
- sp 切换为内核栈指针，
- sscratch 保存 trap 前的用户栈指针。

### 从 U 态进入 S 态是哪一条指令发生的？
从 U 态进入 S 态是在用户程序执行 ecall 指令时发生的（或其他 trap/中断）。

## 荣誉准则

在完成本次实验的过程（含此前学习的过程）中，我曾分别与 以下各位 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

《你交流的对象说明》

此外，我也参考了 以下资料 ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

《你参考的资料说明》

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。

