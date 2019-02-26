---
layout:     post
title:      "Background on Pointer Checking"
subtitle:   ""
author:     "blankaiwang"
header-img: "img/bg-4.jpeg"
header-mask:  0.5
catalog: true
tags:

    - Security
    - Translation
    - Pointer Checking


---



# Background on Pointer Checking

本节介绍基于指针的检查方式 [1-4, 8-10, 12, 13, 17]，并且介绍我们在软件和硬件方向部署防护的首要考虑的问题。尽管已经有很多内存检测的方案，但是基于不连续存放的元数据的指针检查方式 [3, 4, 8, 15]与已有代码具有高度兼容性，因此本讨论侧重分析这类防护方案。在综述 [16]中，作者认为这类防护方案是**唯一**可以对内存安全提供全面的、不基于概率学的防护方式(见Table 2 [43])。这类防护方式可以提供高级别的内存安全属性 [18]，并且已经被Intel用在了商用软件 [4]和专利应用 [12]中。

## 基于不连续存放的元数据的指针检查

应用内存安全意味着可以防护所有的内存安全漏洞 [16]。这类内存安全漏洞，包括缓冲区溢出和UAF漏洞至今依旧普遍存在，尽管他们已经出现很长时间了 [14, 16]。从广泛意义上来说，内存安全防护包括两个主要的组件：阻止分区越界(越界内存访问和缓冲区溢出类)和阻止违反时间安全限制的操作(如UAF和dangling pointer)。

### 基于指针的元数据

在基于指针的内存安全策略中，元数据被用来为每个指针维护一系列的属性值。元数据的访问安全由其所使用的语言特性所保障。但是同时，C/C++的越界访问和指向数据结构及数组的内部指针并没有队元数据的访问安全进行保护。图1展示使用C语言进行元数据初始化和操作的示意。元数据——包括基地址、边界、lock和key，与指针相关——无论指针何时被创建。同时，元数据在进行指针操作时(如指针复制或指针运算)时，也会被相应的更新和赋值。

![Figure 1](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-26-Background-on-Pointer-Checking.assets/Figure%201.png)

元数据可以被使用内联的方式维护，即fat-pointer [1, 2, 10]；也可以被存放在内存中不连续的专用空间中 [3, 5, 7-9]。将元数据存放在不连续的专用空间中可以避免元数据被恶意破坏，同时不破坏程序自身的内存布局，具备与已有代码的兼容性。在指针被写入/读取时，也相应对影子内存中的元数据进行写入/读取(图2，图3)。这类不连续存放的内存空间可以使用多种方式进行部署，包括使用内存中的一块连续区域 [3, 4, 8]，哈希表 [8]，或多级数据结构 [4, 9, 11]。

![Figure 2](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-26-Background-on-Pointer-Checking.assets/Figure%202.png)

![Figure 3](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-26-Background-on-Pointer-Checking.assets/Figure%203.png)

### 边界检查

为了保证边界安全，指针可以合法访问的基址和边界从指针创建的时候就同该指针联系起来。指针的基址和边界通常为64位的值，这个长度可以被用来编码任意细粒度的边界信息。这些基址和边界的元数据被从来在执行内存访问之前进行全面的边界检查(图 4)。

![Figure 4](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-26-Background-on-Pointer-Checking.assets/Figure%204.png)

### 时间错误检查

为了保证时间逻辑正确，需要为每次内存分配操作设计一个特殊的识别位(图5)。在每次内存分配的时候，都会被分配一个额外的64位的字符串，并且这个字符串的数值是独一无二的。并且这个值同指针相关，从而保证了在该内存被释放掉之后依旧可以对识别位进行匹配。在指针解引用的时候，系统会检查与该指针相联系的识别位是否依旧有效。

![Figure 5](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-26-Background-on-Pointer-Checking.assets/Figure%205.png)

对每次内存访问都使用哈希表或树结构的性能开销相对很高 [1, 19]，一个替代的方式就是为每个指针的元数据分配两组值：分配识别符 _key_ 和指向内存地址 _lock location_ 的 _lock_ [2, 7, 9, 13, 17]。只有在对象依旧有效(还没有被释放掉)的时候，_key_ 和 _lock location_ 的值才会匹配。相比于哈希表需要进行哈希查找工作，这种解引用检查将时间错误检查简化为一次简单的比对操作，只需要一次对 _lock location_ 的读操作和将读取的值与 _key_ 进行比对(图 6)。释放已分配的内存会改变 _lock location_ 的值，并且使得其他对该区域的指针访问变为非法访问(dangling pointer)(图7)。因为识别符是独一无二的，因此当 _lock location_ 对应保护的结构被释放掉之后，_lock location_ 可以被复用。

![Figure 6](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-26-Background-on-Pointer-Checking.assets/Figure%206.png)

![Figure 7](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-26-Background-on-Pointer-Checking.assets/Figure%207.png)

## 基于编译器实现的指针检查

基于指针的检查可以在程序源码进行翻译的时候实现 [10, 17]或由编译器实现 [4, 8, 9, 15]。基于编译器实现的方法有三项主要的优点：

1. 指针检查可以建立在经过编译器一系列优化后的代码之上
2. 编译器可以根据得到的信息，精确定位到指针和内存的分配/回收
3. 大量冗余的指针检查可以被移除

基于编译器实现的指针检查 [4, 8, 9, 15]可以将对内存的全面保护的性能开销降低在平均2x，但是依旧对产品化应用而言，这个开销依旧很高。

## 基于硬件实现的指针检查

为了解决纯软件实现的指针检查的高性能开销问题，研究人员提出了基于硬件的指针检查方案 [2, 3, 5, 7, 12]。可以从三个方向来对基于硬件实现的指针检查方法进行分类：

1. 进行的是隐式检查还是显式检查；
2. 指针的识别方法
3. 内存中的元数据的管理方法

这些设计上的思路决定了安全方案可以提供的安全性保证、所需的硬件结构以及可以在多大程度上对程序进行优化。Table 1比较了已有的基于硬件的防护方式和他们的分类，Table 2列出了这些防护方案所使用的硬件结构。

Table 1：

|                   | 安全检查      | 插桩方式    | 元数据管理                           | 避免了新的硬件状态 | 静态优化 | 检查方法 | 性能开销 |
| ----------------- | ------------- | ----------- | ------------------------------------ | ------------------ | -------- | -------- | -------- |
| Chuang et al. [2] | 边界+时间错误 | 编译器+硬件 | 内联(fat pointer)                    | 否                 | 否       | 隐式     | 30%      |
| HardBound [3]     | 边界错误      | 硬件        | 不连续存放(影子内存)                 | 否                 | 否       | 隐式     | 5-9%     |
| SafeProc [5]      | 边界+时间错误 | 编译器      | 不连续存放(256位内容可寻址内存 [^2]) | 否                 | 是       | 显式     | 93%      |
| Watchdog [7]      | 边界+时间错误 | 硬件        | 不连续存放(影子内存)                 | 否                 | 否       | 隐式     | 25%      |
| Intel MPX [6]     | 边界错误      | 编译器      | 不连续存放(多级地址表)               |                    | 是       | 显式     | 14%[^1]  |
| WatchdogLite      | 边界+时间错误 | 编译器      | 不连续存放(影子内存)                 | 是                 | 是       | 显式     | 29%      |

[^1]:为Nginx吞吐量测试，取GCC-MPX和ICC-MPX的大致平均值，见翻译：[Intel MPX Explained](https://blankaiwang.github.io/2019/01/16/Intel-MPX-Explained.html)，实验讨论：[Intel-MPX.github.io](https://intel-mpx.github.io)，论文：[Intel MPX Explained: A Cross-layer analysis of the Intel MPX's System Stack](https://intel-mpx.github.io/code/submission.pdf) (Available on 26/02/2019)
[^2]: CAM： **C**ontent **A**ddressable **M**emory

Table 2：

|                   | 硬件结构                                                     |
| ----------------- | ------------------------------------------------------------ |
| Chuang et al. [2] | 1. 微指令插入；2. 32位元数据检查表；3. 基于元数据的寄存器映射(对所有的寄存器) |
| HardBound [3]     | 1. 微指令插入；2. 为每次内存访问引入指针标签缓存             |
| SafeProc [5]      | 1. 256位内容可寻址内存(在每次内存访问时进行关联检索)；2. 硬件哈希表；3. 256位FIFO缓冲区 |
| Watchdog [7]      | 1. 微指令注入；2. 为每次内存访问引入 _lock location_ 缓存；3. 寄存器重命名 |

使用隐式检查的硬件结构通过微指令插入来对元数据进行检查和传递。已有的工作中, [2, 3, 7]属于这一类型。为了降低在每次内存访问时对元数据进行访问带来的额外性能开销，文献 [3]提出了指针标签，文献 [7]引入了 _lock_ 缓存机制。尽管通过引入对应的硬件结构，可以有效实施隐式检查，但是这类检查无法利用到寄存器对代码的优化功能。相反的，使用显式检查的防护方案，如SafeProc [5]修改了软件工具链对指令进行插入。尽管在他们的论文中，作者没有对编译器进行静态优化以消除冗余的检查，但是显式检查本身允许编译器在使用硬件加速的同时，对冗余检查进行清除。

基于硬件的指针检查需要对指针进行识别，而指针信息通常在二进制文件中是缺失的。HardBound [3]在每次内存访问时都需要元数据进行检查，因此他们引入了指针标签缓存机制来降低通常情况下的性能开销。Watchdog [7]使用了启发式的方法过滤内存操作中所有非指针大小的操作，从而减少了对元数据的访问次数。SafeProc [5]以及Chuang et al. [2]的工作提供了新的指令扩展，从而使编译器能够准确识别指针操作。

对于提供完备的安全性保护而言，元数据管理是非常重要的一环。Chuang et al. [2]使用的fat-pointer，以及其他使用内联形式管理元数据的安全方案 [10]中，元数据会因为类型转换而被破坏。而使用影子内存或硬件哈希表等形式，就可以抵抗这类攻击，从而提供更完备的安全性保护。在Chuang et al.的工作中，元数据只能被存放在内存中，而将内存中的元数据加载到内核中进行边界和UAF检查必然会带来高额的性能开销(对每次内存访问的检查都带来了额外的4次内存访问，而且该防护方案默认对每次内存访问都进行检查)。

SafeProc [5]使用指令扩展来识别指针并且检测时间错误。SafeProc使用 `bounds violation`来检测时间错误，当一个对象被回收时，系统必须找到所有指向该对象的指针并将这些指针的边界都设为非法，因此SafeProc无法应对并发访问 [5, 15]。SafeProc将所有指针的元数据存放到了256位的内容可寻址内存(CAM)中，并且在每次对内存的访问和回收时，对CAM进行关联检索。由于程序可以具备多于256个指针，SafeProc硬件将指针记录重写在一个内存中的双索引哈希表中，硬件可以遍历哈希表来找到所需要的元数据(进行边界检查)或所有指向某一特定对象的指针元数据(进行对象回收)。这种实现方式使得它必须依赖于其他的硬件扩展如内存更新缓冲区(memory update buffer) [5]来降低高额的性能开销。



## 参考文献[^3]

1. T. M. Austin, S. E. Breach, and G. S. Sohi. Efficient Detection of All Pointer and Array Access Errors. In Proceedings of the SIGPLAN 1994 Conference on Programming Language Design and Implementation,
   June 1994.

2. W. Chuang, S. Narayanasamy, and B. Calder. Accelerating Meta Data Checks for Software Correctness and Security. Journal of Instruction-Level Parallelism, 9, June 2007.
3. J. Devietti, C. Blundell, M. M. K. Martin, and S. Zdancewic. Hardbound: Architectural Support for Spatial Safety of the C Programming Language. In Proceedings of the 13th International Conference on Architectural Support for Programming Languages and Operating Systems, Mar. 2008.
4. K. Ganesh. Pointer Checker: [Easily Catch Out-of-Bounds Memory Accesses](http://software.intel.com/sites/products/parallelmag/singlearticles/issue11/7080_2_IN_ParallelMag_Issue11_Pointer_Checker.pdf). Intel Corporation, 2012.
5. S. Ghose, L. Gilgeous, P. Dudnik, A. Aggarwal, and C. Waxman. Architectural Support for Low Overhead Detection of Memory Viloations. In Proceedings of the Design, Automation and Test in Europe, Mar. 2009.
6. Intel Corporation. [Intel Architecture Instruction Set Extensions Programming Reference](http://download-software.intel.com/sites/default/files/319433-015.pdf), 319433-015 edition, July 2013. 
7. S. Nagarakatte, M. M. K. Martin, and S. Zdancewic. Watchdog: Hardware for Safe and Secure Manual Memory Management and Full Memory Safety. In Proceedings of the 39th Annual International Symposium on Computer Architecture, June 2012.
8. S. Nagarakatte, J. Zhao, M. M. K. Martin, and S. Zdancewic. SoftBound: Highly Compatible and Complete Spatial Memory Safety for C. In Proceedings of the SIGPLAN 2009 Conference on Programming Language Design and Implementation, June 2009.
9. S. Nagarakatte, J. Zhao, M. M. K. Martin, and S. Zdancewic. CETS: Compiler Enforced Temporal Safety for C. In Proceedings of the 2010 International Symposium on Memory Management, June 2010.
10. G. C. Necula, J. Condit, M. Harren, S. McPeak, and W. Weimer. CCured: Type-Safe Retrofitting of Legacy Software. ACM Transactions on Programming Languages and Systems, 27(3), May 2005.
11. N. Nethercote and J. Seward. How to Shadow Every Byte of Memory Used by a Program. In Proceedings of the 3rd ACM SIGPLAN/SIGOPS International Conference on Virtual Execution Environments, 2007.
12. B. V. Patel, R. Gopalakrishna, A. F. Glew, R. J. Kushlis, D. A. V. Dyke, J. F. Cihula, A. K. Mallick, J. B. Crossland, G. Nelger, S. D. Rodgers, M. G. Dixon, M. J. Charney, and J. Gottelieb. Managing and Implementing Metadata in Central Processing Unit Using Register Extensions, Mar. 2011. US Patent Pub No: US 2011/0078389 A1.
13. H. Patil and C. N. Fischer. Low-Cost, Concurrent Checking of Pointer and Array Accesses in C Programs. Software — Practice & Experience, 27(1):87–110, 1997.
14. J. Pincus and B. Baker. Beyond Stack Smashing: Recent Advances in Exploiting Buffer Overruns. IEEE Security & Privacy, 2(4):20–27, 2004.
15. M. S. Simpson and R. K. Barua. MemSafe: Ensuring the Spatial and Temporal Memory Safety of C at Runtime. In IEEE International Workshop on Source Code Analysis and Manipulation, 2010.
16. L. Szekeres, M. Payer, T. Wei, and D. Song. SoK: Eternal War in Memory. In Proceedings of the 2013 IEEE Symposium on Security and Privacy, 2013.
17. W. Xu, D. C. DuVarney, and R. Sekar. An Efficient and Backwards-Compatible Transformation to Ensure Memory Safety of C Programs. In Proceedings of the 12th ACM SIGSOFT International Symposium on Foundations of Software Engineering, 2004.
18. J. Zhao, S. Nagarakatte, M. M. K. Martin, and S. Zdancewic. Formalizing the LLVM Intermediate  representation for Verified Program Transformations. In Proceedings of The 39th ACM SIGPLAN/SIGACT Symposium on Principles of Programming Languages, Jan. 2012.
19. R. W. M. Jones and P. H. J. Kelly. Backwards-Compatible Bounds Checking for Arrays and Pointers in C Programs. In Third International Workshop on Automated Debugging, Nov. 1997.

[^3]: WatchdogLite: Hardware-Accelerated Compiler-Based Pointer Checking.