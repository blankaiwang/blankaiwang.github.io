---
layout:     post
title:      "Intel MPX Security Test"
subtitle:   ""
author:     "blankaiwang"
header-img: "img/bg-2.jpg"
header-mask:  0.5
catalog: true
tags:

    - Intel MPX
    - Security
    - Test

---



# Intel MPX Security Test

## 实验环境

### RIPE Test Bed：

git clone https://github.com/johnwilander/RIPE.git

### 系统配置

Ubuntu 16.04 i386  2GB内存

Ubuntu 16.04 x64 2GB内存

## 参数修改

通过对MAKEFILE中的CFLAGS和LDFLAGS进行修改，添加MPX编译支持，比较运行结果

```bash
CFLAGS+=-g -Wall -O2 -mmpx -fcheck-pointer-bounds
LDFLAGS+=-lmpx -lmpxwrappers
```



## 问题修正

1. 在64位OS中，由于RIPE在MAKEFILE中加入了 `-m32`选项强制进行32位编译，直接make会报错.

   Fedora 29:

```bash
gnu/stubs-32.h:No such file or directory
```

​	添加支持 `yum install glibc-devel.i686`

​	Ubuntu 16.04:

```bash
fatal error: sys/cdefs.h: No such file or directory
```

​	添加支持 `sudo apt-get install glibc6-dev-i386`

2. Fedora 29下，添加MPX编译选项后，make报错：

   ![error2.1](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-12-Intel-MPX-Security-Test.assets/Figure%201.jpg)

   直接将`/usr/lib64`路径下对应的两个.so文件拷贝到`/usr/lib`下并不能解决问题：

   ![error2.2](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-02-12-Intel-MPX-Security-Test.assets/Figure%202.jpg)

   此时研判是缺失32位的MPX连接库文件，访问https://fedora.pkgs.org/29/fedora-x86-64/libmpx-8.2.1-2.fc29.i686.rpm.html并安装rpm包，解决问题。

## 运行结果

### Ubuntu 16.04 i386

Native

|          | OK/SOME | FAIL | NP   |
| -------- | ------- | ---- | ---- |
| direct   | 89/0    | 241  | 1590 |
| indirect | 0/0     | 520  | 1400 |
| both     | 89/0    | 761  | 2990 |



MPX

|          | OK/SOME | FAIL | NP   |
| -------- | ------- | ---- | ---- |
| direct   | 60/0    | 270  | 1590 |
| indirect | 0/0     | 520  | 1400 |
| both     | 60/0    | 790  | 2990 |

### Ubuntu 16.04 x64

Native

|          | OK/SOME | FAIL | NP   |
| -------- | ------- | ---- | ---- |
| direct   | 89/1    | 240  | 1590 |
| indirect | 0/0     | 520  | 1400 |
| both     | 89/1    | 760  | 2990 |



MPX

|          | OK/SOME | FAIL | NP   |
| -------- | ------- | ---- | ---- |
| direct   | 60/0    | 270  | 1590 |
| indirect | 0/0     | 520  | 1400 |
| both     | 60/0    | 790  | 2990 |

