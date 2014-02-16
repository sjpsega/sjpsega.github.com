---
layout: post
title: "[译]iOS7最佳实践：一个天气App案例(二)"
date: 2014-02-15 16:46:02 +0800
comments: true
categories: study
keywords: iOS7, Cocoapods, ReactiveCocoa, Mantle
---

## 开始

你有两个选择开始本教程：您可以使用你本教程的第1部分已完成的项目，或者你可以在这里下载[已完成的项目](http://cdn4.raywenderlich.com/wp-content/uploads/2013/11/SimpleWeather-Part-1.zip)。 

在前面的教程中你创建了你的App的天气模型 - 现在你需要使用OpenWeatherMap API为你的App来获取一些数据。你将使用两个类抽象数据抓取、分析、存储：`WXClient`和`WXManager`。

`WXClient`的唯一责任是创建API请求，并解析它们；别人可以不用担心用数据做什么以及如何存储它。划分类的不同工作职责的设计模式被称为关注点分离。这使你的代码更容易理解，扩展和维护。

## 与ReactiveCocoa工作
确保你使用`SimpleWeather.xcworkspace`,打开`WXClient.h`并增加imports

```objc
@import CoreLocation;
#import <ReactiveCocoa/ReactiveCocoa/ReactiveCocoa.h>
```

```
注意：您可能之前没有见过的@import指令，它在Xcode5中被引入，是由苹果公司看作是一个现代的，更高效的替代 #import。有一个非常好的教程，涵盖了最新的Objective-C特性-[What’s New in Objective-C and Foundation in iOS 7](http://www.raywenderlich.com/49850/whats-new-in-objective-c-and-foundation-in-ios-7)。
```

在`WXClient.h`中添加下列四个方法到接口申明：

```objc
@import Foundation;
- (RACSignal *)fetchJSONFromURL:(NSURL *)url;
- (RACSignal *)fetchCurrentConditionsForLocation:(CLLocationCoordinate2D)coordinate;
- (RACSignal *)fetchHourlyForecastForLocation:(CLLocationCoordinate2D)coordinate;
- (RACSignal *)fetchDailyForecastForLocation:(CLLocationCoordinate2D)coordinate;
```

现在，似乎是一个很好的机会来介绍[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)！

ReactiveCocoa（RAC）是一个Objective-C的框架，用于函数式反应型编程，它提供的组合和转化数据流的API。代替专注于编写串行的代码 - 执行有序的代码队列 - 可以响应非确定性事件。

Github上提供的[a great overview of the benefits](https://github.com/blog/1107-reactivecocoa-for-a-better-world)： 

* 对未来数据的进行组合操作的能力。 
* 减少状态和可变性。 
* 用声明的形式来定义行为和属性之间的关系。 
* 为异步操作带来一个统一的，高层次的接口。 
* 一个良好的API建立在KVO的基础上。

例如，你可以监听`username`属性的变化，用这样的代码:

```objc
[RACAble(self.username) subscribeNext:^(NSString *newName) {
    NSLog(@"%@", newName);
}];
```

`subscribeNext`这个block会在`self.username`属性变化的时候执行。新的值会传递给这个block。

您还可以合并信号并组合数据到一个组合数据中。下面的示例取自于ReactiveCocoa的Github页面：

```objc
[[RACSignal 
    combineLatest:@[ RACAble(self.password), RACAble(self.passwordConfirmation) ] 
           reduce:^(NSString *currentPassword, NSString *currentConfirmPassword) {
               return [NSNumber numberWithBool:[currentConfirmPassword isEqualToString:currentPassword]];
           }] 
    subscribeNext:^(NSNumber *passwordsMatch) {
        self.createEnabled = [passwordsMatch boolValue];
    }];
```

[RACSignal](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/ReactiveCocoaFramework/ReactiveCocoa/RACSignal.h)对象捕捉当前和未来的值。信号可以被观察者链接，组合和反应。信号实际上不会执行，直到它被订阅。

这意味着调用`[mySignal fetchCurrentConditionsForLocation：someLocation];`不会做什么，但创建并返回一个信号。你将看到之后如何订阅和反应。

打开`WXClient.m`加入以下imports:

```objc
#import "WXCondition.h"
#import "WXDailyForecast.h"
```

在imports下，添加私有接口：

```objc
@interface WXClient ()
 
@property (nonatomic, strong) NSURLSession *session;
 
@end
```

这个接口有个属性，用来管理API请求的URL session。

添加以下`init`放到到`@implementation`和`@end`之间：

```objc
- (id)init {
    if (self = [super init]) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        _session = [NSURLSession sessionWithConfiguration:config];
    }
    return self;
}
```

使用`defaultSessionConfiguration`为您创建了的session。

```
注意：如果你以前没有了解过NSURLSession，看看我们的[NSURLSession教程](http://www.raywenderlich.com/51127/nsurlsession-tutorial)，了解更多信息。
```

## 构建信号
你需要一个主方法来建立一个信号从URL中取数据。你已经知道，需要三种方法来获取当前的状况下，逐时预报及每日预报。

而不是写三个独立的方法，你可以按照[DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself)（Don’t Repeat Yourself）的软件设计理念，使您的代码容易维护。

第一次看，以下的一些ReactiveCocoa部分可能看起来相当陌生。别担心，你会一块一块理解他。

增加下列方法到`WXClient.m`:

```objc
- (RACSignal *)fetchJSONFromURL:(NSURL *)url {
    NSLog(@"Fetching: %@",url.absoluteString);
 
    // 1
    return [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        // 2
        NSURLSessionDataTask *dataTask = [self.session dataTaskWithURL:url completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
            // TODO: Handle retrieved data
        }];
 
        // 3
        [dataTask resume];
 
        // 4
        return [RACDisposable disposableWithBlock:^{
            [dataTask cancel];
        }];
    }] doError:^(NSError *error) {
        // 5
        NSLog(@"%@",error);
    }];
}
```

通过一个一个注释，你会看到代码执行以下操作：

1. 返回信号。请记住，这将不会执行，直到这个信号被订阅。 `- fetchJSONFromURL：`创建一个对象为其他方法和对象使用;这种行为有时也被称为[工厂模式](https://developer.apple.com/library/ios/documentation/general/conceptual/CocoaEncyclopedia/ClassFactoryMethods/ClassFactoryMethods.html)。 
2. 创建一个[NSURLSessionDataTask](https://developer.apple.com/library/IOS/documentation/Foundation/Reference/NSURLSessionDataTask_class/Reference/Reference.html)（在iOS7中加入）从URL取数据。你会在以后添加的数据解析。 
3. 启动网络请求一旦有人订阅了信号。 
4. 创建并返回[RACDisposable](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/ReactiveCocoaFramework/ReactiveCocoa/RACDisposable.h)对象，它处理当信号摧毁时的清理的工作。 
5. 增加了一个“side effect”，以记录发生的任何错误。side effect不订阅信号，相反，他们返回被连接到方法链的信号。你只需添加一个side effect来记录错误。

```
如果你觉得需要更多一些背景知识，看看由Ash Furrow编写的[这篇文章](http://www.teehanlax.com/blog/getting-started-with-reactivecocoa/)，以便更好地了解ReactiveCocoa的核心概念。
```


在`-fetchJSONFromURL:`中找到`// TODO: Handle retrieved data` ，替换为:

```objc
if (! error) {
    NSError *jsonError = nil;
    id json = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:&jsonError];
    if (! jsonError) {
        // 1
        [subscriber sendNext:json];
    }
    else {
        // 2
        [subscriber sendError:jsonError];
    }
}
else {
    // 2
    [subscriber sendError:error];
}
 
// 3
[subscriber sendCompleted];
```

1. 当JSON数据存在并且没有错误，发送给订阅者序列化后的JSON数组或字典。 
2. 在任一情况下如果有一个错误，通知订阅者。 
3. 无论该请求成功还是失败，通知订阅者请求已经完成。

`-fetchJSONFromURL：`方法有点长，但它使你的特定的API请求方法变得很简单。

## 获取当前状况

还在`WXClient.m`中，添加如下方法：

```objc
- (RACSignal *)fetchCurrentConditionsForLocation:(CLLocationCoordinate2D)coordinate {
    // 1
    NSString *urlString = [NSString stringWithFormat:@"http://api.openweathermap.org/data/2.5/weather?lat=%f&lon=%f&units=imperial",coordinate.latitude, coordinate.longitude];
    NSURL *url = [NSURL URLWithString:urlString];
 
    // 2
    return [[self fetchJSONFromURL:url] map:^(NSDictionary *json) {
        // 3
        return [MTLJSONAdapter modelOfClass:[WXCondition class] fromJSONDictionary:json error:nil];
    }];
}
```

1. 使用`CLLocationCoordinate2D`对象的经纬度数据，格式化URL。 
2. 用你刚刚建立的创建信号的方法。由于返回值是一个信号，你可以调用其他ReactiveCocoa的方法。 在这里，您将返回值映射到一个不同的值 - 一个NSDictionary实例。 
3. 使用`MTLJSONAdapter`来转换JSON到`WXCondition`对象 - 使用`MTLJSONSerializing`协议创建的`WXCondition`。


## 获取逐时预报

现在添加根据坐标获取逐时预报的方法到`WXClient.m`:

```objc
- (RACSignal *)fetchHourlyForecastForLocation:(CLLocationCoordinate2D)coordinate {
    NSString *urlString = [NSString stringWithFormat:@"http://api.openweathermap.org/data/2.5/forecast?lat=%f&lon=%f&units=imperial&cnt=12",coordinate.latitude, coordinate.longitude];
    NSURL *url = [NSURL URLWithString:urlString];
 
    // 1
    return [[self fetchJSONFromURL:url] map:^(NSDictionary *json) {
        // 2
        RACSequence *list = [json[@"list"] rac_sequence];
 
        // 3
        return [[list map:^(NSDictionary *item) {
            // 4
            return [MTLJSONAdapter modelOfClass:[WXCondition class] fromJSONDictionary:item error:nil];
        // 5
        }] array];
    }];
}
```

1. 再次使用`-fetchJSONFromUR`方法，映射JSON。注意：重复使用该方法节省了多少代码！ 
2. 使用JSON的"list"key创建`RACSequence`。 `RACSequences`让你对列表进行ReactiveCocoa操作。 
3. 映射新的对象列表。调用`-map：`方法，针对列表中的每个对象，返回新对象的列表。 
4. 再次使用MTLJSONAdapter`来转换JSON到`WXCondition`对象。 
5. 使用`RACSequence`的`-map`方法，返回另一个`RACSequence`，所以用这个简便的方法来获得一个`NSArray`数据。

## 获取每日预报
最后，添加如下方法到`WXClient.m`:

```objc
- (RACSignal *)fetchDailyForecastForLocation:(CLLocationCoordinate2D)coordinate {
    NSString *urlString = [NSString stringWithFormat:@"http://api.openweathermap.org/data/2.5/forecast/daily?lat=%f&lon=%f&units=imperial&cnt=7",coordinate.latitude, coordinate.longitude];
    NSURL *url = [NSURL URLWithString:urlString];
 
    // Use the generic fetch method and map results to convert into an array of Mantle objects
    return [[self fetchJSONFromURL:url] map:^(NSDictionary *json) {
        // Build a sequence from the list of raw JSON
        RACSequence *list = [json[@"list"] rac_sequence];
 
        // Use a function to map results from JSON to Mantle objects
        return [[list map:^(NSDictionary *item) {
            return [MTLJSONAdapter modelOfClass:[WXDailyForecast class] fromJSONDictionary:item error:nil];
        }] array];
    }];
}
```

是不是看起来很熟悉？是的，这个方法与`-fetchHourlyForecastForLocation:`方法非常像。除了它使用`WXDailyForecast`代替`WXCondition`，并获取每日预报。

构建并运行您的App，现在你不会看到任何新的东西，但这是一个很好机会松一口气，并确保没有任何错误或警告。

![Labels and Views](http://cdn4.raywenderlich.com/wp-content/uploads/2013/11/built-layout.jpg =320x)

## 管理并存储你的数据
现在是时间来充实`WXManager`，这个类会把所有东西结合到一起。这个类实现您App的一些关键功能：

* 它使用[单例设计模式](http://www.raywenderlich.com/46988/ios-design-patterns)。 
* 它试图找到设备的位置。 
* 找到位置后，它获取相应的气象数据。

打开`WXManager.h`使用以下代码来替换其内容：

```objc
@import Foundation;
@import CoreLocation;
#import <ReactiveCocoa/ReactiveCocoa/ReactiveCocoa.h>
// 1
#import "WXCondition.h"
 
@interface WXManager : NSObject
<CLLocationManagerDelegate>
 
// 2
+ (instancetype)sharedManager;
 
// 3
@property (nonatomic, strong, readonly) CLLocation *currentLocation;
@property (nonatomic, strong, readonly) WXCondition *currentCondition;
@property (nonatomic, strong, readonly) NSArray *hourlyForecast;
@property (nonatomic, strong, readonly) NSArray *dailyForecast;
 
// 4
- (void)findCurrentLocation;
 
@end
```

1. 请注意，你没有引入`WXDailyForecast.h`，你会始终使用`WXCondition`作为预报的类。 `WXDailyForecast`的存在是为了帮助Mantle转换JSON到Objective-C。 
2. 使用`instancetype`而不是`WXManager`，子类将返回适当的类型。 
3. 这些属性将存储您的数据。由于`WXManager`是一个单例，这些属性可以任意访问。设置公共属性为只读，因为只有管理者能更改这些值。 
4. 这个方法启动或刷新整个位置和天气的查找过程。

现在打开` WXManager.m`并添加如下imports到文件顶部：

```objc
#import "WXClient.h"
#import <TSMessages/TSMessage.h>
```

在imports下方，粘贴如下私有接口：

```objc
@interface WXManager ()
 
// 1
@property (nonatomic, strong, readwrite) WXCondition *currentCondition;
@property (nonatomic, strong, readwrite) CLLocation *currentLocation;
@property (nonatomic, strong, readwrite) NSArray *hourlyForecast;
@property (nonatomic, strong, readwrite) NSArray *dailyForecast;
 
// 2
@property (nonatomic, strong) CLLocationManager *locationManager;
@property (nonatomic, assign) BOOL isFirstUpdate;
@property (nonatomic, strong) WXClient *client;
 
@end
```

1. 声明你在公共接口中添加的相同的属性，但是这一次把他们定义为读写，因此您可以在后台更改他们。 
2. 为查找定位和数据抓取声明一些私有变量。

添加如下通用的单例构造器到@implementation与@end中间：

```objc
+ (instancetype)sharedManager {
    static id _sharedManager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedManager = [[self alloc] init];
    });
 
    return _sharedManager;
}
```

然后，你需要设置你的属性和观察者。

添加如下方法到`WXManager.m`:

```objc
- (id)init {
    if (self = [super init]) {
        // 1
        _locationManager = [[CLLocationManager alloc] init];
        _locationManager.delegate = self;
 
        // 2
        _client = [[WXClient alloc] init];
 
        // 3
        [[[[RACObserve(self, currentLocation)
            // 4
            ignore:nil]
            // 5
           // Flatten and subscribe to all 3 signals when currentLocation updates
           flattenMap:^(CLLocation *newLocation) {
               return [RACSignal merge:@[
                                         [self updateCurrentConditions],
                                         [self updateDailyForecast],
                                         [self updateHourlyForecast]
                                         ]];
            // 6
           }] deliverOn:RACScheduler.mainThreadScheduler]
           // 7
         subscribeError:^(NSError *error) {
             [TSMessage showNotificationWithTitle:@"Error" 
                                         subtitle:@"There was a problem fetching the latest weather."
                                             type:TSMessageNotificationTypeError];
         }];
    }
    return self;
}
```

你正使用更多的ReactiveCocoa方法来观察和反应数值的变化。上面这些你做了：

1. 创建一个位置管理器，并设置它的delegate为`self`。 
2. 为管理器创建`WXClient`对象。这里处理所有的网络请求和数据分析，这是关注点分离的最佳实践。 
3. 管理器使用一个返回信号的ReactiveCocoa脚本来观察自身的`currentLocation`。这与[KVO](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html)类似，但更为强大。 
4. 为了继续执行方法链，`currentLocation`必须不为`nil`。 
5. `- flattenMap：`非常类似于`-map：`，但不是映射每一个值，它把数据变得扁平，并返回包含三个信号中的一个对象。通过这种方式，你可以考虑将三个进程作为单个工作单元。 
6. 将信号传递给主线程上的观察者。 
7. 这不是很好的做法，在你的模型中进行UI交互，但出于演示的目的，每当发生错误时，会显示一个banner。

接下来，为了显示准确的天气预报，我们需要确定设备的位置。

## 查找你的位置
下一步，你要添加当位置查找到，触发抓取天气数据的代码。

添加如下代码到`WXManager.m`的实现块中:

```objc
- (void)findCurrentLocation {
    self.isFirstUpdate = YES;
    [self.locationManager startUpdatingLocation];
}
 
- (void)locationManager:(CLLocationManager *)manager didUpdateLocations:(NSArray *)locations {
    // 1
    if (self.isFirstUpdate) {
        self.isFirstUpdate = NO;
        return;
    }
 
    CLLocation *location = [locations lastObject];
 
    // 2
    if (location.horizontalAccuracy > 0) {
        // 3
        self.currentLocation = location;
        [self.locationManager stopUpdatingLocation];
    }
}
```

1. 忽略第一个位置更新，因为它一般是缓存值。 
2. 一旦你获得一定精度的位置，停止进一步的更新。 
3. 设置`currentLocation`，将触发您之前在init中设置的RACObservable。

## 获取气象数据
最后，是时候添加在客户端上调用并保存数据的三个获取方法。将三个方法捆绑起来，被之前在`init`方法中添加的RACObservable订阅。您将返回客户端返回的，能被订阅的，相同的信号。

所有的属性设置发生在`-doNext:`中。

添加如下代码到`WXManager.m`:

```objc
- (RACSignal *)updateCurrentConditions {
    return [[self.client fetchCurrentConditionsForLocation:self.currentLocation.coordinate] doNext:^(WXCondition *condition) {
        self.currentCondition = condition;
    }];
}
 
- (RACSignal *)updateHourlyForecast {
    return [[self.client fetchHourlyForecastForLocation:self.currentLocation.coordinate] doNext:^(NSArray *conditions) {
        self.hourlyForecast = conditions;
    }];
}
 
- (RACSignal *)updateDailyForecast {
    return [[self.client fetchDailyForecastForLocation:self.currentLocation.coordinate] doNext:^(NSArray *conditions) {
        self.dailyForecast = conditions;
    }];
}
```

它看起来像将一切都连接起来，并蓄势待发。别急！这App实际上并没有告诉管理者做任何事情。 
打开`WXController.m`并导入这管理者到文件的顶部，如下所示：

```objc
#import "WXManager.h"
```

添加如下代码到`-viewDidLoad:`的最后:

```objc
[[WXManager sharedManager] findCurrentLocation];
```

这告诉管理类，开始寻找设备的当前位置。 

构建并运行您的App，系统会提示您是否允许使用位置服务。你仍然不会看到任何UI的更新，但检查控制台日志，你会看到类似以下内容：

```
2013-11-05 08:38:48.886 WeatherTutorial[17097:70b] Fetching: http://api.openweathermap.org/data/2.5/weather?lat=37.785834&lon=-122.406417&units=imperial
2013-11-05 08:38:48.886 WeatherTutorial[17097:70b] Fetching: http://api.openweathermap.org/data/2.5/forecast/daily?lat=37.785834&lon=-122.406417&units=imperial&cnt=7
2013-11-05 08:38:48.886 WeatherTutorial[17097:70b] Fetching: http://api.openweathermap.org/data/2.5/forecast?lat=37.785834&lon=-122.406417&units=imperial&cnt=12
```

这些输出代表你的代码工作正常，网络请求正常执行。

## 连接接口
这是最后一次显示所有你获取，映射和存储的数据。您将使用ReactiveCocoa来观察`WXManager`单例的变化和当新数据到达时更新界面。 

还在`WXController.m`，到`- viewDidLoad`的底部，并添加下面的代码到`[[WXManager sharedManager] findCurrentLocation];`之前：

```objc
// 1
[[RACObserve([WXManager sharedManager], currentCondition)
  // 2
  deliverOn:RACScheduler.mainThreadScheduler]
 subscribeNext:^(WXCondition *newCondition) {
     // 3
     temperatureLabel.text = [NSString stringWithFormat:@"%.0f°",newCondition.temperature.floatValue];
     conditionsLabel.text = [newCondition.condition capitalizedString];
     cityLabel.text = [newCondition.locationName capitalizedString];
 
     // 4
     iconView.image = [UIImage imageNamed:[newCondition imageName]];
 }];
```

1. 观察`WXManager`单例的currentCondition。 
2. 提供在主线程上的任何变化，因为你正在更新UI。 
3. 使用气象数据更新的文本标签;你为文本标签使用`newCondition`的数据，而不是单例。订阅者的参数保证是最新值。 
4. 使用映射的图像文件名来创建一个图像，并将其设置为视图的图标。

构建并运行您的App，你会看到当前温度，当前状况和表示当前状况的图标。所有的数据都是实时的。但是，如果你的位置是旧金山，它似乎总是约65度。Lucky San Franciscans! :]
![Wiring up the UI](http://cdn3.raywenderlich.com/wp-content/uploads/2013/11/ui-wiring.jpg =320x)

#ReactiveCocoa的绑定