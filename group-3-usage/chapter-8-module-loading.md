---
description: 在内核启动后，手动挂载 ELF 签名验证模块。
---

# Chapter 8 - 挂载模块

## 8.1 编译模块

在 [模块源代码仓库](https://github.com/mrdrivingduck/linux-kernel-elf-sig-verify-module) 中，我们提供了 `Makefile` 用于编译该模块。编译模块时，需要使用到相应的内核源代码目录。`Makefile` 中默认使用 **当前内核源码** 的路径 \(当然也可以指定其它的内核源码路径\)。

## 8.2 使用模块

使用 `insmod` 命令挂载模块，使用 `rmmod` 命令卸载模块：

```bash
$ sudo insmod binfmt_elf_signature_verification.ko
$ sudo rmmod binfmt_elf_signature_verification
```

## 参考资料

[Stackoverflow - How to compile a kernel module](https://stackoverflow.com/questions/37507320/how-to-compile-a-kernel-module)

[CSDN - 如何编译 Linux 内核模块](https://blog.csdn.net/u012247418/article/details/83684214)

