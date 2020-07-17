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

其中可自行修改的配置信息及其含义：

* `default_bits` 表示密钥长度 \(2048-bit / 4096-bit / ...\)
* `O` 表示密钥所属组织的名称
* `CN` 代表密钥的通用名称
* `emailAddress` 代表密钥所属组织的电子邮箱

## 6.2 密钥生成

有了 [上述配置文件](chapter-6-key-generation.md#61-mi-yao-sheng-cheng-de-pei-zhi-wen-jian) 之后，通过 OpenSSL 工具，使用如下命令产生 **公私钥** 并导入 **证书**：

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

命令将使用配置文件 `x509.genkey` 中的信息，产生一对 RSA 公私钥；生成一个名为 `kernel_key.pem` 的 PEM 格式的 X.509 自签名证书，过期日期为 36500 天后 \(永不过期\)；生成的 RSA 私钥与公钥证书全部导入到 `kernel_key.pem` 中。

该文件可以作为 ELF 签名程序的输入 \(签名需要私钥和 X.509 格式的公钥证书\)，同时也是编译内核时 `CONFIG_SYSTEM_TRUSTED_KEYS` 选项指向的文件。

{% hint style="info" %}
我们在 [签名验证内核模块代码仓库](https://github.com/mrdrivingduck/linux-kernel-elf-sig-verify-module) 与 [签名程序代码仓库](https://github.com/mrdrivingduck/linux-elf-binary-signer) 中的 `certs/` 目录下放置了同一个 PEM 文件，其中的公私钥仅用于测试，请不要在生产环境中直接使用。[暴露私钥会导致所有的机制失效](https://www.kernel.org/doc/html/v4.15/admin-guide/module-signing.html#administering-protecting-the-private-key)。

可自行修改上述 `openssl` 命令参数，将 RSA 私钥与公钥证书保存到不同的文件中。编译内核时，只需要用到公钥证书；对 ELF 文件签名时，同时需要私钥和公钥证书。
{% endhint %}

## 6.3 Let's Encrypt 密钥与证书

[Let's Encrypt](https://letsencrypt.org/) 是一个由非营利性组织 - 互联网安全研究小组 \(ISRG\) 提供的免费、自动化和开放的证书颁发机构 \(CA\)，能够为网站提供免费的 [SSL/TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) 数字证书，以促进 Web 的安全化发展。要从 Let’s Encrypt 获取网站域名的证书，用户必须 **证明对域名的实际控制权**，并在 Web 主机上运行使用 **ACME 协议** 的软件来获取 Let’s Encrypt 证书。

Let's Encrypt 官方推荐的证书签发工具是 [Certbot](https://certbot.eff.org/)。只需要满足以下三个要求，该工具就能自动为用户生成有效期为三个月的数字公钥证书与私钥，并配置一个 [cron](https://baike.baidu.com/item/crontab/8819388?fr=aladdin) 定时任务在证书过期前自动刷新证书：

1. 一台开放 `80` 端口的服务器
2. 一个合法域名，且已被 [DNS](https://en.wikipedia.org/wiki/Domain_Name_System) 解析到该服务器
3. 用户对该服务器有 `sudo` 控制权

经过测试，Let's Encrypt 签发的公钥证书 `cert.pem` 和私钥 `privkey.pem` 完全能够在本解决方案中使用。用户只需要将公钥证书 [与内核一起编译](chapter-7-kernel-compilation.md)，再 [使用公钥证书与私钥共同对 ELF 进行签名](chapter-9-elf-sign.md) 即可。

## 6.4 参考资料

[OpenSSL 命令 - req](https://www.iteye.com/blog/ctwen-2028630)

[Generating signing keys](https://www.kernel.org/doc/html/v4.15/admin-guide/module-signing.html#generating-signing-keys)

[Administering/protecting the private key](https://www.kernel.org/doc/html/v4.15/admin-guide/module-signing.html#administering-protecting-the-private-key)

[About Certbot](https://certbot.eff.org/about/)

[GitHub - certbot](https://github.com/certbot/certbot)

[Let's Encrypt](https://letsencrypt.org/)

[Mr Dk.'s blog - notes/Cryptography - Let's Encrypt](https://mrdrivingduck.github.io/blog/#/markdown?repo=notes&path=Cryptography%2FCryptography%20Let%27s%20Encrypt.md)

