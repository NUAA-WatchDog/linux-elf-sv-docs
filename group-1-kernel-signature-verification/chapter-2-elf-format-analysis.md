---
description: 通过对 ELF 文件的格式进行分析，内核中的 ELF 签名验证处理模块能够成功取得签名程序附在 ELF 文件中的数字签名，从而能够进一步进行签名验证。
---

# Chapter 2 - ELF 文件格式分析

## 2.1 关于 ELF 格式

**可执行与可链接格式** \(Executable and Link-able Format, ELF\) 是一种可执行文件、目标代码、共享库、核心转储文件的通用标准文件格式。ELF 标准最早发布于一个名为 System V Release 4 \(SVR4\) 的 Unix 操作系统版本的应用二进制接口 \(Application Binary Interface, ABI\) 标准规范中，并迅速被各大 Unix 版本所接受。1999 年，ELF 格式被选为 Unix 或类 Unix 系统在 x86 处理器平台上的标准二进制文件格式。

在设计上，ELF 格式具有 **灵活、可扩展、跨平台** 的特点。比如，其支持不同的字节顺序 \(大小端\)、不同的地址空间 \(32/64\)、不排斥任何特定的 CPU 或指令集体系结构 \(Instruction Set Architecture, ISA\)。因此，ELF 格式能够在很多不同硬件平台的不同操作系统上运行。

## 2.2 ELF 文件的视角与格式

ELF 文件具有三种主要的目标类型：

* 可重定位文件 - 可与其它目标文件静态链接在一起的代码和数据
* 可执行文件 - 可被操作系统装载到进程地址空间中并执行
* 共享对象文件 - 与其它可重定位文件或共享库文件链接为另一个目标；或与可执行文件和其它共享库文件一起形成进程映像

根据 ELF 文件被使用的方式，文件内容可以有两种视角：

* 链接视角 - 链接器将 ELF 文件链接在一起时使用的视角
* 执行视角 - 操作系统将 ELF 文件装载到进程地址空间时使用的视角

两种视角如下图所示：

![ELF &#x89C6;&#x89D2;](../.gitbook/assets/elf-view.png)

Section 主要用于链接器对代码的重定位，如汇编程序中的 `.text` `.data`。而当文件载入内存执行时，目标代码中的 section 会被链接器组织到可执行文件的各个 segment 中。

如上图所示，ELF 文件除去各 section/segment 以外，主要包含三个重要部分：

1. ELF header
2. Program header table
3. Section header table

ELF header 指明了 ELF 文件的整体信息，如 ELF 文件的 magic value、类型、版本、目标机器等。另外，ELF 还指明了 program header table 与 section header table 两个表在文件中的偏移位置、条目个数、条目大小。这两个表的位置和长度随着 section/segment 的个数而变化，而 ELF header 总是位于文件最开头，且长度固定。显然，如果想要访问 program header table 和 section header table 中的信息，必须通过 ELF header 来找到它们在文件中的确切位置。

Program header table 主要描述了将哪一个或哪几个 section 组织为一个 segment，以及各个 segment 的描述信息。Section header table 描述了 ELF 文件中所有的 section，以及每个 section 的类型、长度等描述信息。

{% hint style="info" %}
Section header table 中并不存储每个 section 的名称。所有 section 的名称全部存储在一个名为 section header string table 的 section 中，名称之间用 `\0` 分隔。在 ELF header 中，记录了该 section 在 section header table 中的索引。
{% endhint %}

## 2.3 内核从 ELF 中取得数字签名的步骤

内核在对 [二进制文件处理函数链表](chapter-1-binary-execution-procedure.md#15-dui-elf-wen-jian-jin-hang-qian-ming-yan-zheng-de-si-lu) 进行遍历时，已经读取了该文件的 [头 128 字节](chapter-1-binary-execution-procedure.md#12-nei-he-ru-he-shi-bie-bu-tong-ge-shi-de-er-jin-zhi-wen-jian)。如果该二进制文件是一个 ELF 文件，那么已读取的内容中已经包含了 ELF 文件的 ELF header。由此，首先通过 ELF header 中的 magic value 检验二进制文件是否是一个 ELF 文件；判断 ELF 文件类型是否为 `ET_EXEC` \(可执行文件\) 或 `ET_DYN` \(动态链接文件\)。

其次，根据 ELF header 中指示的 section header table 的位置、条目个数、每个条目的大小，可以将 section header table 装载到内存；根据 ELF header 中指示的 section header string table 的索引，以及已经装入内存的 section header table，可以将 section header string table 装载到内存。

![ELF &#x6587;&#x4EF6;&#x4E2D;&#x7684;&#x7ED3;&#x6784;&#x5E03;&#x5C40;](../.gitbook/assets/elf-layout.png)

同时遍历 section header table \(每个 section 的描述信息\) 和 section header string table \(每个 section 的名称\)，可以定位到与签名程序约定好的签名信息 section 与被签名 section，如：

* 被签名 section `.text` 与签名数据 section `.text_sig`
* ...

在找到这两个相互对应的 section 之后，再根据 section header table 中指示的这两个 section 在文件中的偏移与长度，将这两个 section 的具体数据装入内存。

最终，基于每对匹配的 section 数据进行签名验证。如果所有的签名验证都正确，那么 [ELF 签名验证模块](chapter-1-binary-execution-procedure.md#15-dui-elf-wen-jian-jin-hang-qian-ming-yan-zheng-de-si-lu) 会返回 `-ENOEXEC` 错误码，使内核随后调用真正的 ELF 处理模块完成相应的工作；如果签名验证错误，那么模块返回其它错误码，内核将无法继续执行这个 ELF 文件。

## 2.4 验证 ELF 的动态链接依赖

一个 ELF 可执行文件中还包含了其依赖的 **共享对象** \(动态链接库\) 的信息。其中：

* `.interp` section 指明了该文件使用的动态链接器的位置
* `.dynstr` section 存放了所有动态链接信息中需要用到的字符串
* `.dynamic` section 存放了所有动态链接相关的信息

其中，`.dynamic` section 是以结构体数组的形式存在的，结构体定义如下：

```c
typedef struct
{
  Elf64_Sxword	d_tag;			/* Dynamic entry type */
  union
    {
      Elf64_Xword d_val;		/* Integer value */
      Elf64_Addr d_ptr;			/* Address value */
    } d_un;
} Elf64_Dyn;
```

`d_tag` 指明结构体自身的类型，并决定了 `d_un` 的用途与含义。这里我们只关注 `d_tag` 为 `DT_NEEDED` 的结构体元素。此时，`d_un` 中有效的元素是 `d_val`，含义是 `.dynstr` 字符串表中的偏移，也就是所依赖的共享对象名称。在完成 ELF 文件本身的数字签名验证工作后，程序还将提取上述信息，依次验证 ELF 依赖的每一个共享对象中的数字签名。如果 ELF 依赖的某个共享对象没有附带数字签名，或数字签名被篡改，内核将拒绝执行这个 ELF 文件。

通过提取 ELF 文件中的信息，内核只能获得共享对象的名称 \(如 `libcrypto.so.1.1`\)，还无法定位到共享对象的确切位置，因此无法打开共享对象文件并验证数字签名。这里我们实现了 **动态链接器** 的部分功能，到 `/etc/ld.so.cache` 中查找共享对象所在的绝对路径。这个文件由特定的格式组织，我们自行实现了解析格式的代码。当然，共享对象的相对位置还可以通过 **环境变量** 等途径指定，这部分我们暂未实现。在获得共享对象的绝对路径后 \(如 `/usr/lib/x86_64-linux-gnu/libcrypto.so.1.1`\)，内核就可以打开这个共享对象文件，并像上述 [验证 ELF 文件数字签名](chapter-2-elf-format-analysis.md#23-nei-he-cong-elf-zhong-qu-de-shu-zi-qian-ming-de-bu-zhou) 那样，验证共享对象中的数字签名 - 因为共享对象也遵循 ELF 格式标准。

{% hint style="info" %}
我们认为，如果引入对动态链接库的数字签名验证机制，虽然更加安全，但有悖于动态链接的设计初衷，因此可能存在一些性能问题。
{% endhint %}

## 2.5 参考资料

[PDF - Executable and Linking Format \(ELF\) Specification](http://www.skyfree.org/linux/references/ELF_Format.pdf)

[Stackoverflow - Read/write files within a Linux kernel module](https://stackoverflow.com/questions/1184274/read-write-files-within-a-linux-kernel-module)

[LD\_LIBRARY\_PATH 和 /etc/ld.so.cache 文件](http://blog.chinaunix.net/uid-25304914-id-3046279.html)

[Apache JIRA - Add utility for parsing ld.so.cache on linux](https://issues.apache.org/jira/browse/MESOS-5399)

