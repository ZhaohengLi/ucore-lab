# 操作系统第一次实验报告
李曌珩 2017050025 计74

##练习1：理解通过 make 生成执行文件的过程。

###操作系统镜像文件 ucore.img 是如何一步一步生成的？

生成 ucore.img 的流程：<br>
```
为了生成 ucore.img 需要先生成 kernel 和 bootblock.o
$(UCOREIMG): $(kernel) $(bootblock)

    为了生成 bootblock 需要先生成 bootasm.o bootmain.o sign
    $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
        由 bootasm.S 生成 bootasm.o (1)
        由 bootmian.c 生成 bootmain.o (2)
        生成 sign (3)
        由 bootasm.o bootmain.o sign 生成 bootblock.o (4)
        由 bootblock.o 生成 bootblock

    为了生成kernel，首先需要 kernel.ld init.o readline.o stdio.o kdebug.o kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o trapentry.o vectors.o pmm.o  printfmt.o string.o
        仅以 intr.o 为例 由 intr.c 生成 intr.o (5)
        ......
        由上述文件生成 kernel

    
    由以上文件生成 ucore.img
    生成一个有10000个块的文件，每个块默认512字节，用0填充 (6)
    把bootblock中的内容写到第一个块 (7)
    从第二个块开始写kernel中的内容 (8)

```

在以上流程中实际使用的命令以及参数：<br>
(1) 由 bootasm.S 生成 bootasm.o 的命令以及关键参数：
```
gcc -Iboot/ -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o

-fno-builtin 不进行 builtin 函数优化
-ggdb 生成可供gdb调试的信息
-m32 生成32位代码
-nostdinc 不使用标准库代码 操作系统使用的服务均由自己实现
-fno-stack-protector  不生成用于检测缓冲区溢出的代码
-Os  为减小代码大小而进行优化
```

(2) 由 bootmian.c 生成 bootmain.o 的命令以及关键参数：
```
gcc -Iboot/ -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o

未出现新的参数
```

(3) 生成 sign
```
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign

未出现新的参数
```

(4) 由 bootasm.o bootmain.o sign 生成 bootblock.o 的命令以及关键参数
```
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o

-m 指定模拟器类型
-N 设置代码段和数据段均可读写
-e 指定代码入口
-Ttext 指定代码段开始位置
```

(5) 由 bootblock.o 生成 bootblock 的命令以及相关参数
```
gcc -Ikern/driver/ -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/driver/intr.c -o obj/kern/driver/intr.o

未出现新参数
```

(6) 生成一个有10000个块的文件，每个块默认512字节，用0填充
```
dd if=/dev/zero of=bin/ucore.img count=10000

未出现新参数
```

(7) 把bootblock中的内容写到第一个块
```
dd if=bin/bootblock of=bin/ucore.img conv=notrunc

未出现新参数
```

(8) 从第二个块开始写kernel中的内容
```
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc

未出现新参数
```

### 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

扇区大小为512字节，并且最后两位分别为 `0x55` 和 `0xAA`。

##练习2：使用 qemu 执行并调试 lab1 中的软件。

在文件 `tools/gdbinit` 中删除了 `continue` 并运行 `make debug` 命令后，可以看到程序从 `0x0000fff0` 处开始运行，过程如下：
```
0x0000fff0 in ?? ()
Breakpoint 1 at 0x100000: file kern/init/init.c, line 17.

(gdb) x/i 0xfffffff0
   0xfffffff0:  ljmp   $0x3630,$0xf000e05b

(gdb) si
0x0000e05b in ?? ()

(gdb) si   
0x0000e062 in ?? ()

(gdb) b *0x00007c00
Breakpoint 2 at 0x7c00

(gdb) c
Continuing.
Breakpoint 2, 0x00007c00 in ?? ()

(gdb) x/16i 0x00007c00
=> 0x7c00:      cli
   0x7c01:      cld
   0x7c02:      xor    %eax,%eax
   0x7c04:      mov    %eax,%ds
   0x7c06:      mov    %eax,%es
   0x7c08:      mov    %eax,%ss
   0x7c0a:      in     $0x64,%al
   0x7c0c:      test   $0x2,%al
   0x7c0e:      jne    0x7c0a
   0x7c10:      mov    $0xd1,%al
   0x7c12:      out    %al,$0x64
   0x7c14:      in     $0x64,%al
   0x7c16:      test   $0x2,%al
   0x7c18:      jne    0x7c14
   0x7c1a:      mov    $0xdf,%al
   0x7c1c:      out    %al,$0x60

```
可以看到，程序成功跳转到 `0x0000e05b` 开始加载 bootloader 中的内容，并从 `0x00007c00` 处开始运行 `bootblock.asm` 中的 `.globl start` 代码。

##练习3：分析 bootloader 进入保护模式的过程。
分析过程如下：
```
.globl start
start:

    # 首先在实模式下执行16位汇编代码 初始化环境 将段寄存器置零
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment

    # 下面进行开启A20的工作 通过将键盘控制器上的A20线置于高电位
    # 全部32条地址线可用 可以访问4G的内存空间
seta20.1:
    inb $0x64, %al                                  # 等待8042键盘控制器不忙
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 发送写指令
    outb %al, $0x64                                 
    to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # 等待8042键盘控制器不忙
    input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 打开A20
    outb %al, $0x60                                 
    set P2's A20 bit(the 1 bit) to 1

    # 加载引导区内的GDT表以及其描述符
    lgdt gdtdesc

    # 将cr0寄存器中的PE位置1 开启保护模式
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0

    # 跳转到下一步代码地址 并开启32位模式
    ljmp $PROT_MODE_CSEG, $protcseg

.code32                                             # Assemble for 32-bit mode
protcseg:
    
    # 设置段寄存器
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    # 建立堆栈
    movl $0x0, %ebp
    movl $start, %esp

    # 进入C代码
    call bootmain

```

##练习4：分析 bootloader 加载 ELF 格式的 OS 的过程。

如何加载 ELF 格式的 OS ：
```c
void bootmain(void) {
    // 首先读取ELF的头部
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // 判断是否为合法的ELF
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // 按照ELF头部的描述 将ELF文件中的数据载入到内存中
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // 根据ELF头部的描述 找到内核的入口
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);
    while (1);
}
```

如何读取硬盘扇区：

1. 等待 IO 直到不忙
2. 依次向 IO 地址 `0x1f2 - 0x1f7` 写入读取参数
3. 等待 IO 直到不忙
4. 使用 `insl` 命令读取硬盘数据至指定内存区域中

##练习5：实现函数调用堆栈跟踪函数。
代码实现为：
```c
void
print_stackframe(void) {
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (uint32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
    uint32_t ebp, eip, i;
    eip = read_eip();
    ebp = read_ebp();
    for (i = 0; i < STACKFRAME_DEPTH; ++i) {
        cprintf("ebp:0x%08x eip:0x%08x args:0x%08x 0x%08x 0x%08x 0x%08x", ebp, eip, *((uintptr_t*)ebp+2), *((uintptr_t*)ebp+3), *((uintptr_t*)ebp+4), *((uintptr_t*)ebp+5));
        cprintf("\n");
        print_debuginfo(eip - 1);
        eip = *((uintptr_t*)ebp+1);
        ebp = *((uintptr_t*)ebp);
        if (ebp == 0) break;
    }
}
```
输出为：
```
(THU.CST) os is loading ...

Special kernel symbols:
  entry  0x00100000 (phys)
  etext  0x00103290 (phys)
  edata  0x0010ea16 (phys)
  end    0x0010fd20 (phys)
Kernel executable memory footprint: 64KB
ebp:0x00007b28 eip:0x00100a59 args:0x00010094 0x00010094 0x00007b58 0x00100092
    kern/debug/kdebug.c:0: print_stackframe+11
ebp:0x00007b38 eip:0x00100d36 args:0x00000000 0x00000000 0x00000000 0x00007ba8
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b58 eip:0x00100092 args:0x00000000 0x00007b80 0xffff0000 0x00007b84
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b78 eip:0x001000bc args:0x00000000 0xffff0000 0x00007ba4 0x00000029
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b98 eip:0x001000db args:0x00000000 0x00100000 0xffff0000 0x0000001d
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007bb8 eip:0x00100101 args:0x001032bc 0x001032a0 0x0000130a 0x00000000
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007be8 eip:0x00100055 args:0x00000000 0x00000000 0x00000000 0x00007c4f
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d72 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d71 --
++ setup timer interrupts
```

最后一行对应的是第一个入栈的函数，即 `bootmain` 。`0x00007bf8` 为 `bootmain` 的栈帧基址

`0x00007d6e` 是 ELF 跳转语句的后一条语句的地址，也是 `bad` 标签地址。<br>

由于 `bootmain` 并没有参数，因此 args 为调用该函数之前栈上的值，即代码的指令编码。


##练习6：

1：中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

每一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移，两者联合便是中断处理程序的入口地址。

2：请编程完善 kern/trap/trap.c 中对中断向量表进行初始化的函数 idt_init。在 idt_init 函数中，依次对所有中断入口进行初始化。使用 mmu.h 中的 SETGATE 宏，填充 idt 数组内容。每个中断的入口由 tools/vectors.c 生成，使用 trap.c 中声明的 vectors 数组即可。
```c
/* idt_init - initialize IDT to each of the entry points in kern/trap/vectors.S */
void idt_init(void) {
     /* LAB1 YOUR CODE : STEP 2 */
     /* (1) Where are the entry addrs of each Interrupt Service Routine (ISR)?
      *     All ISR's entry addrs are stored in __vectors. where is uintptr_t __vectors[] ?
      *     __vectors[] is in kern/trap/vector.S which is produced by tools/vector.c
      *     (try "make" command in lab1, then you will find vector.S in kern/trap DIR)
      *     You can use  "extern uintptr_t __vectors[];" to define this extern variable which will be used later.
      * (2) Now you should setup the entries of ISR in Interrupt Description Table (IDT).
      *     Can you see idt[256] in this file? Yes, it's IDT! you can use SETGATE macro to setup each item of IDT
      * (3) After setup the contents of IDT, you will let CPU know where is the IDT by using 'lidt' instruction.
      *     You don't know the meaning of this instruction? just google it! and check the libs/x86.h to know more.
      *     Notice: the argument of lidt is idt_pd. try to find it!
      */
    extern uintptr_t __vectors[];
    int i;
    for (i = 0; i < 256; ++i)
        SETGATE(idt[i], 0, KERNEL_CS, __vectors[i], DPL_KERNEL); // 填充数组
    SETGATE(idt[T_SYSCALL], 1, KERNEL_CS, __vectors[T_SYSCALL], DPL_USER);
    lidt(&idt_pd);
}
```


3：请编程完善 trap.c 中的中断处理函数 trap，在对时钟中断进行处理的部分填写 trap 函数中处理时钟中断的部分，使操作系统每遇到 100 次时钟中断后，调用 print_ticks 子程序，向屏幕上打印一行文字”100 ticks”。
```c
case IRQ_OFFSET + IRQ_TIMER:
        /* LAB1 YOUR CODE : STEP 3 */
        /* handle the timer interrupt */
        /* (1) After a timer interrupt, you should record this event using a global variable (increase it), such as ticks in kern/driver/clock.c
         * (2) Every TICK_NUM cycle, you can print some info using a funciton, such as print_ticks().
         * (3) Too Simple? Yes, I think so!
         */
        ++ticks; // 添加秒数
        if (ticks % TICK_NUM == 0) print_ticks(); // 如果比对成功 打印信息
        break;
```

执行效果：
```
++ setup timer interrupts
100 ticks
100 ticks
kbd [097] a
100 ticks
qemu-system-i386: terminating on signal 2
```

## 总结

#### 本实验中重要的知识点，以及与对应的OS原理中的知识点

1. Makefile：生成镜像、bootloader的过程
2. A20
3. GDT
4. 中断描述符表
5. 中断处理过程
6. ELF格式
7. 磁盘读取
8. 函数堆栈
9. 特权转换、中断堆栈切换

#### 本实验中没有对应的

无
