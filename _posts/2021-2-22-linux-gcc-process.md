---
layout: post
title: Linking 链接的过程
tags: [linux]
---

### 编译链接的过程

linking是将分散的代码包合并到可以被加载到内存中执行的可执行文件的处理过程；

假如有a.c及b.c两个c代码文件，一般编译用gcc就很简单
```
gcc a.c b.c
```
将上述步骤拆解如下
```
# 预处理，生成预处理文件
cpp other-command-line options a.c /tmp/a.i
# 编译器，生成汇编文件
cc1 other-command-line options /tmp/a.i  -o /tmp/a.s
# 汇编器，生成目标文件object
as other-command-line options /tmp/a.s  -o /tmp/a.o
```
cpp/cc1/as 是gnu标准预处理器、编译器、汇编器，它们都是gcc发行版本的一部分。  

同样的道理，对b.c也是执行上面的三步，生成目标文件，得到b.o的目标文件；  
```
# 链接器的作用就是将目标文件链接，最终生成可执行文件
ld other-command-line-options /tmp/a.o /tmp/b.o -o a.out

```
生成的a.out就是可执行文件了，通过shell可以直接执行
```
./a.out
```
shell将会调用loader加载器，加载a.out中的内容到内存中，并跳转到a.out的程序入口处，开始执行用户程序。

![compiler_driver.png](http://www.mrzzjiy.cn/assets/compiler_driver.png)

### Linkers vs Loaders   
链接器与加载器的作用是息息相关的；  

* 程序加载，将程序从磁盘加载到内存以便程序能够被执行；有些情况下，加载器会处理存储空间申请及虚拟内存映射磁盘页。
* 重定位，目标文件起始地址都是0；重定位的目的就是将同类型的不同部分合并为同一个segment并为它们分配加载地址；代码及数据section也会进行调整。
* 符号定位，符号可以简单理解为一个全局变量名，或者一个函数名； 连接器定位符号是通过记录符号位置，并修正调用者的代码引用。

链接器和加载器有相当大部分的重叠功能，加载器进行程序加载，链接器进行符号定位，而重定位两种都可以。
目标文件分为三种类型：  
* 可重定位的目标文件， 包含二进制代码及数据，并且可以与其他可重定位目标文件在编译时合并，生成可执行文件；
* 可执行目标文件， 可加载进内存并执行；
* 共享目标文件，可以加载进内存，并且可以在加载或者运行时进行动态链接。

系统不同目标文件的格式不同，window NT使用PE格式，IBM使用IBM 360格式，linux及solaris使用elf格式；
如下是elf可重定位目标文件中多种sections的简介：

* .text 程序的机器代码 ro；
* .rodata 只读数据，程序中的一些常量string等；
* .data  初始化的全局变量, rw;
* .bss 未初始化的全局变量,在目标文件中，这个段并不占据实际空间，它仅仅只是一个占位符; 
* .symtab 符号表，是包含了函数及全局变量的符号位置表，函数中的临时变量是不包含在这个表中，它们是由栈来管理的。
* .rel.text 链接的时候.text中需要重定位的位置列表；
* .rel.data 当前模块中已引用但未定义的全局变量信息，需要重定位；
* .debug debug符号列表，编译器指定了-g选项才会有这一项；
* .line 原始代码的行号与机器码之前的映射；
* .strtab .symtab和.debug的字符串表；

### 符号及符号定位

每一个可重定位的目标文件都有符号表及关联的符号；在链接器中，可能会出现如下几种符号：  
* 当前模块定义的全局变量并被其他模块引用的. 所有非静态的函数及全局变量属于这一类。
* 输入模块引用的全局变量且在别处定义。所有函数或变量被extern修饰的
* 本地符号被输入模块引用的. 所有静态函数及静态变量属于这一类。

linker中出现的符号类型，有栈管理的函数临时变量不会出现在linker中。
![link_symbols](http://www.mrzzjiy.cn/assets/link_symbols.png)


链接器通过将每个引用都关联到可重定位目标文件中的符号表中某一项定义来进行符号解析。解析一个模块的本地符号是很简单的，因为同一个模块不能定义相同的符号。解析全局符号要稍微复杂一点，在编译时，编译器将每个全局符号导出为强或弱符号。函数和初始化的全局变量是强符号，而全局为初始化的变量是弱符号。链接器使用如下规则来解析符号：
* 多个强符号是不允许的，即相同的强符号是不允许的；
* 一个强符号多个弱符号，选择强符号；
* 多个弱符号，任意选择一个；

这里的规则需要举个例子说明下：  如果两个c文件都定义了int foo() {}函数，而这个时候编译是会报错的，链接会报错，foo这个强符号重复定义了，是不允许的；理解了这个，其他两条规则也就很好理解了。

### 链接静态库  

静态库是一些类似目标文件的集合，这些文件是已归档文件的格式存储在磁盘上的。每一个elf归档文件都是以magic string: !<arch>\n 开头的。
编译的时候静态库通过命令行参数传递给连接器，并且只会复制程序引用的模块，而不会复制所有的静态库的内容。在unix系统中，libc.a包含了所有的c库函数，如printf fopen等。

```
gcc foo.o bar.o /usr/lib/libc.a /usr/lib/libm.a
```
libm.a是unix的数学库，包括sin\cos等数学计算函数等。  
连接器按顺序扫描命令行传入的可重定位文件或者静态库文件，扫描的过程中链接器会维护三个集合，分别是集合O 重定位文件需要加入到可执行文件中的，集合U 未解析出的符号集合，集合D 引入的模块已定义的符号集合。初始三个集合都是空的。

* 每个命令行的输入文件，链接器都会先判断是可重定位文件还是归档静态库文件；如果是可重定位文件，会放到O集合，并更新U/D两个集合，然后开始解析下一个文件
* 如果是归档静态库文件，链接器会扫描库中所有的模块，以匹配集合U未解析出的符号,如果在库中某个模块中匹配中了未定义的符号，则将该模块加入到集合O中，并更新集合U/D.
* 以上两步扫描完所有的文件后，如果集合U不是空的，则链接器会报错；如果是空的，则会将O集合中的所有模块重定位合并为可执行文件。


这也解释了为啥静态库文件放在了linker命令行的结尾。得特别小心库之间的循环依赖问题。静态库也是需要排序的，加入一个符号在多个库中都有定义，会使用第一个找到的库。

### 重定位

链接器解决了符号定位后，每一个符号都可以找到一个定义. 链接器就开始进行重定位，包含如下两个步骤。
* 重定位sections和符号定义，链接器merge所有相同的sections为一个大的section,比如将所有可重定位的文件的.data section merge为一个.data section。链接器会重新分配运行时内存地址给聚合后的sections、每个引入模块的section及每个符号。进过这一步之后，每个结构和全局变量都有一个加载时的地址了。
* 重定位section中符号引用，这一步，链接器会修改代码或数据中每一处的符号引用，指向加载时地址。

可重定位文件.o --> 可执行文件.exe
![relocating](http://www.mrzzjiy.cn/assets/relocating.png)

当汇编器碰到一个未定位的符号的时候，它会生成一个重定位的入口数据存入到.rel.text/.rel.data section中。重定位信息包含了如何定位到这个符号引用，或者说是这个符号的位置信息。一个典型的elf重定位信息包含了如下信息：  
* offset, section offset, 需要重定位的引用的区偏移量，对于可重定位文件，这个偏移量是从section开头的位置到引用的字节偏移量，可以理解为section存储位置开始的字节偏移量。
* 符号信息。
* 类型，重定位类型，通常是R_386_PC32，表示pc相对寻址；R_386_32表示绝对寻址。
```
Relocation section '.rela.text' at offset 0x2b8 contains 5 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000009  00050000000a R_X86_64_32       0000000000000000 .rodata + 0
00000000000e  000a00000002 R_X86_64_PC32     0000000000000000 puts - 4
00000000001a  000b00000002 R_X86_64_PC32     0000000000000000 hellow - 4
```

linker遍历所有的可重定位项，计算需要重定位的符号的准确的loading time地址或者说运行时地址
```
foreach section s {              //iterate over each section
    foreach relocation entry r { //iterate over each reloc entry in each sec
        refptr=s+r.offset;       //ptr to ref to be relocated
        
        //relocate a PC-relative ref
        if(r.type == R_386_PC32) {
            //ADDR(s): run-time addr for the sec
            refaddr = ADDR(s)+r.offset; //ref's runtime addr
            //ADDR(r.symbol): run-time addr of the symbol
            *refptr = (unsigned) (ADDR(r.symbol)) + *refptr - refaddr);
        }
    
        //relocate an absolute ref
        if(r.type == R_386_32)
            *refptr = (unsigned) (ADDR(r.symbol) + *refptr);
    }
}
```

### 动态链接：动态库

动态库是运行时进行链接运行的，可以被加载到内存中，如很常用printf/scanf等函数，都是通过动态库来共享的。
```
# 创建动态库
gcc -shared -fPIC -o libfoo.so a.o b.o
```
![sharedLD](http://www.mrzzjiy.cn/assets/sharedLD.jpeg)

以上命令创建了一个动态库libfoo.so, 由a.o\b.o两个模块组成，而-fPIC则告诉编译器生成位置无关的代码。

假设程序main.o模块依赖了libfoo.so,这个时候编译可以gcc main.o ./libfoo.so, 这个时候可执行文件中并不会有a.o/b.o模块，可执行文件只是简单的包含了一些需要重定位的项及.interp，linux上一般是ld-linux.so，同样是个动态库。而在执行的时候，loader加载器会将控制权转移给动态链接器，动态链接器会有一些初始启动的代码将动态库的内容映射到程序地址空间里面去。基本上有如下两个操作，这两个操作都是对已经加载到内存中的程序进行操作：
* 重定位动态库的text/data
* 重定位所有引用了动态库的符号
完成上述操作后，动态链接器将控制权转移给程序，开始执行程序main。

从应用中加载动态库   
应用程序在执行中，也可以动态的选择加载动态库，编译的时候可以选择不指定动态库。linux提供了系统调用dlopen, dlsym and dlclose来加载、调用及关闭等操作。


### examples
如下为简单例子：  
main.c
```
#include <stdio.h>
#include "hello.h"

int main(void) {
	printf("hello, world \n");
	char s[20];
	hellow(s);
    printf("hi2,  %s",s);
    return 0;
}
```
hello.c
```
#include <string.h>

void hellow(char str[20]) {
	strcpy(str, "hello, wolrd1\n");
	return;
}
```
hello.h
```
void hellow(char str[20]);
```

编译的过程
```
#预处理
cpp  -o main.i main.c
cpp -o hello.i hello.c

# 编译汇编.s
cc -S -o main.s main.i
cc -S -o hello.s hello.i

# 编译可重定位文件
as -o main.o main.s
as -o hello.o hello.s

# 链接成可执行文件，这里没有用ld，有一些静态依赖库需要解决，简单用gcc代替了哈，后续再研究下。
gcc -o hello main.o hello.o

```
看一眼main.o的rel内容，需要重定位的项，hellow/printf都是需要重定位的，在main.c中这些符号都是未定义的
```
readelf -r main.o

Relocation section '.rela.text' at offset 0x2b8 contains 5 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000009  00050000000a R_X86_64_32       0000000000000000 .rodata + 0
00000000000e  000a00000002 R_X86_64_PC32     0000000000000000 puts - 4
00000000001a  000b00000002 R_X86_64_PC32     0000000000000000 hellow - 4
000000000026  00050000000a R_X86_64_32       0000000000000000 .rodata + e
000000000030  000c00000002 R_X86_64_PC32     0000000000000000 printf - 4

Relocation section '.rela.eh_frame' at offset 0x330 contains 1 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000020  000200000002 R_X86_64_PC32     0000000000000000 .text + 0
```


### 参考文档

[Linker and Loader](https://www.linuxjournal.com/article/6463)