---
description: >-
  通过分析内核源码，理解内核是如何执行一个二进制文件的。特别地，通过利用内核对不同二进制文件格式的处理机制，使得内核开始处理 ELF
  文件之前，在不修改内核原有代码的前提下，找到一个能够验证 ELF 文件中的数字签名的机会。
---

# Chapter 1 - 二进制文件执行过程

## 内核如何执行一个二进制文件

当我们在 shell 中敲入命令，试图执行一个二进制文件时，shell 会调用 Linux 内核的 `execve` 系统调用。该系统调用会将二进制文件载入内存，判断其格式是否合法、是否具有执行权限。如果一切正常，则将进程的代码段、数据段替换为新程序的段，并设置好命令行参数、环境变量、新的程序入口，完成新程序开始运行前的一切准备工作。

如果 `execve` 的执行一切正常，那么该进程原有程序的运行环境都会被新进程完全替换，因此 `execve` 永远不会返回。对于使用 shell 的用户来说，希望能够在运行完一条命令后再次回到 shell 中继续输入下一条命令。为了避免 shell 进程在调用 `execve` 之后被新程序覆盖，在 terminal 的实现中，通常会通过 `fork` 系统调用，先复制出一个当前进程的副本，然后在副本进程中调用 `execve`。执行结束后，回到 shell 的原进程中，使得用户可以继续执行下一个程序。

![Shell &#x6267;&#x884C;&#x7A0B;&#x5E8F;&#x7684;&#x5927;&#x81F4;&#x6D41;&#x7A0B;](../.gitbook/assets/shell.png)

## 内核如何识别不同格式的二进制文件

内核支持执行多种不同格式的二进制文件。比如一个 ELF 文件，或是一个 shell script，甚至还可以是一幅图片，或一个 PDF 文档。对于不同的二进制文件格式，内核需要为其准备不同的运行环境：

* 对于一个 ELF 文件，内核需要找到该文件的代码段和数据段，以及程序的首条指令地址，并将这些信息设置在进程控制块中
* 对于一个 shell script，内核需要寻找脚本指定的解释器，并将脚本文件名作为启动解释器的参数
* 对于一个 PDF 文档或图片，内核也需要寻找对应的解释器，并将文件作为解释器参数

为了识别二进制文件的格式，内核会将二进制文件的前 128 个字节读到如下的结构体中：

```c
#define CORENAME_MAX_SIZE 128

/*
 * This structure is used to hold the arguments that are used when loading binaries.
 */
struct linux_binprm {
    char buf[BINPRM_BUF_SIZE];
    /* ... */
} __randomize_layout;
```

根据 magic value \(每个格式特有的字节序列\) 等信息，判断出二进制文件的格式。比如：

* 通过文件开头是否是 `#!` 来判断是否是一个脚本文件
* 根据文件开头是否是 `0x7f` `0x45(E)` `0x4c(L)` `0x46(F)` 来判断是否是一个 ELF 文件
* ......

## 二进制文件格式处理程序 \(Binary Format Handler\)

在 Linux 4.15.0 中，内核能够执行 **哪些格式** 的二进制文件呢？

在该版本的内核源代码中，已经内置了部分二进制文件格式的处理程序 \(handler\)，位于内核代码的 `fs/` 目录下：

* `binfmt_aout` - a.out 格式
* `binfmt_elf` - ELF \(Execute and Linkable Format\) 格式
* `binfmt_elf_fdpic`
* `binfmt_em86`
* `binfmt_flat`
* `binfmt_misc` - 可在运行期自行配置解释程序的二进制文件格式
* `binfmt_script` - 脚本文件格式，用于执行 shell、Perl 等格式的脚本

{% hint style="info" %}
编译内核前，根据目标硬件平台进行编译配置后，有些格式将不会被编译进内核。
{% endhint %}

其中，每一种格式对应的处理程序，在编码上都以模块的形式实现 \(虽然编译后并不一定是模块\)，并且都用类似面向对象中多态的形式实现了一个统一的接口：

```c
struct linux_binfmt {
    struct list_head lh;
    struct module *module;
    int (*load_binary)(struct linux_binprm *);
    int (*load_shlib)(struct file *);
    int (*core_dump)(struct coredump_params *cprm);
    unsigned long min_coredump;    /* minimal dump size */
} __randomize_layout;
```

其中，关注这三个函数指针：

* `*load_binary` 指向装入该格式二进制文件的函数
* `*load_shlib` 指向装入该格式共享库的函数
* `*core_dump` 指向装入该格式核心转储文件的函数

在操作系统初始化阶段，这些二进制格式的处理函数被依次添加到了一个链表上。当执行一个二进制文件时，内核实现将其头 128 字节以及相关信息读入 `struct linux_binprm` 结构体，然后依次遍历链表上的每一个 `struct linux_binfmt` 结构体，将 `struct linux_binprm` 作为参数，依次调用每种二进制文件格式对应的 `*load_binary` 函数。这一过程在 `fs/exec.c` 的如下函数中实现：

```c
int search_binary_handler(struct linux_binprm *bprm)
{
    /* ... */
    list_for_each_entry(fmt, &formats, lh) {
        /* ... */
        retval = fmt->load_binary(bprm);
        /* ... */
        if (retval != -ENOEXEC || !bprm->file) {
            read_unlock(&binfmt_lock);
            return retval;
        }
    }
    /* ... */
    return retval;
}
```

在这个函数中，可能会有以下三种情况出现：

1. 当前处理函数不能识别这个二进制文件的格式，则返回 `-ENOEXEC` 的错误码，继续尝试用下一个处理函数识别该文件的格式
2. 当前处理函数成功识别了这个二进制文件的格式，并顺利完成了执行这个二进制文件的准备工作，则当前进程的执行流已经被设置了一个新方向，再也不会返回这里了
3. 当前处理函数成功识别了这个二进制文件的格式，但在执行该文件准备工作的过程中发生了错误 \(比如动态分配内存失败\)，则将相应错误码返回并停止遍历链表

示意图如下所示：

![&#x4E8C;&#x8FDB;&#x5236;&#x6587;&#x4EF6;&#x5904;&#x7406;&#x51FD;&#x6570;&#x94FE;&#x8868;](../.gitbook/assets/search-binary-handler.png)

## 内核识别二进制文件格式的次序

上面的图片是典型的 Intel 64-bit 机器中，内核默认编译配置下，识别一个二进制文件格式的次序。哪些因素决定了内核以不同的优先级依次识别各个二进制文件格式呢？

内核是在 **初始化阶段** 逐步构造二进制格式处理函数的 **链表** 的。每个格式的处理函数被挂上链表的时机和方式由如下因素决定：

1. 每个处理函数模块的初始化等级
2. 每个处理函数模块被挂上链表的方式 \(在链表头部插入 / 在链表尾部插入\)
3. 每个处理函数模块被编译的顺序



