---
description: 在内核启动后，手动挂载 ELF 签名验证模块。
---

# Chapter 8 - 挂载模块

## 8.1 编译模块

在 [模块源代码仓库](https://github.com/mrdrivingduck/linux-kernel-elf-sig-verify-module) 中，我们提供了 `Makefile` 用于编译该模块。编译模块时，需要使用到相对应的内核源代码目录。`Makefile` 中默认使用 **系统当前内核源码** 的路径 \(当然也可以指定其它的内核源码路径\)。

```bash
$ make
make -C /lib/modules/4.15.0+/build M=/home/mrdrivingduck/Desktop/linux-kernel-elf-sig-verify/linux-kernel-elf-sig-verify-module modules
make[1]: Entering directory '/home/mrdrivingduck/Desktop/linux-kernel-elf-sig-verify'
  CC [M]  /home/mrdrivingduck/Desktop/linux-kernel-elf-sig-verify/linux-kernel-elf-sig-verify-module/binfmt_elf_signature_verification.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/mrdrivingduck/Desktop/linux-kernel-elf-sig-verify/linux-kernel-elf-sig-verify-module/binfmt_elf_signature_verification.mod.o
  LD [M]  /home/mrdrivingduck/Desktop/linux-kernel-elf-sig-verify/linux-kernel-elf-sig-verify-module/binfmt_elf_signature_verification.ko
make[1]: Leaving directory '/home/mrdrivingduck/Desktop/linux-kernel-elf-sig-verify'
```

## 8.2 使用模块

使用 `modinfo` 命令查看模块附带信息：

```bash
$ modinfo binfmt_elf_signature_verification.ko
filename:       /home/mrdrivingduck/Desktop/linux-kernel-elf-sig-verify/linux-kernel-elf-sig-verify-module/binfmt_elf_signature_verification.ko
alias:          fs-binfmt_elf_signature_verification
version:        1.0
description:    Binary handler for verifying signature in ELF section
author:         zonghuaxiansheng <zonghuaxiansheng@outlook.com>
author:         mrdrivingduck <mrdrivingduck@gmail.com>
license:        Dual MIT/GPL
srcversion:     24C778301DE1DD13C1BB3CF
depends:
name:           binfmt_elf_signature_verification
vermagic:       4.15.0+ SMP mod_unload
```

使用 `insmod` 命令挂载模块：

```bash
$ sudo insmod binfmt_elf_signature_verification.ko
```

{% hint style="info" %}
如果内核中的模块签名验证选项 \(`CONFIG_MODULE_SIG`\) 被开启，则内核会在挂载模块之前，验证模块中的数字签名。如果模块签名验证失败，内核将会被标记为 tainted \(被污染\)。

因此，在挂载模块之前，需要使用内核源码树下的 `scripts/sign-file` 手动对模块进行签名。更多信息可见：[https://www.kernel.org/doc/html/v4.15/admin-guide/module-signing.html\#configuring-module-signing](https://www.kernel.org/doc/html/v4.15/admin-guide/module-signing.html#configuring-module-signing)
{% endhint %}

使用 `lsmod` 命令确认模块是否被装载：

```bash
$ lsmod | grep binfmt_elf_signature_verification
binfmt_elf_signature_verification    16384  0
```

使用 `rmmod` 命令卸载模块：

```bash
$ sudo rmmod binfmt_elf_signature_verification
```

## 参考资料

[Stackoverflow - How to compile a kernel module](https://stackoverflow.com/questions/37507320/how-to-compile-a-kernel-module)

[CSDN - 如何编译 Linux 内核模块](https://blog.csdn.net/u012247418/article/details/83684214)

[CSDN - Linux 内核模块查看命令](https://blog.csdn.net/zwmnhao1980/article/details/81029038)

