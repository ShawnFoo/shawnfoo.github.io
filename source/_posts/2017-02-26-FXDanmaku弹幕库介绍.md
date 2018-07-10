---
title: FXDanmaku弹幕库介绍
date: 2017-02-26
categories: 
- 开源
- iOS
- 2017
tags: 
- 弹幕
---
## 前言

去年, 2016年, 一大波直播平台在移动端涌出, 直播慢慢步入了人们的视角. 网上如今能够看到各式各样的直播, 如秀场直播、游戏直播、体育直播、娱乐直播等等. 

在各种类型的直播中, 弹幕在PC、移动端都几乎成为了标配, 今天在这里主要介绍一下个人开源的iOS弹幕, 以及提前为`实现一款弹幕库涉及的相关技术分享`的相关篇章占坑, 虽不细至于手把手教如何实现, 但关键点都会有所涉及且不仅限于实现弹幕, 如iOS中用pthread实现生产者消费者模型、响应正在执行动画对象的点击事件、实现某类对象复用的ReuseQueue、使用GCD封装实现可取消未执行代码块的OperationQueue等等, *对这些更感兴趣的朋友麻烦直接滑至最后一段.*

<!-- more -->

欢迎各位大神指点一二. 

#### 广告

统计了各渠道的一周浏览记录, 以及github浏览次数、评论留言数, 感觉弹幕估计是有点过时了..感兴趣的朋友比较少, 所以**弹幕相关技术分享打算暂时一缓**. 准备开源并分享一下可能更多人感兴趣的序列帧动画引擎, Demo会通过分别Core Animation以及个人FXAnimationEngine来实现花椒礼物动画的效果(资源花椒ipa中提取), 比较内存占用, 动画被系统打断情况下的表现, 图片解码相关知识等. 此外, 还会分享一下礼物资源热更的方案.

## Github

[Talk is cheap, I'll show you the code](https://github.com/ShawnFoo/FXDanmaku).

请大力点击上方超链接⬆️⬆️⬆️⬆️⬆️

## 特性

1. 除了UI操作, 其他操作都以代码块交给异步队列处理了.(使用GCD提交的代码块, 最终会由XNU kernel根据CPU使用情况创建新的线程去执行或分配给其他线程执行)
2. 遵循 生产者消费者模式, 通过pthread去阻塞队列而非使用timer或异步队列开启runloop空转
3. 定义了包含 弹幕块点击、将出现、已消失事件的delegate
4. 提供 注册复用 自定义弹幕块 的方法
5. 各种自定义参数, 如弹幕块移速, 弹幕库插入方向(从上, 从下, 随机), 弹幕库移动方向(左到右, 右到左), 重置弹道位移百分比系数(防前后弹幕块碰撞)、弹幕队列容量控制
6. 简单易用, 控制方法就三个 start(同时也是恢复), pause, stop. 另外大部分方法都是线程安全的
7. 轻易适配设备方向旋转
8. 设置单行配置即可作为 跑马灯、直播间公告 使用

## 预览图

![](http://upload-images.jianshu.io/upload_images/303892-2d797e38efceb2f4.gif?imageMogr2/auto-orient/strip)
![](http://upload-images.jianshu.io/upload_images/303892-824590dfc5af0cf6.gif?imageMogr2/auto-orient/strip)

## 示例

弹幕设置

```
// Configuration
FXDanmakuConfiguration *config = [FXDanmakuConfiguration defaultConfiguration];
config.rowHeight = [DemoDanmakuItem itemHeight];
config.dataQueueCapacity = 500;
config.itemMinVelocity = 80;  // set random velocity between 80 and 120 pt/s
config.itemMaxVelocity = 120;
self.danmaku.configuration = config;

// Delegate
self.danmaku.delegate = self;

// Reuse
[self.danmaku registerNib:[UINib nibWithNibName:NSStringFromClass([DemoDanmakuItem class]) bundle:nil]
   forItemReuseIdentifier:[DemoDanmakuItem reuseIdentifier]];
[self.danmaku registerClass:[DemoBulletinItem class] 
     forItemReuseIdentifier:[DemoBulletinItem reuseIdentifier]];
```

数据添加

```
// add data for danmaku view to present
DemoDanmakuItemData *data = [DemoDanmakuItemData data];
[self.danmaku addData:data];

// start running
if (!self.danmaku.isRunning) {
	[self.danmaku start];
}
```

代理事件

```
- (void)danmaku:(FXDanmaku *)danmaku didClickItem:(FXDanmakuItem *)item withData:(DemoDanmakuItemData *)data {
    // 此处 处理点击
}

- (void)danmaku:(FXDanmaku *)danmaku willDisplayItem:(FXDanmakuItem *)item withData:(FXDanmakuItemData *)data {
    // 此处 处理弹幕块将要出现/展示
}

- (void)danmaku:(FXDanmaku *)danmaku didEndDisplayingItem:(FXDanmakuItem *)item withData:(FXDanmakuItemData *)data {
    // 此处 处理弹幕块完全离开视线，结束展示
}
```
更多详情 麻烦参照 [Gitbub Demo project](https://github.com/ShawnFoo/FXDanmaku) `FXDanmakuDemo.xcworkspace`. 

## 有关弹幕库使用问题答疑

#### 1. rowHeight、estimatedRowSpace and rowSpace 三者之间的关系

![](http://upload-images.jianshu.io/upload_images/303892-7c9d2ff6d846ccc3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2. 如何使用nib创建自定义弹幕块

![](http://upload-images.jianshu.io/upload_images/303892-a9224a1153c59bac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/303892-66430222e3e71551.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3. 如何适配设备屏幕旋转

如果你的弹幕View 在横竖屏状态下 高度不一样, 比如竖屏高200pt, 横屏约束却是100pt, 那么需要在对应的controller.m文件中加入以下代码(否则 当你的弹幕块使用AutoLayout进行布局时, 横竖屏切换后, 由于视图的frame会变化多次，导致正在展示的弹幕块 出现布局约束冲突的报错)

*iOS8+*

```
- (void)viewWillTransitionToSize:(CGSize)size withTransitionCoordinator:(id<UIViewControllerTransitionCoordinator>)coordinator {
    [self.danmaku pause];
    [self.danmaku cleanScreen];
    
    [coordinator animateAlongsideTransition:nil
		        	             completion:^(id<UIViewControllerTransitionCoordinatorContext>  _Nonnull context) {
                                     // resume danmaku after orientation did change
                                     [self.danmaku start];
                                 }];
}
```

*系统版本小于iOS8*

```
- (void)willRotateToInterfaceOrientation:(UIInterfaceOrientation)toInterfaceOrientation duration:(NSTimeInterval)duration {
    [self.danmaku pause];
    [self.danmaku cleanScreen];
}

- (void)didRotateFromInterfaceOrientation:(UIInterfaceOrientation)fromInterfaceOrientation {
    [self.danmaku start];
}
```

## 安装

#### Cocoapods(iOS7+)

1. Podfile中 视情况对应添加以下内容
	```
	platform :ios, 'xxx'
	target 'xxx' do
	  pod 'FXDanmaku'
	end
	```
2. `pod install`

#### Manually(iOS7+)

直接拖动 `FXDanmaku` 文件夹 到你的项目 对应结构下

## 介绍结尾

欢迎各位 提出宝贵的issues, 更多功能建议, 或者改进之处等等. 同时若各位想要了解弹幕库具体实现的其他相关点, 也可在评论区留言.