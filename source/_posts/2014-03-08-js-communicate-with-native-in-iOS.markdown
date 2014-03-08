---
layout: post
title: "iOS js与native相互通信"
date: 2014-03-08 21:53:16 +0800
comments: true
categories: study
keywords: iOS, js, native, WebViewJavascriptBridge

---

## js与navive相互通信的机制

### js -> native

目前，截止至iOS7，iOS原生并没有提供js直接调用native的方式，只能通过UIWebView相关的UIWebViewDelegate协议的

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
虽然通过这个方式，js调用native是`异步`的，但是效率还是很高，我通过在js调用端，把time传入navive然后相减的方式计算，平均只有5ms的时间间隔。


#### 如何触发这个方法拦截

最最简单且实用的方法莫过于用js创建一个隐藏的iframe设置src了，代码：

```js
function js2native(){
  	var iframe = document.createElement("iframe");
  	iframe.src="callFunction://";
  	iframe.style.display = 'none';
  	document.body.appendChild(iframe);
  	iframe.parentNode.removeChild(iFrame);
  	iframe = null;
}
```

通过查看phoneGap源码的iOSExec方法，还有使用XMLHttpRequest或修改hash的方式来触发方法拦截，但是因为有bug或其他原因，不推荐。

### native -> js

native调用js非常简洁方便，只需要

```objc
[webView stringByEvaluatingJavaScriptFromString:@"alert('hello world!')"];
```
并且该方法是`同步`的。

## 调试
虽然现在能直接用Safari的开发模式直接查看模拟器中的webView页面，但是经过亲自尝试，最想要的也是最重要的js调试，还是不支持，不能进行js断点调试，还是要依赖console来弄……当然css样式调试支持的不错。

### 相关资料

[唐巧-关于UIWebView和PhoneGap的总结](http://blog.devtang.com/blog/2012/03/24/talk-about-uiwebview-and-phonegap/)

[Github上的WebViewJavascriptBridge项目](https://github.com/marcuswestin/WebViewJavascriptBridge)

[大名鼎鼎的phonegap](http://phonegap.com/)