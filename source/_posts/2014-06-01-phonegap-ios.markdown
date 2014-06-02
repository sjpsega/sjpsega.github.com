---
layout: post
title: "iOS版PhoneGap原理分析"
date: 2014-06-01 00:59:15 +0800
comments: true
categories: study
keywords: PhoneGap iOS
---

PhoneGap，著名的跨平台Hybrid框架，旨在让开发者使用HTML、Javascript、CSS开发跨平台的App。

最近的工作，就是做Hybrid方面的，很自然，方案就从PhoneGap入手。

下面就切入正题，分析下PhoneGap的原理，需要说明的是，我只针对iOS版本的PhoneGap做分析，android版本的原理大同小异。

## 安装PhoneGap
现在使用PhoneGap非常方便，只需要安装node，用简单的命令就能完成安装和使用的工作。

安装PhoneGap:
```objc
sudo npm install -g phonegap
```

创建phoneGap应用:
```objc
phonegap create my-app
cd my-app
phonegap run ios
```

具体可看[phonegap官网](http://phonegap.com/)进行学习。

## PhoneGap与Cordova的关系
Cordova是PhoneGap贡献给Apache后的开源项目，是从PhoneGap中抽离出的核心代码，是驱动PhoneGap的核心引擎。有点类似Webkit和Google Chrome的关系。

渊源就是：早在2011年10月，Adobe收购了Nitobi Software和它的PhoneGap产品，然后宣布这个移动Web开发框架将会继续开源，并把它提交到Apache Incubator，以便完全接受ASF的管治。当然，由于Adobe拥有了PhoneGap商标，所以开源组织的这个PhoneGap v2.0版产品就更名为Apache Cordova。

为什么说这个？因为下面的文章中，会出现Cordova这个命令，大家不要觉得奇怪。

## js与native通信的原理

但在切入正题前，需要先了解下iOS js与native通信的原理。了解这个原理，是理解PhoneGap代码的`关键`。

具体可以看我之前写的[iOS Js与native相互通信](http://sjpsega.com/blog/2014/03/08/js-communicate-with-native-in-iOS/)，这里做简单说明。

### js -> native

在iOS中，js调用native并没有提供原生的实现，只能通过UIWebView相关的UIWebViewDelegate协议的

```objc
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
```
方法来做拦截，并在这个方法中，根据url的协议或特征字符串来做调用方法或触发事件等工作，如

```objc
/*
* 方法的返回值是BOOL值。
* 返回YES：表示让浏览器执行默认操作，比如某个a链接跳转
* 返回NO：表示不执行浏览器的默认操作，这里因为通过url协议来判断js执行native的操作，肯定不是浏览器默认操作，故返回NO
* /
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

native调用js非常简单直接，所以PhoneGap解决的主要是js调用native的问题。

## PhoneGap js -> native
我们通过一个js调用native的Dialog的例子做说明。

Dialog是一个PhoneGap的插件，可以看[dialog 插件文档](https://github.com/apache/cordova-plugin-dialogs/blob/master/doc/index.md)，学习下载并使用该插件。

```
这里有个很重要的事需要说明一下：
目前PhoneGap的文档更新非常不及时，特别是插件的使用方面，比如Dialog插件的使用，文档中写的是使用navigator.notification.alert，但是经过我的摸索，因为现在PhoneGap使用AMD的方式来管理插件，所以应该是使用cordova.require("cordova/plugin/notification").alert的方式来调用。

插件的合并方面，也有很多坑，主要是文档不全 - -|||
```

### js部分
在html上添加一个button，然后通过下列代码调用：
```javascript
function alertDismissed() {
    // do something
}

function showAlert() {
    cordova.require("cordova/plugin/notification").alert(
        'You are the winner!',  // message
        alertDismissed,         // callback
        'Game Over',            // title
        'Done'                  // buttonName
    );
}
```

再看下对应的`cordova/plugin/notification`的代码：
```javascript
var exec = cordova.require('cordova/exec');
var platform = cordova.require('cordova/platform');

module.exports = {
    
    /**
     * Open a native alert dialog, with a customizable title and button text.
     *
     * @param {String} message              Message to print in the body of the alert
     * @param {Function} completeCallback   The callback that is called when user clicks on a button.
     * @param {String} title                Title of the alert dialog (default: Alert)
     * @param {String} buttonLabel          Label of the close button (default: OK)
     */
    alert: function(message, completeCallback, title, buttonLabel) {
        var _title = (title || "Alert");
        var _buttonLabel = (buttonLabel || "OK");
        exec(completeCallback, null, "Notification", "alert", [message, _title, _buttonLabel]);
    }
}

....
```

可以看到alert最终其实是调用了`exec`来调用native代码的，`exec`非常关键，是PhoneGap js调用native的核心代码。

然后在源码中搜索`cordova/exec`，查看exec方法的源码。

因为对应的`cordova/exec`源码非常长，我只能截取最关键的代码并做说明：
```javascript
define("wing/exec", function(require, exports, module) {

    ...

    function iOSExec() {
        ...

        var successCallback, failCallback, service, action, actionArgs, splitCommand;
        var callbackId = null;
        
        ...

        // 格式化传入参数
        successCallback = arguments[0]; //成功的回调函数
        failCallback = arguments[1];    //失败的回调函数
        service = arguments[2];         //表示调用native类的类名
        action = arguments[3];          //表示调用native类的一个方法
        actionArgs = arguments[4];      //参数

        //默认callbackId为'INVALID'，表示不需要回调
        callbackId = 'INVALID';
        
        ...

        //如果传入参数有successCallback或failCallback，说明需要回调，就设置callbackId，并存储对应的回调函数
        if (successCallback || failCallback) {
            callbackId = service + wing.callbackId++;
            wing.callbacks[callbackId] =
                {success:successCallback, fail:failCallback};
        }

        //格式化传入的service、action、actionArgs，并存储，准备native代码来调用
        actionArgs = massageArgsJsToNative(actionArgs);

        var command = [callbackId, service, action, actionArgs];

        commandQueue.push(JSON.stringify(command));

        ...

        //通过创建一个iframe并设置src，给native代码一个指令，开始执行js调用native的过程
        execIframe = execIframe || createExecIframe();
        if (!execIframe.contentWindow) {
            execIframe = createExecIframe();
        }
        execIframe.src = "gap://ready";

        ...
    }

    module.exports = iOSExec;

});
```

为了调用native方法，iOSExec方法做了大量初始化的工作，这么做的原因，还是因为`iOS没有提供直接的方法来执行js调用native，不能把参数直接传递给native，所以只能通过js端存储对应操作的所有参数，然后通过指令来让native代码来回调的方式间接完成。`

### native部分
之后，就走到了native代码的部分。

#### CDVViewController
前面js通过创建一个iframe并发送`gap://ready`这个指令来告诉native开始执行操作。native中对应的操作在`CDVViewController.m`文件中的`webView:shouldStartLoadWithRequest:navigationType:`方法：

```objc
- (BOOL)webView:(UIWebView*)theWebView shouldStartLoadWithRequest:(NSURLRequest*)request navigationType:(UIWebViewNavigationType)navigationType
{
    NSURL* url = [request URL];

    /*
     * 判断url的协议以"gap"开头
     * 执行在js端调用cordova.exec()的command队列
     * 注：这里的command表示js调用native
     */
    if ([[url scheme] isElaqualToString:@"gap"]) {
       //_commandQueue即CDVCommandQueue类
        //从js端拉取command，即存储在js端commandQueue数组中的数据
        [_commandQueue fetchCommandsFromJs];
        //开始执行command
        [_commandQueue executePending];
        return NO;
    }
...
}
```

到这里，其实已经走完js调用native的主要过程了。

之后，让我们再看下`CDVCommandQueue`中的fetchCommandsFromJs方法与executePending方法中做的事。

#### CDVCommandQueue
```objc
- (void)fetchCommandsFromJs
{
    // 获取js端存储的command，并在native暂存
    NSString* queuedCommandsJSON = [_viewController.webView stringByEvaluatingJavaScriptFromString:
        @"cordova.require('cordova/exec').nativeFetchMessages()"];
    [self enqueueCommandBatch:queuedCommandsJSON];
}
```
fetchCommandsFromJs方法非常简单，不细说了。

executePending方法稍微复杂些，因为js是单线程的，而iOS是典型的多线程，所以executePending方法做的工作主要是让command一个一个执行，防止线程问题。

executePending方法其实与之后的execute方法紧密相连，这里一起列出，只保留关键代码：

```objc
- (void)executePending
{
    ...
    //_queue即command队列，依次执行
    while ([_queue count] > 0) {
        ...
        //取出从js中获取的command字符串，解析为native端的CDVInvokedUrlCommand类
        CDVInvokedUrlCommand* command = [CDVInvokedUrlCommand commandFromJson:jsonEntry];
        ...
        //执行command
        [self execute:command])
        ...
    }
}

- (BOOL)execute:(CDVInvokedUrlCommand*)command
{
    ...
    BOOL retVal = YES;
    //获取plugin对应的实例
    CDVPlugin* obj = [_viewController.commandDelegate getCommandInstance:command.className];
    //调用plugin实例的方法名
    NSString* methodName = [NSString stringWithFormat:@"%@:", command.methodName];
    SEL normalSelector = NSSelectorFromString(methodName);
    if ([obj respondsToSelector:normalSelector]) {
        //消息发送，执行plugin实例对应的方法，并传递参数
        objc_msgSend(obj, normalSelector, command);
    } else {
        // There's no method to call, so throw an error.
        NSLog(@"ERROR: Method '%@' not defined in Plugin '%@'", methodName, command.className);
        retVal = NO;
    }
    ...
    return retVal;
}
```

可以看到js调用native plugin最终执行的是`objc_msgSend(obj, normalSelector, command);`这块代码，这里我们再拿js端的代码来进行理解。

之前js中的showAlert方法中我们书写了
`exec(completeCallback, null, "Notification", "alert", [message, _title, _buttonLabel]);`

故，这里的对应关系：

* obj:"Notification"
* normalSelector:"alert"
* command:[message, _title, _buttonLabel]

#### CDVNotification
"Notification"真正对应的iOS类是CDVNotification。js端调用的插件名字"Notification"与真正的native类名并非完全对应，因为native因为平台的不同，有不同的命名规范。

看下CDVNotification的代码：

```objc
- (void)alert:(CDVInvokedUrlCommand*)command
{
    NSString* callbackId = command.callbackId;
    NSString* message = [command argumentAtIndex:0];
    NSString* title = [command argumentAtIndex:1];
    NSString* buttons = [command argumentAtIndex:2];

    [self showDialogWithMessage:message title:title buttons:@[buttons] defaultText:nil callbackId:callbackId dialogType:DIALOG_TYPE_ALERT];
}
```

前面用`objc_msgSend(obj, normalSelector, command);`做消息发送，执行的便是这块代码，代码很好理解，就是对command再做解析，并显示。

到此为止，我们已经走完完从js端调用native alert的全部过程了。

列下过程的核心代码：

* js部分：
  * cordova.js中的iOSExec()方法，指定js调用native的初始化工作，并发送开始执行的指令
* native部分：
  * CDVViewController：拦截js调用native的url协议，执行调用
  * CDVCommandQueue：执行js调用native的队列，调用对应的plugin

## 时序图

## 结语
PhoneGap还是很给力的，能做到主流平台全兼容着实不容易。

iOS端因为没有提供js调用native的直接方法，做的处理也算合理到位。

特别是插件化的支持做的很好，但是文档着实不够给力。
