---
title: Objective-C 之消息机制
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

具体操作如下：

先在接收者所属的类中搜寻其“方法列表”(list of methods)，如果能找到与选择子名称相符的方法，就跳至其实现代码。若是找不到，那就站着继承体系继续向上查找，等找到合适的方法之后再跳转。如果最终还是找不到相符的方法，那就执行“消息转发”(message forwarding)操作。

##### 消息转发

由于 Objective-C 的动态性，编译时期不能确定某类是否能执行某个方法(运行期可以向类中添加方法)。

消息转发分为两大阶段。第一阶段先征询接收者所属的类，看其是否能动态添加方法，已处理当前这个“未知的选择子”，这叫做“动态方法解析”(dynamic method resolution)。第二个阶段设计“完整的消息转发机制”。如果运行期系统已经把第一阶段执行完了，那么接收者自己就无法再以动态新增方法的手段来响应包含该选择子的消息了。此时，运行期系统会请求接收者以其他手段来处理与消息相关的方法调用。这又分为两小步：首先，请接收者看看有没有其它对象能处理这条消息。若有，则运行期系统会把消息转给那个对象，于是消息转发过程结束，一切如常。若没有，则启动完整的消息转发机制，运行期系统会把与消息有关的全部细节都封装到 `NSInvocation`对象中，再给接收者最后一次机会，令其设法解决当前还未处理的这条消息。

**动态方法解析**
对象在收到无法解读的消息后，首先将调用其所遇类的下列类方法：
```
+ (BOOL)resolveInstanceMethod:(SEL)selector
```
该方法的参数就是那个未知的选择子，其返回值为`Boolean`类型，表示这个类是否能新增一个实例方法用以处理此选择子。在继续往下执行转发机制之前，本类有机会新增一个处理此选择子的方法。如果尚未实现的方法是类方法，那么就会使用
```
+ (BOOL)resolveClassMethod:(SEL)selector
```
使用这种办法的前提是：相关方法的实现代码已经写好，只等着运行的时候动态插在类中就可以了。此方法常用来实现`@dynamic 属性`。

**备援接收者**
当前接收者还有第二次机会能处理未知的选择子，在这一步中，运行期系统会问它：能不能把这条消息转给其他接收者来处理。与该步骤对应的处理方法如下：
```
- (id)forwardingTargetForSelector:(SEL)selector
```
方法参数代表未知的选择子，若当前接收者能找到备援对象，则将其返回，若找不到，就返回 nil。通过此方法，我们可以用“组合”(composition)来模拟出“多重继承”(multiple inheritance)的某些特性。注意，我们无法操作经由这一步所转发的消息。若是想在发送给备援接收者之前先修改消息内容，那就得通过完整的消息转发机制来做了。

**完整的消息转发**
如果转发算法已经来到这一步的话，那么唯一能做的就是启用完整的消息转发机制了。首先创建`NSInvocation`对象，把与尚未处理的那条消息有关的全部细节都封于其中。词对象包含选择子、目标(target)及参数。在触发`NSInvocation`对象时，“消息派发系统”(message-dispatch system)将亲自出马，把消息指派给目标对象。
此步骤会调用下列方法来转发消息：
```
- (void)forwardInvocation:(NSInvocation*)invocation
```
这个方法可以实现得很简单：只需改变调用目标，使消息在新目标上得以调用即可。然而这样实现出来的方法与“备援接收者”方案所实现的方法等效，所以很少有人采用这么简单的实现方式。比较有用的实现方式为：在触发消息前，先以某种方式改变消息内容，比如追加另外一个参数，或是改换选择子，等等。
实现此方法时，若发现某调用操作不应该由本类处理，则需要调用超类的同名方法。这样的话，继承体系中的每个类都有机会处理此调用请求，直到`NSObject`。如果最后调用了 NSObject 类的方法，那么该方法还会继而调用`doesNotRecognizeSelector:`以抛出异常，此异常表名选择子最终最能得到处理。


#### 参考文档
[Apple Documentation](https://developer.apple.com/documentation/objectivec/)
[Source Browser](https://opensource.apple.com/tarballs/objc4/)
[Objective-C Runtime 运行时之三：方法与消息](http://southpeak.github.io/2014/11/03/objective-c-runtime-3/)
[Objective-C中的Selector和SEL](http://limite.me/blog/2015/01/07/objective-czhong-de-selectorhe-sel/)





