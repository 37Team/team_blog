---
title: iOS之模块化
date: 2021-2-19 12:43:06
category: iOS
tags: 
- 模块化
- 组件化
- 解耦
- JLRoutes
---
### 背景
当团队发展到一定规模，各业务之间相互影响问题剧增（合作成本变高），就开始进行模块化之路。
- 模块化的说法：
更准确的说法是模块化，而不是组件化。
1. 模块化：是横向划分，各业务模块之间有依赖关系，往往通过路由器解耦。
2. 组件化：是纵向划分，更多的是基础组件，各组件相互独立。
### 原理
主流都是通过路由器进行解耦，通过CocoaPods、git进行业务模块分割，对应技术框架有Mediator、JLRouter、CRN等。
### 路由
- 我们先看看目前的状态，引用一张图说明：
  {% asset_img  1602580864719.png 路由使用前 %}

  没错，就是相互引用，耦合很严重。我们想做什么？VC都不要依赖其它的VC。引用一张图说明：
  {% asset_img  602580962971.png 路由使用后 %}

我们就希望VC就只引入中间类（比如Router）就好了，不需要知道其它的控制器。这样是不是很爽？让我们看看怎么设计。

我们需要通过中间类跳转，那么A->中间类->B的过程：
1. A->中间类：必然有参数的传递（商品ID、闭包等），也包含B所对应的标识（类名、路径、其它标识）。
2. 中间类->B：通过获取前面拿到的信息，我们根据B的标识，对应到正确的控制器，获取到界面所需参数，并通过闭包，回调给A。

### 预期结果
我用路由就可以模块化了吗？不行，你代码写一块，可以确保不会改到其它业务模块吗？答案是否定的。所以我们需要通过CocoaPods、git进行管理。我不用CocoaPods，可以进行管理吗？也可以，比如Workspace管理+git形式，RN+git形式等都可以。

### JLRoutes的二次封装思路
{% asset_img  1613707505440.png 二次封装思路 %}
初步的设计思路，所以还有很多没考虑到的地方。仅供参考。

### 扩展
想深入了解的，还是需要了解一下主流的一些方案。
1. [JLRoutes](https://github.com/joeldev/JLRoutes)方案
2. [在现有工程中实施基于CTMediator的组件化方案](https://casatwy.com/modulization_in_action.html)
3. [iOS组件化——蘑菇街案例分析](https://www.jianshu.com/p/ec38c1ee5a2e)

### 参考资料
1. [iOS 组件化-路由解耦思想 JLRoutes 实战篇(一)App内控制器跳转](https://www.jianshu.com/p/c1714707c065)
2. [iOS组件化——蘑菇街案例分析](https://www.jianshu.com/p/ec38c1ee5a2e)
3. [JLRoutes](https://github.com/joeldev/JLRoutes)
4. [JSDRouterGuide](https://github.com/JerseyCoffee/JSDRouterGuide)
5. [iOS应用架构谈 组件化方案](https://casatwy.com/iOS-Modulization.html)
6. [在现有工程中实施基于CTMediator的组件化方案](https://casatwy.com/modulization_in_action.html)
7. [手把手教你搭建cocoapods私有仓库](https://www.jianshu.com/p/9e6fd79294e4)
