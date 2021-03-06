### linux系统缓存机制
#### 1、缓存机制
为了提高文件系统性能，内核利用一部分物理内存分配出缓冲区，用于缓存系统操作和数据文件，当内核收到读写的请求时，内核先去缓存区找是否有请求的数据，有就直接返回，如果没有则通过驱动程序直接操作磁盘。
缓存机制优点：减少系统调用次数，降低CPU上下文切换和磁盘访问频率。
CPU上下文切换：CPU给每个进程一定的服务时间，当时间片用完后，内核从正在运行的进程中收回处理器，同时把进程当前运行状态保存下来，然后加载下一个任务，这个过程叫做上下文切换。实质上就是被终止运行进程与待运行进程的进程切换。
#### 2、查看缓存区及内存使用情况
```
[root@localhost ~]# free -m
      total       used       free     shared    buffers     cached
Mem:    7866       7725       141      19        74       6897
-/+ buffers/cache:     752       7113
Swap:   16382        32      16350
```
可以看到内存总共8G，已使用7725M，剩余141M，不少的人都是这么看的，这样并不能作为实际的使用率。因为有了缓存机制，具体该怎么算呢？
空闲内存=free（141）+buffers（74）+cached（6897）
已用内存=total（7866）-空闲内存
由此算出空闲内存是7112M，已用内存754M，这才是真正的使用率，也可参考-/+ buffers/cache这行信息也是内存正确使用率。
#### 3、可见缓存区分为buffers和cached，他们有什么区别呢？
  内核在保证系统能正常使用物理内存和数据量读写情况下来分配缓冲区大小。buffers用来缓存metadata及pages，可以理解为系统缓存，例如，vi打开一个文件。cached是用来给文件做缓存，可以理解为数据块缓存，例如，dd if=/dev/zero of=/tmp/test count=1 bs=1G 测试写入一个文件，就会被缓存到缓冲区中，当下一次再执行这个测试命令时，写入速度会明显很快。
#### 4、随便说下Swap做什么用的？
  Swap意思是交换分区，通常我们说的虚拟内存，是从硬盘中划分出的一个分区。当物理内存不够用的时候，内核就会释放缓存区（buffers/cache）里一些长时间不用的程序，然后将这些程序临时放到Swap中，也就是说如果物理内存和缓存区内存不够用的时候，才会用到Swap。
#### 5、怎样释放缓存区内存呢？
 5.1 直接改变内核运行参数
 #释放pagecache
 ```
 echo 1 >/proc/sys/vm/drop_caches
 ```
 #释放dentries和inodes
 ```
 echo 2 >/proc/sys/vm/drop_caches
 ```
 #释放pagecache、dentries和inodes
 ```
 echo 3 >/proc/sys/vm/drop_caches 
 ```
 5.2 也可以使用sysctl重置内核运行参数 
 ```
 sysctl -w vm.drop_caches=3 
 ```
注意：这两个方式都是临时生效，永久生效需添加sysctl.conf文件中，一般写成脚本手动清理，建议不要清理。
