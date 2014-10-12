---
layout: post
title: "Objective-C Method Swizzling"
date: 2014-09-17 22:36:40 +0800
comments: true
categories: study
description: Method Swizzling
keywords: Method Swizzling
---

Objective-C 中的 Method Swizzling 是一种可以在程序运行时，修改方法调用的技术。是 OC 作为动态语言的典型证明。

Method Swizzling 是 OC `<objc/runtime.h>` 类库提供的“黑魔法”之一。

## 例子
以替换 NSArray 的 lastObject 方法为例：

* 在 NSArray 中添加需要替换 lastObject 的方法 - xxx_lastObject方法：

```objc
#import "NSArray+Swizzle.h"  

@implementation NSArray (Swizzle)  

- (id)xxx_lastObject  
{  
    id ret = [self xxx_lastObject];  
    NSLog(@"**********  myLastObject ***********");  
    return ret;  
}  
@end  
```

注意这里的写法，xxx_lastObject 方法的方法体中调用了 [self xxx_lastObject]，这样写并不会造成递归，后面会交换 xxx_lastObject 与 lastObject 的 IMP，其实 [self xxx_lastObject] 将会执行 [self lastObject] 。

* 调换 IMP

```objc
#import <objc/runtime.h>  
#import "NSArray+Swizzle.h"

Method ori_Method =  class_getInstanceMethod([NSArray class], @selector(lastObject));  
Method my_Method = class_getInstanceMethod([NSArray class], @selector(xxx_lastObject));  
method_exchangeImplementations(ori_Method, my_Method);  
```
注意这里需要引入 `<objc/runtime.h>`，否则会报错，因为使用了 runtime 中的 method_exchangeImplementations 方法。

使用 `method_exchangeImplementations` 交换了 xxx_lastObject 与 lastObject 的 IMP。后面会讲解更深层次的原理。

* 尝试调用 lastObject

```objc
#import <objc/runtime.h>
#import "NSArray+Swizzle.h"

NSArray *array = @[@"0",@"1",@"2",@"3"];  
NSString *string = [array lastObject];  
NSLog(@"TEST RESULT : %@",string);  
```

输出结果为:

```objc
**********  myLastObject ***********
TEST RESULT : 3
```

很明显，这里调用 lastObject 方法，其实是调用了我们添加的 xxx_lastObject 方法。

## 用处
Method Swizzling 非常强大，主要作用有：

* 在不修改 iOS 系统类库或第三方类库的源码基础上，修改原有调用逻辑
* 动态添加、修改方法，修复线上 bug（如果 Apple 官方允许的话）

## 常用 API
相关常用方法，都在`<objc/runtime.h>`包内：

```objc
//向类中添加Method
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)

//修改类的Method
IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)

//交换2个方法中的IMP
void method_exchangeImplementations(Method m1, Method m2)

//获取类的某个实例方法
Method class_getInstanceMethod(Class aClass, SEL aSelector);
```

## 底层原理

在运行时，OC 的方法是一种叫 `Method` 的结构体，这种 `objc_method` 类型的结构体定义为：

```objc
struct objc_method
   SEL method_name         OBJC2_UNAVAILABLE;
   char *method_types      OBJC2_UNAVAILABLE;
   IMP method_imp          OBJC2_UNAVAILABLE;
}
```

* method_name 是方法的 selector，可以理解为运行时的方法名；
* *method_types 是一个参数和返回值类型[编码](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)的字符串；
* method_imp 是指向方法实现的指针。

`Method Swizzling 的实质是在运行时，访问对象的方法结构体，并改变它的底层实现。`

我们以上节修改 NSArray 的 lastObject 为例做说明。

* 初始时，lastObject 与 xxx_lastObject 方法的 Method 结构体：

```objc
Method lastObject {
      SEL method_name = @selector(lastObject)
      char *method_types = “v@:“ //返回值void, 参数id(self)，selector(_cmd)
      IMP method_imp = 0x000FFFF //指向([NSArray lastObject])
}

Method xxx_lastObject {
       SEL method_name = @selector(swizzle_originalMethodName)
       char *method_types = “v@:”
       IMP method_imp = 0x1234AABA //指向([NSArray xxx_lastObject])
}
```

* 调用 `void method_exchangeImplementations(Method m1, Method m2)` ,交换两者的实现：

```objc
Method ori_Method =  class_getInstanceMethod([NSArray class], @selector(lastObject));  
Method my_Method = class_getInstanceMethod([NSArray class], @selector(xxx_lastObject));  
method_exchangeImplementations(ori_Method, my_Method);
```

* 调换后的 Method 结构体：

```objc
Method lastObject {
      SEL method_name = @selector(lastObject)
      char *method_types = “@@:“ //返回值id, 参数id(self)，selector(_cmd)
      IMP method_imp = 0x1234AABA //指向([NSArray xxx_lastObject])
}

Method xxx_lastObject {
       SEL method_name = @selector(swizzle_originalMethodName)
       char *method_types = “@@:”
       IMP method_imp = 0x000FFFF //指向([NSArray lastObject])
}
```

可以看到使用`void method_exchangeImplementations(Method m1, Method m2)`的实质是交换了xxx_lastObject 与 lastObject 的 IMP，实现了在运行时做方法的替换。使得当执行 [array lastObject] 的时候，实际会去执行 [array xxx_lastObject] 的方法实现。

## 其他提示

### +load
Swizzling 的处理，在类的 `+load` 方法中完成。

因为 `+load` 方法会在类被添加到 OC 运行时执行，且只会被调用一次，保证了 Swizzling 方法的及时处理。

### dispatch_once
Swizzling 的处理，`dispatch_once` 中完成。保证只执行一次。

### prefix
Swizzling 方法添加前缀，避免方法名称冲突。

## 代码示例
[代码示例](https://github.com/sjpsega/SwizzleTest)

## 参考资料
[Method Swizzling](http://nshipster.com/method-swizzling/)

[How to swizzle a class method on iOS?](http://stackoverflow.com/questions/3267506/how-to-swizzle-a-class-method-on-ios)

[iOS 4.3: imp_implementationWithBlock()](http://www.friday.com/bbum/2011/03/17/ios-4-3-imp_implementationwithblock/)

[Objective-C的hook方案（一）:  Method Swizzling](http://blog.csdn.net/yiyaaixuexi/article/details/9374411)

[Objective-C 的动态提示和技巧](http://blog.jobbole.com/45963/)
