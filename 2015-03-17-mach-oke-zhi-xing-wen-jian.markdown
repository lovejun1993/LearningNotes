---
layout: post
title: "Mach-O可执行文件"
date: 2015-03-17 12:53:37 +0800
comments: true
categories: 
---

###Mac OS X/iOS平台上可执行文件格式是: Mach-O

> 学习资料来源于: http://www.cocoachina.com/mac/20150122/10988.html

***

###每一种操作系统平台上都有对应的可执行文件格式

- Windows上`exe`是可直接执行的文件扩展名
- Linux（以及很多版本的Unix）系统上`ELF`是可直接执行的文件格式
- Mac OS X/iOS上，主要的可执行文件格式是`Mach-O`格式

了解Mach-O的格式，对于调试、自动化测试、安全都有意义

***

###Mach与Mach-O的区别:

- Mach: 是Mac OS X/iOS 操作系统内核
- Mach-O: 是在如上操作系统内核上可以直接运行的文件格式

****

###下载一个App并找到对应的`ipa`文件
	
- ipa文件实际上这只是一个变相的zip压缩包
- 可以把一个ipa文件直接通过unzip命令解压

***

###解压缩ipa包

![](http://i12.tietuku.cn/516dbef594655b2c.png)

***

###可以看解压之后的目录结构

![](http://i4.tietuku.cn/083c117c1a29d266.png)
	
- iTunesArtwork文件
- iTunesMetadata.plist文件
- META-INF目录
	- com.apple.FixedZipMetadata.bin文件
	- com.apple.ZipMetadata.plist文件
- Payload目录
	- Ape.app文件（实际上又是一个目录，或者说是一个完整的App Bundle）

***
		
###查看Ape.app包内容

![](http://i4.tietuku.cn/3197d7ca2c7c32e3.png)

![](http://i4.tietuku.cn/55bc026f8ae11201.png)

- 里面有App所需要的很多东西

	- 签名
	- 图片资源、数据资源文件
	- xib文件

- 这里主要看上面那个黑色的执行文件

![](http://i4.tietuku.cn/4ca21f75c37b03c0.png)

- 从右侧的文件属性可以看到，该文件的属性描述: `Unix executable - 15.8M`

	- 可执行文件
	- 大小是15.8M
	- 其余的大小全是图片资源占用

- 使用`file`命令来看一下这个可执行文件Ape

![](http://i4.tietuku.cn/46c4de6a3f478bff.png)


由此看来，这是一个支持armv7和arm64两种处理器架构的通用程序包，里面包含的两部分都是Mach-O格式.

****

###文件打开这个可执行的Ape文件，看里面的具体内容

```
cafe babe 0000 0002 0000 000c 0000 0009
0000 4000 006d 5130 0000 000e 0100 000c
0000 0000 006d c000 0083 2ab0 0000 000e
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000
0000 0000 0000 0000 0000 0000 0000 0000

.....还有好长，我打开的这个Ape文件，大概有9.8万行这样的数字
```

- 可以看到文件中的内容最开始部分，是以 `cafe babe`开头的

> 对于一个 二进制文件 来讲，每个类型都可以在文件最初几个字节来标识出来，即“魔数”。不同类型的 二进制文件，都有自己独特的"魔数"。

- OS X上，可执行文件的标识有这样几个魔数（不同的魔数代表不同的可执行文件类型）
	
	- cafebabe
	- feedface
	- feadfacf
	- 以 `#!` 开头的脚本

- cafebabe可执行格式:
	- 跨处理器架构的通用格式

- feedface和feedfacf可执行格式:
	- 分别只能针对`某一种处理器架构`下的Mach-O格式

- 脚本
	- `#!/bin/bash`开头的shell脚本

####大致了解了一个ipa包的目录结构，最重要的是.app文件中的Mach-O格式的可执行文件Ape

****

###接下来具体看下Mach-O格式的文件的文件结构

![](http://i12.tietuku.cn/f9adff6361b36c8b.png)

####Mach-o包含三个基本区域:

- 头部（header structure）

- 加载命令（load command）
	- 创建后续的所有Segment段

- 最终的数据（Data）
	- 创建所有的Section区
	- 并将Section区指定到上面创建出来的Segment段
	- 也就是一个Segment段包含多个Section区

####使用otool工具来查看Mach-O格式文件（上面的Ape文件）的头部（header structure）

```
otool -h  Ape
```

输出信息如下

```
xiongzenghuideMacBook-Pro:Ape.app xiongzenghui$ otool -h  Ape
Ape (architecture armv7):
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedface      12          9  0x00           2    52       5496 0x00218085
Ape (architecture arm64):
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf 16777228          0  0x00           2    52       6136 0x00218085
```
其文件格式:

```
Mach-O格式文件名字 (architecture 支持的cpu架构)
Mach header
		magic（魔数） cputype  cpusubtype  caps  filetype  ncmds  sizeofcmds  flags
```

上面主要的几个字段:

- filetype，这个可以有很多类型，静态库（.a）、单个目标文件（.o）都可以通过这个类型标识来区分
	- filetype = 2，代表`可执行`的文件
- ncmds和sizeofcmds，这个cmd就是加载命令，ncmds就是加载命令的个数，而sizeofcmds就是所占的大小

如上输出的Mach-O文件的头部信息，对应的c结构体类型定义:

```c
/*
 * The 32-bit mach header appears at the very beginning of the object file for
 * 32-bit architectures.
 */
struct mach_header {
	uint32_t	magic;		/* mach magic number identifier */
	cpu_type_t	cputype;	/* cpu specifier */
	cpu_subtype_t	cpusubtype;	/* machine specifier */
	uint32_t	filetype;	/* type of file */
	uint32_t	ncmds;		/* number of load commands */
	uint32_t	sizeofcmds;	/* the size of all the load commands */
	uint32_t	flags;		/* flags */
};

/* Constant for the magic field of the mach_header (32-bit architectures) */
#define	MH_MAGIC	0xfeedface	/* the mach magic number */
#define MH_CIGAM	0xcefaedfe	/* NXSwapInt(MH_MAGIC) */

/*
 * The 64-bit mach header appears at the very beginning of object files for
 * 64-bit architectures.
 */
struct mach_header_64 {
	uint32_t	magic;		/* mach magic number identifier */
	cpu_type_t	cputype;	/* cpu specifier */
	cpu_subtype_t	cpusubtype;	/* machine specifier */
	uint32_t	filetype;	/* type of file */
	uint32_t	ncmds;		/* number of load commands */
	uint32_t	sizeofcmds;	/* the size of all the load commands */
	uint32_t	flags;		/* flags */
	uint32_t	reserved;	/* reserved */
};

/* Constant for the magic field of the mach_header_64 (64-bit architectures) */
#define MH_MAGIC_64 0xfeedfacf /* the 64-bit mach magic number */
#define MH_CIGAM_64 0xcffaedfe /* NXSwapInt(MH_MAGIC_64) */
```

####使用otool工具来查看Mach-O格式文件（上面的Ape文件）的加载命令（load command）

```
otool -lv  Ape
```

输出信息如下

```
Load command 22
          cmd LC_LOAD_DYLIB
      cmdsize 84
         name /System/Library/Frameworks/Accelerate.framework/Accelerate (offset 24)
   time stamp 2 Thu Jan  1 08:00:02 1970
      current version 4.0.0
compatibility version 1.0.0
Load command 23
          cmd LC_LOAD_DYLIB
      cmdsize 80
         name /System/Library/Frameworks/Security.framework/Security (offset 24)
   time stamp 2 Thu Jan  1 08:00:02 1970
      current version 0.0.0
compatibility version 1.0.0

....

Section
  sectname __objc_data
   segname __DATA
      addr 0x00000001007c40f0
      size 0x000000000001eaa0
    offset 8143088
     align 2^3 (8)
    reloff 0
    nreloc 0
      type S_REGULAR
attributes (none)
 reserved1 0
 reserved2 0
 
 ....
```

1.cmd 是load command的类型,本文中值＝1就是LC_SEGMENT，,LC_SEGMENT的含义是(将文件中的段映射到进程地址空间)

2.cmdsize 代表load command的大小(0×58个字节)。

3.segname 16字节的段名字，当前是__PAGEZERO。

4.vmaddr 段的虚拟内存起始地址

5.vmsize 段的虚拟内存大小

6.fileoff 段在文件中的偏移量

7.filesize 段在文件中的大小

8.maxprot 段页面所需要的最高内存保护（4=r,2=w,1=x）

9.initprot 段页面初始的内存保护

10.nsects 段中包含section的数量

11.flags 其他杂项标志位

####对如上Section的c结构体定义:

```c
struct section { /* for 32-bit architectures */
	char		sectname[16];	/* name of this section */
	char		segname[16];	/* segment this section goes in */
	uint32_t	addr;		/* memory address of this section */
	uint32_t	size;		/* size in bytes of this section */
	uint32_t	offset;		/* file offset of this section */
	uint32_t	align;		/* section alignment (power of 2) */
	uint32_t	reloff;		/* file offset of relocation entries */
	uint32_t	nreloc;		/* number of relocation entries */
	uint32_t	flags;		/* flags (section type and attributes)*/
	uint32_t	reserved1;	/* reserved (for offset or index) */
	uint32_t	reserved2;	/* reserved (for count or sizeof) */
};

struct section_64 { /* for 64-bit architectures */
	char		sectname[16];	/* name of this section */
	char		segname[16];	/* segment this section goes in */
	uint64_t	addr;		/* memory address of this section */
	uint64_t	size;		/* size in bytes of this section */
	uint32_t	offset;		/* file offset of this section */
	uint32_t	align;		/* section alignment (power of 2) */
	uint32_t	reloff;		/* file offset of relocation entries */
	uint32_t	nreloc;		/* number of relocation entries */
	uint32_t	flags;		/* flags (section type and attributes)*/
	uint32_t	reserved1;	/* reserved (for offset or index) */
	uint32_t	reserved2;	/* reserved (for count or sizeof) */
	uint32_t	reserved3;	/* reserved */
};
```

1.sectname，section的名字
2.segname，该section所属的segment名字

```
Section
   sectname __text
   segname __TEXT
   ...
```

####常用的Segment、以及其所属的Section情况

- __TEXT

```
__text, __cstring, __picsymbol_stub, __symbol_stub, __const, __litera14, __litera18
```

- __DATA

```
__data, __la_symbol_ptr, __nl_symbol_ptr, __dyld, __const, __mod_init_func, __mod_term_func, __bss, __commom
```

- __IMPORT

```
__jump_table, __pointers
```

- __RESTRICT
- __LINKEDIT


> `__TEXT`段中的`__text`是实际上的代码部分；`__DATA`段的`__data`是实际的初始数据.

__TEXT段中的 __text格式例子

```
Section
  sectname __text
   segname __TEXT//TEXT则是主体代码段
      addr 0x0000000100006030
      size 0x000000000054b4f0
    offset 24624
     align 2^2 (4)
    reloff 0
    nreloc 0
      type S_REGULAR
attributes PURE_INSTRUCTIONS SOME_INSTRUCTIONS
 reserved1 0
 reserved2 0
```

Load command时创建初始化 __TEXT Segment（代码段）的例子

```
Load command 1
      cmd LC_SEGMENT_64
  cmdsize 952
  segname __TEXT//Segment名
   vmaddr 0x0000000100000000
   vmsize 0x000000000063c000
  fileoff 0
 filesize 6537216
  maxprot r-x//不包含w（写），即不能篡改代码
 initprot r-x
   nsects 11
    flags (none)
```

####__TEXT segment下的 常用section用途图示

![](http://i12.tietuku.cn/afe2e2ba2fd931e6.png)

####__DATA segment下的 常用section用途图示

####通过otool –s查看某segment的某个section

```
otool -s __TEXT __text Ape
```

####通过otool –t直接查看代码段（__TEXT）的 `反汇编`代码

```
otool -tv Ape
```

输出如下汇编形式的代码

```
00152262	    a045	adr	r0, #276
00152264	    95d7	str	r5, [sp, #0x35c]
00152266	    c134	stm	r1!, {r2, r4, r5}
00152268	    0667	lsls	r7, r4, #0x19
0015226a	    a41c	adr	r4, #112
0015226c	    35b8	adds	r5, #0xb8
0015226e	    a209	adr	r2, #36
00152270	    71e3	strb	r3, [r4, #0x7]
00152272	    095b	lsrs	r3, r3, #0x5
00152274	    1c36	adds	r6, r6, #0x0
00152276	    33ea	adds	r3, #0xea
00152278	f85886a4	.long	0xf85886a4
0015227c	    319f	adds	r1, #0x9f
0015227e	    4054	eors	r4, r2
00152280	    c55c	stm	r5!, {r2, r3, r4, r6}
00152282	    b01a	add	sp, #0x68
00152284	    13aa	asrs	r2, r5, #0xe
00152286	    826a	strh	r2, [r5, #0x12]
00152288	    b4eb	push	{r0, r1, r3, r5, r6, r7}
0015228a	    7bd6	ldrb	r6, [r2, #0xf]
0015228c	    bd9e	pop	{r1, r2, r3, r4, r7, pc}
0015228e	    db40	blt	0x152312
00152290	    086b	lsrs	r3, r5, #0x1
...

```

***

###小结:

- 将我们写好的App代码，最终编译成一个`二进制的Unix可执行文件`，其格式为Mach-O格式
	- iOS系统每次运行手机上的App，实际上就是直接运行ipa中的这个`二进制可执行文件`
- 该Mach-O可执行文件包含了整个App大部分东西
	- 程序代码
	- 各种资源文件
- 可以通过otool命令直接将ipa包中的Mach-O格式的可执行文件Ape，直接反编译成汇编代码