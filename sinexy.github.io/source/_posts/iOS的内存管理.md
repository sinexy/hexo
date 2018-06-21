---
title: Objective-C 的内存管理
date: 2018-06-08 18:26:29
tags:
---

#### 前言：

iOS 内存管理相关的内容其实已经一大堆了，为了加强理解，写了这篇读书笔记。有任何不对的地方，还望大家批评指正。

#### 正文：

##### C 语言内存模型
说 Objective-C 的内存管理之前，可以了解一下 C 的内存模型

内存分区有哪些？
栈、全局(静态)区、堆、字符串常量区

* 栈(stack):C语言函数内部变量包括局部变量和形式参数等。在进入函数的时候自动分配，在离开函数时自动清除的变量存储区。

* 全局/静态区(global/static):存放全局变量和静态变量的存储区。全局变量也称为外部变量，它是在函数外部定义的变量。全局变量是所有函数的公用变量，整个程序中任何一个函数都可以任意地调用它。静态变量和全局变量被分配到同一块内存中，静态局部变量只限于在定义处的函数使用，但是离开函数后数值一直保留，直到程序退出。

* 堆(heap):由调用malloc函数分配的内存块，一般每一次malloc函数分配的内存块，最后都要对应调用一次free函数释放这个内存块。如果程序员没有释放掉，那么在程序结束后，操作系统会自动释放。

* 常量储存区:就是存放程序内所有字符串常量的内存区域，这个内存区域上存储的内容不允许修改，直到程序退出为止。

```
int a = 0;  // 全局
char *p;    // 全局

int main(int argc, const char * argv[]) {
    
    int b;  // 栈
    char s[] = "abc";   // 栈
    char *p1;   // 栈
    char *p2 = "123456";    // 123456/0在常量区，p2在栈上
    static int c = 0;   // 全局
    p = (char *)malloc(10);    // 堆
    p1 = (char *)malloc(20);    // 堆
    return 0;
}
```

**区别:**
1. 栈由系统自动分配，不用担心释放问题；堆需要程序员自己申请，那就需要程序员来释放。
2. 栈由系统自动分配，速度就比较快；堆是`new`或者`malloc`出来，一般速度比较慢，而且容易产生内存碎片（话说 OC 中的`allocWithZone:`中这个'zone'，貌似就是减少碎片产生的存在）。
3. 栈是一块连续的内存区域，内存大小比较小；堆是不连续的，比较大。


##### 引用计数
先来说说垃圾回收机制。如上所说，虽然栈中内存会自动释放，但堆中的已分配内存需要我们自己来释放。自己来管理内存的话，释放时机的把握就很重要了。如果释放早了，还有其他地方在使用这块内存中的对象，那就形成了垂悬指针。或者本来这块内存中的对象已经没地方用了，你又没释放，本来就很着急的内存空间，怕是要溢出了。所以，垃圾回收机制就是解决这个问题的。

传统算法：

* 引用计数(Reference Counting)算法
* 标记-清除(Mark-Sweep)算法
* 复制(Copying)算法

标记-清除算法：简单来说，就是轮询所有对象，把被引用的对象标记起来，轮询结束后就清除所有没有被标记的对象。

复制算法：把堆空间分成A、B两份，每次只使用其中之一。假设先使用 A 区域，当系统认为需要回收垃圾时，把所有正在被使用的对象复制到 B 区域。然后清空 A。到下一次需要收集时，再在 B 中正在被使用的对象复制到 A，然后清空 B。

Objective-C 中的内存管理用的就是引用计数：每次有对象被引用，那么对象的引用计数就+1，当你不再引用这个对象时，就告诉对象，让他引用计数-1。当引用计数为0时，就释放该对象。

有这样一个口诀：

* 自己生成的对象，自己持有
* 非自己生成的对象，自己也能持有。
* 不再需要自己持有的对象时释放。
* 非自己持有的对象无法释放。

##### 内存管理的具体实现

以一个对象的生成、持有、释放、销毁为一个完整的生命周期来探讨苹果的实现。
通过 GNUstep 源码中的 [NSObject](https://github.com/gnustep/libs-base/blob/master/Source/NSObject.m) 类来学习。

**生成**
首先，看`alloc`方法。这是一个类方法。
```
+ (id) alloc
{
  return [self allocWithZone: NSDefaultMallocZone()];
}

+ (id) allocWithZone: (NSZone*)z
{
  return NSAllocateObject(self, 0, z);
}


NSAllocateObject(Class aClass, NSUInteger extraBytes, NSZone *zone)
{
  id	new;
#ifdef OBJC_CAP_ARC
  if ((new = class_createInstance(aClass, extraBytes)) != nil)
    {
      AADD(aClass, new);
    }
#else
  int	size;
  NSCAssert((!class_isMetaClass(aClass)), @"Bad class for new object");
  size = class_getInstanceSize(aClass) + extraBytes + sizeof(struct obj_layout);
  if (zone == 0)
    {
      zone = NSDefaultMallocZone();
    }
  new = NSZoneMalloc(zone, size);
  if (new != nil)
    {
      memset (new, 0, size);
      new = (id)&((obj)new)[1];
      object_setClass(new, aClass);
      AADD(aClass, new);
    }
  if (0 == cxx_construct)
    {
      cxx_construct = sel_registerName(".cxx_construct");
      cxx_destruct = sel_registerName(".cxx_destruct");
    }
  callCXXConstructors(aClass, new);
#endif
  return new;
}
```
注意到`obj_layout`了没，他里面有个`retained`这就是用来存引用计数的
```
struct obj_layout {

  char	padding[__BIGGEST_ALIGNMENT__ - ((UNP % __BIGGEST_ALIGNMENT__)

    ? (UNP % __BIGGEST_ALIGNMENT__) : __BIGGEST_ALIGNMENT__)];

  gsrefcount_t retained;

};
typedef	struct obj_layout *obj;

```

其实，说了这么多，就是想说，在创建对象的时候，有个保存引用计数的变量。感觉说了一堆废话...

**持有**
接下来，了解一下持有对象：
```
- (id) retain
{
  return retain_fast(self);
}

static id retain_fast(id anObject)
{
#ifdef SUPPORT_WEAK
  if (objc_retain_fast_np)
    {
      return objc_retain_fast_np(anObject);
    }
  else
#endif
    {
      return objc_retain_fast_np_internal(anObject);
    }
}
static id objc_retain_fast_np_internal(id anObject)
{
  BOOL  tooFar = NO;
#if	defined(GSATOMICREAD)

  if (GSAtomicIncrement((gsatomic_t)&(((obj)anObject)[-1].retained)) > 0xfffffe)
    {
      tooFar = YES;
    }

#else	/* GSATOMICREAD */
  pthread_mutex_t *theLock = GSAllocationLockForObject(anObject);
  pthread_mutex_lock(theLock);
  if (((obj)anObject)[-1].retained > 0xfffffe)
    {
      tooFar = YES;
    }
  else
    {
      ((obj)anObject)[-1].retained++;
    }
  pthread_mutex_unlock(theLock);
#endif	/* GSATOMICREAD */
  if (YES == tooFar)
    {
      static NSHashTable        *overrun = nil;
      [gnustep_global_lock lock];
      if (nil == overrun)
        {
          overrun = NSCreateHashTable(NSNonRetainedObjectHashCallBacks, 0);
        }
      if (0 == NSHashGet(overrun, anObject))
        {
          NSHashInsert(overrun, anObject);
        }
      else
        {
          tooFar = NO;
        }
      [gnustep_global_lock lock];
    	if (YES == tooFar)
        {
          NSString      *base;
			  base = [NSString stringWithFormat: @"<%s: %p>",
          class_getName([anObject class]), anObject];
          [NSException raise: NSInternalInconsistencyException
            format: @"NSIncrementExtraRefCount() asked to increment too far"
            @" for %@ - %@", base, anObject];
        }
    }
  return anObject;
}
```
简单来说，就是在持有对象的时候，让对象的引用计数++。


**释放**
限于篇幅，释放就不贴代码了。就是每次调用的时候让对象的引用计数--。
这里 GNUstep 做了一个简单的容错，当引用计数<0时，让其=0。
当引用计数为0时，调用`[self dealloc]`方法，也就是销毁对象的方法。

**销毁**
`dealloc`方法里面又调用了`NSDeallocateObject(self)`方法。
```
NSDeallocateObject(id anObject)
{
  Class aClass = object_getClass(anObject);
  if ((anObject != nil) && !class_isMetaClass(aClass))
    {
#ifndef OBJC_CAP_ARC
      obj	o = &((obj)anObject)[-1];
      NSZone	*z = NSZoneFromPointer(o);
#endif
      (*finalize_imp)(anObject, finalize_sel);
      AREM(aClass, (id)anObject);
      if (NSZombieEnabled == YES)
	{
#ifdef OBJC_CAP_ARC
	  if (0 != zombieMap)
	    {
              pthread_mutex_lock(&allocationLock);

	      NSMapInsert(zombieMap, (void*)anObject, (void*)aClass);

              pthread_mutex_unlock(&allocationLock);
	    }
	  if (NSDeallocateZombies == YES)
	    {
	      object_dispose(anObject);
	    }
	  else
	    {
	      object_setClass(anObject, zombieClass);
	    }
#else
	  GSMakeZombie(anObject, aClass);
	  if (NSDeallocateZombies == YES)
	    {
	      NSZoneFree(z, o);
	    }
#endif
	}
      else
	{
#ifdef OBJC_CAP_ARC
	  object_dispose(anObject);
#else
	  object_setClass((id)anObject, (Class)(void*)0xdeadface);
	  NSZoneFree(z, o);
#endif
	}
    }
  return;
}
```
上面这么一大堆就是销毁最开始`alloc`分配的内存。

以上就是一个对象完整的生命周期了。

##### autorelease
说到这个内存管理，autorelease 也需要了解一下。
autorelease 与作用域的概念很类似，当超出其作用范围时，销毁作用域里面的变量。
其步骤如下：

1. 生成并持有 NSAutoreleasePool 对象
2. 调用已分配对象的 autorelease 实例方法
3. 废弃 NSAutoreleasePool 对象

有个很典型的例子：
在大量读入图片改变其尺寸时，图像文件读入到 NSData 对象，并从中生成 UIImage 对象，改变该对象尺寸后生成新的 UIImage 对象。这种情况下，就会大量产生 autorelease 对象。
在这种情况下，有必要在适当的地方生成、持有或废弃 NSAutoreleasePool 对象。
```
for (int i = 0; i < 图片数量; i++){
	NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
	/**
	 * 读入图像
	 * 产生大量 autorelease 对象
	 */
	 [pool drain];	// autorelease 的对象被一起 release
}
```
以上就是 Objective-C 的内存管理。其实无论 ARC 还是 MRC 都一样，区别就是 ARC 的情况下，编译器帮我们在合适的地方加入了 retain 和 release 操作。

##### 内存管理语义

还记得上一篇讲声明成员变量的两种方式那篇文章么，其中在用`@property`这种方式的时候，可以指定该属性的所有权修饰符。
ARC 下常用的修饰符有4种：

* __autoreleasing
* __strong
* __unsafe_unretained
* __weak

是不是有点疑惑，和我们常用的不一样。别慌[文档](http://clang.llvm.org/docs/AutomaticReferenceCounting.html)告诉我们该怎么用。

* assign 和 __unsafe_unretained 一样，表示非拥有关系。
* copy 和 __strong 一样表示拥有关系，还多出来了 setter 上复制语义的通常行为。
* retain 和 __strong 一样表示拥有关系。
* strong 和 __strong 一样表示拥有关系。
* weak 和 __weak 一样，表示非拥有关系。
* unsafe_unretained 和 __unsafe_unretained 一样，表示非拥有关系。
 
**__strong**
__strong是 id 类型和对象类型的默认修饰符。strong顾名思义"强引用"。超出其作用域时，强引用失效。
```
// objA 持有 对象 A 的强引用
id __strong objA = [[NSObject alloc] init];   // 对象 A
// objB 持有 对象 B 的强引用
id __strong objB = [[NSObject alloc] init];   // 对象 B
// objC 不持有任何对象
id __strong objC = nil;
/**
 * objA 持有objB 对象的强引用
 * 所以 原来由 objA 持有的对象A，强引用失效，对象A 的持有者不存在，所以废弃对象 A
 * 此时，对象 B 的强引用变量变为 objA 和 objB。
 */
objA = objB;

/**
 * objC 持有objA 的对象 B 的强引用
 * 此时，对象 B 的强引用变量变为 objA， objB 和 objC。
 */
objC = objA;

/**
 * objB 被赋值为 nil，不再持有原对象 B
 * 此时，对象 B 的强引用变量变为 objA 和 objC。
 */
objB = nil;

/**
 * objA 被赋值为 nil，不再持有原对象 B
 * 此时，对象 B 的强引用变量变为 objC。
 */
objA = nil;

/**
 * objC 被赋值为 nil，不再持有原对象 B
 * 此时，对象 B 的强引用不存在，因此废弃对象 B
 */
objC = nil;

```

**__weak**
引用计数管理有个绕不开的问题就是循环引用，当对象 A 和对象 B 都互相持有对方，那他们的引用计数就永远不可能为0，那就不能被释放，导致内存泄露。所以内存泄露就是指应该被废弃的对象在超出其生命周期后继续存在。

__weak 除了能消除循环引用外， 还有一个优点就是当持有的对象被销毁是，此弱引用将自动失效且处于 被nil 复制的状态。

所有被 __weak 修饰的对象，都会被加入到自动释放池中。(每次访问该对象，都会注册到自动释放池，开销更大，所以在 block 外部使用 __weak修饰的对象打破循环引用后，在 block 内部改为用 __strong修饰，以节省开销)

**__unsafe_unretained**
__unsafe_unretained 和 __weak 很像，但是当对象被销毁后，再访问该对象会出现奔溃(悬垂指针)。所以一般来说，对象用 weak 修饰，标量类型由 assign 修饰。

**__autoreleasing**
这个类型和上述的 atuorelease 的作用一样，就是将被修饰的对象，放入自动释放池中。

Objective-C 的内存管理大概就是这么回事了吧。

#### 参考文献
* [自动引用计数](http://clang.llvm.org/docs/AutomaticReferenceCounting.html)
[GNUstep 中的 NSObject](https://github.com/gnustep/libs-base/blob/master/Source/NSObject.m)
* 《Objective-C 高级编程》



