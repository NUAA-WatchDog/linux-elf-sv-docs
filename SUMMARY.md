# Table of contents

* [Linux ELF 文件完整性保护解决方案](README.md)
* [背景](background.md)
* [运行环境与依赖](environment-and-dependencies.md)

## 第一部分 - 内核 ELF 签名验证机制 <a id="group-1-kernel-signature-verification"></a>

* [Chapter 1 - 二进制文件执行过程](group-1-kernel-signature-verification/chapter-1-binary-execution-procedure.md)
* [Chapter 2 - ELF 文件格式分析](group-1-kernel-signature-verification/chapter-2-elf-format-analysis.md)
* [Chapter 3 - 内核密钥保留服务](group-1-kernel-signature-verification/chapter-3-kernel-trusted-keys.md)

## 第二部分 - ELF 文件签名程序 <a id="group-2-elf-signer"></a>

* [Chapter 4 - 摘要与 PKCS \#7 签名](group-2-elf-signer/chapter-4-openssl.md)
* [Chapter 5 - ELF 签名注入](group-2-elf-signer/chapter-5-elf-signature-injection.md)

## 第三部分 - 使用方式 <a id="group-3-usage"></a>

* [Chapter 6 - 密钥/证书生成](group-3-usage/chapter-6-key-generation.md)
* [Chapter 7 - 编译内核](group-3-usage/chapter-7-kernel-compilation.md)
* [Chapter 8 - 挂载模块](group-3-usage/chapter-8-module-loading.md)
* [Chapter 9 - ELF 签名](group-3-usage/chapter-9-elf-sign.md)

## 第四部分 - 评估与测试 <a id="group-4-evaluation-and-testing"></a>

* [Chapter 10 - 内核签名验证开销评估](group-4-evaluation-and-testing/chapter-10-kernel-performance-overhead-evaluation.md)
* [Chapter 11 - ELF 签名开销评估](group-4-evaluation-and-testing/chapter-11-elf-signature-overhead-evaluation.md)

