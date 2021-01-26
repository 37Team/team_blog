---
title: iOS静态库与动态库集成问题
date: 2021-1-26 20:55:06
category: iOS
tags: 
- 静态库
- 动态库
- 集成
- Coacoapod
---
### 情况一：第三方静态库，被自己的动态库、App同时集成：
- 经典警告：
`One of the two will be used. Which one is undefined.`
{% asset_img 1611651302481.png 控制台报错信息 %}
- 具体情况：
`第三方静态库`（比如RSStaticPrint）同时被`APP`、`自己的动态库SDK`集成，`APP`又嵌入`自己的动态库SDK`
- 分析：
【现象】存在两份静态库，各自load方法都会执行，根据调用位置各自调用所在位置的`第三方静态库`（比如RSStaticPrint）。
【对象情况简单剖析】
	1. `自己的动态库SDK`调用的是`自己的动态库SDK`里面的类对象RSStaticPrint A(即`自己的动态库SDK`.framework里的代码)
	2. `App`调用的实际上是类对象RSStaticPrint B(即.app里的二进制代码)
- 经典应用：
【无法调起微信登录问题（微信登录通过Pod只能静态库形式集成）】

**【问题】**
	`自己的动态库`和`APP`同时集成微信登录的静态库，导致Appdelegate的回调无法进行。因为微信初始化和回调在Appdelegate，而微信登录调用位置在`自己的动态库SDK`，由于Appdelegate（即`App`的位置）和自己的动态库SDK用的不是同一个类对象，所以由于未初始化，无法调起微信登录
	
**【处理】**
	`动态库`直接集成微信登录的静态库，`App`不要静态集成

### 情况二：第三方动态库（比如RSStaticPrint）同时被APP、自己的动态库SDK集成嵌入
结论：不会有问题，实际上都是同一份，即`*.app/frameworks/*.framework`的这份

### 进一步探究(选看)
[Using Firebase from a framework or a library](https://github.com/firebase/firebase-ios-sdk/blob/master/docs/firebase_in_libraries.md)
