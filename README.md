# supervisor-la32r：LoongArch 32 Reduced 监控程序

## 介绍

监控程序分为两个部分，Kernel 和 Term。其中 Kernel 使用 LoongArch 32 Reduced 汇编语言编写，运行在学生实现的 CPU 中，用于管理硬件资源；Term 是上位机程序，使用 Python 语言编写，有基于命令行的用户界面，达到与用户交互的目的。Kernel 和 Term 直接通过串口通信，即用户在 Term 界面中输入的命令、代码经过 Term 处理后，通过串口传输给 Kernel 程序；反过来，Kernel 输出的信息也会通过串口传输到 Term，并展示给用户。

## Kernel

Kernel 使用汇编语言编写，使用到的指令均符合 LoongArch 32 Reduced 规范。Kernel 提供了三种不同的版本，以适应不同的档次的 CPU 实现。它们分别是：第一档为基础版本，直接基本的 I/O 和命令执行功能，不依赖异常、中断等处理器特征，适合于最简单的 CPU 实现；第二档支持中断，使用中断方式完成串口的 I/O 功能，需要处理器实现中断处理机制；第三档在第二档基础上进一步增加了 TLB 的应用，要求处理器支持基于 TLB 的内存映射，更加接近于操作系统对处理器的需求。

为了在硬件上运行 Kernel 程序，我们首先要对 Kernel 的汇编代码进行编译。编译时需要龙芯提供的 LoongArch 32 Reduced 工具链。将下载的压缩包解压到任意目录后，设置环境变量 `GCCPREFIX` 以便 make 工具找到编译器，例如：

`export GCCPREFIX=/usr/local/loongarch32r-linux-gnusf/bin/loongarch32r-linux-gnusf-gcc`

下面是编译监控程序的过程。在 `kernel` 文件夹下面，有汇编代码和 Makefile 文件，我们可以使用 make 工具编译 Kernel 程序。

假设当前目录为 `kernel`，目标版本为基础版本，在终端中运行命令

`make`

即可编译基础版本的监控程序。如果顺利结束，将生成 `kernel.elf` 和 `kernel.bin` 文件。要在模拟器中运行它，可以使用命令

`make sim`

它会在 QEMU 中启动监控程序，并等待 Term 程序连接。本文后续章节介绍了如何使用 Term 连接模拟器。

若要在开发板上运行 kernel，使用开发板提供的工具，将 `kernel.bin` 写入内存 0 地址（物理地址）位置，并让处理器复位从 0x8000000 地址（LoongArch 32 Reduced 中对应物理地址为 0 的虚地址）处开始执行，Kernel 就运行起来了。

Kernel 运行后会先通过串口输出版本号，该功能可作为检验其正常运行的标志。之后 Kernel 将等待 Term 从串口发来的命令，关于 Term 的使用将在后续章节描述。

接下来我们分别说明三个档次的监控程序对于硬件的要求，及简要的设计思想。

### 基础版本

基础版本的 Kernel 共使用了 18 条不同的指令，它们是：

1. `addi.w`
1. `andi`
1. `b`
1. `beq`
1. `bl`
1. `bne`
1. `csrwr`
1. `jirl`
1. `ld.b`
1. `ld.w`
1. `lu12i.w`
1. `move`
1. `or`
1. `ori`
1. `slli.w`
1. `srli.w`
1. `st.b`
1. `st.w`

根据 LoongArch 32 Reduced 规范正确实现这些指令后，程序才能正常工作。

监控程序支持三种启动方式：

1. 监控程序被加载到物理地址 0x1c000000。处理器处于直接地址翻译模式，监控程序从 0x1c000000 地址开始执行。
2. 监控程序被加载到物理地址 0x00000000。处理器处于直接地址翻译模式，监控程序从 0x00000000 地址开始执行。
3. 监控程序被加载到物理地址 0x00000000，处理器处于映射地址翻译模式，且 0x80000000 被映射到 0x00000000。监控程序从 0x80000000 地址开始执行。

无论哪种启动方式，监控程序在启动时，会配置 CSR.DMW0、CSR.DMW1 和 CSR.CRMD，并进入映射地址翻译模式。具体地，监控程序会按照下列内存映射来配置 CSR.DMW0 和 CSR.DMW1：

1. 0x80000000-0x9FFFFFFF: 映射到 0x00000000-0x1FFFFFFF，访问类型为一致可缓存，对应 CSR.DMW0=0x80000011
2. 0xA0000000-0xBFFFFFFF: 映射到 0x00000000-0x1FFFFFFF，访问类型为强序非缓存，对应 CSR.DMW1=0xA0000001

监控程序使用了 8 MB 的内存空间，其中约 1 MB 由 Kernel 使用，剩下的空间留给用户程序。此外，为了支持串口通信，还设置了一个内存以外的地址区域，用于串口收发。具体内存地址的分配方法如下表所示：


| 虚地址区间 | 说明 |
| --- | --- |
| 0x80000000-0x800FFFFF | 监控程序代码 |
| 0x80100000-0x803FFFFF | 用户代码空间 |
| 0x80400000-0x807EFFFF | 用户数据空间 |
| 0x807F0000-0x807FFFFF | 监控程序数据 |
| 0xBFE001E0-0xBFE001E8 | 串口数据及状态 |

串口控制器访问的代码位于 `kern/utils.S`，其寄存器定义与 UART 16550 一致。串口的物理地址为 0x1FE001E0。

Kernel 的入口地址为 0x80000000，对应汇编代码 `kern/init.S` 中的 `START:` 标签。在完成必要的初始化流程后，Kernel 输出版本信息，随后进入 shell 线程，与用户交互。shell 线程会等待串口输入，执行输入的命令，并通过串口返回结果，如此往复运行。

当收到启动用户程序的命令后，用户线程代替 shell 线程的活动。用户程序的寄存器，保存在从 0x807F0000 到 0x807F007B 的连续 124 字节中，依次对应 \$r1 到 \$r31 用户寄存器，每次启动用户程序时从上述地址装载寄存器值，用户程序运行结束后保存到上述地址。

因为监控程序会把指令写入内存，然后跳转到内存上执行指令，所以如果实现了指令和数据缓存，需要添加指令来保证指令缓存可以得到内存中新写入的指令。具体地，有两种方法：

1. 使用 `ibar` 指令，可以在编译时指定 `EN_IBAR=y` 来打开。
2. 使用 `cacop` 指令，可以在编译时指定 `EN_CACOP=y` 来打开。

### 进阶一：中断和异常支持

作为扩展功能之一，Kernel 支持中断方式的 I/O，和 Syscall 功能。要启用这一功能，编译时的命令变为：

`make EN_INT=y`

这一编译选项，会使得代码编译时增加宏定义 `ENABLE_INT`，从而使能中断相关的代码。

为支持中断和异常，CPU 要额外实现以下指令

1. `csrrd`
1. `csrwr`
1. `ertn`
1. `syscall`

此外还需要实现 CSR 寄存器的部分字段：

1. CSR.EENTRY：异常入口地址
2. CSR.ESTAT：Ecode（异常类型）
3. CSR.ERA：异常返回地址
4. CSR.ECFG：LIE（使能串口中断，LIE[3]，复位值为 0）
5. CSR.PRMD：PIE
6. CSR.CRMD：IE（全局中断使能，复位值为 0）
7. CSR.SAVE0

CSR 寄存器字段功能定义参见 LoongArch 32 Reduced 特权态规范（在参考文献中）。

进阶一不涉及特权态切换，所有操作都在 PLV0 上进行。

监控程序实现了简单的线程调度，系统中只有两个线程：

1. thread0：idle，响应串口中断（CSR.CRMD.IE=1，CSR.ECFG.LIE[3]=1）
2. thread1：user/shell，不响应中断（CSR.CRMD.IE=0，CSR.ECFG.LIE[3]=0）

启动时，监控程序会运行 thread1。thread1 会尝试从串口读取数据，如果发现没有数据可以读取，就会调用 wait 系统调用，此时监控程序会调度到 thread0。thread0 打开了串口中断，因此当串口上有数据可以读取的时候，监控程序会响应中断，调度到 thread1，thread1 就可以从串口读取数据。

监控程序对于异常、中断的使用方式如下：

- 异常入口地址设置为 EXCEPTIONHANDLER，只考虑两种异常：
	- 串口硬件中断：中断号为 3，目的是为了唤醒 thread1(user/shell) 线程。具体地，它仅在 thread0(idle) 线程中打开。
	- 系统调用：支持两个系统调用：wait 和 putc。当 shell 线程调用 wait 系统调用时，CPU 控制权转交给 thread0(idle) 线程。当 shell 线程调用 putc 系统调用时，会向串口发送 a0 寄存器的低八位。
- 异常帧保存 31 个通用寄存器及 CSR.ECFG、CSR.ERA 和 CSR.PRMD 三个 CSR。
- 禁止发生嵌套异常。
- 当发生不能处理的异常时，出现严重错误，终止当前任务，自行重启。并且发送错误信号 0x80 提醒 Term。

### 进阶二：TLB 支持

在支持异常处理的基础上，可以进一步使能 TLB 支持，从而实现虚实地址映射。要启用这一功能，编译时的命令变为：

`make EN_INT=y EN_TLB=y`

CPU 要额外实现以下指令

1. `tlbrd`
1. `tlbfill`
1. `invtlb`

此外还需要实现 CSR：

1. CSR.TLBRENTRY
2. CSR.SAVE0
3. CSR.SAVE1
4. CSR.BADV
5. CSR.TLBELO0
6. CSR.TLBELO1

以及 TLB Refill 异常，其中 Refill 异常入口地址为 TLBREFILL，与其它异常的入口地址不同。

为了简化，虽然 TLB 可以支持灵活的虚实地址转换，监控程序只实现了简单的线性映射：

- va[0x00000000, 0x002FFFFF] = pa[0x00100000, 0x003FFFFF] 对应用户代码空间
- va[0x7FC10000, 0x7FFFFFFF] = pa[0x00400000, 0x007EFFFF] 对应用户数据空间

监控程序为这两个区间内的每个页保存了一个对应的 EntryLo 项，并连续地保存在 PTECODE 和 PTESTACK 中。在处理 TLB Refill 异常的时候，监控程序会找到对应的 EntryLo0 和 EntryLo1 取值，然后用 tlbfill 指令写入随机 TLB 表项。

此时用户栈的地址初始化为 0x80000000。

虽然使用了 TLB，但是没有进行特权态的切换。
 
## Term

Term 程序运行在实验者的电脑上，提供监控程序和人交互的界面。Term 支持7种命令，它们分别是

- R：按照\$r1至\$r31的顺序返回用户程序寄存器值。
- D：显示从指定地址开始的一段内存区域中的数据。
- A：用户输入汇编指令或者数据，并放置到指定地址上。输入行只有数值时视为数据，否则为指令。
- F：从文件读入汇编指令或者数据，并放置到指定地址上，格式与 A 命令相同。
- U：从指定地址读取一定长度的数据，并显示反汇编结果。
- G：执行指定地址的用户程序。
- T：查看指定的 TLB 条目。本功能仅在 Kernel 支持 TLB 时有效。
- Q：退出 Term

利用这些命令，实验者可以输入一段汇编程序，检查数据是否正确写入，并让程序在处理器上运行验证。

Term 程序位于 `term` 文件夹中，可执行文件为 `term.py`。对于本地的 Thinpad，运行程序时用 -s 选项指定串口。例如：

`python term.py -s COM3` 或者 `python term.py -s /dev/ttyACM0`（串口名称根据实际情况修改）

连接远程实验平台的 Thinpad，或者 QEMU 模拟器时，使用 -t 选项指定 IP 和端口。例如：

`python term.py -t 127.0.0.1:6666`

### 测试程序

监控程序附带了几个测试程序，代码见 `kern/test.S`。我们可以通过命令

`make show-utest`

来查看测试程序入口地址。记下这些地址，并在 Term 中使用 G 命令运行它们。

### 用户程序编写

根据监控程序设计，用户程序的代码区为 0x80100000-0x803FFFFF，实验时需要把用户程序写入这一区域。用户程序的最后需要以 `jr $r1` 结束，从而保证正确返回监控程序。

在输入用户程序的过程中，既可以用汇编指令，也可以直接写 16 进制的数据（机器码）。空行表示输入结束。

以下是一次输入用户程序并运行的过程演示：

	MONITOR for LA32R - initialized.
	>> a
	>>addr: 0x80100000
	one instruction per line, empty line to end.
	[0x80100000] ori $r4,$r0,5
	[0x80100004] xor $r12,$r12,$r12
	[0x80100008] xor $r13,$r13,$r13
	[0x8010000c] loop:
	[0x8010000c] add.w $r13,$r13,$r12
	[0x80100010] addi.w $r12,$r12,1
	[0x80100014] bne $r4,$r12,loop
	[0x80100018] jr $r1
	[0x8010001c] 
	>> u
	>>addr: 0x80100000
	>>num: 64
	0x80100000: ori $r4,$r0,0x5
	0x80100004: xor $r12,$r12,$r12
	0x80100008: xor $r13,$r13,$r13
	0x8010000c: add.w       $r13,$r13,$r12
	0x80100010: addi.w      $r12,$r12,1(0x1)
	0x80100014: bne $r4,$r12,-8(0x3fff8) # 0x8010000c
	0x80100018: jirl        $r0,$r1,0
	0x8010001c: 0x00000000
	0x80100020: 0x00000000
	0x80100024: 0x00000000
	0x80100028: 0x00000000
	0x8010002c: 0x00000000
	0x80100030: 0x00000000
	0x80100034: 0x00000000
	0x80100038: 0x00000000
	0x8010003c: 0x00000000
	>> g
	>>addr: 0x80100000

	elapsed time: 0.000s
	>> r
	R1 (ra)    = 0x807f0000
	R2 (tp)    = 0x00000000
	R3 (sp)    = 0x807f0000
	R4 (a0)    = 0x00000005
	R5 (a1)    = 0x00000000
	R6 (a2)    = 0x00000000
	R7 (a3)    = 0x00000000
	R8 (a4)    = 0x00000000
	R9 (a5)    = 0x00000000
	R10(a6)    = 0x00000000
	R11(a7)    = 0x00000000
	R12(t0)    = 0x00000005
	R13(t1)    = 0x0000000a
	R14(t2)    = 0x00000000
	R15(t3)    = 0x00000000
	R16(t4)    = 0x00000000
	R17(t5)    = 0x00000000
	R18(t6)    = 0x00000000
	R19(t7)    = 0x00000000
	R20(t8)    = 0x00000000
	R21(x)     = 0x00000000
	R22(fp)    = 0x807f0000
	R23(s0)    = 0x00000000
	R24(s1)    = 0x00000000
	R25(s2)    = 0x00000000
	R26(s3)    = 0x00000000
	R27(s4)    = 0x00000000
	R28(s5)    = 0x00000000
	R29(s6)    = 0x00000000
	R30(s7)    = 0x00000000
	R31(s8)    = 0x00000000
	>> q


当处理器和 Kernel 支持异常功能时（即上文所述 `EN_INT=y`），用户还可以用 Syscall 的方式打印字符。打印字符的系统调用号为 30。使用时，用户把调用号保存在 a7($r11) 寄存器，打印字符参数保存在 a0($r4) 寄存器，并执行 syscall 指令，a0($r4) 寄存器的低八位将作为字符打印。例如：
	
	li.w $r11, 30            # 系统调用号
	li.w $r4, 0x4F           # 'O'
	syscall 0

	li.w $r4, 0x4B           # 'K'
	syscall 0
	jr $r1

## 参考文献

- [LoongArch 32 Reduced 指令集标准](https://www.loongson.cn/uploads/images/2023041918122813624.%E9%BE%99%E8%8A%AF%E6%9E%B6%E6%9E%8432%E4%BD%8D%E7%B2%BE%E7%AE%80%E7%89%88%E5%8F%82%E8%80%83%E6%89%8B%E5%86%8C_r1p03.pdf)

## 项目作者

- 初始版本：韦毅龙，李成杰，孟子焯
- 后续维护：张宇翔，董豪宇
- 移植到 LoongArch 32 Reduced：陈嘉杰
