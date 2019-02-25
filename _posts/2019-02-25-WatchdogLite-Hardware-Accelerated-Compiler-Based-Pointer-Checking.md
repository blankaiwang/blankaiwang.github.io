---
layout:     post
title:      "WatchdogLite: Hardware-Accelerated Compiler-Based Pointer Checking"
subtitle:   ""
author:     "blankaiwang"
header-img: "img/bg-3.jpg"
header-mask:  0.5
catalog: true
tags:

    - 读书笔记
    - Security
    - 边界检查

---



# WatchdogLite: Hardware-Accelerated Compiler-Based Pointer Checking



## 文章信息

Santosh Nagarakatte, Rutgers University,

Milo M.K. Martin, Steve Zdancewic, University of Pennsylvania,

CGO '14



## 简介

WatchdogLite提供了新的指令扩展，对基于编译器实现的指针检查方案提供硬件加速。通过对硬件和编译器进行分工，在已有架构寄存器的基础上实现了硬件加速。WatchdogLite利用编译器识别指针，去除多余的检查，插入新的指令。该方案实现了与之前通过添加新的硬件来追踪指针的安全方案相近的性能。



## 设计思想

WatchdogLite实现三项主要功能

1. 元数据读/写
2. 边界检查
3. UAF检查

实现的关键在于通过合理对编译器和硬件进行分工，避免额外的硬件状态及对应的硬件处理机制，并结合硬件和编译器的优点。

### 元数据读/写

`MetaLoad`和`MetaStore`指令负责从影子区域读/写指针所需的元数据。每次从内存中读取指针时，在读操作前插入一个`MetaLoad`，类似的，在写操作前插入`MetaStore`。`MetaLoad`和`MetaStore`中，指针对应的影子信息可以根据指针地址计算出来，也可以采用内存地址移位得到影子内存中地址的方式。

如果没有硬件的支持，主程序内存到影子区域的地址映射会带来高额的性能开销。`MetaLoad`和`MetaStore`将地址映射操作进行硬编码，并且在地址生成阶段就使用硬件进行对应的位操作，从而显著降低了性能开销。

![MetaLoad and MetaStore](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-25-WatchdogLite-Hardware-Accelerated-Compiler-Based-Pointer-Checking.assets/Figure%201.png)

### 边界检查

WatchdogLite引入了一个新的指令`SChk`进行边界检查。`SChk`替换了在x86下进行边界检查的5条指令（`cmp,br,lea,cmp,br`)，`SChk`需要三个参数

1. 被检查的地址
2. 基址(边界下界)
3. 边界(边界上界)

在边界检查失败后抛出异常。

`SChk`并行进行下界和上界的检查：



* 地址>下界？
* 地址+数据结构的大小<=上界？

当其中的一项检查失败后，抛出异常，实现逻辑如下图所示：

![SChk](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-25-WatchdogLite-Hardware-Accelerated-Compiler-Based-Pointer-Checking.assets/Figure%202.png)



### 时间错误检查(UAF检查)

WatchdogLite引入了`TChk`指令进行时间错误检查。`TChk`替换了3条x86指令(`load,compare,branch`)，从而加速了对*key*和*lock*的检查：

1. 从寄存器读取64位的*key*和64位的*lock*
2. 根据*lock*进行一次64位的读操作
3. 检查读取的值与_key_是否相等

实现逻辑如下图所示：

![TChk](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-25-WatchdogLite-Hardware-Accelerated-Compiler-Based-Pointer-Checking.assets/Figure%203.png)



## 实验评估

实验环境：SoftBound+CETS

平均开销：

narrow mode：45%

wide mode：29%

![Performance](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-25-WatchdogLite-Hardware-Accelerated-Compiler-Based-Pointer-Checking.assets/Figure%204.png)



## 评价

WatchdogLite作为2014年和Intel MPX同期的工作，在不具备专用寄存器等硬件扩展的情况下，提出了一种硬件和编译器的分工方案，从而加速了边界检查速度，并且比Intel MPX额外支持了UAF检查。同时，WatchdogLite也存在着一定的不足：

1. 依旧在复用已有的通用寄存器进行元数据的操作，存在通用寄存器竞争的问题
2. 虽然影子内存地址映射提出了可以使用地址+偏移量的方式解决，在文章中提到的默认的配置方式依旧根据地址解码出影子内存中的地址，与MPX中边界表(Bound Table)采用的寻址方式相同，实际上会带来较高的性能开销



