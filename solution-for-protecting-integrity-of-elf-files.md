---
description: English version introduction.
---

# Solution for protecting integrity of ELF files on Linux

## Author

Jingt. Zhang \([@mrdrivingduck](https://github.com/mrdrivingduck)\), Nanjing University of Aeronautic and Astronautics \(NUAA\)

H. Zong \([@zonghuaxiansheng](https://github.com/zonghuaxiansheng)\), University of Science and Technology of China \(USTC\)

## Introduction

The solution aims at protecting integrity of ELF files with modern cryptography techniques, designed for operating systems based on Linux kernel under \(but not limited to\) Intel® x86 architecture. The solution consists of two parts:

1. An ELF signing program based on message digest and asymmetric encryption algorithm, in user space
2. A kernel module for verifying signature of ELF files based on Linux key retention service, in kernel space

Two parts are both implemented in GNU C, together with some Python / shell scripts for auditing, testing or batch job. All code will be available under the permission of [MIT License](https://www.mit-license.org/).

The following parts of this document will introduce the theory and the usage.

### Signing program for ELF files

The three main job for this program:

* Compute the message digest for instructions and data which is necessary for the execution of an ELF file
* Extract the public & private key from certificate in X.509 format, and compute digital signature with message digest
* Attach the digital signature to the original ELF file as verification information

{% hint style="info" %}
The attachment of digital signature should not break the format of the original ELF file, especially for the instructions and data which will be used by operating system. For an OS without verifying the integrity of an ELF file, it should also normally and correctly execute the ELF file.
{% endhint %}

Repository:

* [ELF binary signer](https://github.com/NUAA-WatchDog/linux-elf-binary-signer)

### Mechanism for verifying integrity of ELF file inside the kernel

When the OS executes an ELF file, the kernel will firstly load the ELF file into memory, parse and extract the information under protection \(instructions, data\) and corresponding digital signatures. Then, the kernel will compute the message digest, decrypt the signature with kernel-trusted public key, and compare the plain-text signature to the digest. If they are exactly the same, it means the ELF file is not tampered, the kernel will move on to the preparation for execution; if they are different from each other, it means the ELF file is tampered, the kernel will refuse to run this ELF file, for the sake of security.

{% hint style="info" %}
The above-mentioned actions done by the kernel are completely transparent to users. After inputting the command to run an ELF file, the user doesn't need to perform any more actions. The final execution result should only be one of the following:

1. The kernel executes the ELF file normally, and output a result as expected.
2. The kernel refuses to execute an ELF file, and shows the reason of the error.
{% endhint %}

Repositories:

* [The kernel source tree](https://github.com/NUAA-WatchDog/linux-kernel-elf-sig-verify) \(Linux kernel 4.15.0 release\)
* [Loadable standalone kernel module for ELF file verification](https://github.com/NUAA-WatchDog/linux-kernel-elf-sig-verify-module)

### Design and Documentations

This document is under the management of both [GitHub](https://github.com/) and [GitBook](https://www.gitbook.com/).

Repositories:

* [Online document on GitBook](https://app.gitbook.com/@mrdrivingduck/s/linux-elf-sv/)
* [Document on GitHub repository](https://github.com/NUAA-WatchDog/linux-elf-sv-docs)

