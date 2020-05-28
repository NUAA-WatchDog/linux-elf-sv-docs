---
description: 使用 ELF 文件签名程序，对一个未被签名的 ELF 文件进行签名。
---

# Chapter 9 - ELF 签名

## 8.1 ELF 文件签名程序的构建

首先，需要安装签名程序依赖的库。基于 Debian 系列的 Linux 发行版可以使用 **APT** \(Advanced Package Tool\) 工具轻易安装这些依赖：

```bash
$ sudo apt install libssl-dev
```

在 GitHub 上克隆 [ELF 文件签名程序](https://github.com/mrdrivingduck/linux-elf-binary-signer) 的代码仓库，通过 `make` 命令自动编译、构建签名程序，并对构建出的签名程序本身进行签名：

```bash
$ make
cc -o elf-sign elf_sign.c -lcrypto
cc -o sign-target sign_target.c
./elf-sign.signed sha256 certs/kernel_key.pem certs/kernel_key.pem elf-sign
 --- 64-bit ELF file, version 1 (CURRENT).
 --- Little endian.
 --- 29 sections detected.
 --- Section 0014 [.text] detected.
 --- Length of section [.text]: 10192
 --- Signature size of [.text]: 465
 --- Writing signature to file: .text_sig
 --- Removing temp signature file: .text_sig
```

{% hint style="info" %}
由于自行构建得到的 `elf-sign` 也是一个 ELF 程序，因此，在它可以用于对其它 ELF 文件进行签名之前，其自身必须先被签名，否则内核将拒绝执行这个 ELF 程序。在仓库中，我们提供了一个已经被测试密钥 \(`certs/kernel_key.pem`\) 签名后的签名程序 `elf-sign.signed`，并用这个签名程序对 `make` 命令构建的 `elf-sign` 进行签名。

如果您自行生成了私钥与公钥证书，那么您可能需要在一个 **没有 ELF 签名验证机制** 且 **确认安全** 的内核上，对自行构建得到的 `elf-sign` 进行自签名。然后，这个被签名后的签名程序可以使用在具有 ELF 签名验证机制的内核上。
{% endhint %}

如果一切正常，通过 `readelf` 或 `objdump` 命令，可以在构建成功的 `elf-sign` 中看到名为 `.text_sig` 的 section；而签名前被备份的原始版本 ELF 文件 `elf-sign.old` 中则没有这个 section：

```bash
$ readelf -a elf-sign
...
  [26] .text_sig         PROGBITS         0000000000000000  00004039
       00000000000001d1  0000000000000000           0     0     1
...
```

```bash
$ objdump -s elf-sign
...
Contents of section .text_sig:
 0000 308201cd 06092a86 4886f70d 010702a0  0.....*.H.......
 0010 8201be30 8201ba02 0101310d 300b0609  ...0......1.0...
 0020 60864801 65030402 01300b06 092a8648  `.H.e....0...*.H
 0030 86f70d01 07013182 01973082 01930201  ......1...0.....
 0040 01306e30 56311130 0f060355 040a0c08  .0n0V1.0...U....
 0050 57617463 68446f67 31193017 06035504  WatchDog1.0...U.
 0060 030c1045 4c462076 65726966 69636174  ...ELF verificat
 0070 696f6e31 26302406 092a8648 86f70d01  ion1&0$..*.H....
 0080 09011617 6d726472 6976696e 67647563  ....mrdrivingduc
 0090 6b40676d 61696c2e 636f6d02 144879f3  k@gmail.com..Hy.
 00a0 22f1671d f368169f 7f3ed0f8 4150a2cc  ".g..h...>..AP..
 00b0 86300b06 09608648 01650304 0201300d  .0...`.H.e....0.
 00c0 06092a86 4886f70d 01010105 00048201  ..*.H...........
 00d0 00ad85d9 206b0473 2742b31b 0a22e22f  .... k.s'B..."./
 00e0 27693ef2 c5b128d1 699adef7 0217b02d  'i>...(.i......-
 00f0 5296daf7 cc3bc6a6 b876928f a459d69f  R....;...v...Y..
 0100 625fc1ae dbcb383d f7070aad a41dd3a1  b_....8=........
 0110 820054d3 1c971e96 be2c858b cc439625  ..T......,...C.%
 0120 5e228ef2 623f2087 3ab349d9 4c3906db  ^"..b? .:.I.L9..
 0130 58ecbbac 43ccd826 8ca5bd8e 16194514  X...C..&......E.
 0140 021b8b5d 72a67370 bf5f33f3 a4b4a824  ...]r.sp._3....$
 0150 78505c7d 4ef054a6 2b622152 589008fe  xP\}N.T.+b!RX...
 0160 121d3d32 2aca10c3 4d6800c7 386887e4  ..=2*...Mh..8h..
 0170 af9b3649 c77a7813 40a75d55 64f69c2a  ..6I.zx.@.]Ud..*
 0180 51d34e1e cd242dec bda586b5 2027071b  Q.N..$-..... '..
 0190 8f1d903b c84197ee ce293e23 187fbf8c  ...;.A...)>#....
 01a0 5f7c3df3 4055130f c769a797 3b9b2b63  _|=.@U...i..;.+c
 01b0 a6f4754f 8e32036d d27243f5 a6a38017  ..uO.2.m.rC.....
 01c0 2745de68 5723b263 c17f9921 5571c6da  'E.hW#.c...!Uq..
 01d0 54                                   T
 ...
```

## 8.2 签名程序的使用方式

```bash
$ ./elf-sign
Usage: elf-sign [-ch] <hash-algo> <key> <x509> <elf-file> [<dest-file>]
  -c,         compact signing mode for old ELF binary
  -h,         display the help and exit

Sign the <elf-file> to an optional <dest-file> with
private key in <key> and public key certificate in <x509>
and the digest algorithm specified by <hash-algo>. If no
<dest-file> is specified, the <elf-file> will be backup to
<elf-file>.old, and the original <elf-file> will be signed.
```

`elf-sign` 的参数含义：

1. `hash-algo` - 摘要算法 \(可以选用其它内核内置支持的摘要算法\)
2. `key` - 存放用于签名的私钥的文件路径
3. `x509` - 存放用于签名的公钥证书的文件路径
4. `elf-file` - 待签名的目标 ELF 文件
5. `dest-file` \(可选\) - 签名后的输出文件名

比如，用测试证书中的 RSA-2048 私钥与 SHA-256 摘要算法，对一个名为 `sign-target` 的 ELF 文件进行签名：

```bash
$ ./elf-sign sha256 certs/kernel_key.pem certs/kernel_key.pem sign-target
 --- 64-bit ELF file, version 1 (CURRENT).
 --- Little endian.
 --- 29 sections detected.
 --- Section 0014 [.text] detected.
 --- Length of section [.text]: 418
 --- Signature size of [.text]: 465
 --- Writing signature to file: .text_sig
 --- Removing temp signature file: .text_sig
```

对于布局中只有 section header string table 而没有 symbol table 和 string table 的 ELF 程序，使用 [兼容模式选项](../group-2-elf-signer/chapter-5-elf-signature-injection.md#53-jian-rong-mo-shi) `-c` 来进行签名：

```bash
$ ./elf-sign -c sha256 certs/kernel_key.pem certs/kernel_key.pem /bin/cat mycat
 --- 64-bit ELF file, version 1 (CURRENT).
 --- Little endian.
 --- 30 sections detected.
 --- Section 0014 [.text] detected.
 --- Length of section [.text]: 16569
 --- Signature size of [.text]: 465
 --- Writing signature to file: .text_sig
 --- Removing temp signature file: .text_sig
```

## 8.3 参考资料

[Manually signing modules](https://www.kernel.org/doc/html/latest/admin-guide/module-signing.html#manually-signing-modules)

