---
layout: post
title: "Category添加实例变量的故事"
date: 2014-04-07 20:55:16 +0800
comments: true
categories: study
keywords: iOS, Category, Instance variables

---

## 起因
有一个需求，针对 url 字符串，需要根据传入的key获取对应参数键值对的value。例如：

```objc
NSString *urlString = @"http://www.1688.com?key1=val1&key2=val2";
NSURL *url = [NSURL URLWithString:urlString];
NSString *val = [url paramWithKey:@"key1"];
```

NSURL 类虽然自带方法很多，但是却没有这个有用的方法。故需要做类扩展（Category）。

实现的时候，需要一个 NSDictionary 类型的实例变量来存储 url 字符串的参数键值对，便于多次调用 -paramWithKey 的时候，提高性能。

## 经过
写的时候，闹了几个笑话，走了弯路，记录下来。

### 在头文件中添加变量

```objc
@interface NSURL (WingQueryDictionary)
@property (nonatomic, copy) NSDictionary *queryDictionary;
@end
```

结果：编译不报错，但是实际执行中，实例中，找不到 self.queryDictionary 这个变量。

### 直接在 Category 中添加实例变量

```objc
@implementation NSURL (QueryDictionary){
	NSDictionary *queryDictionary;
}
@end
```

结果：编译直接报错，因为`Category不允许为已有的类添加新的属性或者实例变量，只能添加新方法`。

### 在匿名 Category 中添加实例变量

```objc
@interface NSURL(){
    NSDictionary *queryDictionary;
}
@end

@implementation NSURL (QueryDictionary){
	NSDictionary *queryDictionary;
}

- (NSString *)paramWithKey:(NSString *)key{
    return self.queryDictionary[key];
}
@end
```

结果：编译直接报错，原因同上。

### 添加一个类变量

```objc
static NSDictionary *queryDictionary;

@implementation NSURL (WingQueryDictionary)
@end
```

结果：能够完成需求，但是若新建多个 NSURL 并调用 -paramWithKey 方法，参数获取会有bug。

### 利用 runtime.h 来实现添加实例变量

最后，实在没辙了，只能 google 找方法。

发现可以`利用 runtime.h 中 objc_getAssociatedObject / objc_setAssociatedObject 方法来模拟生成实例变量`。

关键代码：
```objc
#import <objc/runtime.h>

static const char *varKey = "queryDictionary";

- (NSDictionary *)queryDictionary {
    return (NSDictionary *)objc_getAssociatedObject(self, &varKey);
}

- (void)setQueryDictionary:(NSDictionary *)queryDictionary {
    objc_setAssociatedObject(self, &varKey, queryDictionary, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```

### API 解析

设置实例对象：
```objc
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
```
用该方法来完成实例变量的 setter 实现。

* 参数 id：设置实例对象相关的实例，通常为self
* 参数 key：设置实例对象的key，即name
* 参数 value：设置实例对象的值
* 参数 policy：设置实例对象的内存对策，可选值如下：
  * (weak) / (assign) OBJC_ASSOCIATION_ASSIGN
  * (strong) / (retain) OBJC_ASSOCIATION_RETAIN
  * (copy) OBJC_ASSOCIATION_COPY
  * (nonatomic,strong) OBJC_ASSOCIATION_RETAIN_NONATOMIC
  * (nonatomic,copy) OBJC_ASSOCIATION_COPY_NONATOMIC


获得实例对象：
```objc
id objc_getAssociatedObject(id object, void *key)
```
完成实例变量的 getter实现。看了 setter 实现，这个方法的参数就很容易理解了。

## 参考资料

[让Category支持添加属性与成员变量](http://www.cnblogs.com/wupher/archive/2013/01/05/2845338.html)

[Adding Properties to Objective-C Categories](http://kaspermunck.github.io/2012/11/adding-properties-to-objective-c-categories/)
