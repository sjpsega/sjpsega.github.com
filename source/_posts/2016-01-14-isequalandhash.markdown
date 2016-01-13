---
layout: post
title: "isEqual 与 hash"
date: 2016-01-14 01:17:56 +0800
comments: true
categories: study
description: isEqual hash
keywords: isEqual hash
---

## 最佳实践
* 实现 `isEqualTo__ClassName__`
* 重写 isEqual 方法，同时重写 hash 方法。


## 关系
* 两个对象 isEqual 相等，hash 必须也相等
* 两个对象 hash 相同，isEqual 则不一定相等

## 碰撞
当对象存入集合对象时（Array，Set，HashTable等），内部会使用对象的 hash 值来作为 key 来存入。
当两个不相等的对象，有相同的 hash 值，存入集合对象，就会发生`碰撞`现象。
发生碰撞现象，对使得存取数据变慢，所以需要尽量避免这个现象，但不可能完全避免。

## 作为集合对象 key 的注意点
* 实现 isEqual 和 hash 方法，遵守两者关系，尽量避免 hash 碰撞的情况发生
* 实现 NSCopying 协议，接口：`- (void)setObject:(ObjectType)anObject forKey:(KeyType <NSCopying>)aKey;`

## 参考资料
[Equality](http://nshipster.com/equality/)

[Implementing Equality and Hashing](https://mikeash.com/pyblog/friday-qa-2010-06-18-implementing-equality-and-hashing.html)