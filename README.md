# Linux ELF 文件数据完整性保护系统

## 作者

张靖棠 \([@mrdrivingduck](https://github.com/mrdrivingduck)\)，南京航空航天大学，计算机科学与技术学院，网络空间安全专业

宗华 \([@zonghuaxiansheng](https://github.com/zonghuaxiansheng)\)，中国科学技术大学，计算机科学与技术学院，计算机技术专业

## 简介

本解决方案旨在通过密码学技术，对 \(包括但不限于\) Intel® x86 平台下基于 Linux 内核的操作系统中的 ELF 文件实现数据完整性保护。方案主要分为两个部分：

1. 用户空间下，基于信息摘要算法和非对称加密算法的 ELF 文件数字签名程序
2. 内核空间下，基于内核密钥保留服务的 ELF 文件数据完整性验证模块

上述两部分全部由 GNU C 语言实现，另有少量用于审计、测试或批处理的 Python / shell 脚本文件。所有代码将在 [MIT License](https://www.mit-license.org/) 的许可下开源。

文档的后续内容主要介绍这两个部分的实现原理，以及使用说明。

### ELF 文件数字签名程序

该程序主要包含三个主要工作：

* 对于 ELF 文件执行时所需要使用的指令、数据等信息，分别生成固定长度的 **信息摘要**
* 从 X.509 格式的证书中读取公私钥，根据摘要内容计算 **数字签名**
* 将数字签名附加到原 ELF 文件中，作为 ELF 完整性的验证信息

{% hint style="info" %}
数字签名的附加不能破坏原 ELF 文件的格式，尤其不能破坏会被操作系统使用的关键执行信息 \(指令、数据等\)。对于一个不检验 ELF 文件完整性的操作系统，也能够正常、正确地执行这个 ELF 文件。
{% endhint %}

代码仓库地址：

* ELF 二进制文件签名程序：[https://github.com/mrdrivingduck/linux-elf-binary-signer](https://github.com/mrdrivingduck/linux-elf-binary-signer)

### ELF 文件数据完整性验证内核模块

当操作系统执行一个 ELF 文件时，内核首先将 ELF 文件载入内存，解析出文件中的 **被保护信息** \(代码、数据等\) 和对应的数字签名。内核计算出被保护信息的摘要，再使用内核信任的内置密钥对数字签名进行解密，比对解密后的签名与信息摘要是否完全一致。如果数据完全一致，说明 ELF 文件的被保护信息未经篡改，内核继续完成 ELF 文件开始执行前的准备工作；如果数据不一致，说明 ELF 文件遭到篡改，为了维护操作系统的安全性，内核拒绝继续执行这个 ELF 文件。

{% hint style="info" %}
内核所进行的上述动作对用户来说完全透明。用户在输入执行 ELF 文件的命令后，不需要任何额外的操作。最终的执行结果只可能为以下两种：

1. 内核正常执行 ELF 文件，输出预期结果
2. 内核拒绝执行 ELF 文件，并显示错误原因
{% endhint %}

代码仓库地址：

* 内核源码树 \(基于 Linux kernel 4.15.0 release\)：[https://github.com/mrdrivingduck/linux-kernel-elf-sig-verify](https://github.com/mrdrivingduck/linux-kernel-elf-sig-verify)
* 可动态装载的独立模块：[https://github.com/mrdrivingduck/linux-kernel-elf-sig-verify-module](https://github.com/mrdrivingduck/linux-kernel-elf-sig-verify-module)

