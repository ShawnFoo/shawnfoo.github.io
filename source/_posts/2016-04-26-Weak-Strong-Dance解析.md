---
title: Weak-Strong Dance解析
date: 2016-04-26
tags: [iOS, block, 2016]
categories: [小结]
---
我们在使用Block时常常看到`Weak-Strong Dance`的用法, 很多的文章以及[官方文档](https://developer.apple.com/library/mac/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html#//apple_ref/doc/uid/TP40011226-CH1-SW4)都举例了这样做的原因. 但是还尚未发现有对strong进行讲解的. 下面就举个栗子具体分析下**为什么加strong**以及**何时起作用**

首先放上两个类似ReactiveCocoa中定义weakify和strongify的宏, 以便下文用到
> \#define WeakObj(o) autoreleasepool{} \__weak typeof(o) weak##o = o
> \#define StrongObj(o) autoreleasepool{} \__strong typeof(o) o = weak##o

<!-- more -->

---
#### 一、weak的作用(代码+注解 简单跳过)
防止被block捕获(会导致引用计数加1), 打破循环引用(retain cycle)

```
// DeallocMonitor继承NSObject, 仅重写其dealloc方法, 并在其中打印其被释放日志
DeallocMonitor *object1 = [DeallocMonitor new];
DeallocMonitor *object2 = [DeallocMonitor new];
@WeakObj(object2);//__weak typeof(object2) weakobject2 = object2;
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
        // object1被block捕获, 引用计数加1, 外部作用域结束时仍未被释放, 直至该Block执行完毕才被释放
        NSLog(@"5s已到, %@该被释放勒", object1);
        // weakobject2被weak 修饰, 其指向的object2对象的引用计数不会增加, 当外部作用域结束时就已被释放
        NSLog(@"5s已到, %@早已被释放, 此处为null", weakobject2);
});
// 外部作用域结束
```

---
#### 二、为何要加strong, 其何时才起作用？
加strong的原因想必大家都知道是为了**防止block执行过程中 `__weak typeof(object) weakObject`指向的对象突然被释放了**, 这就会导致block中的代码运行结果出现意想不到的结果(比如一些代码执行有效, 其余代码执行无效; 弱引用的对象因为为nil而导致的crash等.)

##### 2.1 即使加了strong, 也不能保证weakObject指向的对象不会被释放
只能确保在block执行期间, weakObject指向的对象有效(不会被释放)

下面这段代码就是在block中用strong申明的对象强引用一次weakObject, 但修饰对象在block执行前就已经被释放的栗子

```
// DeallocMonitor继承NSObject, 仅重写其dealloc方法, 并在其中打印其被释放日志
DeallocMonitor *object = [DeallocMonitor new];
@WeakObj(object);// weakobject
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
        // 该strongObj的申明仅在block执行时才见效, 而外部作用域一结束object就已经被释放了, 所以然并卵
       @StrongObj(object);
        /* weakobject用 weak修饰, 故其引用计数不变, 
          上边的宏本意是申明一个新的object局域变量对weakobject指向的原object进行强引用..
          按理 原object引用计数应该会加1, 可是它还没等到被强引用时就已经挂掉了
        */
        NSLog(@"5s已到, %@然后早已被释放, 此处为null", object);
});
// 外部作用域结束
```

##### 2.2 Block内部申明的强引用指针变量指向weakObject仅在block执行时才有效
定义该Block的时strongObj宏还尚未使原对象引用计数加1! 那么strongObj宏生效时的表现是什么样子的呢? 继续上代码

```
// 该段代码主要是打了一个时间差, 以模拟strong申明起作用的情形

// DeallocMonitor继承NSObject, 仅重写其dealloc方法, 并在其中打印其被释放日志
DeallocMonitor *object = [DeallocMonitor new];
// 保证外部作用域结束的2.5秒(无限接近..)内object不会被释放
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.5 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
        NSLog(@"果断强引用object: %@\n 还能再多坚持2.5s", object);
});
@WeakObj(object);// weakobject
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
       @StrongObj(object);// __strong typeof(object) object = weakobject
        sleep(3);// 卡个3s
        // 此处就不会像上一段代码那样, 强引用一个为nil的object, 故weakobject指向的对象引用计数加1, 直到该block运行完, 才会被释放
        NSLog(@"5s已到, %@打印完这个日志就飞升了", object);
});
// 外部作用域结束
```

##### 2.3 有多少个嵌套block就应该申明多少对weak-strong
假定我们在最外层block使用的一对weak-strong, 且外层block内还有一个block(没有用weak-strong)引用到了strongObj宏申明的局域变量object, 并假设原对象在外层block开始运行前一直存活, 这就会导致内层block捕获到局域变量object并使其指向对象的引用计数加1, 因为内层block捕获到了外层block中申明的object(强引用), 就跟外层block会捕获到外部强引用变量指向的对象一样一样的

```
DeallocMonitor *object = [DeallocMonitor new];
@WeakObj(object);
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
       @StrongObj(object);// 因为block运行时, weakObject指向对象依旧存在, 故该强引用使其引用计数加1
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_global_queue(0, 0), ^{
            // 这一层block 发现上边的object是强引用, 导致捕获到其指向对象, 使其引用计数在该内层block尚未执行时就加1了
            NSLog(@"打印完这个日志, %@才被释放", object);
        });
        NSLog(@"%@外层block结束, 引用计数减一", object);
});
sleep(3);
// 外部作用域所在线程小歇一会, 确保object存活3s, 作用域结束
```
所以嵌套block时 万万要小心, 不要漏写了. 另外weak-strong要成对出现, 不然少一个strong, 都有可能为此付出代价

##### 2.4 遗漏补缺
1. 在block中对外部weakObject进行强引用(strong修饰)的结果是使weakObject指向的原对象的引用计数加1, 因为weakObject指针指向的是原对象在堆中的存储地址
2. block 不会对弱引用指针变量指向的对象进行捕获

##### 2.5 block的相关知识, 个人推荐书籍章节
* Effective-ObjectiveC(Item 37: Understand Blocks)
* Pro Multithreading and Memory Management for iOS and OS X(Blocks Implementation)

#### 三、题外篇(内存泄露检测工具-妈妈再也不用担心内存泄露)

对于ReactiveCocoa以及各种嵌套Block的常用玩家..想必仅靠Xcode的Instrument去检测memory leak问题是绝对不够的,  个人卖瓜推荐一个检测内存泄露的小工具类:
[FXDeallocMonitor](https://github.com/ShawnFoo/FXCustomTabBarController/blob/master/FXCustomTabBarController/FXDeallocMonitor.m)
拷贝FXDeallocMonitor.h、FXDeallocMonitor.m文件到项目中, 根据头文件中的方法调用就行, 简单易用😄

对于需求更高者, 推荐近期facebook开源的[FBAllocationTracker](https://github.com/facebook/FBAllocationTracker)