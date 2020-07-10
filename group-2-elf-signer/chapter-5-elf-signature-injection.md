---
description: >-
  将产生的数字签名附加到 ELF 文件中，并且不破坏 ELF 原有的结构于信息。使得加入签名信息的 ELF 文件也能够在没有签名验证机制的普通 Linux
  上也可以正确运行。
---

# Chapter 5 - 将签名数据注入 ELF 文件

## 5.1 签名数据存放位置

要将签名数据附在原有的 ELF 文件中，且不破坏 [ELF 的结构](../group-1-kernel-signature-verification/chapter-2-elf-format-analysis.md)，使其能够在没有签名验证机制的 OS 上也能运行，有两种可能的方式：

* 直接在 ELF 文件的尾部附加签名
* 以符合 ELF 规范的方式以 ELF section 为单位添加签名数据

在尾部追加签名的方式比较简单，但从文件整体视角来看，该文件已经不是一个符合 ELF 格式的文件了。如果我们需要对 ELF 文件的多个 section 进行签名，在文件末尾追加的方法将具有较差的可扩展性和灵活性。此外，将无法通过 `readelf` 或 `objdump` 等工具读取签名数据。

以符合 ELF 格式的方式添加数据稍微复杂一些，因为其中涉及到对 ELF 的文件格式进行解析。另外，在 ELF 文件中添加若干新的 section，可能会涉及到对 ELF 文件中其它结构的修改。但这种方法使我们可以在 ELF 文件中添加任意多个 section，并可以在 ELF 的 section header table 结构中记录这些 section 的元信息。这样方便内核从签名后的 ELF 文件中快速获取有效的签名数据。

## 5.2 将签名数据 section 注入 ELF 文件

在向 ELF 文件插入数据时，我们尽可能地在每个 ELF 结构的尾部插入数据。在尾部插入能够最大程度地避免破坏已有的 ELF 结构之间的引用关系。要将数据注入到 ELF 文件中，并维持 ELF 文件的原有格式，需要修改如下几处结构：

1. 插入一个包含签名数据的新 section 数据区
2. Section header table 中应多出一个新的 entry 来描述签名数据在文件中的偏移和长度
3. 将新 section 的名称字符串添加到 `.shstrtab` 中，并更新该 section 的长度
4. 数据插入位置之后的所有 section 的偏移地址都需要被后移更新
5. 在 ELF header 中更新 section header table 的偏移位置和元素数量

另外，还需要保证数据插入后，所有 section 的地址对齐要求得到满足。Section header table 的起始地址也需要对齐 8 字节地址，以充分利用总线宽度提升性能。

由于 ELF 文件的布局方式因编译器而异，在 [签名程序的代码仓库](https://github.com/mrdrivingduck/linux-elf-binary-signer/tree/master/test/func) 中，保存了 GCC 编译出的 ELF 示例文件与 [Golang](https://golang.org/) 编译出的 ELF 示例文件。它们有着完全不同的布局。

如果 section header string table 位于 section 的中间，向其中插入内容会使布局在插入位置之后的所有 section 在文件中后移。不仅会影响到 section header table 中的数据，还需要修正 program header table。为了避免对 program header table 中的内容产生影响，程序将在文件的最底部重新创建一个 section header string table，并使 section header table 中的数据结构指向该位置；原先的 section header string table 已经无效，但不被删除。

另外，如果 section header table 布局于位于任意 section 之前，在其中插入新的 section header entry 也会导致 program header table 中各 segment 的文件偏移与实际位置不一致。我们用了类似的处理方法，将原有 section header table 在文件末尾复制一份，并插入数据。如右下图所示。

图中红色部分为新插入的数字签名信息及其相关的描述信息：

![&#x6CE8;&#x5165;&#x7B7E;&#x540D;&#x540E;&#x7684; ELF &#x6587;&#x4EF6;](../.gitbook/assets/elf-new-section.png)

在签名程序的具体实现中，我们将 ELF 文件中的以下部分装载到内存中。这两部分的内存副本将用于记录和修改各个 ELF 结构在文件中的偏移变化和长度变化，并最终被写回 ELF 文件，覆盖原有内容：

* ELF header
* Section header table

我们为 section header table 多分配了一块 section header entry 的内存，并在最后一个空出的 entry 中为新增的 section 设置信息，该 section 用于保存签名数据。程序会将签名数据插入到从文件末尾开始的第一个能被 8 整除的地址，作为新增 section 的数据。这个 **地址** 以及 **签名数据的长度** 被记录到最后一个 section header entry 结构体中。在这一过程进行的同时，程序顺带计算在文件中需要插入的字节偏移位置与字节数量。

{% hint style="info" %}
这里阐明文章中 **插入** 和 **覆盖** 的概念。对于普通的文件写入，可被理解为从文件中的某个偏移位置开始 **覆盖** 文件中的原有内容，\(如果覆盖内容没有超出文件原有长度\) 将不会改变文件的长度。而插入特指 **将插入位置之后的原有内容向后挤**，从而一定会引发文件长度的变化。

对于文件中位置、长度固定，而内容需要被修改的部分，使用覆盖操作 \(比如 ELF header\)；对于文件中必须新增的数据，使用插入操作 \(比如新增 section 的字符串名\)。
{% endhint %}

接下来将被插入文件的数据由两部分组成：

* 签名 section 的名称字符串，以及用于补齐 8 字节地址的无用字符 \(因为 `.shstrtab` 长度任意\)
* Section header table 内存副本中最后一个新增的 entry

基于上述工作，签名程序能够确定 ELF header 中记录的 section header table 偏移量 \(`e_shoff`\) 和 section header 数量 \(`e_shnum`\) 的最终值。另外，通过遍历 section header table，程序还可以确定在 `.shstrtab` 中插入新 section 名称字符串的位置，以及用于使插入位置之后所有 ELF 结构的地址对齐的填充字节个数。

最后一步，程序将内存中的 ELF header 副本与 section header table 副本 \(不包含新增 entry\) 中的偏移量修正后，覆盖原文件中的相同部分；然后，在计算好的文件偏移处插入准备好的数据。

{% hint style="info" %}
在插入数据时，应当先处理在文件中插入位置靠后的数据。如果先处理在文件中插入位置靠前的数据，将会影响到之后插入的数据在文件中的插入偏移位置。

也就是说，插入的顺序与具体的 ELF 结构无关，只和 ELF 结构在文件中的偏移位置有关，插入位置靠后的数据先被插入。
{% endhint %}

签名程序会在被签名 section 名称的基础上加上 `_sig` 后缀，成为对应的签名数据 section。如，对于保存 ELF 程序指令的 `.text` section，签名程序会将签名数据作为一个名为 `.text_sig` 的 section 附加到 ELF 文件中。如果在解析 ELF section 的过程中发现已经存在名称以 `_sig` 为后缀的 section，则签名程序会中止退出，以防止重复签名。

```bash
$ readelf -a sign-target
...
Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
...
  [29] .text_sig         PROGBITS         0000000000000000  000020c0
       00000000000001d1  0000000000000000   O       0     0     8
...
```

在签名期间，签名程序会产生一些临时文件，用于保存被签名的若干个 section 的数字签名数据；在签名完成之后，这些临时文件将会被清除。

签名程序可以指定一个被签名文件名，以及一个可选的输出文件名。如果不指定输出文件名，对于一个名为 `elf`的 ELF 文件，在签名成功后，签名程序会保留未被签名的旧版本 ELF 文件 `elf.old` 作为备份，而注入签名后的新 ELF 文件将会被命名为原先的 `elf`。具体的使用方法参见 [后续](../group-3-usage/chapter-9-elf-sign.md#82-qian-ming-cheng-xu-de-shi-yong-fang-shi)。

## 5.3 参考资料

[A tutorial introduction to _libelf_](https://sourceforge.net/projects/elftoolchain/files/Documentation/libelf-by-example/20120308/libelf-by-example.pdf/download)\_\_

[Adding section to ELF file](https://stackoverflow.com/questions/1088128/adding-section-to-elf-file)

[ELF format manipulation](https://stackoverflow.com/questions/7601344/elf-format-manipulation)

[How to remove a specific ELF section, without stripping other symbols?](https://stackoverflow.com/questions/31453859/how-to-remove-a-specific-elf-section-without-stripping-other-symbols)

