---
description: 评估在内核中引入了签名验证机制后所带来的各类性能开销。
---

# Chapter 10 - 内核签名验证开销评估

## 10.1 开销分析

我们的解决方案在内核为执行 ELF 文件进行准备的过程中，引入了额外的签名验证机制，从而引入了 CPU、内存、I/O 上的开销：

* CPU 的开销：摘要算法和 _RSA_ 算法所带来的计算开销
* 内存开销：将 ELF 文件中必要的 table 和 section 保存在内存缓冲区所带来的开销
* I/O 开销： 将 ELF 文件中必要的 table 和 section 从磁盘装入内存的开销

这里的开销主要分为两个维度：**时间** 上的开销，以及 **空间** 上的开销。

## 10.2 时间开销

我们评估了签名验证机制所带来的时间开销。具体地，评估带有签名验证机制的内核，在执行一个 ELF 文件前的准备工作中，比不进行签名验证的内核慢多少。

我们选用的测试对象是 `/bin` 下的一些系统内置命令，它们实际上都是 ELF 文件，如 `ls` `mv` 等。我们对它们进行签名后，分别在签名验证模块已装载和未装载的条件下，以空参数执行这些 ELF 程序足够多次。虽然以空参数执行 ELF 文件会导致一些程序错误退出，但我们只关心 **内核为执行二进制文件而进行准备工作** 的时间，而并不关心二进制文件真正开始执行后的时间与状态。

我们以 `fs/exec.c` 文件中实现 `execve` 系统调用具体逻辑的 `do_execveat_common()` 函数作为评估对象。该内核函数主要负责为可执行文件准备运行环境，包括在进程控制块中设置新进程的入口地址、内存等相关工作。最重要的是，该函数中包括了对二进制格式处理程序链表的遍历和使用。

![&#x65F6;&#x95F4;&#x5F00;&#x9500;&#x7684;&#x8861;&#x91CF;&#x6307;&#x6807;](../.gitbook/assets/prev-test-metric.png)

如图所示，虚线指示的区间表示进程没有被 CPU 运行；实线指示的区间表示进程正在被 CPU 运行。Process A 代表启动执行 ELF 文件的进程 \(比如 `bash`\)，Process B 代表执行 ELF 文件的新进程。

* 在 1 号时刻，A 进程接收了执行一个 ELF 文件的命令，于是调用 `fork()`，开始从自身复制一个新的进程 B
* 在 2 号时刻，`fork()` 结束，新复制的进程 B 准备就绪。为了避免 [copy-on-write](https://en.wikipedia.org/wiki/Copy_on_write) 优化机制带来额外的开销，内核通常会调度新的进程 B 首先执行。进程 B 开始执行 `execve()`，载入新的 ELF 文件
* 在 3 号时刻，`execve()` 结束。此时，进程 B 的下一条将被执行的指令被设置为 ELF 文件的入口指令。但由于 load-on-demand 机制，第一条指令所在的内存页还没有被载入内存。此时，所有进程全部处于就绪状态，可以被调度器调度
* 在 4 号时刻，调度器选择执行 B 进程。在访问 ELF 文件的第一条指令时，发生 [**缺页异常 \(Page Fault\)**](https://en.wikipedia.org/wiki/Page_fault)，由异常处理程序从磁盘上将该指令所在的页装入内存；在此过程中，调度器可能会选择其它进程执行
* 在 5 号时刻，B 进程所要访问的第一条指令已经在内存中；当调度器调度到 B 进程时，B 进程真正开始运行

我们评估的是签名验证机制在 `execve()` 的执行过程中引入的开销，即图中 2 号时刻与 3 号时刻之间的时间。我们将每个待测试的 ELF 文件分别在签名验证模块装载和未装载的内核上运行足够多的次数 \(1000 次\)，并统计前后 `execve()` 执行时间增加的百分比。软硬件环境与统计单位：

* Machine：Lenovo® R720
* CPU：Intel® Core™ i7-7700HQ CPU @ 2.80GHz \* 8
* Memory：15.5 GiB
* Disk：60GB SSD
* OS：Deepin 15.11, x64
* 时间单位：微秒 \(μs\)

| ELF 文件 | 平均执行时间 \(无签名验证\) | 平均执行时间 \(签名验证\) | 开销倍数 |
| :--- | :--- | :--- | :--- |
| cp | 3013373                     | 6150166                   | 2.0410   |
| df | 4031915                     | 6397411                   | 1.5867   |
| echo     | 2015491                     | 3531790                   | 1.7523   |
| false    | 1670998                     | 3142193                   | 1.8804   |
| grep     | 2726951                     | 7343783                   | 2.6930   |
| kill     | 4066986                     | 5582955                   | 1.3727   |
| less     | 2637721                     | 5661841                   | 2.1465   |
| ls       | 2765057                     | 5498167                   | 1.9884   |
| mkdir    | 2797265                     | 5024875                   | 1.7963   |
| mount    | 4353229                     | 5878047                   | 1.3503   |
| mv       | 3062660                     | 6194796                   | 2.0227   |
| rm       | 2186421                     | 4062839                   | 1.8582   |
| rmdir    | 2164469                     | 3794732                   | 1.7532   |
| tar      | 3348584                     | 10261805                  | 3.0645   |
| touch    | 2165813                     | 4495568                   | 2.0757   |
| true     | 1692813                     | 3146270                   | 1.8586   |
| umount   | 3341070                     | 4812077                   | 1.4403   |
| uname    | 2013237                     | 3526416                   | 1.7516   |

测试脚本与结果位于 [签名程序的代码仓库](https://github.com/NUAA-WatchDog/linux-elf-binary-signer/tree/master/test/prev) 中。

我们对测试结果进行了分析与解读：

1. 内核内置的 ELF 处理模块仅需要访问 ELF header 和 program header table，而 program header table 在布局上紧接 ELF header 之后，因此当内核将 ELF 的头 128 字节装入主存时，紧接其后的 program header table 也已经位于主存 **高速缓冲 \(buffer\)** 中，访问时不再需要额外的 I/O 开销
2. 加入签名验证机制后，`execve` 系统调用的执行时间与程序代码段长度相关 - 比如 `tar` 程序的代码段特别长，因此需要更长的时间分配内存、计算 `.text` 的摘要
3. 加入签名验证机制后，`execve` 系统调用的执行时间为原先的 1.3 倍至 3 倍不等，开销平均大约增加 60% - 70%。主要原因是将 ELF 文件的 section header table、section header string table、签名 section 与被签名 section 载入内存引入了额外的 I/O 开销

## 10.3 空间开销

这里主要关注内核为待验证的 ELF section 所动态分配的内存空间。由于内存空间会在使用完毕后被回收，这里只关心动态分配内存空间的 **极限值**。即，内核最大能够支持分配多大的内存将一个 ELF section 装入内存进行验证。

在所有目前经过测试的程序中，具有最长的 `.text` section 的 ELF 文件为 [chromium](http://www.chromium.org/Home) \(Version 71.0.3578.80, Developer Build, 64-bit, deepin 15.11 x64\)。该 ELF 文件的大小为 134147808 字节 \(127.9333 MB\)，其中，`.text` section 的长度为 80549925 字节 \(76.8184 MB\)。ELF 数字签名验证模块成功地验证了其中的数字签名。

## 10.4 参考资料

[Wikipedia - Copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write)

[Mr Dk.'s blog - linux-kernel-comments-notes/Chapter 12 - 文件系统/Chapter 12.15 - exec.c 程序](https://mrdrivingduck.github.io/blog/#/markdown?repo=linux_kernel_comments_notes&path=Chapter%2012%20-%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%2FChapter%2012.19%20-%20select.c%20%E7%A8%8B%E5%BA%8F.md)

[Stackoverflow - Does Linux load program-pages on demand?](https://stackoverflow.com/questions/19292744/does-linux-load-program-pages-on-demand)

