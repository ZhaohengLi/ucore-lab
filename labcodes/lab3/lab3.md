# 操作系统第二次实验报告
李曌珩 2017050025 计74

## 练习1
### 设计实现过程

翻阅代码可以看到，`do_pgfault()`在缺页的时候被调用，此时用户访问的虚拟地址在物理内存中没有映射，这种情况可能是非法访存，也有可能是合理访存。`do_pgfault()`首先对非法访存进行检查，即查看vma结构中是否有对应映射表项。

练习1负责处理合理访存的情况，这种合理访存引发的页缺失异常又可以划分为如下情况：

1. 该页被置换算法换到了swap分区中。
2. 第一次访问该页，仅在vma_struct中存在对应，而没有写入页表，分配物理页。

本练习需要处理情况2，这个时候仅需要调用pgdir_alloc_page函数即可，该函数会自动分配相关对应的物理页，如果内存不够还会调用swap_out函数将不需要的页面换出。

相关代码如下：
```c
/*LAB3 EXERCISE 1: 2017050025*/
    ptep = get_pte(mm->pgdir, addr, 1);
    if (ptep == NULL) { goto failed; }
    if (*ptep == 0) { if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) goto failed;
    } else {
        /*LAB3 EXERCISE 2: 2017050025*/
        if(swap_init_ok) {
            struct Page *page=NULL;
            swap_in(mm, addr, &page);
            page_insert(mm->pgdir, page, addr, perm);
            page->pra_vaddr = addr;
            swap_map_swappable(mm, addr, page, 1);
        } else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
   }
```
### 回答问题
#### 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中组成部分对 ucore 实现页替换算法的潜在用处。
在本次实验中，若要实现扩展时钟算法，那么就会需要使用PTE中的Access位和Dirty位进行记录该页的历史访问情况。

PDE对实现页替换算法目前没有过多用处。

项中可能用到的有关位的说明如下，更详细的位说明已经在LAB2中做过。

| 位   | 名称 | 含义 | 用法 |
| ---- | ---- | ---- | --- |
| 0     | P     | 页面是否在内存中，若此位为0则其他位可随意使用                           |存放交换分区信息，可用于页替换算法实现|
| 5     | A     | 在上次清零之后，该页是否被读过或写过  |可用于时钟替换算法和拓展时钟替换算法的实现|
| 6 | D | 在上次清零之后，该页是否被写过 |可用于拓展时钟替换算法实现|

#### 如果 ucore 的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
在正常运行的状态下，出现了访问异常，硬件会做如下处理：

1. 保存现场，存储当前的寄存器到主存储器中；
2. 设置相应的寄存器CR2记录当前出错程序的地址信息，记录页面访问异常类型；
3. 切换特权级；
4. 根据异常号读取IDT表，确定ISR的地址，判断是否有进入中断门的权限；
5. 跳转到ISR起始地址开始执行中断服务例程。

在执行缺页服务例程的时候又出现了访问异常属于较为特别的嵌套异常，除非操作系统内核出现故障，否则这种情况一般不会发生。

x86的CPU在处理该嵌套异常的时候，会类似上述操作保存现场，并进入double_fault异常而非缺页异常供操作系统开发人员捕捉错误并处理。

另外，对于Qemu来说三次出现嵌套缺页异常的话，模拟器就会出错退出。

## 练习2
### 设计实现过程

在本次练习中需要处理在访问缺页的时候发现该页被置换算法换到了swap分区中的情况，这个时候需要首先调入该缺失的页，再更新页面置换算法结构体中可以被换出的页。

要注意实现FIFO算法的时候，每次有新页到物理内存的时候，需要先将新页指针连接到FIFO中，再选择链表末端的页进行换出操作。

主要代码如下：
```c
static int
_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    list_entry_t *entry=&(page->pra_page_link);
 
    assert(entry != NULL && head != NULL);
    //record the page access situlation
    /*LAB3 EXERCISE 2: 2017050025*/ 
    //(1)link the most recent arrival page at the back of the pra_list_head qeueue.
    list_add(head, entry);
    return 0;
}


static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    assert(head != NULL);
    assert(in_tick==0);
    /* Select the victim */
    /*LAB3 EXERCISE 2: 2017050025*/ 
    //(1)  unlink the  earliest arrival page in front of pra_list_head qeueue
    struct Page* page = le2page(head->prev, pra_page_link);
    list_del(head->prev);
    //(2)  assign the value of *ptr_page to the addr of this page
    *ptr_page = page;
    return 0;
}
```

### 回答问题
如果要在 ucore 上实现"extended clock 页替换算法"请给你的设计方案，现有的 swap_manager 框架是否足以支持在 ucore 中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题：

回答：现有的 swap_manager 框架可以支持在 ucore 中实现此算法。

#### 需要被换出的页的特征是什么？
在clock链表中进行扫描的时候，如果发现某一个页表项的Access位`PTE_A`和Dirty位`PTE_D`均为0，则换出该页。

#### 在 ucore 中如何判断具有这样特征的页？
通过遍历循环链表，可以依次查找出所有能够被换出的页。

遍历时候根据时钟置换算法修改标志位：

1. `PTE_A=0,PTE_D=0`：说明该页可以被替换，clock指针跳到链表下一项，返回；
2. `PTE_A=0,PTE_D=1`：调用 `swapfs_write()` 函数，将该页面写入交换分区，之后将D修改为0，clock指针跳到链表下一项；
3. `PTE_A=1`：此时将A修改为0，clock指针跳到链表下一项。

#### 何时进行换入和换出操作？
缺页异常发生之后，需要访问的页不在物理内存中，此时进行换入操作。

物理内存已满而需要新的内存空间时，需要换出未来最不可能使用的物理页，此时进行换出操作。


## 总结
#### 本实验中重要的知识点，以及与对应的OS原理中的知识点
1. FIFO
2. 拓展时钟替换算法
3. 缺页异常

#### 本实验中没有对应的
1. 最优算法、LRU、最不常用算法
2. Belady现象
3. 全局页替换算法
4. 抖动和负载控制