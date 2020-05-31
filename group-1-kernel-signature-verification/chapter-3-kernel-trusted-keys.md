---
description: 通过 Linux 的密钥保留服务，在内核代码中访问系统内置密钥，从而对 ELF 文件中的签名数据进行验证。
---

# Chapter 3 - 内核密钥保留服务

## 3.1 Linux 密钥保留服务

**Linux 密钥保留服务 \(Linux key retention service\)** 是在 Linux 2.6 中引入的，它的主要意图是在 Linux 内核中缓存身份验证数据。远程文件系统和其他内核服务可以使用这个服务来管理密码学、身份验证标记、跨域用户映射和其他安全问题。它还使 Linux 内核能够快速访问所需的密钥，并可以用来将密钥操作（比如添加、更新和删除）委托给用户空间。

Root 用户可以用过 proc 文件系统查看内核中的密钥：

```bash
$ cat /proc/keys
125ce30c I------     1 perm 1f030000     0     0 keyring   .dns_resolver: empty
155ea96d I------     1 perm 1f030000     0     0 keyring   .id_resolver: empty
178570f1 I------     1 perm 1f0b0000     0     0 keyring   .builtin_trusted_keys: 1
1ab86c77 I------     1 perm 1f030000     0     0 asymmetri sforshee: 00b28ddf47aef9cea7: X509.rsa []
1abf10d6 I--Q---     1 perm 1f3f0000     0 65534 keyring   _uid_ses.0: 1
390c087a I------     1 perm 1f0b0000     0     0 keyring   .builtin_regdb_keys: 1
3e099031 I--Q---     2 perm 1f3f0000     0 65534 keyring   _uid.0: empty
3e22b50c I------     1 perm 1f030000     0     0 asymmetri WatchDog: ELF verification: 7e0e1ac946e5350460497ba611a475534c9c3ec4: X509.rsa 4c9c3ec4 []
```

上述信息显示了公钥的序号 \(Serial number\)、类型 \(key-ring/asymmetry/...\)、状态、过期时间、描述等信息。

在编译内核时通过配置 `CONFIG_SYSTEM_TRUSTED_KEYS` 选项，引用 [PEM](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail) 格式的证书文件，就能够在内核的系统密钥环 \(`.system_keyring`\) 上添加额外的 [X.509](https://en.wikipedia.org/wiki/X.509) 公钥证书。关于该编译选项的说明如下：

{% code title="certs/Kconfig" %}
```text
config SYSTEM_TRUSTED_KEYS
	string "Additional X.509 keys for default system keyring"
	depends on SYSTEM_TRUSTED_KEYRING
	help
	  If set, this option should be the filename of a PEM-formatted file
	  containing trusted X.509 certificates to be included in the default
	  system keyring. Any certificate used for module signing is implicitly
	  also trusted.

	  NOTE: If you previously provided keys for the system keyring in the
	  form of DER-encoded *.x509 files in the top-level build directory,
	  those are no longer used. You will need to set this option instead.
```
{% endcode %}

在内核启动之后，也可以通过 keyctl 系统调用向内核中动态添加新的公钥。但内核只允许 X.509 封装信息已经被 `.system_keyring` 中已有密钥进行合法签名后的新密钥加入系统密钥环。

我们为 ELF 的签名与验证生成了一对 [RSA](https://en.wikipedia.org/wiki/RSA_%28cryptosystem%29) 公私钥，将公私钥以符合 X.509 标准的方式导入到一个 PEM 编码的证书中。并通过上述机制，将证书编译进内核的系统密钥环上。这样，在 [ELF 签名验证模块](chapter-1-binary-execution-procedure.md#15-dui-elf-wen-jian-jin-hang-qian-ming-yan-zheng-de-si-lu) 中，可以通过使用系统密钥环中的公钥，对 ELF 文件中的签名信息进行验证。

## 3.2 访问系统内置密钥进行签名验证

首先，我们对 Linux 内核中已有的 [内核模块签名](https://www.kernel.org/doc/html/v4.15/admin-guide/module-signing.html) 验证机制的代码进行了分析。在内核源代码目录 `certs/system_keyring.c` 中，定义了内核内置的受信密钥：

{% code title="certs/system\_keyring.c" %}
```c
static struct key *builtin_trusted_keys;
```
{% endcode %}

但由于这个变量没有被声明为 `extern`，因此无法在其它内核代码中直接引用这个变量。但是在这个源文件中，开放了 `verify_pkcs7_signature()` 函数，使得其它内核代码能够通过这个函数，间接使用内置密钥环的签名验证功能：

{% code title="certs/system\_keyring.c" %}
```c
/**
 * verify_pkcs7_signature - Verify a PKCS#7-based signature on system data.
 * @data: The data to be verified (NULL if expecting internal data).
 * @len: Size of @data.
 * @raw_pkcs7: The PKCS#7 message that is the signature.
 * @pkcs7_len: The size of @raw_pkcs7.
 * @trusted_keys: Trusted keys to use (NULL for builtin trusted keys only,
 *					(void *)1UL for all trusted keys).
 * @usage: The use to which the key is being put.
 * @view_content: Callback to gain access to content.
 * @ctx: Context for callback.
 */
int verify_pkcs7_signature(const void *data, size_t len,
			   const void *raw_pkcs7, size_t pkcs7_len,
			   struct key *trusted_keys,
			   enum key_being_used_for usage,
			   int (*view_content)(void *ctx,
					       const void *data, size_t len,
					       size_t asn1hdrlen),
			   void *ctx)
{
...
```
{% endcode %}

在内核代码中，通过 `#include <linux/verification.h>` 使用该函数时，输入 **签名数据** 与 **被签名数据** 的 **缓冲区内存地址** 和 **缓冲区长度**，就能够使用内置密钥完成签名认证。因此，[ELF 签名验证模块](chapter-1-binary-execution-procedure.md#15-dui-elf-wen-jian-jin-hang-qian-ming-yan-zheng-de-si-lu) 只要能够从 ELF 文件中正确提取 [PKCS \#7](https://tools.ietf.org/html/rfc2315) 格式的签名数据，以及签名保护的目标数据，就可以通过这个函数验证数字签名是否正确。

## 3.3 参考资料

[IBM Developer - Linux 密钥保留服务入门](https://www.ibm.com/developerworks/cn/linux/l-key-retention.html#artrelatedtopics)

[The Linux Kernel - Kernel module signing facility](https://www.kernel.org/doc/html/v4.15/admin-guide/module-signing.html)

[Signature verification of kernel module and kexec](https://www.slideshare.net/joeylikernel/signature-verification-of-kernel-module-and-kexec)

