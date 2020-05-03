# 操作系统第七次实验报告
李曌珩 2017050025 计74

## 练习1
#### 给出内核级信号量的设计描述，并说明其大致执行流程。

主要设计思想为：利用等待队列和定义的信号量来实现对某个信号等待的进程的管理、互斥与同步

设计描述如下：
```c
typedef struct {
    struct proc_struct *proc; // 等待进程的指针
    uint32_t wakeup_flags; // 进程被放入等待队列的原因标记
    wait_queue_t *wait_queue; // 指向此wait结构所属于的wait_queue
    list_entry_t wait_link; // 用来组织wait_queue中wait节点的连接
} wait_t;

typedef struct {
    list_entry_t wait_head; // wait_queue的队头
} wait_queue_t;

le2wait(le, member) // 实现wait_t中成员的指针向wait_t 指针的转化
    
typedef struct {
    int value; // 信号量的当前值
    wait_queue_t wait_queue; // 信号量对应的等待队列
} semaphore_t;
```

主要步骤为：

1. Init操作：初始化信号量，将该信号量的数值初始化为给定的值，等待队列初始化为单个元素的队列。
```c
sem_init(semaphore_t *sem, int value) {
    sem->value = value;
    wait_queue_init(&(sem->wait_queue));
}
```

2. Up操作：
    + 关中断保证互斥操作。
    + 检查等待队列是否为空。如果等待队列为空，那么信号数值加1并返回，表示该信号有富余的量；如果等待队列不为空，则选取等待队列队首的元素唤醒。
    + 恢复中断。


```c
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        } else {
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}
```

3. Down操作：
    + 关中断保证互斥操作。
   + 若信号量大于0，减少1并返回，表示信号量被进程消耗。
   + 若信号量等于0，则将当前进程放入等待队列，运行调度器以调度到其他进程。
   + 当该进程被再次调度回来时，如果等待进程的`wakeup_flags`没有改变，则信号量被该进程所消耗，从而可以将该进程从等待队列中删除。若等待进程的`wakeup_flags`改变，说明进程因为其他原因被唤醒，此时仍然需要将进程从等待队列中移除，并且返回唤醒原因。
   + 恢复中断


```c
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);

    schedule();

    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
```


#### 给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

可以使用在用户态使用生产者/消费者模式来解决进程同步相关问题。

此时在内核中设置同步锁，当某进程使用共享变量时为变量上锁，同时用户访问共享变量时通过中断来进入内核态，并在内核态判断和解决该进程对该共享变量的权限问题，当访问结束后，解锁该共享变量。


## 练习2

#### 给出内核级条件变量的设计描述，并说明其大致执行流程。

设计描述如下：
```c
typedef struct condvar{
  semaphore_t sem;        // 条件变量中的信号量，通过信号量的操作可以间接完成条件变量的操作
  int count;              // 挂在该条件变量中的进程或者线程数量
  monitor_t * owner;      // 条件变量所属的管程
} condvar_t;

typedef struct monitor{
  semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
  semaphore_t next;       // the next semaphore is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
  int next_count;         // the number of of sleeped signaling proc
  condvar_t *cv;          // the condvars in monitor
} monitor_t;
```
主要流程如下：

1. `monitor_init`函数：
    - 用于初始化。
    - 将管程的`next`的值置0，`mutex`的值置1，表示没有进程进入、也没有进程在等待。
    - 为条件变量分配内存并初始化，将`sem`信号量的值置0，表示没有进程在等待。

```c
void     
monitor_init (monitor_t * mtp, size_t num_cv) {
  int i;
  assert(num_cv>0);
  mtp->next_count = 0;
  mtp->cv = NULL;
  sem_init(&(mtp->mutex), 1); //unlocked
  sem_init(&(mtp->next), 0);
  mtp->cv =(condvar_t *) kmalloc(sizeof(condvar_t)*num_cv);
  assert(mtp->cv!=NULL);
  for(i=0; i<num_cv; i++){
      mtp->cv[i].count=0;
      sem_init(&(mtp->cv[i].sem),0);
      mtp->cv[i].owner=mtp;
  }
}
```

2. `cond_signal`函数：
    - 用于激活某一个在等待管程的进程。
    - 首先进程B判断`cv.count`,如果大于0，这表示当前有执行`cond_wait`而睡眠的进程A，因此需要唤醒等待在`cv.sem`上睡眠的进程A。由于只允许一个进程在管程中执行，所以一旦进程B唤醒了别人（进程A），那么自己就需要睡眠。故让`monitor.next_count`加一，且让自己（进程B）睡在信号量`monitor.next`上。如果睡醒了，表示进程A已经访问结束，则让`monitor.next_count`减1。


```c
void 
cond_signal (condvar_t *cvp) {
  if (cvp->count > 0) {
      ++cvp->owner->next_count;
      up(&(cvp->sem));
      down(&(cvp->owner->next));
      --cvp->owner->next_count;
  }
}
```

3. `cond_wait`函数：

* 用于等待某个条件变量。

* 如果进程A执行了`cond_wait`函数，表示此进程等待某个条件`Cond`不为真，需要睡眠。因此表示等待此条件的睡眠进程个数`cv.count`要加一。接下来会出现两种情况:

* 如果`monitor.next_count`大于0，表示有大于等于1个进程执行`cond_signal`函数且睡了，就睡在了`monitor.next`信号量上（假定这些进程挂在`monitor.next`信号量相关的等待队列Ｓ上），因此需要唤醒等待队列Ｓ中的一个进程B；然后进程A睡在`cv.sem`上。如果进程A醒了，则让`cv.count`减一，表示等待此条件变量的睡眠进程个数少了一个，可继续执行。

* 如果`monitor.next_count`如果小于等于0，表示目前没有进程执行`cond_signal`函数且睡着了，那需要唤醒的是由于互斥条件限制而无法进入管程的进程，所以要唤醒睡在`monitor.mutex`上的进程。然后进程A睡在`cv.sem`上，如果睡醒了，则让`cv.count`减一，表示等待此条件的睡眠进程个数少了一个，可继续执行。


```c
void
cond_wait (condvar_t *cvp) {
    ++cvp->count;
    if (cvp->owner->next_count > 0) {
      up(&(cvp->owner->next));
    } else {
      up(&(cvp->owner->mutex));
    }
    down(&(cvp->sem));
    --cvp->count;
}
```


#### 给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。

`cond_signal`函数和`cond_wait`函数导致的进程阻塞与唤醒都可以由信号量的相关接口进行完成，因此可以考虑设计用户态的管程机制，由语言内部维护或是用户手动维护这两个必要函数的运作，通过系统调用完成对信号量的P操作或者V操作。

#### 能否不用基于信号量机制来完成条件变量？如果不能，请给出理由，如果能，请给出设计说明和具体实现。
可以。

使用等待队列+锁机制来完成条件变量，这样可以保证某个条件变量只能有一个进程占有，其余进程都挂在等待队列上。

1. `init` 函数：初始化等待队列。
2. `signal`函数：首先关闭中断，如果等待队列不为空，那么选择等待队列中第一个进程，移出等待队列并唤醒该进程，最后打开中断。
3. `wait`函数：首先关闭中断，当前进程释放对等待队列的锁，sleep并加到等待队列尾部，最后打开中断。再次切换到该进程时，表示该进程已经被`signal`操作唤醒，从等待队列中移除，此时进程重新占有对等待队列的锁即可。



## 总结

#### 本实验中重要的知识点，以及与对应的OS原理中的知识点

1. 禁用中断的同步方法
2. 信号量
3. 条件变量、管程
4. 哲学家就餐问题

#### 本实验中没有对应的

1. 纯软件实现的同步方法
2. 读者-写者问题
3. 死锁
4. 进程间通信

