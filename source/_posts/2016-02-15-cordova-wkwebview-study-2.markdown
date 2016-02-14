---
layout: post
title: "Cordova WKWebView 学习笔记（二）"
date: 2016-02-15 00:11:47 +0800
comments: true
categories: study
description: cordova wkwebview
keywords: cordova wkwebview
---
这篇主要是代码学习。

官方代码地址：[cordova-plugin-wkwebview-engine](https://github.com/apache/cordova-plugin-wkwebview-engine)

我的测试代码：[CordovaWKWebViewTest](https://github.com/sjpsega/CordovaWKWebViewTest)，融合了一个 Device JSBridge 的例子，该例子参考自[CrossWalk - Cordova Plugin Support](https://crosswalk-project.org/documentation/ios/cordova_plugin_support.html)

### WKWebView JSBridge 事件传递
在 WKWebView 中，开始原生支持 web 向 Native 发送消息，是通过 messageHandler 实现的。

在 UIWebView 时代，实现 web 向 app 发送消息，通常是通过 iframe 发起一个特殊请求，UIWebView 通过在拦截方法中，拦截这个特殊请求，使用这种类似 hack 的方式曲线救国的。

#### 实现步骤
* Native 端添加 messageHandler
```objectivec
WKUserContentController* userContentController = [[WKUserContentController alloc] init];
// 关键代码，添加 messageHandler，名字为 cordova
[userContentController addScriptMessageHandler:self name:@"cordova"];

WKWebViewConfiguration* configuration = [[WKWebViewConfiguration alloc] init];
configuration.userContentController = userContentController;

WKWebView* wkWebView = [[WKWebView alloc] initWithFrame:frame configuration:configuration];
```

* JS 端调用
```javascript
//调用名为 cordova 的 messageHandler，与 Native 端进行通信
window.webkit.messageHandlers.cordova.postMessage(command);
```

* Native 接收
```objectivec
//实现 WKScriptMessageHandler 接口
- (void)userContentController:(WKUserContentController*)userContentController didReceiveScriptMessage:(WKScriptMessage*)message
{
    NSLog(@"%@",message.name); //cordova
    NSLog(@"%@",message.body); //JSBridge 内容体
    ... //实现调用 Native 代码
}
```

一个 messageHandler 一个通道，Cordova 统一使用 cordova 这个通道来通信。

### 新增一个插件的实现步骤
目录结构（列出关键目录与文件）：
```
|wkwvtest
    |Staging
        config.xml
        |www
            cordova_plugins.js
            |plugins //JS Plugihns
    |Plugins // Native Plugins
```

已实现一个 Device 插件为例：

* 在 Staging/www/plugins 目录下，新增一个 device.js 文件，添加 JS 代码
```javascript
cordova.define("org.apache.cordova.device.device", function(require, exports, module) { 
    function Device() {
    //...
    }
    //...
    Device.prototype.getInfo = function(successCallback, errorCallback) {
        //调用 JSBridge，调用名为 Device 的 Native 类 getDeviceInfo 方法
        exec(successCallback, errorCallback, "Device", "getDeviceInfo", []);
    };
    module.exports = new Device();
});
```

* 在 Staging/www/cordova_plugins.js 文件中注册 device.js
```javascript
cordova.define('cordova/plugin_list', function(require, exports, module) {
    module.exports = [
        {
            "file": "plugins/device.js",//写明相对 cordova_plugins.js 的文件路径
            "id": "org.apache.cordova.device.device",
            "clobbers": [
                 "device" //全局命名空间，会注册为 window.device
            ]
        }
    ];
    module.exports.metadata =
    {
        "org.apache.cordova.device": "0.3.0"
    }
});
```

* 添加在 plugins 目录下，新增 CDVDevice 类，继承于 CDVPlugin
```objectivec
@interface CDVDevice : CDVPlugin
- (void)getDeviceInfo:(CDVInvokedUrlCommand*)command;
@end
```

* 在 Staging/config.xml 中注册 CDVDevice
```xml
<widget id="my.project.wkwvtest" version="0.0.1" xmlns="http://www.w3.org/ns/widgets" xmlns:cdv="http://cordova.apache.org/ns/1.0">
    <feature name="Device">
        <param name="ios-package" value="CDVDevice" />
    </feature>
</widget>
```

完成这步骤，便可以通过 web 端，调用 `exec(successCallback, errorCallback, "Device", "getDeviceInfo", []);`，调用 Device 这个 Native 类的 getDeviceInfo 方法

详见：[CordovaWKWebViewTest](https://github.com/sjpsega/CordovaWKWebViewTest)

### "同步" JSBridge 代码执行
这里的"同步"需要打引号，虽然 WKWebView 提供了原生 web 向 Native 发送消息的方案，但是注意，这里是发送消息，还不是同步回调。

Cordova 新版本的插件机制，通过多重事件机制的方式，使得一种类型的 JSBridge 同步成为可能 - webView 打开页面后，需要调用 Native 端获取环境变量的 JSBridge。比如上例的 Device 这个 JSBridge 就是典型。

Cordova 启动的几个重要事件：

* onNativeReady
* onDOMContentLoaded
* onPluginsReady
* onCordovaReady
* onDeviceReady

通常是从上往下依次进行，并且 web 开始调用 JSBridge，必须监听 `deviceready` 事件，所以 onDeviceReady 始终最后执行，代表 Cordova 插件环境完全准备完毕。

所以，这种"同步" JSBridge 就是利用这个特性，做了一些文章。

看关键代码：
```javascript
cordova.define("org.apache.cordova.device.device", function(require, exports, module) { 
    channel = require('cordova/channel'),
    exec = require('cordova/exec'),
    //注册名为 onCordovaInfoReady 的 Sticky 类型的事件
    channel.createSticky('onCordovaInfoReady');
    //告诉系统，必须等待 CordovaInfoReady 事件发送
    channel.waitForInitialization('onCordovaInfoReady');

    function Device() {
        var me = this;
        //在 onCordovaReady 事件中，注册回调，使得系统初始化便调用该回调，并且 onCordovaReady 在 onPluginsReady 事件后 fire
        channel.onCordovaReady.subscribe(function() {
            //调用 JSBridge
            me.getInfo(function(info) {
                //调用 JSBridge 成功回调用，fire onCordovaInfoReady 事件
                channel.onCordovaInfoReady.fire();
            },function(e) {
            });
        });
    }

    Device.prototype.getInfo = function(successCallback, errorCallback) {
        exec(successCallback, errorCallback, "Device", "getDeviceInfo", []);
    };

    module.exports = new Device();
});
```

web 端调用 Device 的代码：
```javascript
document.addEventListener("deviceready", onDeviceReady, false);
function onDeviceReady() {
    console.log(device.uuid);
}
```
可以看到，deviceready 的时候，便可以直接调用 device 这个全局变量。
这是因为之前在 cordova_plugins.js 这个文件中，做了申明，会把 device 变为全局变量
```javascript
"clobbers": [
    "device" //全局命名空间，会注册为 window.device
]
```

这种设计非常巧妙，但是以我的经验，这种形式的 JSBridge 使用比较局限，主要缺点有：

* 只能对整个 App 周期，固定不变的值使用。如果使用中，这个值会变化则无法使用，如地理位置信息就不行
* 只能针对属性使用，不能对方法使用