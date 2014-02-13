---
layout: post
title: "[译]iOS7最佳实践：一个天气App案例(一)"
date: 2014-02-11 22:21:53 +0800
comments: true
categories: study
keywords: iOS7, Cocoapods, ReactiveCocoa
---

注：本文由译自：[raywenderlich ios-7-best-practices-part-1](http://www.raywenderlich.com/55384/ios-7-best-practices-part-1)，去除了跟主题无关的寒暄部分。
欢迎转载，保持署名


在这个两部分的系列教程中，您将探索如何使用以下工具和技术来创建自己的App：

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

构建并运行您的App，以确保一切工作正常：
![Blank App](http://cdn1.raywenderlich.com/wp-content/uploads/2013/11/blank-app.jpg)

~~~
提示：您可能会注意到一些项目生成警告。由Cocoapods引入的项目，是由不同的开发者开发，并且不同的开发者对生成警告有不同的态度。通常，你应该可以忽略它们。只要确保没有任何编译器错误！
~~~

## 创建你的主视图控制器
虽然App看起来复杂，但它会通过一个单一的View Controller完成。现在，你将添加他。 

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

标号注释的解释：

1. 初始化并设置W`XController`实例作为App的根视图控制器。通常这个控制器是一个的`UINavigationController`或`UITabBarController`，但在当前情况下，你使用`WXController`的单个实例。
2. 设置默认的视图控制器来显示你的TSMessages。通过这样做，你将不再需要手动指定要使用的控制器来显示警告。

构建并运行，看看你的新视图控制器起作用了。
![WXController](http://cdn1.raywenderlich.com/wp-content/uploads/2013/11/wxcontroller-red.jpg)

在红色背景下，状态栏有点难以阅读。幸运的是，有一个简单的方法，使状态栏更清晰易读。

在iOS7，UIViewController有一个新的API，用来控制状态栏的外观。打开`WXController`，直接添加下面的代码到`-viewDidLoad:`方法下：

~~~
- (UIStatusBarStyle)preferredStatusBarStyle {
    return UIStatusBarStyleLightContent;
}
~~~

再次构建并运行，你将看到状态栏如下的变化:
![Create WXController with Light Status Bar](http://cdn1.raywenderlich.com/wp-content/uploads/2013/11/wxcontroller-red-status.jpg)

## 设置你的App视图
现在是时候让你的App接近生活。下载这个项目的[图片](http://cdn1.raywenderlich.com/wp-content/uploads/2013/11/Images.zip)，并解压缩到一个合适的位置。这个压缩包的背景图片出自Flickr用户[idleformat](http://www.flickr.com/photos/52547323@N00/5637648252)之手，天气图片出自Dribbble用户[heeyeun](http://dribbble.com/shots/1247177-Weather-icons?list=users)之手。

切换回Xcode，单击`File\Add Files to “SimpleWeather”`....定位到你刚刚解压缩的图片文件夹并选择它。选择`Copy items into destination group’s folder (if needed)`，然后单击`Add`。

打开`WXController.h`, 添加如下委托协议：

~~~
<UITableViewDataSource, UITableViewDelegate, UIScrollViewDelegate>
~~~

现在打开`WXController.m`。 小提示：你可以使用`Control-Command-Up`的快捷键来实现`.h`和`.m`文件之间的快速切换。

添加如下代码到`WXController.m`顶部:
~~~
#import <LBBlurredImage/UIImageView+LBBlurredImage.h>
~~~

`LBBlurredImage.h`包含在Cocoapods引入的`LBBlurredImage`项目，你会使用这个库来模糊背景图片。

应该有一个空的私有接口样板在`WXController` imports的下方。它具有以下属性：

~~~
@interface WXController ()
 
@property (nonatomic, strong) UIImageView *backgroundImageView;
@property (nonatomic, strong) UIImageView *blurredImageView;
@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, assign) CGFloat screenHeight;
 
@end
~~~

现在，是时候在项目中创建并设置视图。

下面是你App的分解图，记住，表视图将是透明的：
![Exploded Screens](http://cdn1.raywenderlich.com/wp-content/uploads/2013/11/screens.jpg)

为了实现动态模糊效果，在你的App中，你会根据App的滚动来改变模糊图像的alpha值。

打开`WXController.m`，使用如下代码来，替换掉`-viewDidLoad`中设置背景色的代码：

~~~
// 1
self.screenHeight = [UIScreen mainScreen].bounds.size.height;
 
UIImage *background = [UIImage imageNamed:@"bg"];
 
// 2
self.backgroundImageView = [[UIImageView alloc] initWithImage:background];
self.backgroundImageView.contentMode = UIViewContentModeScaleAspectFill;
[self.view addSubview:self.backgroundImageView];
 
// 3
self.blurredImageView = [[UIImageView alloc] init];
self.blurredImageView.contentMode = UIViewContentModeScaleAspectFill;
self.blurredImageView.alpha = 0;
[self.blurredImageView setImageToBlur:background blurRadius:10 completionBlock:nil];
[self.view addSubview:self.blurredImageView];
 
// 4
self.tableView = [[UITableView alloc] init];
self.tableView.backgroundColor = [UIColor clearColor];
self.tableView.delegate = self;
self.tableView.dataSource = self;
self.tableView.separatorColor = [UIColor colorWithWhite:1 alpha:0.2];
self.tableView.pagingEnabled = YES;
[self.view addSubview:self.tableView];
~~~

这是非常简单的代码：

1. 获取并存储屏幕高度。之后，你将在用分页的方式来显示所有天气​​数据时，使用它。`貌似没有使用到`
2. 创建一个静态的背景图，并添加到视图上。
3. 使用LBBlurredImage来创建一个模糊的背景图像，并设置alpha为0，使得开始`backgroundImageView`是可见的。
4. 创建tableview来处理所有的数据呈现。 设置WXController的delegate和dataSource，以及滚动视图的delegate。请注意，设置`pagingEnabled`为`YES`。

添加如下UITableView的delegate和dataSource的代码到`WXController.m`的`@implementation`块中：

~~~
// 1
#pragma mark - UITableViewDataSource
 
// 2
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return 2;
}
 
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    // TODO: Return count of forecast
    return 0;
}
 
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    static NSString *CellIdentifier = @"CellIdentifier";
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:CellIdentifier];
 
    if (! cell) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleValue1 reuseIdentifier:CellIdentifier];
    }
 
    // 3
    cell.selectionStyle = UITableViewCellSelectionStyleNone;
    cell.backgroundColor = [UIColor colorWithWhite:0 alpha:0.2];
    cell.textLabel.textColor = [UIColor whiteColor];
    cell.detailTextLabel.textColor = [UIColor whiteColor];
 
    // TODO: Setup the cell
 
    return cell;
}
 
#pragma mark - UITableViewDelegate
 
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    // TODO: Determine cell height based on screen
    return 44;
}
~~~

1. `Pragma mark`是[组织代码](http://nshipster.com/pragma/)的很好的一种方式。
2. 你的table view有两个部分，一个是每小时的天气预报，另一个用于每日播报。table view的section数目，返回2。 
3. 天气预报的cell不应该是可选择的。给他们一个半透明的黑色背景和白色文字。

~~~
注意：使用格式化的注释`// TODO：`帮助Xcode找到需要你以后完成的代码。你还可以使用`Show Document Items`(Control-6)来查看TODO项。~~~

最后，添加如下代码到`WXController.m`:

~~~
- (void)viewWillLayoutSubviews {
    [super viewWillLayoutSubviews];
 
    CGRect bounds = self.view.bounds;
 
    self.backgroundImageView.frame = bounds;
    self.blurredImageView.frame = bounds;
    self.tableView.frame = bounds;
}
~~~

在`WXController.m`中，你的视图控制器调用该方法来编排其子视图。 
构建并运行你的App，看看你的视图如何堆叠。
![Image Background](http://cdn1.raywenderlich.com/wp-content/uploads/2013/11/background.jpg)

仔细看，你会看到所有空的table cell的单独的cell分隔线。
仍然在`-viewDidLoad`中，添加下面的代码来设置你的布局框架和边距：

~~~
// 1
CGRect headerFrame = [UIScreen mainScreen].bounds;
// 2
CGFloat inset = 20;
// 3
CGFloat temperatureHeight = 110;
CGFloat hiloHeight = 40;
CGFloat iconHeight = 30;
// 4
CGRect hiloFrame = CGRectMake(inset, 
                              headerFrame.size.height - hiloHeight,
                              headerFrame.size.width - (2 * inset),
                              hiloHeight);
 
CGRect temperatureFrame = CGRectMake(inset, 
                                     headerFrame.size.height - (temperatureHeight + hiloHeight),
                                     headerFrame.size.width - (2 * inset),
                                     temperatureHeight);
 
CGRect iconFrame = CGRectMake(inset, 
                              temperatureFrame.origin.y - iconHeight, 
                              iconHeight, 
                              iconHeight);
// 5
CGRect conditionsFrame = iconFrame;
conditionsFrame.size.width = self.view.bounds.size.width - (((2 * inset) + iconHeight) + 10);
conditionsFrame.origin.x = iconFrame.origin.x + (iconHeight + 10);
~~~

这是相当常规设置代码，但这里是怎么回事： 

1. 设置table的header与屏幕相同的大小。你将利用的UITableView的分页来分隔页面页眉和每日每时的天气预报部分。 
2. 创建inset（或padding）变量，以便您的所有标签均匀分布并居中。
3. 创建并初始化为各种视图创建的高度变量。设置这些值作为常量，可以很容易地在需要配置和更改您的视图设置。
4. 使用常量和inset变量，为label和view创建框架。
5. 复制图标框，调整，使文本具有一定的扩展空间，并将其移动到该图标的右侧。当我们把标签添加到视图，你会看到布局的计算。

添加如下代码到`-viewDidLoad`：

~~~
// 1
UIView *header = [[UIView alloc] initWithFrame:headerFrame];
header.backgroundColor = [UIColor clearColor];
self.tableView.tableHeaderView = header;
 
// 2
// bottom left
UILabel *temperatureLabel = [[UILabel alloc] initWithFrame:temperatureFrame];
temperatureLabel.backgroundColor = [UIColor clearColor];
temperatureLabel.textColor = [UIColor whiteColor];
temperatureLabel.text = @"0°";
temperatureLabel.font = [UIFont fontWithName:@"HelveticaNeue-UltraLight" size:120];
[header addSubview:temperatureLabel];
 
// bottom left
UILabel *hiloLabel = [[UILabel alloc] initWithFrame:hiloFrame];
hiloLabel.backgroundColor = [UIColor clearColor];
hiloLabel.textColor = [UIColor whiteColor];
hiloLabel.text = @"0° / 0°";
hiloLabel.font = [UIFont fontWithName:@"HelveticaNeue-Light" size:28];
[header addSubview:hiloLabel];
 
// top
UILabel *cityLabel = [[UILabel alloc] initWithFrame:CGRectMake(0, 20, self.view.bounds.size.width, 30)];
cityLabel.backgroundColor = [UIColor clearColor];
cityLabel.textColor = [UIColor whiteColor];
cityLabel.text = @"Loading...";
cityLabel.font = [UIFont fontWithName:@"HelveticaNeue-Light" size:18];
cityLabel.textAlignment = NSTextAlignmentCenter;
[header addSubview:cityLabel];
 
UILabel *conditionsLabel = [[UILabel alloc] initWithFrame:conditionsFrame];
conditionsLabel.backgroundColor = [UIColor clearColor];
conditionsLabel.font = [UIFont fontWithName:@"HelveticaNeue-Light" size:18];
conditionsLabel.textColor = [UIColor whiteColor];
[header addSubview:conditionsLabel];
 
// 3
// bottom left
UIImageView *iconView = [[UIImageView alloc] initWithFrame:iconFrame];
iconView.contentMode = UIViewContentModeScaleAspectFit;
iconView.backgroundColor = [UIColor clearColor];
[header addSubview:iconView];
~~~
这是相当长的一块代码，但它真的只是在做设置各种控件的繁重工作。总之：

1. 设置当前view为你的table header。
2. 构建每一个显示气象数据的标签。
3. 添加一个天气图标的图像视图。

构建并运行你的App，你应该可以看到你之前布局的所有所有view。下面的屏幕截图显示了使用手工布局的、所有标签框在视觉上的显示。
![Labels and Views](http://cdn1.raywenderlich.com/wp-content/uploads/2013/12/built-layout.jpg)
用手指轻轻推动table，当你滚动它的时候，应该会反弹。

## 获取气象数据
你会注意到，App显示“Loading...”，但它不是真正在做事情。是时候获取一些真正的天气数据。

你会从[OpenWeatherMap](http://openweathermap.org/)的API拉取数据。 OpenWeatherMap是一个非常棒的服务，旨在提供实时，准确，免费的天气数据给任何人。虽然有很多的天气API，但他们大多要么使用较旧的数据格式，如XML，或是有偿服务 - 并且有时还相当昂贵。

你会遵循以下基本步骤，来获你设备的位置的气象数据： 

1. 找到设备的位置 
2. 从[API端](http://api.openweathermap.org/data/2.5/weather?lat=37.785834&lon=-122.406417&units=imperial)下载JSON数据 
3. 映射JSON到`WXConditions`和`WXDailyForecasts` 
4. 告诉UI有新数据了

开始创建你的天气模型和数据管理类。单击`File\New\File…`并选择`Cocoa Touch\Objective-C class`。命名为`WXClient`并使其为`NSObject`的子类。

这样做三次创建以下类：

* `WXManager`作为NSObject`的子类 
* `WXCondition`作为`MTLModel`的子类 
* `WXDailyForecast`作为`WXCondition`的子类

全部完成？现在，你可以开始下一节，其中涉及映射和转换您的天气数据。

## 创建你的天气模型
你的模型将使用[Mantle](https://github.com/github/Mantle)，这使得数据映射和转型非常简单。