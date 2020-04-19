# 操作系统第五次实验报告
李曌珩 2017050025 计74

## 练习1
### 设计实现过程
在加载程序并运行之前，需要先设置好 `proc_struct` 结构中的成员变量 `trapframe` 中的内容，包括代码段、地址段、栈地址、入口和用户信息等。具体设置参照如下列表和代码。

- `tf_cs`: 用户态的代码段寄存器，需要设置为`USER_CS`
- `tf_ds, tf_es, tf_ss`: 用户态数据段寄存器，需要设置为`USER_DS`
- `tf_esp`: 用户态的栈指针，需要设置为`0xB0000000`
- `tf_eip`: 用户态的代码指针，需要设置为用户程序的起始地址
- `tf_eflags`: 一些用户态的设置信息

相关代码如下：
```c
/* LAB5:EXERCISE1 2017050025
     * should set tf_cs,tf_ds,tf_es,tf_ss,tf_esp,tf_eip,tf_eflags
     * NOTICE: If we set trapframe correctly, then the user level process can return to USER MODE from kernel. So
     *          tf_cs should be USER_CS segment (see memlayout.h)
     *          tf_ds=tf_es=tf_ss should be USER_DS segment
     *          tf_esp should be the top addr of user stack (USTACKTOP)
     *          tf_eip should be the entry point of this binary program (elf->e_entry)
     *          tf_eflags should be set to enable computer to produce Interrupt
     */
    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP;
    tf->tf_eip = elf->e_entry;
    tf->tf_eflags = FL_IF;
```
### 回答问题
#### 当创建一个用户态进程并加载了应用程序后，CPU 是如何让这个应用程序最终在用户态执行起来的。

在`schedule`函数中，会调用函数`proc_run`：

```c
current = proc;
load_esp0(next->kstack + KSTACKSIZE);
lcr3(next->cr3);
switch_to(&(prev->context), &(next->context));
```
可以看到步骤如下：

1. 首先关闭中断，避免出现不可恢复的错误
2. 修改 `current`，令其指向该进程
3. 载入该进程内核栈的 `esp` 和页目录表起始地址 `cr3`
4. 调用 `switch_to` 函数进行上下文切换，由于在之前的 `copy_thread` 中设置了 `context` 的 `esp` 和 `eip`，`ret` 会跳转到 `forkret`

```c
// return falls through to trapret...
.globl __trapret
__trapret:
// restore registers from stack
popal

// restore %ds, %es, %fs and %gs
popl %gs
popl %fs
popl %es
popl %ds

// get rid of the trap number and error code
addl $0x8, %esp
iret

.globl forkrets
forkrets:
// set stack to this new process's trapframe
movl 4(%esp), %esp
jmp __trapret
```

5. 由汇编代码可以看到，`forkret` 跳转到 `__trapret`
6. `__trapret` 根据 `tf` 的信息恢复上下文，恢复各个段选择子，最后使用 `iret` 指令回到用户态


## 练习2
### 设计实现过程

创建子进程的函数 do_fork 在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。资源的肤质分为四部分，获取源物理页的虚拟地址，获取目的物理页的虚拟地址，拷贝内容，设置页表的映射关系。

主要代码如下：
```c
/* LAB5:EXERCISE2 2017050025
         * replicate content of page to npage, build the map of phy addr of nage with the linear addr start
         *
         * Some Useful MACROs and DEFINEs, you can use them in below implementation.
         * MACROs or Functions:
         *    page2kva(struct Page *page): return the kernel vritual addr of memory which page managed (SEE pmm.h)
         *    page_insert: build the map of phy addr of an Page with the linear addr la
         *    memcpy: typical memory copy function
         *
         * (1) find src_kvaddr: the kernel virtual address of page
         * (2) find dst_kvaddr: the kernel virtual address of npage
         * (3) memory copy from src_kvaddr to dst_kvaddr, size is PGSIZE
         * (4) build the map of phy addr of  nage with the linear addr start
         */
        void* src_kvaddr = page2kva(page);
        void* dst_kvaddr = page2kva(npage);
        memcpy(dst_kvaddr, src_kvaddr, PGSIZE);
        ret = page_insert(to, npage, start, perm);
```

### 回答问题
#### 简要说明如何设计实现“Copy on Write 机制”，给出概要设计。

1. 若实现机制，需要修改 `do_pgfault` 中的相关代码。
2. 在使用memcpy拷贝页时，需要对当前操作进行判断。如果当前为读操作那么将原来的页W置为0，刷新TLB，直接返回原来页信息；如果当前是写操作那么进行拷贝操作。另外，如果某个页面引用计数为1，说明并没有其他进程共享该页面，在写操作的时候可以直接写，而不必复制。
3. 如果有某个进程尝试写一个只读页面，那么会抛出异常。此时说明页表中的权限和进程中的权限不一致，在异常处理的时候拷贝这个页面给该进程，修改页表，重新映射虚拟地址，并且W置为1。

## 练习3
### 分析

#### fork

`fork` 此操作会创建新的进程，并设置进程状态为 `UNINIT`。同时创建PCB、分配内核栈、拷贝虚拟内存映射关系、设置trapframe、维护进程块的相互关系。调用函数 `wakeup_proc` 之后进程状态为 `RUNNABLE`。

#### exec

`exec`操作会清除当前进程的内存布局 `mm`，之后调用 `load_icode` 从二进制ELF格式串中建立起新的内存布局，进程状态不会发生变化。

#### wait

`wait`操作的内容会对当前进程以及子进程产生影响。进程先会被设置成 `SLEEPING` 状态，接着一直等到有 `ZOMBIE` 状态的子进程出现，释放它的占用的资源，然后返回子进程的 `exit_code`。

#### exit

`exit`操作内容基本与`fork`操作相反，销毁相应的内存空间、维护进程链表关系、并将进程转换成 `ZOMBIE`，如果有父进程并且父进程等待子进程退出，则将父进程的状态从 `SLEEPING` 转变为 `RUNNABLE`，随后将该进程所有子进程的父进程设置为 `initproc`，之后唤醒 `initproc`。


### 回答问题
#### 请分析 fork/exec/wait/exit 在实现中是如何影响进程的执行状态的？

通过修改`proc->state`来改变进程的状态，进一步通过调度器的策略选择来影响程序执行。

#### 请给出 ucore 中一个用户态进程的执行状态生命周期图（包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）。

```
  alloc_proc                                 RUNNING
      +                                   +--<----<--+
      +                                   + proc_run +
      V                                   +-->---->--+
PROC_UNINIT -- proc_init/wakeup_proc --> PROC_RUNNABLE -- try_free_pages/do_wait/do_sleep --> PROC_SLEEPING --
                                           A      +                                                           +
                                           |      +--- do_exit --> PROC_ZOMBIE                                +
                                           +                                                                  +
                                           -----------------------wakeup_proc----------------------------------
```

|进程状态         |     意义         | 转换到该状态的函数调用 |
| ---------------|-----------------|------------------|
|PROC_UNINIT     |   未初始化        | alloc_proc |
|PROC_SLEEPING   |   睡眠状态       | try_free_pages, do_wait, do_sleep |
|PROC_RUNNABLE   |   就绪或正在运行  | proc_init, wakeup_proc |
|PROC_ZOMBIE     |   僵尸状态       | do_exit |


## 总结

#### 本实验中重要的知识点，以及与对应的OS原理中的知识点

1. 进程状态模型
2. 用户进程
3. ELF格式

#### 本实验中没有对应的

1. 挂起的进程