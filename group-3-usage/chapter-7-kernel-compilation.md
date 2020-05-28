---
description: 把用于 ELF 文件签名验证的公钥证书作为系统信任证书，编译到内核的内置密钥环中。
---

# Chapter 7 - 编译内核

## 7.1 内核编译配置

在 Linux 内核编译配置 `.config` 中，提供了 `SYSTEM_TRUST_KEYS` 的编译选项。其描述如下：

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

这个编译选项允许一个 PEM 格式的 X.509 证书被添加到系统默认的密钥环上。编辑这个选项，将其设置为 [PEM 格式公钥证书的路径](chapter-6-key-generation.md#62-mi-yao-sheng-cheng)：

{% code title=".config" %}
```text
#
# Certificates for signature checking
#
CONFIG_SYSTEM_TRUSTED_KEYRING=y
CONFIG_SYSTEM_TRUSTED_KEYS="<PATH_TO_CERT>/kernel_key.pem"
```
{% endcode %}

## 7.2 编译内核

完成上述选项的配置后，刷新配置并编译内核：

```bash
$ make oldconfig
$ make -j8
```

在编译过程中，应该可以看到如下信息：

```bash
  ...
  EXTRACT_CERTS   <PATH_TO_CERT>/kernel_key.pem
  AS      certs/system_certificates.o
  AR      certs/built-in.o
  ...
```

编译完成后，安装编译后的新内核：

```text
$ sudo make modules_install
$ sudo make install
```

重启电脑开机运行，查看 proc 文件系统中的 `/proc/keys` \(需要 root 权限\)。如果能够看到自行生成的密钥，那么说明该密钥已经被放置于内核的系统密钥环中。

```text
0c87ab47 I------     1 perm 1f030000     0     0 asymmetri WatchDog: ELF verification: 7e0e1ac946e5350460497ba611a475534c9c3ec4: X509.rsa 4c9c3ec4 []
```

## 7.3 参考资料

[Public keys in the kernel](https://www.kernel.org/doc/html/v4.15/admin-guide/module-signing.html#public-keys-in-the-kernel)

