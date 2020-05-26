---
description: 借助 OpenSSL 工具，产生一对用于 ELF 文件签名和验证的公私钥。
---

# Chapter 6 - 密钥/证书生成

## 6.1 密钥生成的配置文件

Linux 内核中，已经给出了一个可以生成 X.509 格式 **自签名证书** 的配置文件模板 \(`certs/x509.genkey`\)：

```text
[ req ]
default_bits = 2048
distinguished_name = req_distinguished_name
prompt = no
string_mask = utf8only
x509_extensions = myexts

[ req_distinguished_name ]
O = WatchDog
CN = ELF verification
emailAddress = mrdrivingduck@gmail.com

[ myexts ]
basicConstraints=critical,CA:FALSE
keyUsage=digitalSignature
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid
```

配置信息的含义：

* `default_bits` 表示密钥长度
* `O` 表示密钥所属组织的名称
* `CN` 代表密钥的 common name
* `emailAddress` 为电子邮箱

## 6.2 密钥生成

有了 [上述配置文件](chapter-6-key-generation.md#61-mi-yao-sheng-cheng-de-pei-zhi-wen-jian) 之后，通过 OpenSSL 工具的支持，使用如下命令产生 **公私钥** 及 **证书**：

```bash
$ openssl req -new -nodes -utf8 -sha256 -days 36500 -batch -x509 \
    -config x509.genkey -outform PEM -out kernel_key.pem \
    -keyout kernel_key.pem
Generating a RSA private key
........+++++
........................................+++++
writing new private key to 'kernel_key.pem'
-----
```

命令将使用配置文件 `x509.genkey` 中的信息，产生一对 RSA 公私钥；生成一个名为 `kernel_key.pem` 的 PEM 格式的 X.509 自签名证书，过期日期为 36500 天后 \(相当于永不过期\)；生成的 RSA 私钥与公钥证书全部导入到 `kernel_key.pem` 中。

该文件可以作为 ELF 签名程序的输入 \(签名需要私钥和 X.509 格式的公钥证书\)，同时也用于编译到内核的系统内置公钥证书密钥环中。

{% hint style="info" %}
我们在 [**内核源代码仓库**](https://github.com/mrdrivingduck/linux-kernel-elf-sig-verify) 与 [**签名程序代码仓库**](https://github.com/mrdrivingduck/linux-elf-binary-signer) 放置了同一个 PEM 文件，其中的公私钥仅用于测试，请不要在生产环境中直接使用。[暴露私钥会导致所有的机制失效](https://www.kernel.org/doc/html/v4.15/admin-guide/module-signing.html#administering-protecting-the-private-key)。
{% endhint %}

## 6.3 参考资料

[OpenSSL 命令 - req](https://www.iteye.com/blog/ctwen-2028630)

[Generating signing keys](https://www.kernel.org/doc/html/v4.15/admin-guide/module-signing.html#generating-signing-keys)

[Administering/protecting the private key](https://www.kernel.org/doc/html/v4.15/admin-guide/module-signing.html#administering-protecting-the-private-key)

