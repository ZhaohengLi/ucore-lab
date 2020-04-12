# 操作系统第四次实验报告
李曌珩 2017050025 计74

## 练习1
### 设计实现过程
需要初始化的变量作用和初始化方法如下：

- state: 进程状态，初始化为PROC_UNINIT；
- parent: 父进程，初始化为NULL；
- mm: 虚拟内存管理器，初始化为NULL；
- pid: 进程pid，初始化为-1，防止与idle混淆；
- need_resched: 是否需要调度，不需要；
- runs: 进程调度运行次数，初始化为0；
- kstack: 内核栈的虚拟地址（`0xC`开头），初始化为NULL；
- context: 程序运行上下文，初始化为0；
- cr3: 页表起始地址（物理地址），初始化为启动时使用的页表物理地址；
- flags: 进程标志，本Lab无用；
- name: 进程名字，需要清空；
- tf: 中断状态，设为NULL；

相关代码如下：
```c
if (proc != NULL) {
    //LAB4:EXERCISE1 2017050025
    proc->state = PROC_UNINIT;
    proc->parent = NULL;
    proc->mm = NULL;
    proc->pid = -1;
    proc->need_resched = 0;
    proc->runs = 0;
    proc->kstack = 0;
    memset(&(proc->context), 0, sizeof(struct context));
    proc->cr3 = boot_cr3;
    proc->flags = 0;
    proc->tf = NULL;
    memset(proc->name, 0, PROC_NAME_LEN + 1);
}
```
### 回答问题
#### 请说明 proc_struct 中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？

`context`变量用于进程切换，其存储了当前CPU的上下文，包括但不限于寄存器和程序指针，遇到进程切换的时候调用`switch_to`时用到，该变量保证了程序间的独立运行。

`*tf`变量保存了中断发生时进程在被打断之前的状态，保存了进入中断处理历程时用户态的上下文，用于内核态和用户态的转换。

总的来说，`context`变量用于进程切换，`*tf`变量用于中断状态保存。

## 练习2
### 设计实现过程

1. 分配进程控制块；
2. 分配内核栈空间；
3. 拷贝mm_struct，建立新进程的地址映射关系；
4. 拷贝父进程的trapframe，并为子进程设置返回值为0；
5. 需要屏蔽中断，并建立新的哈希链表；
6. 唤醒进程；
7. 父进程返回子进程的pid。

主要代码如下：
```c
if ((proc = alloc_proc()) == NULL) goto fork_out;
if (setup_kstack(proc) != 0) goto bad_fork_cleanup_proc;
if (copy_mm(0, proc) != 0) goto bad_fork_cleanup_kstack;
copy_thread(proc, stack, tf);
bool intr_flag;
local_intr_save(intr_flag);
{
    proc->pid = get_pid();
    hash_proc(proc);
    list_add(&proc_list, &(proc->list_link));
    ++nr_process;
}
local_intr_restore(intr_flag);
wakeup_proc(proc);
ret = proc->pid;
```

### 回答问题
#### 请说明 ucore 是否做到给每个新 fork 的线程一个唯一的 id？请说明你的分析和理由。
是的。

fork过程中分配pid时是不允许中断的，且在`get_pid`函数中进行了相应的处理，不会分配当前未被销毁且已经分配过进程的`pid`。

## 练习3
### 分析

`proc_run`代码与分析如下：

```c
void proc_run(struct proc_struct *proc) {
    if (proc != current) { // 当需要切换运行的目标进程不是当前进程时
        bool intr_flag; // 为屏蔽中断设置的标识位
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag); // 屏蔽中断来确保当前为原子操作
        {
            current = proc; // 切换到目标进程
            load_esp0(next->kstack + KSTACKSIZE); // 指向新进程的堆栈起始地址
            lcr3(next->cr3); // 修改页目录表起始地址
            switch_to(&(prev->context), &(next->context)); // 切换目标进程上下文
        }
        local_intr_restore(intr_flag); // 恢复中断
    }
}
```
它调用了`switch_to`，函数是由汇编代码编写的，功能十分清楚，保存之前进程的数据和恢复目标进程的数据。
```
.text
.globl switch_to
switch_to:                      # switch_to(from, to)

    # 保存之前进程的数据
    movl 4(%esp), %eax          # eax points to from
    popl 0(%eax)                # save eip !popl
    movl %esp, 4(%eax)          # save esp::context of from
    movl %ebx, 8(%eax)          # save ebx::context of from
    movl %ecx, 12(%eax)         # save ecx::context of from
    movl %edx, 16(%eax)         # save edx::context of from
    movl %esi, 20(%eax)         # save esi::context of from
    movl %edi, 24(%eax)         # save edi::context of from
    movl %ebp, 28(%eax)         # save ebp::context of from

    # 恢复目标进程的数据
    movl 4(%esp), %eax          # not 8(%esp): popped return address already
                                # eax now points to to
    movl 28(%eax), %ebp         # restore ebp::context of to
    movl 24(%eax), %edi         # restore edi::context of to
    movl 20(%eax), %esi         # restore esi::context of to
    movl 16(%eax), %edx         # restore edx::context of to
    movl 12(%eax), %ecx         # restore ecx::context of to
    movl 8(%eax), %ebx          # restore ebx::context of to
    movl 4(%eax), %esp          # restore esp::context of to

    pushl 0(%eax)               # push eip

    ret
```

### 回答问题
#### 在本实验的执行过程中，创建且运行了几个内核线程？
创建了idle(pid=0)和init(pid=1)两个内核线程。

#### 语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?
local_intr_save 和 local_intr_restore 两个函数用于关闭/开启中断，保证进程切换时context的切换过程不会被其他中断打断，确保其是一个原子操作，从而避免严重的错误。

## 总结

#### 本实验中重要的知识点，以及与对应的OS原理中的知识点

1. 进程状态模型
2. 内核线程的控制
3. proc和context的理解

#### 本实验中没有对应的

1. 用户进程
2. 挂起的进程