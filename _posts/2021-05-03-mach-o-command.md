---
layout: post
title: "Mach-O文件命令行工具"
date:   2021-05-03
tags: []
comments: true
author: jcexk
---

> ### APP瘦身之清除无用类/协议/方法
> github上有很多类似工具，之所以写这篇文章是为了对Mach-O更进一步的了解，推荐使用工具进行分析
> * **清除无用类：**利用```__objc_classlist-(__objc_classrefs+__objc_superrefs)```的差集，进行清理
> * **清除无用协议：**利用```__objc_protolist - (__objc_classrefs+__objc_superrefs)```的差集，进行清理
> * **清除无用方法：**利用```__objc_classlist的 (instanceMethods - __objc_selrefs) + clasMethods - __objc_selrefs```的差集，进行清理
> * 因OC的动态特性，建议在清理之前先确认是否使用

### 什么是Mach-O
Mach-O是OS X/iOS系统下可执行文件的格式，mach-o，就比如Linux系统下可执行文件格式为ELF，windows是PE格式一样。
Mach-O 由三部分组成：
* **Header：**描述了Mach-O的cpu架构、文件类型以及加载命令等信息。
* **Load commands：**描述了文件中数据的具体组织结构，不同的数据类型使用不同的加载命令表示。
* **Data**：Data中的每个段（segment）的数据都保存在这里，段的概念与ELF文件中段的概念类似。每个段都有一个或多个Section，它们存放了具体的数据与代码；今天要分享的就是Data中的内容。
![Mach-O组成图](https://jcexk-1259114619.cos.ap-shanghai.myqcloud.com/2021/05/03/16199749247499.jpg)

### __DATA
由上图可知，每一个Segment中(暂称呼为段)都包含多个Sectoin(暂称呼为区)
Segment由三部分组成：__TEXT、__DATA、__LINKEDIT。具体分布如下图
![__DATA分布图](https://jcexk-1259114619.cos.ap-shanghai.myqcloud.com/2021/05/03/data-fen-bu-tu.png)

### __Segment

__Segment有两种类型

* **__TEXT：**代码段，包括函数，和只读的常量，例如C字符串,可读可执行
* **__DATA：**数据段，包括可读写的全局变量, 静态变量等，可读可写
* **__LINKEDIT：**如何加载程序, 包含了方法和变量的元数据（位置，偏移量），以及代码签名等信息。

### __Section

下面列举一些常见的 Section

| Secton |作用 | Segment |
|--------|---------|----|
|    __text    |   主程序代码      | __TEXT   |
|    __const    |   const 关键字修饰的常量      | __TEXT   |
|   __stubs    |    用于 Stub 的占位代码，很多地方称之为桩代码     | __TEXT   |
|    __stubs_helper   |    当 Stub 无法找到真正的符号地址后的最终指向     | __TEXT   |
|   __objc_methname    |    Objective-C 方法名称     | __TEXT   |
|    __objc_methtype   |     Objective-C 方法类型   | __TEXT   |
|     __objc_classname  |   Objective-C 类名称      | __TEXT   |
|  __cstring     |     C 语言字符串    | __TEXT   |
|       |         | 空行  |
|   __data    |    初始化过的可变数据     | __DATA   |
|   __la_symbol_ptr    |    lazy binding 的指针表，表中的指针一开始都指向 __stub_helper     | __DATA   |
|   nl_symbol_ptr    |    非 lazy binding 的指针表，每个表项中的指针都指向一个在装载过程中，被动态链机器搜索完成的符号     | __DATA   |
|    __const   |    没有初始化过的常量     | __DATA   |
|     __cfstring  |     程序中使用的 Core Foundation 字符串（CFStringRefs）    | __DATA   |
|    __bss   |     存放为初始化的全局变量，即常说的静态内存分配    | __DATA   |
|    __common   |    没有初始化过的符号声明     | __DATA   |
|   __objc_classlist    |     Objective-C 类列表    | __DATA   |
|    __objc_protolist   |    Objective-C 协议列表     | __DATA   |
|    __objc_imginfo   |     Objective-C 镜像信息    | __DATA   |
|    __objc_selrefs   |    Objective-C 引用的selector     | __DATA   |
|   __objc_protorefs    |     Objective-C 引用的协议    | __DATA   |
|     __objc_superrefs  |     Objective-C 引用的超类    | __DATA   |

上面列举了这些，是不是看着很兴奋？是不是跃跃欲试想看看这些__Section中到底有什么东西呢？
而OSX系统自带的otool刚好可以分析Mach-O可执行文件。
### otool 常用命令
以下终端路径为...../RJEventLinkDemo.app
![-w727](https://jcexk-1259114619.cos.ap-shanghai.myqcloud.com/2021/05/03/16199789975139.jpg)

* 查看fat headers信息

> otool -f RJEventLinkDemo

* 查看archive header信息

> otool -a RJEventLinkDemo

* 查看Mach-O头结构

> otool -h RJEventLinkDemo

* 查看load commands

> otool -l RJEventLinkDemo

* 查看依赖的动态库,包括动态库名称、当前版本号、兼容版本号

> otool -L RJEventLinkDemo

* 查看支持的框架

> otool -D RJEventLinkDemo

* 查看text section

> otool -t -v RJEventLinkDemo

* 查看data section

> otool -d RJEventLinkDemo

* 查看Objective-C segment

> otool -o RJEventLinkDemo

* 查看symbol table

> otool -I RJEventLinkDemo

* 获取所有方法名称

> otool -v -s __TEXT __objc_methname RJEventLinkDemo

* 获取所有类的地址

> otool -v -s __DATA __objc_classlist RJEventLinkDemo

* 获取所有引用到的类的地址

> otool -v -s __DATA __objc_classrefs RJEventLinkDemo

* 获取所有协议的地址

> otool -v -s __DATA __objc_protolist RJEventLinkDemo

* 获取所有引用到的协议的地址

> otool -v -s __DATA __objc_protorefs RJEventLinkDemo

* 获取所有引用到的方法的地址

> otool -v -s __DATA __objc_selrefs RJEventLinkDemo

Mach-o文件中__DATA __objc_classrefs段记录了引用类的地址，__DATA __objc_classlist段记录了所有类的地址，取差集可以得到未使用的类的地址，然后进行符号化，就可以得到未被引用的类信息。

Objective-C 中的方法都会通过 objc_msgSend 来调用，而 objc_msgSend 在 Mach-O 文件里是通过 _objc_selrefs 这个 section 来获取 selector 这个参数的。
所以，_objc_selrefs 里的方法一定是被调用了的。_objc_classrefs 里是被调用过的类， objc_superrefs 是调用过 super 的类（继承关系）。通过 _objc_classrefs 和 _objc_superrefs,我们就可以找出使用过的类和子类。
注意需要人工二次确认
JSONModel 里定义了未使用的协议会被判定为无用协议；
如果子类使用了父类的方法，父类的这个方法不会被认为使用了；
通过点的方式使用属性，该属性会被认为没有使用；
使用 performSelector 方式调用的方法也检查不出来，比如 self performSelector:@selector(arrivalRefreshTime)；
运行时声明类的情况检查不出来。比如通过 NSClassFromString 方式调用的类会被查出为没有使用的类，比如 layerClass = NSClassFromString(@“SMFloatLayer”)。
还有以 [[self class] accessToken] 这样不指定类名的方式使用的类，会被认为该类没有被使用。像 UITableView 的自定义的 Cell 使用 registerClass，这样的情况也会认为这个 Cell 没有被使用。

更多详情请在终端输入`otool`查看
![otool更多功能](https://jcexk-1259114619.cos.ap-shanghai.myqcloud.com/2021/05/03/16199794314400.jpg)
命令行出处[ZhongXi](https://juejin.cn/post/6844903641573261326)，记录在此为了方便下次使用。
