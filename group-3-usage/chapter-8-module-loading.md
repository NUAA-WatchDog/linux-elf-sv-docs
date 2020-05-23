---
description: 在内核启动后，手动挂载 ELF 签名验证模块。
---

# Chapter 8 - 挂载模块

## 8.1 编译模块

在 [模块源代码仓库](https://github.com/mrdrivingduck/linux-kernel-elf-sig-verify-module) 中，我们提供了 `Makefile` 用于编译该模块。

## 8.2 使用模块

使用 `insmod` 命令挂载模块，使用 `rmmod` 命令卸载模块：

```bash
$ sudo insmod binfmt_elf_signature_verification.ko
$ sudo rmmod binfmt_elf_signature_verification
```



