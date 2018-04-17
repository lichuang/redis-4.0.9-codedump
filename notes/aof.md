# aof
aof(Append Only File)

## 一般流程
当开启AOF开关时，任何一次的数据修改，都会同步到AOF文件中。

redis使用aof_buf用来缓冲添加到AOF的数据，根据策略决定将这个buf何时写入文件。

调用堆栈：

```
(gdb) bt
#0  feedAppendOnlyFile (cmd=0x1000d3f70, dictid=0, argv=0x102000190, argc=3) at aof.c:556
#1  0x000000010000c875 in propagate (cmd=0x1000d3f70, dbid=0, argv=0x102000190, argc=3, flags=<optimized out>) at server.c:2112
#2  call (c=0x0, flags=<optimized out>) at server.c:2291
#3  0x000000010000cf27 in processCommand (c=0x0) at server.c:2510
#4  0x000000010001c547 in processInputBuffer (c=0x3) at networking.c:1354
#5  0x0000000100004dfd in aeProcessEvents (eventLoop=0x100625d30, flags=<optimized out>) at ae.c:440
#6  0x000000010000511b in aeMain (eventLoop=0x3) at ae.c:498
#7  0x000000010000febe in main (argc=<optimized out>, argv=<optimized out>) at server.c:3894
```

## 同步aof_buf到磁盘

完成这个操作的核心函数是flushAppendOnlyFile

## rewrite aof文件流程
随着aof文件越来越大，需要进行重写（rewrite），目的在于之前可能存在了多个命令已经是冗余的了，比如针对同一个key的多次修改、删除操作，其实根据当前这个key的值合并成一个操作。这样的目的是为了减少aof文件的大小。

自动触发的条件是：

```
           /* Trigger an AOF rewrite if needed. */
           if (server.aof_state == AOF_ON &&  // 有打开AOF开关
               server.rdb_child_pid == -1 &&  // 没有在进行rdb文件写入
               server.aof_child_pid == -1 &&  // 没有在进行AOF文件写入
               server.aof_rewrite_perc &&     // 有配置这个参数
               server.aof_current_size > server.aof_rewrite_min_size) // 大于aof rewrite的文件大小
           {
              // 计算比例
              long long base = server.aof_rewrite_base_size ?
                              server.aof_rewrite_base_size : 1;
              long long growth = (server.aof_current_size*100/base) - 100;
              if (growth >= server.aof_rewrite_perc) {  // 比例大于配置的值
                  serverLog(LL_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
                  // 后台进行AOF文件的重写
                  rewriteAppendOnlyFileBackground();
              }
           }
```

即：

    1.  打开了AOF开关
    2.  当前没有在进行rdb、aof操作。
    3.  有配置of_rewrite_perc
    4.  aof文件大小大于配置的值aof_rewrite_min_size

此时将计算一个比例，将当前aof大小对比aof_rewrite_base_size大小，如果其比例大于aof_rewrite_perc，就调用rewriteAppendOnlyFileBackground函数进行重写。

###重写的具体流程
rewriteAppendOnlyFileBackground函数流程：

```
// rewrite aof 文件流程：
  // 1）redis server调用rewriteAppendOnlyFileBackground函数，它将fork出一个子进程：
  //  a) 子进程rewrite写请求到一个临时的aof文件中
  //  b) 父进程将rewrite过程中新增的写请求添加到aof_rewrite_buf中
  // 2）子进程完成它的rewrite工作，然后退出
  // 3) 父进程收到子进程退出的信号，把aof_rewrite_buf的内容添加到临时文件中，然后原子的rename该临时文件为aof文件
```

同时，在重写的过程中，父进程redis还会收到新的写请求，这些写请求会同步到rewite diff缓冲区中。

aof_rewrite_buf_blocks这个数据结构，是一个链表，其链表元素定义为：

```
typedef struct aofrwblock {
      unsigned long used, free;
      char buf[AOF_RW_BUF_BLOCK_SIZE];
  } aofrwblock;
```

每次向这里写完一段数据，会通过pipe来通知aof rewrite子进程，diff缓冲区有数据：

```
      if (aeGetFileEvents(server.el,server.aof_pipe_write_data_to_child) == 0) {
          aeCreateFileEvent(server.el, server.aof_pipe_write_data_to_child,
              AE_WRITABLE, aofChildWriteDiffData, NULL);
      }
```

而子进程在正常同步完自己内存里的数据到aof临时文件之后，会尝试读取rewrite diff 通知pipe，从里面尽可能的读取多一些的数据出来再次写入到aof临时文件中。

最后同步完成时，diff缓冲区可能还有数据，但是变化不会太多，这一点为了尽量减少父进程自己同步diff缓冲区数据到文件造成进程卡顿的流程。

子进程将自己内存数据同步到aof文件的流程在rewriteAppendOnlyFile函数中。


