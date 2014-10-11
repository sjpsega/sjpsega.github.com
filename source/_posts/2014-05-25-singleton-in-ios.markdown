---
layout: post
title: "iOS中的单例模式"
date: 2014-05-25 12:25:10 +0800
comments: true
categories: study
keywords: 设计模式 单例 singleton iOS
---

单例模式是指在一个系统中，类有且只有一个实例对象，可以通过全局的一个入口对这个实例对象进行访问。

在 iOS 开发中，单例模式是非常常用的一种设计模式。

## ARC下的实现
iOS5.0 以后就开始可以使用ARC（Automatic Reference Counting：自动引用计数）来代替之前的MRC（Manual Reference Counting：人工引用计数）。

使用 ARC 会减少很多代码和忘了释放对象的苦恼。

慢慢 ARC 会成为主流，我入门 iOS 也是学 ARC 的，所以这里只写 ARC 的单例实现。

`Singleton.h:`
```objc
//
//  Singleton.h
//  Singleton

#import <Foundation/Foundation.h>

@interface Singleton : NSObject

+ (Singleton *)sharedInstance;

@end

```

`Singleton.m:`
```objc

//
//  Singleton.m
//  Singleton

#import "Singleton.h"

@implementation Singleton

+ (Singleton *)sharedInstance{
    static Singleton *singleton;
    static dispatch_once_t token;
    dispatch_once(&token,^{
        singleton = [[Singleton alloc]init];
    });
    return singleton;
}

- (id)init{
    self = [super init];
    if(self){
        //在这里可以进行类的初始化工作
    }
    return self;
}

@end

```

在这个实现中，核心是使用了GCD（Grand Central Dispatch）的`dispatch_once`方法，该方法可以保证Singleton只被实例化一次，并且该方法`线程安全`。

## 纰漏
以上实现有一个纰漏，Singleton 继承于 `NSObject`，`NSObject` 有一个公开的初始化方法 `-(id)init`，所以若使用者不小心，他完全可以使用 `[[Singleton alloc] init]` 来创建多个实例对象，这样轻易就破坏了单例的实现。

例如：
```objc
Singleton *single1 = [[Singleton alloc] init];
Singleton *single2 = [[Singleton alloc] init];
//结果为0，即NO，两者不是同一个实例
NSLog(@"%d",single1 == single2);
```

`2014-10-10 更新：`

`摒弃之前的实现方式：`
```
    ## 更好的实现
    其实是要对 `Singleton.m` 做一点改变，就能封住这个纰漏：

    ```objc
    //
    //  Singleton.m
    //  Singleton

    #import "Singleton.h"

    @implementation Singleton

    + (Singleton *)sharedInstance{
        static Singleton *singleton;
        static dispatch_once_t token;
        dispatch_once(&token,^{
            //这里调用私有的initSingle方法
            singleton = [[Singleton alloc]initSingle];
        });
        return singleton;
    }

    //只是把原来在init方法中的代码，全都搬到initSingle
    - (id)initSingle{
        self = [super init];
        if(self){
            //在这里可以进行类的初始化工作
        }
        return self;
    }

    - (id)init{
        //改为调用[Singleton sharedInstance]
        return [Singleton sharedInstance];
    }

    @end
    ```

    * 新建私有的initSingle方法，将原来init中的实现，全部搬入
    * sharedInstance中的singleton初始化，调用initSingle方法
    * init方法中，调用[Singleton sharedInstance]
```

`更好的实现：`

最佳的实现方式是重写 `+allocWithZone` 方法，使得在给对象分配内存空间的时候，就指向同一份数据，并且 `+alloc` 默认调用的是 `+allocWithZone` 方法，重写 `+allocWithZone` 是最根本的解决方案。

在内存分配阶段就指向同一份数据了，如果外界调用 alloc 与 init 来新建对象，自然还是指向同一份数据，问题自然解决了。

```objc
//
//  Singleton.m
//  Singleton

#import "Singleton.h"
@implementation Singleton

+ (Singleton *)sharedInstance{
    static dispatch_once_t onceToken;
    static Singleton *_singleton = nil;
    dispatch_once(&onceToken,^{
        _singleton = [[super allocWithZone:NULL] init];
    });
    return _singleton;
}

+ (id)allocWithZone:(struct _NSZone *)zone{
    return [self sharedInstance];
}

```

[代码示例](https://github.com/sjpsega/SingletonTest)

## 参考资料
[Creating a Singleton Instance](https://developer.apple.com/legacy/library/documentation/Cocoa/Conceptual/CocoaFundamentals/CocoaObjects/CocoaObjects.html#//apple_ref/doc/uid/TP40002974-CH4-SW32)

[Objective-C中单例模式的实现](http://cocoa.venj.me/blog/singleton-in-objc/)

[从 Objective-C 里的 Alloc 和 AllocWithZone 谈起](http://www.justinyan.me/post/1306)

[iOS_单例](http://www.cnblogs.com/yudigege/p/3943898.html)
