---
layout: post
title: 从 App 启动说起
date: 2017-07-22 11:30:00 +0800
comments: true
categories:
keywords: ios,启动,mach-o,dyld
description: WWDC 2016 Session 406 笔记
---

从点击 App 图标到 main() 执行之前，iOS 系统已经为我们做了很多事情，[WWDC 2016 Session 406](https://developer.apple.com/videos/play/wwdc2016/406/) 以优化启动时间为目的，向我们介绍了在这个过程里系统都做了什么。

## Mach-O

首先，App 的可执行文件是属于 [Mach-O](https://en.wikipedia.org/wiki/Mach-O) 格式的，这是一种用于记录可执行文件、对象代码、共享库、动态加载代码和内存转储的文件格式。Mach-O 文件一般都有一个 ```Header``` ，各种 ```Load Commands``` 以及多个  ```Segment```。其中每个  ```Segment``` 有固定的 page size，64 位系统是 16KB，而其他的都是 4KB。一个可执行文件通常都有以下  ```Segment```：```__PAGEZERO```，```__TEXT```，```__DATA```，```__LINKEDIT```。在每个 ```Segment``` 中又有多个  ```Section```。

安装 Xcode 后可以用命令行工具 ```xcrun``` 调用其他工具查看 Mach-O 信息，只是想简单的查看或编辑的话更推荐有 GUI 界面的 [MachOView](https://sourceforge.net/projects/machoview/)。

### Header

 ```Header ``` 位于 Mach-O 文件的开始位置，包含了该文件的基本信息，如目标架构，文件类型，加载指令数量和 dyld 加载时需要的标志位。

``` c
/*
 * The 64-bit mach header appears at the very beginning of object files for
 * 64-bit architectures.
 */
struct mach_header_64 {
  uint32_t      magic;      /* mach magic number identifier */
  cpu_type_t    cputype;    /* cpu specifier */
  cpu_subtype_t cpusubtype; /* machine specifier */
  uint32_t      filetype;   /* type of file */
  uint32_t      ncmds;      /* number of load commands */
  uint32_t      sizeofcmds; /* the size of all the load commands */
  uint32_t      flags;      /* flags */
  uint32_t      reserved;   /* reserved */
};
```

指示架构类型的是 ```magic``` 字段，64 位系统的值是 ```0xfeedfacf```，而 32 位系统对应的是 ```0xfeedface```。

### Load Commands

在 ```Header``` 之后的是各种加载指令，这些指令详细地定义了文件的逻辑结构和文件在虚拟内存中的布局，并且在加载解析时被内核加载器或动态连接器调用。

其中 ```LC_SEGMENT_64```（或 ```LC_SEGMENT```）指令类型，它们指示加载器如何加载对应的段到虚拟内存。在加载时，将会从 ```fileoff``` 的文件偏移位置，加载 ```fileszie``` 大小的内容映射到 ```vmaddr``` 处。

``` c
/*
 * The 64-bit segment load command indicates that a part of this file is to be
 * mapped into a 64-bit task's address space.  If the 64-bit segment has
 * sections then section_64 structures directly follow the 64-bit segment
 * command and their size is reflected in cmdsize.
 */
struct segment_command_64 { /* for 64-bit architectures */
  uint32_t  cmd;          /* LC_SEGMENT_64 */
  uint32_t  cmdsize;      /* includes sizeof section_64 structs */
  char      segname[16];  /* segment name */
  uint64_t  vmaddr;       /* memory address of this segment */
  uint64_t  vmsize;       /* memory size of this segment */
  uint64_t  fileoff;      /* file offset of this segment */
  uint64_t  filesize;     /* amount to map from the file */
  vm_prot_t maxprot;      /* maximum VM protection */
  vm_prot_t initprot;     /* initial VM protection */
  uint32_t  nsects;       /* number of sections in segment */
  uint32_t  flags;        /* flags */
};
```

其它的指令包括处理：

* dyld 的加载信息
* 符号表和动态符号表的位置和大小
* 引用的共享库（```LC_LOAD_DYLIB``` 指令类型）

### Segment

```__PAGEZERO``` 是第一个段，它的大小在 32 位系统是 4KB+，而在 64 位系统是 4GB+。位于虚拟内存从 0x000000 到 App 真实起始位置之间，都是被标记为不可读写和不可执行。因为里面并没有数据，所以 file size 为 0。主要的作用是为了捕获 NULL 指针使用和指针截断错误，防止引起系统崩溃。开发中常见的 ```EXC_BAD_ACCESS``` 异常都是因为错误访问到这里了。

```__TEXT``` 段包含了 Mach 头部，被执行的代码和只读常量，它被只读和可执行的方式映射。以下是几个常见的 section：

* ```__text``` 里就是程序编译后的机器码。
* ```__stubs``` 和 ```__stub_helper``` 是给动态链接器（dyld）使用的。
* ```__const``` 包含常量变量，所有不需要重定向的常量数据都会被编译器放在这里。
* ```__cstring``` 包含硬编码的字符串常量。

```__DATA``` 段包含了可读写的内容，全局变量和静态变量等，以可读写和不可执行的方式映射。常见的 section 有：

* ```__nl_symbol_ptr``` 和 ```__la_symbol_ptr``` 分别是 non-lazy 和 lazy 符号指针。lazy 符号指针用于可执行文件中调用未定义的函数，例如不包含在可执行文件中的函数，它们将会延迟加载。而针对 non-lazy 符号指针，当可执行文件被加载同时，也会被加载。
* ```__const``` 包含需要重定向的常量数据。
* ```__bss``` 包含没有被初始化的静态变量。

对于 iOS 或 macOS 应用还有一些以 ```__objc``` 开头的 section，例如 ```__objc_classlist``` 和 ```__objc_protolist``` 分别包含指向 Objective-C 的类和协议的指针。

```__LINKEDIT``` 段它的作用是包含如何加载整个文件的「元数据」。例如符号表，字符串表和重定位表。代码签名后每页的加密散列值也会存储到这里。

### Mach-O Universal 文件

[Universal Binary](https://en.wikipedia.org/wiki/Universal_binary) 的目的是把多种架构的 Mach-O 文件合并在一起。同时包含有一个称作 ```Fat Header``` 的结构，用于记录包含的架构类型和对应指令在文件中的偏移量。

## 虚拟内存

[虚拟内存](https://zh.wikipedia.org/wiki/%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98)通过间接寻址的方式让 Mach-O 文件可以被分页映射进物理内存里。这样，不同进程读取的相同页内容可以得到共享。而当某个想要读取的页没有在内存中就会触发 page fault，系统再通过调用 ```mmap()``` 读取指定页实现 lazy reading（懒读取）。

被划分后的内存页还可以根据设定的类型进行相应的权限校验。

对于有写权限的内存页，在需要执行写操作时，会用到 Copy-On-Write（COW）技术。原理就是当进程需要对这一页内容进行修改时，内核会把里面的内容先复制一份，然后把逻辑地址重新映射到新的物理内存去。这被修改过的内存页就叫 dirty page，其中包含着进程特定的信息。相对的 clean page 是可以被内核重新生成的。所以生成 dirty page 的代价远大于 clean page。

## 安全

苹果为了保障用户使用的应用都是安全无恶意的，为此使用了不同层级的保护机制。在应用启动过程中，主要涉及到的就是 ASLR 和 Code Signing。

### ASLR

[ASLR（地址空间布局随机化）](https://zh.wikipedia.org/wiki/%E4%BD%8D%E5%9D%80%E7%A9%BA%E9%96%93%E9%85%8D%E7%BD%AE%E9%9A%A8%E6%A9%9F%E8%BC%89%E5%85%A5)利用随机地址方式加载数据，以防止恶意程序对已知地址进行攻击。虽然是二十多年的技术了，但苹果从 iOS 4.3 后才正式启用。

### Code Signing

Code Signing（代码签名）可以让用户确信应用来自已知来源，并且自最后一次签名后未被修改。在实现上为了有更高的效率，用 ```codesign``` 设置签名时，每页的内容都会生成一个单独的加密散列值，并且储存到 ```__LINKEDIT``` 去，在加载时会校验每页的内容确保不被篡改。

## 从 exec() 到 main()

总的来说，应用的启动是从 ```exec()``` 这个系统调用开始，到 ```main()``` 之前内核会按顺序执行以下步骤：

1. 基于 ASLR 在随机初始位置把应用映射到新的地址空间
2. 接着从初始位置到 **0x00000** 的 **PAGEZERO** 区域置为不可读，不可写和不可执行
3. 用 dyld 加载动态链接库

下面就具体分析 dyld 的工作过程。

## dyld

dyld（dynamic loader）是苹果的动态链接器，当内核完成启动应用的初始准备后，就会将 dyld 映射到程序里的随机地址，并将 PC 寄存器设为该地址然后运行。dyld 将会负责加载应用依赖的所有动态链接库。

加载过程执行的步骤如下：

1. Load dylibs
2. Rebase
3. Bind
4. ObjC
5. Initializers

### Load dylibs

如上文提到，应用的执行文件在 **Load Commands** 中包含了所有依赖的 dylib 信息（```LC_LOAD_DYLIB```）。dyld 会解析这些信息找到对应的库，接着打开和读取它们的起始位置，确认它们是 Mach-O 文件，然后找到代码签名并且注册到内核。最后调用 ```mmap()``` 把里面的 ```Segment``` 全部加载进来。

因为应用依赖的 dylib 也会依赖其它的 dylib，所以 dyld 还需要递归地把全部依赖的 dylib 加载进来。平均一个应用会加载 100 到 400 个 dylib，但大部分都是系统的 dylib，它们会被预计算和预缓存，所以这些系统的 dylib 加载速度非常快。

### Fix-ups

刚被加载进来的 dylib 都是处于相对独立的状态，为了把它们绑定起来，需要经过一个修正（fix-ups）过程。

在 code-gen（编译器处理）时，通过动态 PIC（[Position Independent Code](https://zh.wikipedia.org/wiki/%E5%9C%B0%E5%9D%80%E6%97%A0%E5%85%B3%E4%BB%A3%E7%A0%81)）技术，本来因为代码签名限制不能再修改的代码，可以被加载到间接地址上。一个方法调用会在 ```__DATA``` 段创建一个指向被调用者的指针，程序会加载并且跳转到这个指针去。

所以 dyld 接着做的事情就是修正这些指针和数据。修正主要有两种类型：rebasing 和 binding。

### Rebasing

Rebasing 就是为内部指针加上一个偏移（**slide**）值。

ASLR 会将 dylib 加载到新的随机地址，这个随机地址跟代码和数据指向的旧地址会有偏差，dyld 就要将这些内部指针地址都加上一个偏移量作为修正，偏移量的计算方法如下：

```
Slide = actual_address - preferred_address
```

所有需要 rebase 的指针信息已经被编码到 **__LINKEDIT** segemnt 里。然后就是不断重复地对 __DATA segemnt 中需要 rebase 的指针加上这个偏移量。虽然这又会发生 page fault 和 COW，导致可能的 I/O 瓶颈，不过因为 rebase 是按地址排列进行的，所以从内核的角度来看这是个有次序的任务，它会预先读入数据，减少 I/O 消耗。

### Binding

Binding 负责处理指向 dylib 外部的指针。

外部指针实际上是与字符串符号名称绑定。在运行时，dyld 需要找到符号名对应的实现。而这需要很多计算，包括去符号表里找。找到后就会将对应的值记录到 ```__DATA``` segement 的那个指针里。Binding 的计算量虽然比 Rebasing 更多，但实际需要的 I/O 操作很少，因为之前 Rebasing 已经做过了。

### ObjC

在 ObjC 中有很多数据结构或类结构都已经靠 Rebasing 和 Binding 得到修正了。

但 ObjC 是一个动态语言，可以用类的名字来实例化一个类的对象，所以 ObjC 运行时需要维护一张用于注册类定义的全局表。每当一个 dylib 加载进来时，里面包含的所有类都需要被注册到这个全局表中。

相对于 C++ 有易碎基类（fragile base class）的问题，ObjC 会在加载时通过 fix-ups 动态修改类的实例变量的偏移量来避免。

在 ObjC 中还可以通过定义类别（Category）的方式为一个类添加方法。而有时添加方法的类在另一个 dylib 中，这时也需要做些额外修正。

另外，ObjC 中的 selector 必须是唯一的。

### Initializers

这一步做的是动态对 ```__Data``` 段作修正。

OC 的 ```+load```（建议改用 ```+initialize```） 方法可以像 C++ 的初始化器（initializer）执行一些初始化工作。因为之前的依赖关系，dyld 将会按照自底向上（bottom up）的顺序依次执行初始化器,确保加载某个 dylib 前，依赖的其余 dylib 肯定已经被预先加载。

最后 dyld 会调用 main() 函数。

总的来说， dyld 是一个辅助程序，用于加载依赖动态链接库，修正在 ```__DATA``` 段里的指针，最后调用所有初始化器。

## 减少启动时间

上文已经介绍了整个启动过程需要时间消耗的原因，苹果的工程师推荐最好把这个时间控制在 400ms 左右。下面就针对各个点进行优化。

### 测量启动时间

需要先启用 dyld 内建的方法获得 App 的启动时间：在 Xcode 中 **Edit scheme** -> **Run** -> **Auguments** -> **Environment Variables** 添加变量 ```DYLD_PRINT_STATISTICS``` 并设为 1。

启动类型分为两类：

* 暖启动（Warm launch）：App 和数据已经在内存里；
* 冷启动（Cold launch）：App 不在内核缓冲区缓存。

冷启动耗时需要重启设备才能测量，而且明显冷启动耗时要比暖启动多，所以前者的数据更为重要。

### Dylib oading 优化

上文已经提到系统 dylib 的加载已经被优化过，不过内嵌的（embedded）dylib 仍然很占用时间。另外，虽然可以用 ```dlopen()``` 方法实现懒加载，但实际上这会带来一些问题，而且总的消耗也更多。

关于这一点，可以：

* 合并 dylib
* 用 static archives
* 尽量不用 ```dlopen()``` 方法

### Rebase／Binding 优化

Rebasing 需要把时间花在 IO 上，而 binding 很少 IO 却有很大计算量。它们做的主要是修正 ```__Data``` 段里的指针，所以减少这些指针的数量也会减少对应消耗的时间，推荐的方法有：

* 减少 Objc 的元数据：包括 ```Class```，```selector```和 ```category``` 的数量。有些编程方式或设计模式更推荐短小的类和方法，其实这会不断地增加启动时间。
* 减少 C++ 的虚方法，因为虚方法会创建 [vtable]()，虽然不像 Objc 的元数据那么多，但这也会在 ```__Data` 段创建需要修正的结构体。
* 使用 Swift 的 ```struct``` 类型。Swift 更加内联，而且被很好的优化，所以迁移到 Swfit 会有一定的提升。
* 测试机器生成的代码。这些生成的代码可能会包含很大的结构体。

### Objc setup 优化

可以对这一步做的优化不多，因为前面的优化同时也会减少这里消耗的时间。

### Initializer 优化

对于两种初始化方法：

##### 显式初始化

* 推荐用 ```+initialize``` 代替 ```+load``，让初始化工作推迟到调用时才进行。
* C++ 的 ```__atribute__((constructor))``` 方法同样改为调用点初始化方法（call site initializer），例如使用 ```dispatch_once()```，```pthread_once()``` 或 ```std::once()``` 方法。

##### 隐式初始化：

对于 C++ 的静态复杂构造器：

* 最好也是改为调用点初始化方法
* 在构造器中只对 [POD（Plain Old Data）](https://zh.wikipedia.org/wiki/POD_(%E7%A8%8B%E5%BA%8F%E8%AE%BE%E8%AE%A1)) 类型赋值，静态链接器可以预先计算 ```__DATA``` 段中的数据，无需再进行修正工作。
* 使用编译器标志  ```-Wglobal-constructors``` 来发现隐式初始化代码。
* 使用 Swift 重写代码。前面也提到，因为 Swift 已经被很好得优化了。

dyld 运行在 App 开始之前，默认会是单线程并取消加锁，但如果在初始化方法里调用 ```dlopen()``` 就会开启多线程，然后 dyld 不得不加锁，导致性能严重受到影响，而且还有可能发生死锁或其它未知后果。所以也不要在初始化方法里创建线程。

## Reference

[Mach-O Executables](https://objccn.io/issue-6-3/)

[代码签名探析](https://objccn.io/issue-17-2/)

[优化 App 的启动时间](http://yulingtianxia.com/blog/2016/10/30/Optimizing-App-Startup-Time/)
