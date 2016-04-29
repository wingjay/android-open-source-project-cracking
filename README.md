# Android 优质开源项目剖析与技术进阶
本系列的文章主要面向以下几类读者：

1. 不满足于`简单使用开源库`，想要`通过探究其原理`以`精进自身技术或创建自己的开源库`；
2. 对于`新技术`如`RxJava`等的实践处于摸索状态，希望有`优质的code实例及细致分析`来让你迅速上手一门新技术；
3. 对于一些`底层库`如`网络底层库Retrofit`、`图片加载库Picasso/Glide`等实现原理保持好奇。

_________________________________

比起阅读枯燥文档，独自摸索一门技术的最佳实践，我们还有一种方法能够快速而稳定的精进自身的开发水平，那就是通过`阅读、分析、仿写与理解`优质的开源项目。

### 一、已有文章

#### Application
分析文档 | 作者 | 开源项目 | 介绍 | 添加时间
:------------- | :------------- | :------------- | :------------- | :------------- 
[Meizhi Android之RxJava & Retrofit最佳实践](https://github.com/wingjay/android-open-source-project-cracking/blob/master/application/Meizhi%20Android%E4%B9%8BRxJava%20%26%20Retrofit%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5.md) | [wingjay](https://github.com/wingjay) | [Meizhi](https://github.com/drakeet/Meizhi) | 使用RxJava & Retrofit的sample | 20160416

#### View
分析文档 | 作者 | 开源项目 | 介绍 | 添加时间
:------------- | :------------- | :------------- | :------------- | :-------------
[JJSearchViewAnim源码分析](http://www.jianshu.com/p/a48f4e6cf036)| [Skykai521](https://github.com/Skykai521)| [JJSearchViewAnim](https://github.com/android-cjj/JJSearchViewAnim) | | 20160417
[SwipeBackLayout源码分析](http://www.jianshu.com/p/a91d669421e9)| [Skykai521](https://github.com/Skykai521)| [SwipeBackLayout](https://github.com/ikew0ng/SwipeBackLayout) | | 20160417
[HTextView源码分析](http://www.jianshu.com/p/15358d444800)| [Skykai521](https://github.com/Skykai521)| [HTextView](https://github.com/hanks-zyh/HTextView) | | 20160417

#### Library
分析文档 | 作者 | 开源项目 | 介绍 | 添加时间
:------------- | :------------- | :------------- | :------------- | :-------------
[ButterKnife源码分析](https://github.com/wingjay/android-open-source-project-cracking/blob/master/library/ButterKnife.md)| [BigFootprint](https://github.com/BigFootprint)| [ButterKnife](http://jakewharton.github.io/butterknife/) | | 20160423
[RxPermissions源码解析](http://www.jianshu.com/p/c8a30200e6b2)| [Skykai521](https://github.com/Skykai521)| [RxPermissions](https://github.com/tbruyelle/RxPermissions) | | 20160417
[BarcodeScanner源码分析](http://www.jianshu.com/p/d34383d4cb89)| [Skykai521](https://github.com/Skykai521)| [BarcodeScanner](https://github.com/dm77/barcodescanner) | | 20160417
[ViewAnimator源码分析](http://www.jianshu.com/p/749c4531d108)| [Skykai521](https://github.com/Skykai521)| [ViewAnimator](https://github.com/florent37/ViewAnimator) | | 20160417
[uCrop源码分析](http://www.jianshu.com/p/523e77a10321)| [Skykai521](https://github.com/Skykai521)| [uCrop](https://github.com/Yalantis/uCrop) | | 20160417
[Picasso源代分析](http://www.jianshu.com/p/3c36382bc1cd)| [Skykai521](https://github.com/Skykai521)| [Picasso](https://github.com/square/picasso) | | 20160417
[EventBus 3.0源码分析](http://www.jianshu.com/p/f057c460c77e)| [Skykai521](https://github.com/Skykai521)| [EventBus](https://github.com/greenrobot/EventBus) | | 20160417

### 二、我们会挑选哪些`优质开源项目`
#### 1. 新技术
我们会挑选覆盖`RxJava`、`React Native`、`Dynamic load`、`Dagger`、`Retrofit`等新技术的开源项目，分析总结出`新技术最佳实践`供读者阅读仿写，快速将新技术应用到自身项目开发中，不用反复踩坑。

#### 2. 底层库
初级程序员会调用API、实现基本功能；
中级程序员开始`封装`，消除`ugly`代码；
高级程序员能够`设计架构`，重构出`优雅`代码。

我们会挑选一些优秀底层库，`深入浅出`的去分析它们的`设计思想`，阐述如何把这些设计思想融入到自身实际项目中。

#### 3. 自定义View
很多人习惯了在Github寻找通用的UI库。

坏消息是，UI的变化千千万，迟早有一天我们会不得不由于自己项目的特殊性，而要自己来实现自定义view。

好消息是，自定义view虽然变化万千，但却不离其宗，而我们的分析就是尝试向你讲述`如何理解自定义view的原理`。

### 三、加入我们
如果你对本项目有兴趣，你可以选择以下方式之一加入进来：

1. `阅读者`。`start & watch`这个项目，关注微信公众号`CoolCoder`，我们会在第一时间推送。
2. `写作者`。如果你热爱分析开源项目，热爱分享与写作。那就挑选一个你认为优质的开源项目进行写作，创建pull request。
3. `评论者`。阅读中遇到问题？直接创建[issue](https://github.com/wingjay/android-open-source-project-cracking/issues)，作者会快速回答你。
4. `翻译者`。如果你还不具备分析开源项目的能力，那可以来对我们的中文文章进行翻译。这个翻译过程会让你受益匪浅的。
5. `校对者`。如果你技术过硬，愿意帮助新手程序员，可以发邮件给我<mailto:yinjiesh@126.com>，我相信"校对者"三个字会让很多年轻程序员记住你。

### 四、写作者
[wingjay](https://github.com/wingjay) 

![](https://avatars0.githubusercontent.com/u/9619875?v=3&s=460)

[达庆凯 Skykai521](https://github.com/Skykai521) 

![](https://avatars3.githubusercontent.com/u/8402109?v=3&s=100)

...


