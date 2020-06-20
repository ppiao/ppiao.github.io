---
layout: post
title: 学习Linux源码fs/pipe.c
---

# TL;DR

1. 概要
    * fs/pipe.c是一个内核模块，初始化了pipe（伪）文件系统
    * pipe系统调用在pipe文件系统中新建一个pipe文件
    * pipe文件包括struct inode和struct pipe_inode_info数据结构

2. pipefs初始化
    * 内核调用init_pipe_fs初始化pipefs：[register_filesystem][register-filesystem]

3. pipefs创建文件
    * pipe系统调用内核入口：[SYSCALL_DEFINE1(pipe)][pipe-syscall-enter]
    * 创建PIPE文件的函数：[create_pipe_files][create-pipe-files]
    * 在内存创建inode：[alloc_inode][alloc-inode]（[pipe_fs_type][pipefs-type]）
    * 初始化pipe文件：[alloc_pipe_info][alloc-pipe-info]
    * 绑定pipe文件pipefifo_fops：[pipefifo_fops][pipefifo-fops]
    * 打开新创建的pipe文件：[create_pipe_files][open-pipe-file]
    * 返回打开的文件给用户：[copy_to_user][copy-file-to-user]

3. pipe write
    * write系统调用入口：[SYSCALL_DEFINE3(write)][pipe-write-enter]
    * 根据fd获取到对应的数据结构struct fd和struct file：[ksys_write][ksys-write]
    * VFS write函数调用对应文件系统的write函数：[__vfs_write][vfs-write]
    * pipefs的write函数把用户数据写到内核内存中：[pipe_write][pipe-write]

4. 总结
    * 管道是一个文件系统，分配一块内存供进程读写
    * 管道的数据保存在inode->i_pipe中，结构为struct pipe_inode_info

[register-filesystem]: https://github.com/torvalds/linux/blob/v5.7/fs/pipe.c#L1352
[pipe-syscall-enter]: https://github.com/torvalds/linux/blob/v5.7/fs/pipe.c#L1011
[create-pipe-files]: https://github.com/torvalds/linux/blob/v5.7/fs/pipe.c#L901
[alloc-inode]: https://github.com/torvalds/linux/blob/v5.7/fs/inode.c#L233
[pipefs-type]: https://github.com/torvalds/linux/blob/v5.7/fs/pipe.c#L1344
[alloc-pipe-info]: https://github.com/torvalds/linux/blob/v5.7/fs/pipe.c#L871
[pipefifo-fops]: https://github.com/torvalds/linux/blob/v5.7/fs/pipe.c#L878
[open-pipe-file]: https://github.com/torvalds/linux/blob/v5.7/fs/pipe.c#L909-L930
[pipe-write-enter]: https://github.com/torvalds/linux/blob/v5.7/fs/read_write.c#L621
[ksys-write]: https://github.com/torvalds/linux/blob/v5.7/fs/read_write.c#L601
[vfs-write]: https://github.com/torvalds/linux/blob/v5.7/fs/read_write.c#L601
[pipe-write]: https://github.com/torvalds/linux/blob/v5.7/fs/pipe.c#L439
[copy-file-to-user]: https://github.com/torvalds/linux/blob/v5.7/fs/pipe.c#L992
