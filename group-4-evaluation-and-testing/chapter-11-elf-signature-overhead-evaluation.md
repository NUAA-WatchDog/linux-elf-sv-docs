---
description: 评估在 ELF 文件中附加签名后所带来的各类开销。
---

# Chapter 11 - ELF 签名开销评估

## 10.1 开销分析

ELF 签名程序所带来的开销主要来自于：

* 遍历 ELF 文件中的每个 section，将需要被签名的 section 装入内存
* 对装入内存的 section 计算摘要，并用公私钥生成数字签名
* 将数字签名作为新的 section 加入 ELF 文件中

## 10.2 时间开销

时间开销与 ELF 文件中的 section 个数，需要被签名的 section 个数呈正相关。

{% hint style="info" %}
在对每个被签名的 section 生成数字签名时，由于针对 section 计算出的摘要是一个定长的序列，因此用于计算数字签名的时间开销可以认为是固定的。
{% endhint %}

由于计算出的 section 摘要长度固定，并且长度较短，因此 ELF 文件的签名工作基本在每一台机器上都能瞬时完成。

## 10.3 空间开销

这里主要关注签名数据作为新的 section 被加入原 ELF 文件后，会使 ELF 文件大小增加多少。

上面已经提到，由于对一个 section 计算出的摘要是一个 **定长** 序列，那么计算得到的数字签名也是定长的。在使用 _SHA-256_ 和 _RSA-2048_ 作为签名工具链的条件下，我们测试得到每个 section 会为 ELF 文件带来 560 Bytes 左右的额外开销 \(签名数据 + section header 中新的 section 项 + string table section 中该 section 的名称\)。

对于现代计算机的硬件来说，这个空间开销基本上可以忽略。规模越大的 ELF 文件，开销所占的百分比越小。

