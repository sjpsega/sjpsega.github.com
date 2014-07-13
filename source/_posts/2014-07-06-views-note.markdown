---
layout: post
title: "《#3 Views》 学习笔记"
date: 2014-07-06 17:15:53 +0800
comments: true
categories: study
keywords: iOS 视图
---
文章地址：[objc第三章](http://objccn.io/issue-3/)

## 绘制像素到屏幕上

## 理解 Scroll Views

### 光栅化和组合

渲染过程的第一部分是众所周知的光栅化(`rasterization`)，光栅化简单的说就是产生一组绘图指令并且生成一张图片。

每个视图都有一个 `bounds` 和 `frame`。当布局一个界面时，我们需要处理视图的 frame。这允许我们放置并设置视图的大小。视图的 frame 和 bounds 的大小总是一样的，但是他们的 origin 有可能不同。

![scrollview_1](/images/2014-07-06-views-note/scrollview_1.png)

视图图片的左上角会根据它 frame 的 origin 进行偏移，并绘制到父视图的图片上：

```objc
CompositedPosition.x = View.frame.origin.x - Superview.bounds.origin.x;

CompositedPosition.y = View.frame.origin.y - Superview.bounds.origin.y;
```

正如之前所说的，如果一个视图 bounds 的 origin 是 {0,0}。那么，我们得到这个公式：

```objc
CompositedPosition.x = View.frame.origin.x;

CompositedPosition.y = View.frame.origin.y;
```

### Scroll View的Content Offset

```objc
CompositedPosition.x = View.frame.origin.x - Superview.bounds.origin.x;

CompositedPosition.y = View.frame.origin.y - Superview.bounds.origin.y;
```

我们减少 `Superview.bounds.origin` 的值(因为他们总是0)。但是如果他们不为0呢？我们用和前一个图例相同的 frames，但是我们改变了紫色视图 bounds 的 origin 为 {-30, -30}。得到下图：

![scrollview_4](/images/2014-07-06-views-note/scrollview_4.png)

现在，巧妙的是通过改变这个紫色视图的 bounds，它每一个单独的子视图都被移动了。事实上，这正是 scroll view 工作的原理。当你设置它的 contentOffset 属性时它改变 scroll view.bounds 的 origin。事实上，contentOffset 甚至不是实际存在的。代码看起来像这样：

```objc
- (void)setContentOffset:(CGPoint)offset
{
    CGRect bounds = [self bounds];
    bounds.origin = offset;
    [self setBounds:bounds];
}
```

### Content Size
content size 定义了可滚动区域。

当 content size 设置为比 bounds 大的时候，用户就可以滚动视图了。

![scrollview_5](/images/2014-07-06-views-note/scrollview_5.png)

### 用Content Insets对窗口稍作调整
contentInset 属性可以改变 content offset 的最大和最小值，这样便可以滚动出可滚动区域。

实际使用：

* 下拉刷新的实现

![scrollview_6](/images/2014-07-06-views-note/scrollview_6.png)

* 键盘出现，不遮挡可视区域

当键盘出现在屏幕上时，设置 content inset 的底部等于键盘的高度。

![scrollview_7](/images/2014-07-06-views-note/scrollview_7.png)

## 自定义 Collection View 布局

### 布局对象 (Layout Objects)
UITableView 和 UICollectionView 都是 data-source 和 delegate 驱动的。

布局继承自 UICollectionViewLayout 抽象基类。

### Cells 和其他 Views
Collection view 的 cell 必须是 UICollectionViewCell 的子类。除了 cell，collection view 额外管理着两种视图：supplementary views 和 decoration views。

`Supplementary views` 相当于 table view 的 section header 和 footer views。像 cells 一样，他们的内容都由数据源对象驱动。然而和 table view 中用法不一样，supplementary view 并不一定会作为 header 或 footer view；他们的数量和放置的位置完全由布局控制。

`Decoration views` 纯粹为一个装饰品。他们完全属于布局对象，并被布局对象管理，他们并不从 data source 获取的 contents。当布局对象指定需要一个 decoration view 的时候，collection view 会自动创建，并将布局对象提供的布局参数应用到上面去。并不需要为自定义视图准备任何内容。

Supplementary views 和 decoration views 必须是 `UICollectionResuableView` 的子类。

### 自定义布局
一般有两种类型的 collection view 布局：

* 独立于内容的布局计算
* 基于内容的布局计算

#### collectionViewContentSize
布局首先要提供的信息就是滚动区域大小，这样 collection view 才能正确的管理滚动。布局对象必须在此时计算它内容的总大小，包括 supplementary views 和 decoration views。

#### layoutAttributesForElementsInRect:
具体实现必须返回一个包含 `UICollectionViewLayoutAttributes` 对象的数组，为每一个 cell 包含一个这样的对象，supplementary view 或 decoration view 在矩形区域内是可见的。UICollectionViewLayoutAttributes 类包含了 collection view 内 item 的所有相关布局属性。默认情况下，这个类包含 frame，center，size，transform3D，alpha，zIndex 和 hidden属性。

这个方法涉及到所有类型的视图，也就是 cell，supplementary views 和 decoration views。

实现需要做这几步：

1. 创建一个空的可变数组来存放所有的布局属性。

2. 确定 index paths 中哪些 cells 的 frame 完全或部分位于矩形中。这个计算需要你从 collection view 的数据源中取出你需要显示的数据。然后在循环中调用你实现的 `layoutAttributesForItemAtIndexPath:` 方法为每个 index path 创建并配置一个合适的布局属性对象，并将每个对象添加到数组中。

3. 如果你的布局包含 supplementary views，计算矩形内可见 supplementary view 的 index paths。在循环中调用你实现的 `layoutAttributesForSupplementaryViewOfKind:atIndexPath:` ，并且将这些对象加到数组中。通过为 kind 参数传递你选择的不同字符，你可以区分出不同种类的supplementary views（比如headers和footers）。当需要创建视图时，collection view 会将 kind 字符传回到你的数据源。记住 supplementary 和 decoration views 的数量和种类完全由布局控制。你不会受到 headers 和 footers 的限制。

4. 如果布局包含 decoration views，计算矩形内可见 decoration views 的 index paths。在循环中调用你实现的 layoutAttributesForDecorationViewOfKind:atIndexPath: ，并且将这些对象加到数组中。

5. 返回数组。

#### layoutAttributesFor…IndexPath

方法|创建布局对象
---|---
layoutAttributesForItemAtIndexPath:|[UICollectionViewLayoutAttributes layoutAttributesForCellWithIndexPath:]
layoutAttributesForSupplementaryViewOfKind:atIndexPath:|[UICollectionViewLayoutAttributes layoutAttributesForSupplementaryViewOfKind:withIndexPath:]
layoutAttributesForDecorationViewOfKind:atIndexPath:|[UICollectionViewLayoutAttributes layoutAttributesForDecorationViewOfKind:withIndexPath:]

#### shouldInvalidateLayoutForBoundsChange
当 collection view 的 bounds 改变时，布局需要告诉 collection view 是否需要重新计算布局。

### 动画
#### 插入和删除
对应实现的方法：

* initialLayoutAttributesForAppearingItemAtIndexPath:
* initialLayoutAttributesForAppearingSupplementaryElementOfKind:atIndexPath:
* initialLayoutAttributesForAppearingDecorationElementOfKind:atIndexPath:
* finalLayoutAttributesForDisappearingItemAtIndexPath:
* finalLayoutAttributesForDisappearingSupplementaryElementOfKind:atIndexPath:
* finalLayoutAttributesForDisappearingDecorationElementOfKind:atIndexPath:

#### 布局间切换
可以通过调用 `setCollectionViewLayout:animated:` 将一个 collection view 布局动态的切换到另外一个布局。

## 自定义控件
### 视图层次概览
#### UIResponder
UIResponder 是 `UIView` 的父类。responder 能够处理触摸、手势、远程控制等事件。

UIResponder 的子类有 `UIView`、`UIApplication`、`UIViewController`。

通过重写 UIResponder 的方法，可以决定一个类是否可以成为第一响应者 (first responder)，例如当前输入焦点元素。

UIResponder 允许自定义输入方法，使用 inputAccessoryView 向键盘添加辅助视图；使用 inputView 提供一个完全自定义的键盘。

#### UIView
UIView 子类处理所有跟内容绘制有关的事情以及触摸时间。

一个普遍错误的概念：视图的区域是由它的 frame 定义的。实际上 `frame` 是一个`派生属性`，是由 `center` 和 `bounds` 合成而来。

#### UIControl
UIControl 建立在视图上，增加了更多的交互支持。

最重要的是，它增加了 target / action 模式。

### 渲染
尽量避免 `drawRect:`，使用现有的视图构建自定义视图。

通常最快速的渲染方法是使用图片视图。

把可拉伸的图片和图片视图一起使用也可以极大的提高效率。可拉伸图片是这些技术中最快的。原因是可拉伸图片在 CPU 和 GPU 之间的数据转移量最小，并且这些图片的绘制是经过高度优化的。(`-[UIImage resizableImageWithCapInsets:resizingMode:]`)

#### 自定义绘制
技术选择：

图片 > Core Animation > Core Graphics > GLKit 和原生 OpenGL

若重写 `drawRect:`，确保检查内容模式。默认的模式是将内容缩放以填充视图的范围，这在当视图的 frame 改变时并不会重新绘制。

### 自定义交互
创建自定义控件时所面对的一个普遍的设计问题是向拥有它们的类中回传返回值。可选择的方式有 target action 模式，代理，block 或者 KVO，甚至通知。

#### 使用 Target-Action
最方便的做法是使用 target-action。

自定义视图中：
```objc
[self sendActionsForControlEvents:UIControlEventValueChanged];
```

视图控制器中：
```objc
- (void)setupPieChart
{
    [self.pieChart addTarget:self
                  action:@selector(updateSelection:)
        forControlEvents:UIControlEventValueChanged];
}

- (void)updateSelection:(id)sender
{
    NSLog(@"%@", self.pieChart.selectedSector);
}
```
这么做的好处是在自定义视图子类中需要做的事情很少，并且自动获得多目标支持。

#### 使用代理
如果你需要更多的控制从视图发送到视图控制器的消息，通常使用代理模式。

自定义视图中：
```objc
[self.delegate pieChart:self didSelectSector:self.selectedSector];
```

视图控制器中:
```objc
@interface MyViewController <PieChartDelegate>

 ...

- (void)setupPieChart
{
    self.pieChart.delegate = self;
}

- (void)pieChart:(PieChart*)pieChart didSelectSector:(PieChartSector*)sector
{
    // 处理区块
}
```
当你想要做更多复杂的工作而不仅仅是通知所有者值发生了变化时，这么做显然更合适。

#### 使用 Block
自定义视图中：

```objc
@interface PieChart : UIControl

@property (nonatomic,copy) void(^selectionHandler)(PieChartSection* selectedSection);

@end
```

`Block` 的好处是可以把相关的代码整合在视图控制器中:

```objc
- (void)setupPieChart
{
    self.pieChart.selectionHandler = ^(PieChartSection* section) {
        // 处理区块
    }
}
```

一旦 block 中的代码要失去控制 (比如 block 中要处理的事情太多，导致 block 中的代码过多)，你还应该将它们抽离成独立的方法，这种情况的话可能用代理会更好一些。

#### 使用 KVO
编写代码:

```objc
self.selectedSegment = theNewSelectedSegment;
```

当使用合成属性，KVO 会拦截到该变化并发出通知。在视图控制器中，编写类似的代码:
```objc
- (void)setupPieChart
{
    [self.pieChart addObserver:self forKeyPath:@"selectedSegment" options:0 context:NULL];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    if(object == self.pieChart && [keyPath isEqualToString:@"selectedSegment"]) {
        // 处理改变
    }
}
```

#### 使用通知

视图头文件中：
```objc
extern NSString* const SelectedSegmentChangedNotification;
```

在实现文件中：
```objc
NSString* const SelectedSegmentChangedNotification = @"selectedSegmentChangedNotification";

...

- (void)notifyAboutChanges
{
    [[NSNotificationCenter defaultCenter] postNotificationName:SelectedSegmentChangedNotification object:self];
}
```

现在订阅通知，在视图控制器中：
```objc
- (void)setupPieChart
{
    [[NSNotificationCenter defaultCenter] addObserver:self
                                       selector:@selector(segmentChanged:)
                                           name:SelectedSegmentChangedNotification
                                          object:self.pieChart];

}

...

- (void)segmentChanged:(NSNotification*)note
{
}
```

### 辅助功能 (Accessibility)
苹果官方提供的标准 iOS 控件均有辅助功能。这也是推荐用标准控件创建自定义控件的另一个原因。

### 本地化
使用 NSLocalizedString 本地化字符串。

### 测试
使用 `UIAutomation` 做视图测试。

## 先进的自动布局工具箱
