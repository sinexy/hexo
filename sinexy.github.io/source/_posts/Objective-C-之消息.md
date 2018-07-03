---
title: Objective-C 之消息
date: 2018-07-03 10:15:16
tags:
---

#### 前言：
初学 Objective-C 时， 不觉得 `[object doSomething]` 与 `doSomething()` 有什么不一样，最多是写法看起来奇葩一点。随着了解的深入，渐渐觉得 Runtime 黑魔法简直不要太好用。

有任何不对的地方，还望大家批评指正。

#### 正文：
调用对象的方法，是再日常不过的操作。其他语言叫做方法调用，在 Objective-C 中叫做“消息传递”。消息有“名称”(name)或“选择子”(selector)，可以接受参数，也可能有返回值。

非动态语言在编译时期就能决定运行时应该调用的函数，而“动态绑定”是在运行期来决定到底该调用哪个方法，这样我们就可以在程序运行时改变其真实调用的方法，这些特性使得 Objective-C 成为一门真正的动态语言。

我们先来了解一下基本概念：

**SEL**: 定义表示方法选择器的不透明类型。方法选择器用于表示运行时方法的名称。方法选择器是已使用Objective-C运行时注册（或“映射”）的C字符串。加载类时，编译器生成的选择器将由运行时自动映射。

可以这么理解，Objective-C 在编译时，会依据每一个方法的名字、参数序列，生成一个唯一的编号，他的行为基本可以等同 C 语言中的函数指针，但 Objective-C 不能直接使用函数指针，只能通过`@selector`来取。
下面代码可能更清楚:
```
SEL sel = @selector(method);
```

其实 SEL 的与类无关，只要方法名相同，SEL 就是一样的。
A 类
```
@implementation ClassA

- (void)print
{
    NSLog(@"打印 A");
    SEL selA = @selector(print);
    NSLog(@"%p",selA);
}

@end
```
B 类
```
@implementation ClassB

- (void)print
{
    NSLog(@"打印 B");
    SEL selB = @selector(print);
    NSLog(@"%p",selB);
}

@end
```
控制器中
```
    ClassA *classA = [ClassA new];
    SEL selA = @selector(print);
    [classA print];
    ClassB *classB = [ClassB new];
    SEL selB = @selector(print);
    [classB print];
    NSLog(@"%p-----%p",selA, selB);
```
打印结果如下：
```
打印 A
0x10d8ceade
打印 B
0x10d8ceade
0x10d8ceade-----0x10d8ceade
```
所以在 Objective-C 中，同一个类不能存在2个同名方法，即使参数类型不同也不行，没有方法重载一说。


**IMP**:指向方法实现开始的指针。其定义如下：
```
id (*IMP)(id, SEL, ...)
```
此数据类型是指向实现该方法的函数的开头的指针。此函数使用为当前CPU体系结构实现的标准C调用约定。第一个参数是指向`self`的指针（即，此类的特定实例的内存，或者，对于类方法，指向元类的指针）。第二个参数是方法选择器。方法参数接在后面。

前面介绍的`SEL`就是为了查找方法的最终实现`IMP`的。每个方法对应唯一的`SEL`，因此通过`SEL`可以快速准确地获得其所对应的`IMP`。拿到了 `IMP`就可以执行对应的方法了。

**Method**:表示类定义中的方法。其定义如下：
```
typedef struct objc_method *Method;

struct objc_method {
		SEL method_name;
		char *method_types;
		IMP method_imp;
	} method_list[1];		/* variable length structure */
```
方法的结构体中包含了一个`SEL`和`IMP`，实际上相当于把 `SEL` 和 `IMP` 建立了一个映射关系，对应起来了。

在 Objective-C 中，给对象发送消息：
```
	ClassA *classA = [ClassA new];
	[classA print];
```
等同于：
```
void objc_msgSend(classA, print);
```
其原型是：
```
id objc_msgSend(id self, SEL cmd, ...)
```

`classA` 叫做“接收者”(receiver), `print` 叫做“选择子”(selector)。
`objc_msgSend` 函数会依据 `self`和`SEL`来调用适当的方法。




#### 参考文档
[Apple Documentation](https://developer.apple.com/documentation/objectivec/)
[Source Browser](https://opensource.apple.com/tarballs/objc4/)
[Objective-C Runtime 运行时之三：方法与消息](http://southpeak.github.io/2014/11/03/objective-c-runtime-3/)
[Objective-C中的Selector和SEL](http://limite.me/blog/2015/01/07/objective-czhong-de-selectorhe-sel/)





