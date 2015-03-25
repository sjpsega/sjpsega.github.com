---
layout: post
title: "重置 iOS UIWebView 的缩放"
date: 2015-03-25 14:01:19 +0800
comments: true
categories: study
description: 重置 iOS UIWebView 的缩放
keywords: iOS UIWebView reset zoom
---

## 需求
遇到一个需求，iOS webView设置了可缩放，用户缩放webView后，做某个操作，需要将webView的缩放重置。

## 经过
在网上找了下，看到很多很多解决方案，不论是国内的，还是国外的解决方案，但是实际使用都没有效果……


## 解决
经过不断尝试，发现以下方式是有效果的：

```objc
//重置 webView 的 html viewport 属性，变成不可缩放，并且缩放比例为1.0，并执行javaScript
#define QUOTE(...) #__VA_ARGS__
const char *webViewHeightJSString = QUOTE(
var viewportmeta = document.querySelector('meta[name="viewport"]');
if (viewportmeta) {
viewportmeta.content = 'width=device-width, initial-scale=1.0, 		minimum-scale=1.0, maximum-scale=1.0, user-scalable=no';
}
);
#undef QUOTE
[webView stringByEvaluatingJavaScriptFromString:[NSString stringWithUTF8String:webViewHeightJSString]];
//设置 webView zoomScale 为 1.0
webView.scrollView.zoomScale = 1.0;
```

此方法在iOS7、8上的 UIWebView 上测试通过，WKWebView 没有测试。


