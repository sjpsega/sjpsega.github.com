---
layout: post
title: "[译]iOS7最佳实践：一个天气App案例(一)"
date: 2014-02-11 22:21:53 +0800
comments: true
categories: study
keywords: iOS7, Cocoapods, ReactiveCocoa
---

注：本文由译自：[raywenderlich ios-7-best-practices-part-1](http://www.raywenderlich.com/55384/ios-7-best-practices-part-1)，只翻译了最精华的部分

在这个两部分的系列教程中，您将探索如何使用以下工具和技术来创建自己的应用程序：

* [Cocoapods](http://cocoapods.org/)
* Manual layout in code(纯代码布局)
* [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)
* [OpenWeatherMap](http://openweathermap.org/)

本教程专为熟悉基本知识的、但还没有接触到太多高级主题的中级开发者而设计。本教程也是想要去探索Objective-C函数编程一个很好的开始。

![Finished Weather App](http://cdn1.raywenderlich.com/wp-content/uploads/2013/11/weather3.gif)

## 开始

打开Xcode并执行File\New\Project。选择Application\Empty Application。将项目命名为SimpleWeather，单击下一步，选择一个目录去保存你的项目，然后点击Create。
现在，你的基础项目已经完成。下一步是集成你的第三方工具。但首先你要确定关闭Xcode，确保他不会影响下一步。

### Cocoapods
你将要下载[Cocoapods](http://cocoapods.org/)的代码，将文件添加到你的Xcode项目中来使用，以及配置这些项目需要的设置。

### Mantle
[Mantle](https://github.com/MantleFramework/Mantle)是由于Github团队开发的，用来去除所有的需要Objective-C把JSON数据转为NSObject子类的样板代码。Mantle也做数据转换，通过一种神奇的方式把JSON原始数据(strings, ints, floats)转换为复杂数据，比如NSDate, NSURL, 甚至是自定义类。

### LBBlurredImage
[LBBlurredImage](https://github.com/lukabernardi/LBBlurredImage)是一个继承自UIImageView，使图像模糊变得轻而易举的项目。你将仅仅用一行代码来创建一个神奇的模糊效果。

### TSMessages
[TSMessages](https://github.com/toursprung/TSMessages) 是另一个非常简单的库，用来显示浮层警告和通知。当出现错误信息而不直接影响用户的时候，最好使用浮层来代替模态窗口(例如UIAlertView)，这样你将尽可能少影响用户。
你将只用TSMessages，在网络失去连接或API错误的时候。如果发生错误，你将看到类似这样的一个浮层：
![TSMessages Error](http://cdn1.raywenderlich.com/wp-content/uploads/2013/12/TSMessage.png)

### ReactiveCocoa
最后，你将使用到[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa),他也来自于GitHub团队。ReactiveCocoa给Objective-C带来了函数编程，类似与[Reactive Extensions](http://msdn.microsoft.com/en-us/data/gg577609.aspx)在.NET的模式。你将在第二部分花费大部分时间去实现ReactiveCocoa。

## 设置你的Cocoapods
设置你的Cocoapods，先要确保你已经安装了Cocoapods。为此，打开命令行程序，并输入。

~~~
which pod
~~~

你将会看到类似这样的输出:

~~~
/usr/bin/pod
~~~

这依赖与你如何管理Ruby gems，例如你使用[rbenv](http://rbenv.org/)或[RVM](http://rvm.io/),路径可能有所不同。

如果terminal简单的返回提示，或显示`pod not found`,Cocoapods未安装在你的机器上。可以查看我们的[Cocoapods教程](http://www.raywenderlich.com/12139/introduction-to-cocoapods)作为安装说明。这也是一个很好的资源，如果你想更多得了解Cocoapods的话。

[Podfiles](http://guides.cocoapods.org/syntax/podfile.html)是用来告诉Cocoapods哪些开源项目需要导入。

要创建你的第一个Cocoapod，首先在命令行中用`cd`命令导航到你的XCode项目所在的文件夹，在命令行中启动编辑器，输入

~~~
platform :ios, '7.0'
 
pod 'Mantle'
pod 'LBBlurredImage'
pod 'TSMessages'
pod 'ReactiveCocoa'
~~~

这文件做了两件事情：

* 告诉Cocoapods你的目标平台与版本，这里的你目标是iOS 7.0。
* 列给Cocoapods一个项目所有需要引入和安装的三方库清单。

在命令行中输入`pod install`进行安装。

这可能需要话一到两分钟的时间去安装各种包。你的终端应该输出如下所示:

~~~
$ pod install
Analyzing dependencies
 
CocoaPods 0.28.0 is available.
 
Downloading dependencies
Installing HexColors (2.2.1)
Installing LBBlurredImage (0.1.0)
Installing Mantle (1.3.1)
 
Installing ReactiveCocoa (2.1.7)
Installing TSMessages (0.9.4)
Generating Pods project
Integrating client project
 
[!] From now on use `SimpleWeather.xcworkspace`.
~~~

Cocoapods会在你的项目目录中创建一堆新文件，但是，只有一个你会关心，SimpleWeather.xcworkspace。

用Xcode打开SimpleWeather.xcworkspace。看看你的项目设置，现在有一个Pods项目在你的项目工作区，以及在Pods文件夹放着每一个你引入的库，如下所示：

![Cocoapods Project](http://cdn1.raywenderlich.com/wp-content/uploads/2013/12/SimpleWeather-Cocoapods.jpg)

确保你已经选择SimpleWeather项目，如图所示：
![Select SimpleWeather Project](http://cdn1.raywenderlich.com/wp-content/uploads/2013/12/SimpleWeather-Project.jpg)

生成并运行您的应用程序，以确保一切工作正常：
![Blank App](http://cdn1.raywenderlich.com/wp-content/uploads/2013/11/blank-app.jpg)

~~~
提示：您可能会注意到一些项目生成警告。由Cocoapods引入的项目，是由不同的开发者开发，并且不同的开发者对生成警告有不同的态度。通常，你应该可以忽略它们。只要确保没有任何编译器错误！
~~~

## 创建你的主View Controller
虽然应用程序看起来复杂，但它会通过一个单一的View Controller完成。现在，你将添加他。 

选中SimpleWeather项目，单击`File\New\File`，并且选择`Cocoa Touch\Objective-C class`. 命名为`WXController`，并设置为`UIViewController`的子类。

确保`Targeted for iPad`和`With XIB for user interface`都没有选中，如下图所示：
![Create WXController](http://cdn1.raywenderlich.com/wp-content/uploads/2013/11/create-controller.jpg)

打开` WXController.m`然后用如下所示替换`-viewDidLoad`方法：

~~~
- (void)viewDidLoad {
    [super viewDidLoad];
 
    // Remove this later
    self.view.backgroundColor = [UIColor redColor];
}
~~~

现在打开`AppDelegate.m`，并且引入如下两个class:

~~~
#import "WXController.h"
#import <TSMessage.h>
~~~

眼尖的读者会注意到`WXController`使用引号引入，`TSMessage`使用单括号引入。

回头看看，当你创建Podfile的时候，你使用Cocoapods引入TSMessage。Cocoapods创建TSMessage项目，并把它加入到工作空间。既然你从工作区的其他项目导入，可以使用尖括号代替引号。

代替`-application:didFinishLaunchingWithOptions`的内容：

~~~
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    self.window = [[UIWindow alloc] initWithFrame:[[UIScreen mainScreen] bounds]];
    // 1
    self.window.rootViewController = [[WXController alloc] init];
    self.window.backgroundColor = [UIColor whiteColor];
    [self.window makeKeyAndVisible];
    // 2
    [TSMessage setDefaultViewController: self.window.rootViewController];
    return YES;
}
~~~