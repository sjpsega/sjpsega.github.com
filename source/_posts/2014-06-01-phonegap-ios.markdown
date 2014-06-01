---
layout: post
title: "iOS版PhoneGap原理分析"
date: 2014-06-01 00:59:15 +0800
comments: true
categories: study
keywords: PhoneGap iOS cordova
---

PhoneGap，就是现在的[Cordova](http://cordova.apache.org/)，著名的跨平台Hybrid框架，旨在让开发者使用HTML、Javascript、CSS开发跨平台的App。

新名字好难记，所以这篇文章我还是叫PhoneGap，比较顺口。（总觉得PhoneGap是个很好的名字，改名为Cordova真是一个大败笔……）

最近的工作，就是做Hybrid方面的，很自然，方案就从PhoneGap入手。

下面就切入正题，分析下PhoneGap的原理，需要说明的是，我只针对iOS版本的PhoneGap做分析，android版本的原理大同小异。

## js与native通信的原理

但在切入正题前，需要先了解下iOS js与native通信的原理。了解这个原理，是理解PhoneGap代码的关键。

具体可以看我之前写的[iOS Js与native相互通信](http://sjpsega.com/blog/2014/03/08/js-communicate-with-native-in-iOS/)，这里做简单说明。

### js -> native

在iOS中，js调用native并没有提供原生的实现，只能通过UIWebView相关的UIWebViewDelegate协议的

```objc
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
```
方法来做拦截，并在这个方法中，根据url的协议或特征字符串来做调用方法或触发事件等工作，如

```objc
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
    NSURL *url = [request URL];
    if ([[url scheme] isEqualToString:@"callFunction") {
        //调用原生方法

        return NO;
    } else if (([[url scheme] isEqualToString:@"sendEvent") {
        //触发事件

        return NO;
    } else {
        return YES;
    }
}
```

### native -> js
native调用js非常简洁方便，只需要
```objc
[webView stringByEvaluatingJavaScriptFromString:@"alert('hello world!')"];
```

## PhoneGap js -> native
PhoneGap是开源的，iOS版本的代码托管在GitHub上，可以从[这里下载](https://github.com/apache/cordova-ios)。

clone代码到本地后，打开项目，可以看到PhoneGap为了兼容iOS平台，不仅为iOS优化了js代码，还写了大量的Objective-C代码来完成兼容。

项目的代码量有点大，说下核心的代码：

* js部分：cordova.js中的iOSExec()方法，用来js调用native
* oc部分：
  * CDVViewController：拦截js调用native的url协议，执行调用规则
  * CDVCommandQueue：执行js调用native的队列，调用对应的plugin
  * CDVCommandDelegateImpl：执行回调js的工具类

## 时序图

## 结语