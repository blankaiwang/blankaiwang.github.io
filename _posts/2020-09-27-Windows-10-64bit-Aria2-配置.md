```
layout:     post
title:      "Windows 10 64bit Aria2配置"
subtitle:   ""
author:     "blankaiwang"
header-img: "img/Aria2.jpg"
header-mask:  0.5
catalog: true
tags:

    - Windows 10
    - Aria2
    

```







# Windows 10 64bit Aria2 配置

## 1. 下载并解压Aria2

访问Aria2的[GitHub博客](https://aria2.github.io/)，跳转到对应的版本下载页。本文以version 1.35.0为例。

在下载页的assets处，选择下载系统适配的细分版本，此处选择下载 aria2-1.35.0-win-64bit-build1.zip。

下载完成后，将zip包解压至目录，如C:/Aria2/



## 2. 配置Aria2

在C:/Aria2/下，新建如下几个空白文件：

* aria2.conf
* aria2.log
* aria2.session
* HideRun.vbs



编辑aria2.conf，输入以下配置项：

```
dir=C:\downloads\    /*配置下载文件保存路径*/
log=C:\aria2\aria2.log   /*配置日志路径*/
input-file=C:\aria2\aria2.session
save-session=C:\aria2\aria2.session
save-session-interval=60
force-save=true
log-level=error

enable-rpc=true
pause=false
rpc-allow-origin-all=true
rpc-listen-all=true
rpc-save-upload-metadata=true
rpc-secure=false

daemon=true
disable-ipv6=true
enable-mmap=true
file-allocation=falloc
max-download-result=120
#no-file-allocation-limit=32M
force-sequential=true
parameterized-uri=true
```



编辑HideRun.vbs：

```vbscript
CreateObject("WScript.Shell").Run "C:\aria2\aria2c.exe --conf-path=aria2.conf",0
//应修改Run后的aria2c.exe路径为实际aria2c.exe路径
```

**此后使用Aira2前，需运行HideRun.vbs开启Aria2c服务**，也可将该vbs脚本置于启动文件夹，设置为开机自启动。



## 3. 通过Aria2 Web控制台使用Aria2

浏览器访问[Aria2 Web控制台](http://aria2c.com/)，即可对下载任务进行查看、添加、删除操作。在使用Aria2前，需**确保Aria2c服务已启用**。



## 4. Aria2控制台可视化改进

由于Aria2自身的Web控制台相对简陋，GitHub上已有相当数量的任务可视化项目，此处选用[webui-aria2](https://github.com/ziahamza/webui-aria2)。webui-aria2提供了下载速度的可视化显示，下载任务批量管理、检索，下载设置配置等功能，相对于命令行模式和Aria2自身提供的Aria2 Web控制台提升了Aria2的用户友好度。

将webui-aria2的代码下载到本地并解压后，无需再次配置，访问 ../docs/index.html即可打开aria控制台，也可将index.html创建桌面快捷方式或设置浏览器书签，实现快速访问。



## 参考资料

* https://www.cnblogs.com/banxiancode/p/12604034.html
* https://www.cnblogs.com/mlgjb/p/9144575.html



