---
title: Objective-C Runtime 之 Class
date: 2018-06-15 11:11:21
tags:
---

#### 前言：

与 C 等面向过程的语言不同，用 Objective-C 、C++ 等面向对象语言编程时，**“对象”(object)**就是“基本构造单元”，开发者使用对象来传递、存储数据。在对象之间传递数据并执行操作的过程叫做**消息传递**或者**方法调用**。

Objective-C 是 C 的超集。所有 C 的代码都可以直接放在 Objective-C 程序里经过编译。而且 Objective-C 的 OOP 就是使用 C 结构体模拟的。

既然说到了 OO 编程，那就先聊聊 Objective-C 中的类吧

有任何不对的地方，还望大家批评指正。

#### 正文：

`Class` 是一个指向 `objc_class`结构体的指针：
```
typedef struct objc_class *Class;
```
`objc_class`结构体的定义如下：
```
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
    Class super_class                       OBJC2_UNAVAILABLE;  
    const char *name                        OBJC2_UNAVAILABLE;  
    long version                            OBJC2_UNAVAILABLE; 
    long info                               OBJC2_UNAVAILABLE;      
    long instance_size                      OBJC2_UNAVAILABLE;  
    struct objc_ivar_list *ivars            OBJC2_UNAVAILABLE;  
    struct objc_method_list **methodLists   OBJC2_UNAVAILABLE;  
    struct objc_cache *cache                OBJC2_UNAVAILABLE;  
    struct objc_protocol_list *protocols    OBJC2_UNAVAILABLE;  
#endif
} OBJC2_UNAVAILABLE;
```
下面依次了解一下各成员变量的意义：

`isa`: 大名鼎鼎的 isa 指针，相信面试中大多数人都遇到过这个问题，说说什么 isa
描述 Objective-C 对象所用的数据结构在 runtime.h 中可以看到，id 类型本身也定义在这里：
```
typedef struct objc_object {
	Class isa;
}  *id;
```
其实顾名思义，is a ，该变量定义了对象所属的类，该对象属于哪个类，isa 就指向那个类，![这张图片](https://raw.githubusercontent.com/WiInputMethod/interview/master/img/ios-runtime-class.png)
这张图早已说明了一切：
类中的 super_class 指针用来追溯整个继承链。isa 指针找到其所属的类。

`super_class`: 指向该类的父类，如果该类已是根类，则为 NULL
`name`: 类名
`version`: 版本信息
`info`: 类信息
`instance_size`: 实例大小
`ivars`: 成员变量地址列表，还记得之前说过的声明成员变量的两种常见方式么，里面提到把成员变量当做一种储存偏移量的特殊变量，交由类对象保管。就保管在这里呢
`methodLists`: 方法列表。
`cache`: 方法缓存。
`protocols`: 协议列表。

##### 元类(Meta Class)
所有的类自身也是一个对象。元类存储着一个类的所有类方法。元类也是一个类，它的`isa`指针，指向基类的元类，基类的元类的`isa`指针指向它自己，这就形成了一个闭环。


 
 
 
 
#### 参考文献
* [From C++ to Objective-C](http://chachatelier.fr/programmation/fichiers/cpp-objc-en.pdf)
* [Objective-C Runtime 运行时之一：类与对象](http://southpeak.github.io/2014/10/25/objective-c-runtime-1/)
* [Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/#Class)
* 《Effective+Objective-C+2.0++编写高质量iOS与OS+X代码的52个有效方法》

 

