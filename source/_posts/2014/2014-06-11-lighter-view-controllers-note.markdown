---
layout: post
title: "《#1 更轻量的 View Controllers》 学习笔记"
date: 2014-06-11 12:01:53 +0800
comments: true
categories: study
keywords: iOS ViewController
---
文章地址：[objc第一章](http://objccn.io/issue-1/)

## 更轻量的 View Controllers

* 针对`UITableViewController`等VC，可以把对应的`UITableViewDataSource`、`UITableViewDelegate`等`Protocol`代码，写成单独的类，再设置给VC的dataSource或deletege
* 将业务逻辑、网络请求逻辑移到 Model 中

## 整洁的 Table View 代码

### 添加Child View Controllers的步骤
```objc
ChildViewController *childVC = [[ChildViewController alloc] init];
[self addChildViewController:childVC];
CGRect frame = self.view.bounds;
childVC.view.frame = frame;
[self.view addSubview:childVC.view]; 
[childVC didMoveToParentViewController:self];
```

### 移除Child View Controllers的步骤
```objc
[childVC willMoveToParentViewController:nil];
[childVC.view removeFromSuperview];
[childVC removeFromParentViewController];
```
源自：[Custom Container View Controller](http://geeklu.com/2014/05/custom-container-view-controller/)

### 分离关注点（Separating Concerns）
这块最好对照看下文章的代码实践：[http://objccn.io/issue-1-2/](http://objccn.io/issue-1-2/)

## 测试 View Controllers

## View Controller 容器

### 过场动画 - 新老VC替换
```objc
toController.view.frame = fromController.view.bounds;                           
[self addChildViewController:toController];                                     
[fromController willMoveToParentViewController:nil];   
[self transitionFromViewController:fromController
                  toViewController:toController
                          duration:0.2
                           options:direction | UIViewAnimationOptionCurveEaseIn
                        animations:nil
                        completion:^(BOOL finished) {
                            [toController didMoveToParentViewController:self];  
                            [fromController removeFromParentViewController];    
                        }];
```

## 代码学习：
代码地址：[https://github.com/objcio/issue-1-lighter-view-controllers](https://github.com/objcio/issue-1-lighter-view-controllers)

### 反序列化
```objc
NSData *data = [NSData dataWithContentsOfURL:archiveURL options:0 error:NULL];
NSKeyedUnarchiver *unarchiver = [[NSKeyedUnarchiver alloc] initForReadingWithData:data];
_users = [unarchiver decodeObjectOfClass:[NSArray class] forKey:@"users"];
_photos = [unarchiver decodeObjectOfClass:[NSArray class] forKey:@"photos"];
[unarchiver finishDecoding];
```

[When to use NSSecureCoding](http://stackoverflow.com/questions/17301769/when-to-use-nssecurecoding)

[NSSecure​Coding](http://nshipster.com/nssecurecoding/)

