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