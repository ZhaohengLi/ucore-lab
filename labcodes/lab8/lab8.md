# 操作系统第六次实验报告
李曌珩 2017050025 计74

## 练习1
#### 文件读写操作的过程分析

```c
if (offset % SFS_BLKSIZE) {
    size = nblks ? SFS_BLKSIZE - offset % SFS_BLKSIZE : (endpos - offset);
    if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) goto out;
    if ((ret = sfs_buf_op(sfs, buf, size, ino, offset % SFS_BLKSIZE)) != 0) goto out;
    alen += size;
    if (nblks == 0) goto out;
    buf += size, ++blkno, --nblks;
}
while (nblks > 0) {
    if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) goto out;
    if ((ret = sfs_block_op(sfs, buf, ino, 1)) != 0) goto out;
    alen += SFS_BLKSIZE, buf += SFS_BLKSIZE, ++blkno, --nblks;
}
if (endpos % SFS_BLKSIZE) {
    if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0) goto out;
    if ((ret = sfs_buf_op(sfs, buf, endpos % SFS_BLKSIZE, ino, 0)) != 0) goto out;
    alen += endpos % SFS_BLKSIZE;
}
```

`sfs_io_nolock`中的部分代码如上。
该函数主要调用的是以下两个函数，对给定一个文件的`inode`以及需要读写的偏移量和大小，转换成数据块级别，进行读写操作。

1. `sfs_bmap_load_nolock`: 该函数能够将文件数据块的便宜转换成硬盘空间块的数据块编号；
2. `sfs_block_op` / `sfs_buf_op`：该函数接受硬盘空间数据块编号，并进行读写操作；

另外要注意当读取的文件大小和偏移并非是和数据块一一对齐的时候我们需要对其进行特殊处理，需要判断偏移的开头未对齐部分和结尾未对齐部分专门进行处理，而中间部分则调用对齐的`block`级别操作进行处理。


#### 实现“UNIX 的 PIPE 机制”的概要设方案

主要思想为将管道文件映射到一块内存区域，每次在文件系统清理的时候就将管道清空。

为了创建管道，与UNIX系统相同，需要增加相关的系统调用`write`和`read`接口进行读和写操作了。

在uCore中增设标记位来确定当前读写的是否为管道文件。同时使用互斥锁来进行同步互斥的控制（“生产者-消费者”）。

## 练习2
#### 完成基于文件系统的执行程序机制的实现

主要修改如下：

1. 读取ELF文件不再从内存中读取，而是通过已经实现好的文件系统的`read`操作进行硬盘文件读取。

通过调用`load_icode_read`函数完成，在文件系统之上调用了`read`和`seek`函数，基于SFS对文件进行读取和寻址操作，在实现的时候，需要依次将ELF头，`program header`以及真正的各个代码段数据段读入内存。

2. 加入了任意大小参数`argc`和`argv`的功能，使得应用程序能够接受命令行参数输入。

将相关的字符串拷贝到用户栈的顶端，并将最开始的栈指针放在`argc`的起始地址即可。


#### 设计实现基于“UNIX 的硬链接和软链接机制”的概要设方案

硬链接机制的创建： 将目录项的名字设定为传入参数，目录项的inode号设置为目标文件的inode号，且使目标inode的引用计数增加。

硬链接机制的删除：减少目标inode的引用计数。若减为0，清除目标inode及其数据块。

软链接机制的创建：将inode的类型设置为符号链接，文件内容（数据）设置为目标路径字符串。

软链接机制的删除：不需要额外的操作。

## 总结

#### 本实验中重要的知识点，以及与对应的OS原理中的知识点

1. 管道
2. 虚拟文件系统框架
3. 文件描述符
4. 目录
5. inode、打开的文件等结构体
6. inode缓存
7. 简单文件系统
8. ucore特定的文件系统架构

#### 本实验中没有对应的

1. 其他进程间通信机制，例如信号、消息队列和共享内存
2. RAID
3. 磁盘调度算法
4. 磁盘缓存
5. IO