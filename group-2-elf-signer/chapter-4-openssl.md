---
description: '通过 OpenSSL 库以及一对公私钥，应用指定的摘要算法和加密算法，产生 PKCS #7 格式的数字签名。'
---

# Chapter 4 - 摘要与 PKCS \#7 签名

## 4.1 数字摘要

数字摘要能够将任意长度的消息变成固定长度的短消息。数字摘要就是采用单向 hash 函数将需要加密的明文 **摘要** 成一串固定长度的密文，称为数字指纹。不同的明文摘要成密文，其结果总是不同的；而对于同样的明文，摘要必定一致。 一个 hash 函数的好坏是由发生碰撞的概率决定的。如果攻击者能够轻易地构造出两个具有相同 hash 值的消息，那么这样的 hash 函数是很危险的。

一些常见的摘要算法有：

* SHA-1
* SHA-224
* SHA-256
* SHA-384
* SHA-512

## 4.2 数字签名

数字签名 \(又称公钥数字签名\) 是只有信息的发送者才能产生的，别人无法伪造的一段数字串。这段数字串同时也是对信息发送者发送信息真实性的有效证明。它是一种类似写在纸上的普通的物理签名，但是使用了公钥加密领域的技术实现，用于鉴别数字信息。一套数字签名通常定义两种互补的运算：

* 签名
* 验证

数字签名是 **非对称密钥加密技术** 与 **数字摘要技术** 的应用。发送报文时，发送方用一个 hash 函数从报文文本中生成摘要，然后用发送方的私钥对这个摘要进行加密，这个加密后的摘要将作为报文的数字签名和报文一起发送给接收方；接收方使用与发送方相同的 hash 函数从接收到的原始报文中计算出报文摘要，再使用发送方的公钥对报文附加的数字签名进行解密。如果这两个摘要相同，那么接收方就能确认原始报文未经篡改。

## 4.3 PKCS \#7

[PKCS \(Public Key Cryptography Standards\) \#7](https://tools.ietf.org/html/rfc2315) 被命名为 [CMS](https://en.wikipedia.org/wiki/Cryptographic_Message_Syntax) \(Cryptographic Message Syntax Standard\)，是 [RSA 公司](https://www.rsa.com/) 提出的最著名的标准。这一标准也是 [S/MIME](https://en.wikipedia.org/wiki/S/MIME) \(Secure/Multipurpose Internet Mail Extensions\) 的基础。PKCS \#7 提供了一种用途广泛的创建数字签名的语法和格式，其中包含的信息有：

* 产生签名的 hash 算法
* Detached / Attached 格式
* 签名公钥证书
* 签名者信息
* 证书签发者信息
* 认证属性 \(签名时间、消息摘要等\)
* 加密算法
* 签名数据

Linux 内核中的 `verify_pkcs7_signature()` 函数能够以 PKCS \#7 的格式解析签名，并使用内核内置密钥验证签名。

## 4.4 产生 ELF 文件的签名

ELF 文件签名程序需要使用 OpenSSL 库函数。首先，通过 OpenSSL 的函数，实例化所有的加密算法 \(`OpenSSL_add_all_algorithms()`\) 和摘要算法 \(`OpenSSL_add_all_digests()`\)。然后，根据命令行参数指定的摘要算法、公钥证书文件路径、私钥文件路径，分别获得摘要算法实例 \(`EVP_get_digestbyname(hash_algo)`\)、读取私钥 \(`read_private_key(private_key_name)`\)、读取公钥证书 \(`read_x509(x509_name)`\)。最终，通过以下的代码，产生最终的数字签名：

{% code title="elf\_sign.c" %}
```c
#ifndef USE_PKCS7
	/* Load the signature message from the digest buffer. */
	cms = CMS_sign(NULL, NULL, NULL, NULL,
				CMS_NOCERTS | CMS_PARTIAL | CMS_BINARY |
				CMS_DETACHED | CMS_STREAM);
	ERR(!cms, "CMS_sign");

	ERR(!CMS_add1_signer(cms, x509, private_key, digest_algo,
					CMS_NOCERTS | CMS_BINARY |
					CMS_NOSMIMECAP | use_keyid |
					use_signed_attrs),
		"CMS_add1_signer");
	ERR(CMS_final(cms, bm, NULL, CMS_NOCERTS | CMS_BINARY) < 0,
		"CMS_final");

#else
	pkcs7 = PKCS7_sign(x509, private_key, NULL, bm,
				PKCS7_NOCERTS | PKCS7_BINARY |
				PKCS7_DETACHED | use_signed_attrs);
	ERR(!pkcs7, "PKCS7_sign");
#endif
```
{% endcode %}

产生的数字签名首先会被导入到一个临时文件中。并在随后被注入到 ELF 文件中，称为 ELF 文件中的一个新的 section。

## 4.5 参考资料

[Manually signing modules](https://www.kernel.org/doc/html/latest/admin-guide/module-signing.html#manually-signing-modules)

[PKCS \#7: Cryptographic Message Syntax Version 1.5](https://tools.ietf.org/html/rfc2315)

[Digital signature - Wikipedia](https://en.wikipedia.org/wiki/Digital_signature)

[What Is a Digital Signature?](https://www.instantssl.com/digital-signature)

[Introduction to Digital Signatures and PKCS \#7](https://www.cryptomathic.com/news-events/blog/introduction-to-digital-signatures-and-pkcs-7)

[PKCS\#7 - SignedData](http://www.pkiglobe.org/pkcs7.html)

[OpenSSL 之 BIO 系列之15 --- 内存 \(mem\) 类型 BIO](https://blog.csdn.net/fryingpan/article/details/40374813)

[OpenSSL 中文手册之 BIO 库详解](https://blog.csdn.net/liao20081228/article/details/77193729)

