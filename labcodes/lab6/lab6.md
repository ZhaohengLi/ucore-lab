# 操作系统第六次实验报告
李曌珩 2017050025 计74

## 练习1
#### 分析 sched_class 中各个函数指针的用法，并结合 Round Robin 调度算法描 ucore 的调度执行过程。

`sched_class` 一共6个成员，如下：

1. `name`：名字。
2. `init`：初始化调度器的相关参数，包括数据结构、计数器。
3. `enqueue`：添加一个新的进程到等待队列中，并且计数器+1。
4. `dequeue`：从等待队列中删除一个指定的进程，并且计数器-1。
5. `pick_next`：从等待队列中选择下一个被调度的进程。
6. `proc_tick`：响应时钟中断，更新进程时间片信息。


在ucore中，每次时钟中断的时候都会执行`proc_tick`函数，对于当前执行的进程的`time_slice`进行减一操作，当进程的时间片用完之后就需要运行调度器。
具体如下：

1. 在某次时钟中断发现进程的时间片用完时，会将当前进程的 `need_resched` 设置为1。
2. 从 `trap` 函数实现可知，在 `trap_dispatch` 之后，由于 `need_resched = true`，因此进入 `schedule` 进入调度器。
3. 将当前进程塞到等待队列中，并且取出等待队列中优先级最高的进程。
4. 设置取出的进程为当前进程，如果取出的进程为空则设置为 `idleproc`。
5. 最后内核调用 `proc_run` 运行当前被选中进程。


#### 说明如何设计实现”多级反馈队列调度算法“，给出概要设计。

多级反馈队列调度算法设计如下：

1. `init`：在此初始化所有优先级队列，并且把所有进程优先级设置为最高。
2. `enqueue`：如果当前进程时间片为0，不改变优先级并放入对应的优先级队列中，如果当前进程时间片不为0，那么降低优先级并放入对应队列中，将该进程时间片设置为对应优先级拥有的时间片大小。
3. `dequeue`：从对应优先级队列中取出进程。
4. `pick_next`：根据优先级算法，判断是否要转移优先级队列，取出当前维护的队列中最靠前的进程。
5. `proc_tick`：响应时钟中断，更新进程时间片信息。


## 练习2
### 设计实现过程

1. 初始化 `run_pool` 设置为空。
2. 由于 `max_stride - min_stride <= PASS_MAX <= BIG_STRIDE`，并且需要保证两个进程 `stride` 差值在32位有符号整数表示范围内，所以 `max_stride - min_stride <= 0x7fffffff`。
3. 修改 `enqueue`函数，使用左偏树的插入接口。
4. 修改 `dequeue`函数，使用左偏树的删除接口。
4. 修改 `pick_next`函数，选择进程的时候，选位于堆顶的元素，并且计算stride。


## 总结

#### 本实验中重要的知识点，以及与对应的OS原理中的知识点

1. 时间片轮转算法
2. Stride 调度算法
3. 调度框架

#### 本实验中没有对应的

1. 其他调度算法
2. 实时调度
3. 多处理器调度
4. 优先级反置