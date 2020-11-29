---
title: ELF文件格式解析
date: 2020-11-02
tags: 
    - ELF
    - 可执行文件格式
archives: 2020-11
author: Jiajie Li
summary: 讲解ELF文件格式
---

# ELF文件格式解析
编译器编译源代码之后生成的文件叫做**目标文件**。目标文件从结构上来说，和可执行文件已经没有区别了，只是其还没有经过链接过程，其中的部分符号和地址还没有被调整。

**可执行文件格式**涵盖了程序的编译、链接、装载和执行的方方面面，了解可执行文件格式对于认识操作系统、了解程序运行背后的机理很有帮助。

目前Linux平台上可执行文件格式是ELF(Executable and Linkable Format，类似Windows平台上的exe文件格式)，本文就对ELF文件的格式进行讲解。


示例程序
====
为了方便大家理解入门，这里以一个简单的C语言程序为例，来逐一对应ELF中的各部分：
```c
#include <stdio.h>

void say_hello(char *who)
{
    printf("hello, %s!\n", who);
}

char *my_name = "wb";

int main()
{
    say_hello(my_name);
    return 0;
}    
```
我们将其编译生成名为app的可执行程序并运行：
```shell
linux-XqAfQZ:~/tmp_analyze_elf # gcc -o app main.c
linux-XqAfQZ:~/tmp_analyze_elf # ./app
hello, wb!  
```

ELF整体布局
====
ELF格式可以表达四种类型的二进制对象文件(object files)：
  1. 可重定位文件(relocatable file，就是大家平常见到的.o文件)
  2. 可执行文件(executable file, 例上述示例代码生成的app文件)
  3. 共享库文件(shared object file，就是.so文件，用来做动态链接)
  4. 核心转储文件(core dump file，就是core dump文件)

可重定位文件用在编译和链接阶段；可执行文件用在程序运行阶段；共享库则同时用在编译链接和运行阶段。在不同阶段，我们可以用不同视角来理解ELF文件，整体布局如下图所示：

<img src = "./2020-11-03-ELF文件格式解析-1.jpg">

从上图可见，ELF格式文件整体可分为四大部分：
  * ELF Header, ELF头部，定义全局性信息
  * Program Header Table， 描述段(Segment)信息的数组，每个元素对应一个段；通常包含在可执行文件中，可重定文件中可选(通常不包含)
  * Segment and Section，段(Segment)由若干区(Section)组成；段在运行时被加载到进程地址空间中，包含在可执行文件中；区是段的组成单元，包含在可执行文件和可重定位文件中
  * Section Header Table，描述区(Section)信息的数组，每个元素对应一个区；通常包含在可重定位文件中，可执行文件中为可选(通常包含)



ELFHeader实例解析
====
ELF规范中对ELF Header中各字段的定义如下所示：

<img src = "./2020-11-03-ELF文件格式解析-2.jpg">

接下来我们通过readelf -h命令来看看示例程序app中的ELF Header内容，显示结果如下图：

<img src = "./2020-11-03-ELF文件格式解析-3.jpg">

  * **e_ident**含前16个字节，又可细分成class、data、version等字段，具体含义不用太关心，只需知道前4个字节点包含”ELF”关键字，这样可以判断当前文件是否是ELF格式；
  * **e_type**表示具体ELF类型，可重定位文件/可执行文件/共享库文件，显然这里是一个可执行文件；e_machine表示执行的机器平台，这里是x86_64；
  * **e_version**表示文件版本号，这里的1表示初始版本号；
  * **e_entry**对应”Entry point address”，程序入口函数地址，通过进程虚拟地址空间地址(0x400440)表达；
  * **e_phoff**对应“Start of program headers”，表示program header table在文件内的偏移位置，这里是从第64号字节(假设初始为0号字节)开始；
  * **e_shoff**对应”Start of section headers”，表示section header table在文件内的偏移位置，这里是从第4472号字节开始，靠近文件尾部；
  * **e_flags**表示与CPU处理器架构相关的信息，这里为零；
  * **e_ehsize**对应”Size of this header”，表示本ELF header自身的长度，这里为64个字节，回看前面的e_phoff为64，说明ELF header后紧跟着program header table；
  * **e_phentsize**对应“Size of program headers”，表示program header table中每个元素的大小，这里为56个字节；
  * **e_phnum**对应”Number of program headers”，表示program header table中元素个数，这里为9个；
  * **e_shentsize**对应”Size of section headers”，表示section header table中每个元素的大小，这里为64个字节；
  * **e_shnum**对应”Number of section headers”，表示section header table中元素的个数，这里为30个；
  * **e_shstrndx**对应”Section header string table index”，表示描述各section字符名称的string table在section header table中的下标，详见后文对string table的介绍。

Program Header Table实例解析
====
Program Header Table是一个数组，每个元素叫Program Header，规范对其结构定义如下：

<img src = "./2020-11-03-ELF文件格式解析-4.jpg">

同样我们用readelf -l命令查看示例程序的program header table：

<img src = "./2020-11-03-ELF文件格式解析-5.jpg">

上图截取了readelf命令返回的上半部，我们重点看下前面几项：
  * **PHDR**，此类型header元素描述了program header table自身的信息。从这里的内容看出，示例程序的program header table在文件中的偏移(Offset)为0x40，即64号字节处；该段映射到进程空间的虚拟地址(VirtAddr)为0x400040；PhysAddr暂时不用，其保持和VirtAddr一致；该段占用的文件大小FileSiz为00x1f8；运行时占用进程空间内存大小MemSiz也为0x1f8；Flags标记表示该段的读写权限，这里”R E”表示可读可执行，说明本段属于代码段；Align对齐为8，表明本段按8字节对齐。
  * **INTERP**，此类型header元素描述了一个特殊内存段，该段内存记录了动态加载解析器的访问路径字符串。示例程序中，该段内存位于文件偏移0x238处，即紧跟program header table；映射的进程虚拟地址空间地址为0x400238；文件长度和内存映射长度均为0x1c，即28个字符，具体内容为”/lib64/ld-linux-x86-64.so.2”；段属性为只读，并按字节对齐；
  * **LOAD**，此类型header元素描述了可加载到进程空间的代码段或数据段：第三项为代码段，文件内偏移为0，映射到进程地址0x400000处，代码段长度为0x764个字节，属性为只读可执行，段地址按2M边界对齐；第四段为数据段，文件内偏移为0xe10，映射到进程地址为0x600e10处(按2M对齐移动)，文件大小为0x230，内存大小为0x238(因为其内部包含了8字节的bss段，即未初始化数据段，该段内容不占文件空间，但在运行时需要为其分配空间并清零)，属性为读写，段地址也按2M边界对齐。
  * **DYNAMIC**，此类型header元素描述了动态加载段，其内部通常包含了一个名为”.dynamic”的动态加载区；这也是一个数组，每个元素描述了与动态加载相关的各方面信息，我们将在动态加载中介绍。该段是从文件偏移0xe28处开始，长度为0x1d0，并映射到进程的0x600e28；可见该段和上一个数据段是有重叠的。

readelf命令返回内容的下半部分给出了各段(segment)和各区(section)之间的包含关系，如下图所示。INTERP段只包含了”.interp”区；代码段包含”.interp”、”.plt”、”.text”等区；数据段包含”.dynamic”、”.data”、”.bss”等区；DYNAMIC段包含”.dynamic”区。从这里可以看出，有些区被包含在多个段中。

<img src = "./2020-11-03-ELF文件格式解析-6.jpg">

Section Header Table实例解析
====
针对各区的描述信息由Section Header Table提供，该数组中每个元素的定义如下：

<img src = "./2020-11-03-ELF文件格式解析-7.jpg">

下面我们再通过readelf -S命令看看示例程序中section header table的内容，如下图所示。示例程序共生成30个区，Name表示每个区的名字，Type表示每个区的功能，Address表示每个区的进程映射地址，Offset表示文件内偏移，Size表示区的大小，EntSize表示区中每个元素的大小(如果该区为一个数组的话，否则该值为0)，Flags表示每个区的属性(参见图中最后的说明)，Link和Info记录不同类型区的相关信息(不同类型含义不同，具体参见规范)，Align表示区的对齐单位。

<img src = "./2020-11-03-ELF文件格式解析-8.jpg">

String Table实例解析
====
从上述Section Header Table示例中，我们看到有一种类型为STRTAB的区(在Section Header Table中的下标为6,27,29)。此类区叫做String Table，其作用是集中记录字符串信息，其它区在需要使用字符串的时候，只需要记录字符串起始地址在该String Table表中的偏移即可，而无需包含整个字符串内容。

我们使用readelf -x读出下标27区的详细内容观察：

<img src = "./2020-11-03-ELF文件格式解析-9.jpg">

红框内为该区实际内容，左侧为区内偏移地址，后侧为对应内容的字符表示。我们可以发现，这里其实是一堆字符串，这些字符串对应的就是各个区的名字。因此section header table中每个元素的Name字段其实是这个string table的索引。再回头看看ELF header中的e_shstrndx，它的值正好就是27，指向了当前的string table。

同理再来看下29区的内容，如下图所示。这里我们看到了”main”、”say_hello”字符串，这些是我们在示例中源码中定义的符号，由此可以29区是应用自身的String Table，记录了应用使用的字符串。

<img src = "./2020-11-03-ELF文件格式解析-10.jpg">

Symbol Table实例解析
====
Section Header Table中，还有一类SYMTAB(DYNSYM)区，该区叫符号表。符号表中的每个元素对应一个符号，记录了每个符号对应的实际数值信息，通常用在重定位过程中或问题定位过程中，进程执行阶段并不加载符号表。符号表中每个元素定义如下：name表示符号对应的源码字符串，为对应String Table中的索引；value表示符号对应的数值；size表示符号对应数值的空间占用大小；info表示符号的相关信息，如符号类型(变量符号、函数符号)；shndx表示与该符号相关的区的索引，例如函数符号与对应的代码区相关。

<img src = "./2020-11-03-ELF文件格式解析-11.jpg">

我们用readelf -s读出示例程序中的符号表，如下图所示。如红框中内容所示，我们示例程序定义的main函数符号对应的数值为0x400554，其类型为FUNC，大小为26字节，对应的代码区在Section Header Table中的索引为13；say_hello函数符号对应数值为0x400530，其类型为FUNC，大小为36字节，对应的代码区也为13。

<img src = "./2020-11-03-ELF文件格式解析-12.jpg">

代码段实例解析
====
在理解了String Table和Symbol Table的作用后，我们通过objdump反汇编来理解一下.text代码段：

<img src = "./2020-11-03-ELF文件格式解析-13.jpg">

这里截取了与示例程序相关部分，我们看到0x400530和0x400554两处各定义一个函数，其符号分别为say_hello和main，这部分信息实际是通过符号表解析而来的；在涉及到内存地址的指令中，除了对数据段地址的引用是通过绝对地址进行的之外，对于代码段地址的引用都是以相对地址的方式进行的，这样做的好处是在二进制文件的重定位过程中，我们不用修改指令的访问地址(因为相对地址保持不变)；最后，我们看到对于库函数printf的访问指向了代码段地址0x400410，那么这个地址处放的是printf函数么？要回答这个问题就涉及**动态链接**，我们将在下文专题分析。

总结
====
通过以上的定义以及示例讲解，相信大家已经对ELF的文件格式有所了解了，如果想要继续深挖ELF文件的细节，大家可以参考以下这些资料。
1. https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
2. https://linux-audit.com/elf-binaries-on-linux-understanding-and-analysis/
3. https://refspecs.linuxfoundation.org/elf/elf.pdf