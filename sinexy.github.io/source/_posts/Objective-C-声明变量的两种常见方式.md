---
title: Objective-C 声明变量的两种常见方式
date: 2018-06-06 12:11:21
tags:
---

#### 前言：

昨天突然想起前同事问过的一个问题
```
{
	NSString *_aString;
}
```
和
```
@property (nonatomic, copy) NSString *aString;
```
这两种写法有什么区别？
正好在一个开发群里，也看到过类似问题，索性写一段话来表达一下自己的看法。有任何不对的地方，还望大家批评指正。

#### 正文：

声明在`{}`中的，叫做成员变量；而用 `@property`修饰的，顾名思义叫做属性。

先说区别：

* 在'{}'中声明的成员变量，级别是`protected`的。简单来说就是子类和当前类可以使用这个成员变量，其他类不可以。而用'@property'声明的变量,级别是`public`，可以在当前类或者其他类中被访问。

* 在'{}'中声明成员变量，使用变量名进行访问；'@property'声明的变量，使用_变量名(在变量名前加一个下划线),或者self.变量名进行访问。

* 在'{}'中声明成员变量，直接访问保存实例变量的那块内存，没有调用 set/get 方法，不走“方法派发”(method dispatch)，所以要更快。

* 在'{}'中声明成员变量，不走 set 方法，自然就没有一堆属性特质，那也就绕过了内存管理语义。

* 在'{}'中声明成员变量，不会触发 KVO(Key-Value Observing)。

其实，对于写 Java 或者 C++的同学来说，在'{}'中声明成员变量这种写法再正常不过，而 Objective-C 代码通常不这样写。因为这种写法：
> 对象布局在编译期(compile time)就已经固定了。

眼神好的同学应该注意到“编译期”这三个字。想想看，Objective-C 的一项重要特性——运行时。在某个类中动态添加属性是不是很高级？
> 只要碰到访问 _aString 变量的代码，编译器就把其替换为“偏移量”(offset)，这个偏移量是“硬编码”(hardcode)，表示该变量存放对象的内存区域的其实地址有多远。这样做看起来没什么问题，但是如果又加了一个实例变量，那就麻烦了。

当然，这个问题苹果工程师已经解决了，他们的做法是把实例变量当做一种存储偏移量所用的“特殊变量”(special variable)，交由“类对象”(class object)保管。偏移量会在运行期查找，如果类的定义变了，那么存储的偏移量也就变了，无论何时访问实例变量，总能使用正确的偏移量。

而另一个解决方案就是使用'@property'。其实
```
@property (nonatomic) NSString *aString;
```
相当于
```
{
	NSString *_aString;
}
- (NSString *)aString;
- (void)setAString:(NSString *)aString;
```

想要访问属性，可以使用 _aString 或者 self.aString。写 Java 的同学又开心了，因为这个".属性名"和 Java 的中写看起来一样。其实，在 Objective-C 中，没有"点语法"，这是编译器搞出来的。在使用  self.aString 时，编译器会将其转换为对应的存取方法调用。

```
AObject *aObject = [AObject new];
aObject.aString = @"一堆文字";
NSLog(@"%@",aObject.aString);  // 打印结果我不写
```
相当于
```
AObject *aObject = [AObject new];
[aObject setAString:@"一堆文字"];
NSLog(@"%@",[aObject aString]);
```
如果`@property`只有这点作用，存在的意义就不大了。所以，属性特质就出现了。(因果逻辑 doge)
```
@property (nonatomic, readwrite, copy) NSString *aString;
```

说到属性特质，我们可以又扯到 Objective-C 的内存管理去，所以我们下次再聊。

 
 
 
 
#### 参考文献

Effective+Objective-C+2.0++编写高质量iOS与OS+X代码的52个有效方法

 

