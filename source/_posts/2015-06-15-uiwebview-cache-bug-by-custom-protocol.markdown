---
layout: post
title: "iOS UIWebView 自定义协议文件加载缓存问题"
date: 2015-06-15 22:56:31 +0800
comments: true
categories: study
description: iOS UIWebView cache bug
keywords: iOS UIWebView cache bug
---

## 起因
一个简单的需求：为了减少网络请求，一些前端资源会做本地缓存；使用 WebView 访问页面的时候，拦截特定请求，使得特定请求直接加载本地资源。

但是做的过程中发现，当请求的资源为自定义协议的时候，比如加载的 url 为 `abc://xxx.com/xx.js` 的时候，系统会强制对该资源做缓存，并且调用系统提供的清除缓存接口，如 `[[NSURLCache sharedURLCache] removeAllCachedResponses]`， 都无法清除该缓存，除非重启 App。

这就导致该资源内容发生变化的时候，无法立即生效，这在业务中，显然是不能接受的。

`注`：该方案主要针对 `UIWebView` 有效；`WKWebView` 因为不能使用自定义 NSURLProtocol 拦截资源，方法二就不起作用。

## 解决
###方法一：html 模板上，资源 url 加上随机字符串
在 html 页面上修改 url 字符串，如 `abc://xxx.com/xx.js`，需要改成 `abc://xxx.com/xx.js?t=123`，这样系统就不会缓存该资源。
该方法简单直接。

###方法二：创建自定义 NSURLProtocol，拦截请求，并且在创建 response 时，修改自定义协议为 http 或 https

伪代码如下：
```objc
//原请求
NSString *requestURLString = @"abc://xxx.com/xx.js";
//将原请求的自定义协议改成 http
NSString *changeURLString = @"http://xxx.com/xx.js";
//创建 response
NSURL *url = [NSURL URLWithString:changeURLString];
NSURLResponse* response =
[[NSURLResponse alloc] initWithURL:url
                          MIMEType:@"application/x-javascript"
             expectedContentLength:indexJSData.length
                  textEncodingName:@"UTF-8"];
```

本来结合方法一的解决方式，创建 response 的时候，在 url 后加上随机字符串，但是经过试验，该方案`无效`，缓存始终存在。

使用该解决方法解决缓存问题，据个人猜测是因为，iOS 系统默认缓存数据，但针对 http 或 https 协议会根据 Cache-Control 等字段判断是否缓存。

具体可见 [测试demo](https://github.com/sjpsega/CustomProtocolCacheTest)，并注意 log 信息。

## 讨论
解决这个缓存问题，其实我还是有些疑惑，因为试了所有 iOS 给出的数据缓存 API，都不起作用。这些包括：
* 使用 [[NSURLCache sharedURLCache] removeAllCachedResponses] 清理缓存
* 重写了 NSURLCache 的 - (NSCachedURLResponse *)cachedResponseForRequest:(NSURLRequest *)request 方法，不返回 response 对象
* 自定义 NSURLProtocol 中，使用 NSURLConnection 加载数据，实现 NSURLConnectionDelegate 的 - (NSCachedURLResponse *)connection:(NSURLConnection *)connection willCacheResponse:(NSCachedURLResponse *)cachedResponse 方法，返回 nil，不做对应 url 数据的缓存。

然后，方法一与方法二中，我都试验了 url 加随机字符串的方式，但是为什么 html 模板上的资源 url 加上随机字符串有效，创建 response 时候，url 加上随机字符串无效，我也赶到无解。

感觉在 webView 资源缓存的问题上，Apple 官方还是有些接口没开放，或者是我不知道...

## 参考资料
[URL Loading System](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/URLLoadingSystem/URLLoadingSystem.html#//apple_ref/doc/uid/10000165i)

[NSURLCache](http://nshipster.cn/nsurlcache/)
