# 运行环境与依赖

### Linux 发行版

当前解决方案确定可运行于 [Deepin](https://www.deepin.org/)、[UOS](https://www.chinauos.com/home)、[Ubuntu](https://ubuntu.com/) 等基于 Debian 的 Linux 操作系统发行版。

### Linux​ 内核版本

当前解决方案基于 Linux kernel 4.15.0 release。由于没有修改任何内核主线代码，因此可以快速扩展到更高版本的 Linux kernel 中。

### 依赖

本解决方案中，运行于用户空间的 ELF 数字签名程序需要解析 ELF 文件格式，还需要应用一些密码学的算法，因此需要一些 Debian 软件包作为依赖。基于 Debian 的 Linux 发行版系列可以使用 **APT** \(Advanced Package Tool\) 工具轻易安装这些依赖。签名程序依赖的软件包如下：

* `libssl-dev`

解决方案中内核部分的代码不能也不需要依赖任何第三方软件包。

