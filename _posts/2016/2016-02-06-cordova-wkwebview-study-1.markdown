---
layout: post
title: "Cordova WKWebView 学习笔记（一）"
date: 2016-02-06 13:43:47 +0800
comments: true
categories: study
description: cordova wkwebview
keywords: cordova wkwebview
---
## WKWebView
Apple 从 iOS 8 开始，引入了新的 WebView 类 `WKWebView`，试图替换已经老迈的 UIWebView。

### 优点
通过官方的描述，和自己实际测试，WKWebView功能相当强大，对比 UIWebView 的优点很多：

* 更少的内存使用，渲染相同页面，占用内存基本只有 UIWebView 的 1/3 ~ 1/4，并且大大减少了内存泄露的情况
* 使用与 Safari 一样的性能强大的 Nirtro JS 引擎
* 异常强大的 app 与 web 的内容传递
    * 可以使用 `WKUserContentController` 在 Native 端注入用户自定义脚本 JS
    * web 端可以使用 `window.webkit.messageHandlers.{NAME}.postMessage()`，直接向 Native 端发送消息。UIWebView 只能使用 iframe 等方案 hack

### 限制
官方的描述基本是溢美之词，他不会告诉你 WKWebView 的一些问题，但是这些问题，在做 Hybrid 应用的时候，影响很大，纯 wap 无影响。
估计是切换底层实现的关系，这些问题在 UIWebView 上不存在：

* 无能加载本地文件，只能通过内建一个 WebServer 实现功能。（直到 iOS 9 才新开了一个loadFileURL:allowingReadAccessToURL: 接口，原生实现该功能）
* 不能注册自定义 NSURLProtocol，导致大量 URL 拦截功能难以实现，比如页面展示本地图片
* 不能使用 NSHTTPCookieStorage 设置 WebView 的 Cookie
* 必须 iOS8 及以上版本，Cordova 只支持 iOS9 及以上

还有其他在实际开发中，可能爆出的 bug。

##  Cordova WKWebview 现状
Cordova 开发了[一个插件](https://github.com/apache/cordova-plugin-wkwebview-engine)来支持 WKWebview，用户可以自行在 WKWebview 与 UIWebView 中选择。

但因为 WKWebView 存在的种种问题，WKWebView 还不是 Cordova 的默认选择。

[cordova 关于 WKWebView 的一些说明](https://shazronatadobe.wordpress.com/2015/03/03/wkwebview-and-apache-cordova/)

## 结论
* 纯 wap 页面，WKWebView 毫无问题，会有很好的表现
* Hybrid 应用，因为现在存在的一些问题，以及会导致实现 Camera、PhotoList 等 JSBridge 的功能上会有较大限制，选择 WKWebView 需要慎重

## 相关代码
[cordova-plugin-wkwebview-engine](https://github.com/apache/cordova-plugin-wkwebview-engine)

[CrossWalk Cordova Plugin Support](https://github.com/crosswalk-project/ios-extensions-crosswalk)

## 相关资料
[WKWeb​View](http://nshipster.cn/wkwebkit/)

[Telerik Platform Documentation - Configure the Web Views](http://docs.telerik.com/platform/appbuilder/cordova/configuring-your-app/configure-web-views)

[Cordova Plugins - WKWebView](http://plugins.telerik.com/cordova/plugin/wkwebview)

[CrossWalk - Cordova Plugin Support](https://crosswalk-project.org/documentation/ios/cordova_plugin_support.html)

[Is there support in Cordova to WKWebView?](http://stackoverflow.com/questions/29268433/is-there-support-in-cordova-to-wkwebview)

