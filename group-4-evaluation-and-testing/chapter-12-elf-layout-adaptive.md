# Chapter 12 - 适配不同的 ELF 文件布局

## 12.1 目的

ELF 文件的布局因编译器不同而大相径庭。由于签名程序需要对 ELF 文件中的结构进行调整，因此应当适配不同的 ELF 文件布局。在签名程序代码仓库的 `test/func/` [目录](https://github.com/NUAA-WatchDog/linux-elf-binary-signer/tree/master/test/func)下，我们试图收集布局尽可能不同的合法 ELF 文件，用于测试签名程序 `elf-sign` 的健壮性。

## 12.2 ELF by GCC

上述目录中的 `hello-gcc` 是一个由极其简单的 C 程序通过 GCC 编译后得到的 ELF 文件：

```c
#include <stdio.h>

int main()
{
    printf("Hello world\n");
    return 0;
}
```

该 ELF 文件的布局如下，特点为：

* Section header table 布局在所有 section 数据之后
* Section header string table 是布局在最后的 section

```bash
$ readelf -a hello-gcc
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x530
  Start of program headers:          64 (bytes into file)
  Start of section headers:          6440 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         29
  Section header string table index: 28

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000000238  00000238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000000254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000000274  00000274
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000000298  00000298
       000000000000001c  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000000002b8  000002b8
       00000000000000a8  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           0000000000000360  00000360
       0000000000000082  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           00000000000003e2  000003e2
       000000000000000e  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          00000000000003f0  000003f0
       0000000000000020  0000000000000000   A       6     1     8
  [ 9] .rela.dyn         RELA             0000000000000410  00000410
       00000000000000c0  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             00000000000004d0  000004d0
       0000000000000018  0000000000000018  AI       5    22     8
  [11] .init             PROGBITS         00000000000004e8  000004e8
       0000000000000017  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         0000000000000500  00000500
       0000000000000020  0000000000000010  AX       0     0     16
  [13] .plt.got          PROGBITS         0000000000000520  00000520
       0000000000000008  0000000000000008  AX       0     0     8
  [14] .text             PROGBITS         0000000000000530  00000530
       00000000000001a2  0000000000000000  AX       0     0     16
  [15] .fini             PROGBITS         00000000000006d4  000006d4
       0000000000000009  0000000000000000  AX       0     0     4
  [16] .rodata           PROGBITS         00000000000006e0  000006e0
       0000000000000011  0000000000000000   A       0     0     4
  [17] .eh_frame_hdr     PROGBITS         00000000000006f4  000006f4
       000000000000003c  0000000000000000   A       0     0     4
  [18] .eh_frame         PROGBITS         0000000000000730  00000730
       0000000000000108  0000000000000000   A       0     0     8
  [19] .init_array       INIT_ARRAY       0000000000200db8  00000db8
       0000000000000008  0000000000000008  WA       0     0     8
  [20] .fini_array       FINI_ARRAY       0000000000200dc0  00000dc0
       0000000000000008  0000000000000008  WA       0     0     8
  [21] .dynamic          DYNAMIC          0000000000200dc8  00000dc8
       00000000000001f0  0000000000000010  WA       6     0     8
  [22] .got              PROGBITS         0000000000200fb8  00000fb8
       0000000000000048  0000000000000008  WA       0     0     8
  [23] .data             PROGBITS         0000000000201000  00001000
       0000000000000010  0000000000000000  WA       0     0     8
  [24] .bss              NOBITS           0000000000201010  00001010
       0000000000000008  0000000000000000  WA       0     0     1
  [25] .comment          PROGBITS         0000000000000000  00001010
       0000000000000029  0000000000000001  MS       0     0     1
  [26] .symtab           SYMTAB           0000000000000000  00001040
       00000000000005e8  0000000000000018          27    43     8
  [27] .strtab           STRTAB           0000000000000000  00001628
       00000000000001ff  0000000000000000           0     0     1
  [28] .shstrtab         STRTAB           0000000000000000  00001827
       00000000000000fe  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)

There are no section groups in this file.

...
```

## 12.3 ELF by [Golang](https://golang.org/)

上述目录中的 `hello-golang` 是一个由极其简单的 Go 程序编译后得到的 ELF 文件：

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello world!")
}
```

该 ELF 文件的布局如下，特点为：

* Section header table 紧随 program header table 之后，布局在所有 section 数据之前
* Section header string table section 布局在其它 section 中间

```bash
$ readelf -a hello-golang
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x44f4d0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          456 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         7
  Size of section headers:           64 (bytes)
  Number of section headers:         23
  Section header string table index: 3

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000401000  00001000
       000000000008113b  0000000000000000  AX       0     0     16
  [ 2] .rodata           PROGBITS         0000000000483000  00083000
       0000000000041a3a  0000000000000000   A       0     0     32
  [ 3] .shstrtab         STRTAB           0000000000000000  000c4a40
       000000000000010b  0000000000000000           0     0     1
  [ 4] .typelink         PROGBITS         00000000004c4b60  000c4b60
       0000000000000b44  0000000000000000   A       0     0     32
  [ 5] .itablink         PROGBITS         00000000004c56a8  000c56a8
       0000000000000040  0000000000000000   A       0     0     8
  [ 6] .gosymtab         PROGBITS         00000000004c56e8  000c56e8
       0000000000000000  0000000000000000   A       0     0     1
  [ 7] .gopclntab        PROGBITS         00000000004c5700  000c5700
       000000000004ded8  0000000000000000   A       0     0     32
  [ 8] .noptrdata        PROGBITS         0000000000514000  00114000
       000000000000cbdc  0000000000000000  WA       0     0     32
  [ 9] .data             PROGBITS         0000000000520be0  00120be0
       0000000000006b10  0000000000000000  WA       0     0     32
  [10] .bss              NOBITS           0000000000527700  00127700
       000000000001c688  0000000000000000  WA       0     0     32
  [11] .noptrbss         NOBITS           0000000000543da0  00143da0
       0000000000002698  0000000000000000  WA       0     0     32
  [12] .debug_abbrev     PROGBITS         0000000000547000  00128000
       00000000000001b5  0000000000000000           0     0     1
  [13] .debug_line       PROGBITS         00000000005471b5  001281b5
       0000000000010539  0000000000000000           0     0     1
  [14] .debug_frame      PROGBITS         00000000005576ee  001386ee
       0000000000012054  0000000000000000           0     0     1
  [15] .debug_pubnames   PROGBITS         0000000000569742  0014a742
       0000000000007d9c  0000000000000000           0     0     1
  [16] .debug_pubtypes   PROGBITS         00000000005714de  001524de
       000000000000a50d  0000000000000000           0     0     1
  [17] .debug_gdb_script PROGBITS         000000000057b9eb  0015c9eb
       000000000000002d  0000000000000000           0     0     1
  [18] .debug_info       PROGBITS         000000000057ba18  0015ca18
       0000000000063dcf  0000000000000000           0     0     1
  [19] .debug_ranges     PROGBITS         00000000005df7e7  001c07e7
       0000000000005f80  0000000000000000           0     0     1
  [20] .note.go.buildid  NOTE             0000000000400f9c  00000f9c
       0000000000000064  0000000000000000   A       0     0     4
  [21] .symtab           SYMTAB           0000000000000000  001c7000
       0000000000012018  0000000000000018          22   101     8
  [22] .strtab           STRTAB           0000000000000000  001d9018
       00000000000121c4  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)

There are no section groups in this file.
```

## 12.4 更多 ELF 布局

后续我们将会收集更多不同布局的 ELF 文件，作为 `elf-sign` 程序的回归测试集合。

