---
layout: post
title: "《#2 并发编程》 学习笔记"
date: 2014-06-12 12:49:46 +0800
comments: true
categories: study
keywords: iOS GCD 并发
---
文章地址：[objc第二章](http://objccn.io/issue-2/)

## 并发编程：API 及挑战

### OS X 和 iOS 中的并发编程
#### Grand Central Dispatch(GCD)

GCD 在后端管理着一个线程池，可以将开发者从线程管理的工作中解放出来，通过集中的管理线程，来缓解大量线程被创建的问题。在日常开发中，最好使用 GCD 来做线程方面的工作。

GCD 公开有 5 个不同的队列：运行在主线程中的 main queue，3 个不同优先级的后台队列，以及一个优先级更低的后台队列（用于 I/O）

使用不同优先级的若干个队列乍听起来非常直接，不过，我们强烈建议，`在绝大多数情况下使用默认的优先级队列就可以了`。

![http://img.objccn.io/issue-2/gcd-queues.png](http://img.objccn.io/issue-2/gcd-queues.png)

#### Operation Queues

操作队列（operation queue）是由 GCD 提供的一个队列模型的 Cocoa 抽象。GCD 提供了更加底层的控制，而操作队列则在 GCD 之上实现了一些方便的功能，这些功能对于 app 的开发者来说通常是最好最安全的选择。

`NSOperationQueue` 有两种不同类型的队列：主队列和自定义队列。主队列运行在主线程之上，而自定义队列在后台执行。在两种类型中，这些队列所处理的任务都使用 `NSOperation` 的子类来表述。

#### Run Loops
Run loop并不像 GCD 或者操作队列那样是一种并发机制，因为它并不能并行执行任务。不过在主 dispatch/operation 队列中， run loop 将直接配合任务的执行，它提供了一种异步执行代码的机制。

Run loop 比起操作队列或者 GCD 来说容易使用得多，因为通过 run loop ，你不必处理并发中的复杂情况，就能异步地执行任务。

### 并发编程中面临的挑战
#### 资源共享
并发编程中许多问题的根源就是在多线程中访问共享资源。

在多线程中任何一个共享的资源都可能是一个潜在的冲突点，你必须精心设计以防止这种冲突的发生。

为了防止出现这样的问题，多线程需要一种互斥的机制来访问共享资源。

#### 互斥锁
互斥访问的意思就是同一时刻，只允许一个线程访问某个特定资源。

从语言层面来说，在 Objective-C 中将属性以 atomic 的形式来声明，就能支持互斥锁了。事实上在默认情况下，属性就是 atomic 的。将一个属性声明为 atomic 表示每次访问该属性都会进行隐式的加锁和解锁操作。虽然最把稳的做法就是将所有的属性都声明为 atomic，但是加解锁这也会付出一定的代价。

#### 死锁
互斥锁解决了竞态条件的问题，但很不幸同时这也引入了一些其他问题，其中一个就是死锁。当多个线程在相互等待着对方的结束时，就会发生死锁，这时程序可能会被卡住。

#### 资源饥饿（Starvation）
大多数情况下，限制资源一次只能有一个线程进行读取访问其实是非常浪费的。因此，在资源上没有写入锁的时候，持有一个读取锁是被允许的。这种情况下，如果一个持有读取锁的线程在等待获取写入锁的时候，其他希望读取资源的线程则因为无法获得这个读取锁而导致资源饥饿的发生。

#### 优先级反转
优先级反转是指程序在运行时低优先级的任务阻塞了高优先级的任务，有效的反转了任务的优先级。

解决这个问题的方法，通常就是不要使用不同的优先级。通常最后你都会以让高优先级的代码等待低优先级的代码来解决问题。`当你使用 GCD 时，总是使用默认的优先级队列（直接使用，或者作为目标队列）`。如果你使用不同的优先级，很可能实际情况会让事情变得更糟糕。

### 总结

建议采纳的安全模式：从主线程中提取出要使用到的数据，并利用一个操作队列在后台处理相关的数据，最后回到主队列中来发送你在后台队列中得到的结果。使用这种方式，你不需要自己做任何锁操作，这也就大大减少了犯错误的几率。


## 常见的后台实践
### 操作队列 (Operation Queues) 还是 GCD ?
`GCD 是基于 C 的底层的 API ，而操作队列则是 GCD 实现的 Objective-C API。`

`操作队列可以取消在任务处理队列中的任务`，而在 GCD 中不容易实现。

#### 后台的 Core Data
`Core Data`不熟悉

### 后台 UI 代码
`UIKit 只能在主线程上运行。`而那部分不与 UIKit 直接相关，却会消耗大量时间的 UI 代码可以被移动到后台去处理。

### 异步网络请求处理
* 所有网络请求都应该采取异步的方式完成。
* 可以使用 `NSURLConnection` 的异步方法，并且把所有操作转化为 operation 来执行。通过这种方法，我们可以从操作队列的强大功能和便利中获益良多：我们能轻易地控制并发操作的数量，添加依赖，以及取消操作。
* 使用像 [AFNetworking](http://afnetworking.com/) 这样的框架：建立一个独立的线程，为建立的线程设置自己的 run loop，然后在其中调度 URL 连接。但是并不推荐你自己去实现这些事情。

### 进阶：后台文件 I/O
大体上逐行读取一个文件的模式是这样的：

1. 建立一个中间缓冲层以提供，当没有找到换行符号的时候可以向其中添加数据
2. 从 stream 中读取一块数据
3. 对于这块数据中发现的每一个换行符，取中间缓冲层，向其中添加数据，直到（并包括）这个换行符，并将其输出
4. 将剩余的字节添加到中间缓冲层去
5. 回到 2，直到 stream 关闭

### 总结
在主队列中接收事件或者数据，然后用后台操作队列来执行实际操作，然后回到主队列去传递结果，遵循这样的原则来编写尽量简单的并行代码，将是保证高效正确的不二法则。

## 底层并发 API
### dispatch_once
```objc 
+ (UIColor *)boringColor;
{
    static UIColor *color;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        color = [UIColor colorWithRed:0.380f green:0.376f blue:0.376f alpha:1.000f];
    });
    return color;
}
```
上面的 block 只会运行一次。并且在连续的调用中，这种检查是很高效的。你能使用它来初始化全局数据比如单例。要注意的是，使用 `dispatch_once_t` 会使得测试变得非常困难（单例和测试不是很好配合）。

`要确保 onceToken 被声明为 static ，或者有全局作用域。任何其他的情况都会导致无法预知的行为。`

### dispatch_after
它使工作延后执行。它是很强大的，但是要注意：你很容易就陷入到一堆麻烦中。
```objc
- (void)foo
{
    double delayInSeconds = 2.0;
    dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t) (delayInSeconds * NSEC_PER_SEC));
    dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
        [self bar];
    });
}
```
这里存在一些缺点。我们`不能（直接）取消我们已经提交到 dispatch_after 的代码，它将会运行`。

如果你需要一些事情在某个特定的时刻运行，那么 `dispatch_after` 或许会是个好的选择。确保同时考虑了 `NSTimer`，这个API虽然有点笨重，但是它`允许你取消定时器的触发`。

### 队列
GCD 中一个基本的代码块就是队列。

```objc
- (id)init
{
    self = [super init];
    if (self != nil) {
        NSString *label = [NSString stringWithFormat:@"%@.isolation.%p", [self class], self];
        self.isolationQueue = dispatch_queue_create([label UTF8String], 0);

        label = [NSString stringWithFormat:@"%@.work.%p", [self class], self];
        self.workQueue = dispatch_queue_create([label UTF8String], 0);
    }
    return self;
}
```

队列可以是并行也可以是串行的，也可以是并行的。默认情况下，是串行的。

GCD 队列的内部使用的是线程。

#### 目标队列
你能够为你创建的任何一个队列设置一个目标队列。

为一个类创建它自己的队列而不是使用全局的队列被普遍认为是一种好的风格。这种方式下，你可以设置队列的名字，这让调试变得轻松许多—— Xcode 可以让你在 Debug Navigator 中看到所有的队列名字，如果你直接使用 lldb。(lldb) thread list 命令将会在控制台打印出所有队列的名字。一旦你使用大量的异步内容，这会是非常有用的帮助。

### 隔离
#### 资源保护
多线程编程中，最常见的情形是你有一个资源，每次只有一个线程被允许访问这个资源。

#### 全都使用异步分发
#### 如何写出好的异步 API
如果你的方法或函数有一个返回值，异步地将其传递给一个回调处理程序。这个 API 应该是这样的，你的方法或函数同时持有一个结果 block 和一个将结果传递过去的队列。
```objc
- (void)processImage:(UIImage *)image completionHandler:(void(^)(BOOL success))handler;
{
    dispatch_async(self.isolationQueue, ^(void){
        // do actual processing here
        dispatch_async(self.resultQueue, ^(void){
            handler(YES);
        });
    });
}
```

### 迭代执行

dispatch_apply
```objc
for (size_t y = 0; y < height; ++y) {
    for (size_t x = 0; x < width; ++x) {
        // Do something with x and y here
    }
}
```

小小的改动或许就可以让它运行的更快：
```objc
dispatch_apply(height, dispatch_get_global_queue(0, 0), ^(size_t y) {
    for (size_t x = 0; x < width; x += 2) {
        // Do something with x and y here
    }
});
```

### 组
```objc
dispatch_group_t group = dispatch_group_create();

dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
dispatch_group_async(group, queue, ^(){
    // Do something that takes a while
    [self doSomeFoo];
    dispatch_group_async(group, dispatch_get_main_queue(), ^(){
        self.foo = 42;
    });
});
dispatch_group_async(group, queue, ^(){
    // Do something else that takes a while
    [self doSomeBar];
    dispatch_group_async(group, dispatch_get_main_queue(), ^(){
        self.bar = 1;
    });
});

// This block will run once everything above is done:
dispatch_group_notify(group, dispatch_get_main_queue(), ^(){
    NSLog(@"foo: %d", self.foo);
    NSLog(@"bar: %d", self.bar);
});
```

#### 对现有API使用 dispatch_group_t
```objc
+ (void)withGroup:(dispatch_group_t)group 
        sendAsynchronousRequest:(NSURLRequest *)request 
        queue:(NSOperationQueue *)queue 
        completionHandler:(void (^)(NSURLResponse*, NSData*, NSError*))handler
{
    if (group == NULL) {
        [self sendAsynchronousRequest:request 
                                queue:queue 
                    completionHandler:handler];
    } else {
        dispatch_group_enter(group);
        [self sendAsynchronousRequest:request 
                                queue:queue 
                    completionHandler:^(NSURLResponse *response, NSData *data, NSError *error){
            handler(response, data, error);
            dispatch_group_leave(group);
        }];
    }
}
```
为了能正常工作，你需要确保:
* dispatch_group_enter() 必须要在 dispatch_group_leave()之前运行。
* dispatch_group_enter() 和 dispatch_group_leave() 一直是成对出现的。

### 事件源
GCD 有一个较少人知道的特性：事件源 `dispatch_source_t`。

当你需要用到它时，它会变得极其有用。它的一些使用是秘传招数，我们将会接触到一部分的使用。但是大部分事件源在 iOS 平台不是很有用，因为在 iOS 平台有诸多限制，你无法启动进程（因此就没有必要监视进程），也不能在你的 app bundle 之外写数据（因此也就没有必要去监视文件）等等。

GCD 事件源是以极其资源高效的方式实现的。

#### 监视进程
#### 监视文件
#### 定时器
#### 取消

### 输入输出
写出能够在繁重的 I/O 处理情况下运行良好的代码是一件非常棘手的事情。

#### GCD 和缓冲区
最直接了当的方法是使用数据缓冲区。GCD 有一个 `dispatch_data_t` 类型，在某种程度上和 Objective-C 的 NSData 类型很相似。但是它能做别的事情，而且更通用。

#### 读和写 
在 GCD 的核心里，调度 I/O（译注：原文为 Dispatch I/O） 与所谓的通道有关。调度 I/O 通道提供了一种与从文件描述符中读写不同的方式。

### 基准测试
能够测量给定的代码执行的平均的纳秒数。
```objc
uint64_t dispatch_benchmark(size_t count, void (^block)(void));
```
### 原子操作
头文件 `libkern/OSAtomic.h` 里有许多强大的函数，专门用来底层多线程编程。尽管它是内核头文件的一部分，它也能够在内核之外来帮助编程。

#### 计数器
#### 原子队列
#### 自旋锁
OSAtomic.h 头文件定义了使用自旋锁的函数：OSSpinLock。

## 线程安全类的设计
### 线程安全
#### Apple 的框架
一般来说除非特别声明，大多数的类默认都`不是`线程安全的。

Apple没有把 UIKit 设计为线程安全的类是有意为之的，将其打造为线程安全的话会使很多操作变慢。而事实上 UIKit 是和主线程绑定的，这一特点使得编写并发程序以及使用 UIKit 十分容易的，你唯一需要确保的就是对于 UIKit 的调用总是在主线程中来进行。

#### 为什么 UIKit 不是线程安全的？
对于一个像 UIKit 这样的大型框架，确保它的线程安全将会带来巨大的工作量和成本。

#### 内存回收 (deallocation) 问题
另一个在后台使用 UIKit 对象的的危险之处在于“内存回收问题”。

#### Collection 类
总的来说，比如 `NSArry 这样不可变类是线程安全的`。然而它们的可变版本，比如 `NSMutableArray 是线程不安全的`。

一个好习惯是写类似于 `return [array copy]` 这样的代码来确保返回的对象事实上是不可变对象。

#### 原子属性 (Atomic Properties)

##### 为何不用 @synchronized ？
`@synchonized(self)` 更适合使用在你需要确保在发生错误时代码不会死锁，而是抛出异常的时候。

#### 你自己的类
单独使用原子属性并不会使你的类变成线程安全。它不能保护你应用的逻辑，只能保护你免于在 setter 中遭遇到竞态条件的困扰。

一个简单的解决办法是使用 `@synchronize`。其他的方式都非常非常可能使你误入歧途，已经有太多聪明人在这种尝试上一次又一次地以失败告终。

##### 可行的线程安全设计
对于那些肯定应该线程安全的代码（一个好例子是负责缓存的类）来说，一个不错的设计是使用并发的 dispatch_queue 作为读/写锁，并且确保只锁着那些真的需要被锁住的部分，以此来最大化性能。

### GCD 的陷阱
对于大多数上锁的需求来说，GCD 就足够好了。它简单迅速，并且基于 block 的 API 使得粗心大意造成非平衡锁操作的概率下降了不少。

#### 将 GCD 当作递归锁使用
GCD 是一个对共享资源的访问进行串行化的队列。这个特性可以被当作锁来使用，但实际上它和 @synchronized 有很大区别。 

#### 用 dispatch_async 修复时序问题

#### 在性能关键的代码中混用 dispatchsync 和 dispatchasync

#### 使用 dispatch_async 来派发内存敏感的操作

## 测试并发程序
以 SenTestingKit 为例子，在XCode5中已经不再使用。

http://www.cnblogs.com/yjg2014/p/yjg.html

http://m.blog.csdn.net/blog/againbike/11870459