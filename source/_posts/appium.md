---
title: appium 入门参考
date: 2020-11-05 15:21:06
category: iOS
tags: 
- appium 
- iOS
- 自动化测试
---
## 前言
游戏发行业务中，对游戏进行测试是保证游戏质量重要的一环。传统人工测试的方法费时费力、容易出错，自动化测试才是更好的解决方案。`appium`则是自动化测试的优秀方案，新手可以通过官网的[开始文档](http://appium.io/docs/en/about-appium/getting-started/)快速入门。
本文定位为：新手阅读完开始文档的**第二篇文档**。重点介绍了appium方案与其他方案的**优缺点对比**，以及在**环境配置**、**原生控件查找**、**图片识别**方面的关键知识和常用问题解决方法。
本文**适合**只有单一iOS开发或者自动化测试背景的人员，阅读完开始文档，作为**辅助文档**阅读；**不适合**作为新手第一篇文档，或者已同时熟练掌握iOS开发、appium自动化测试的人员阅读。

## iOS UI测试方案对比
iOS的UI测试的技术方案首先有两个大的方向：原生方向以及跨平台的方向。简单表格对比如下：

| 方向   | 框架    | 编程语言           | 原生控件查找 | 图片识别 | 更新维护    | 开发体验 |
| ------ | ------- | ------------------ | ------------ | -------- | ----------- | -------- |
| 原生   | XCTest  | Objective-C、Swift | 最优         | **没有** | 最优        | 最优     |
| 跨平台 | appium  | Python、JS等       | 有           | 有       | 优          | 一般     |
| 跨平台 | airtest | Python             | 有           | 有       | 一般（iOS） | 优       |

<!-- more -->
### 原生
原生方向优势在于原生控件识别的速度以及准确度，以及框架更新维护的及时。劣势在于没有直接集成**图片识别**的功能，而图片识别是游戏的自动化UI测试关键的一环。因此原生方向是适合App的自动化UI测试方案，不适合直接应用于游戏。此外，原生方向还对使用人员有掌握iOS开发的要求，不便于我们（笔者是iOS开发）与测试人员合作开发。
### 跨平台
跨平台方向是实现iOS手游UI自动化测试更好的选择。此方向的具体实现有很多，表格举了两个较为典型的例子：`appium`以及`airtest`。
#### appium
appium是开源社区最为流程的移动UI测试框架，支持多种编程语言编写脚本。项目更新频繁，bug修复及时。功能方面，原生控件识别、图片识别都齐全。使用过程遇到的问题在社区中能较快找到解决方法。缺点在于appium的IDE等配套（指免费方案）不完善，且没有针对手游进行专门优化，实际使用需要自己实现较多的脚手架以及轮子。
#### airtest
airtest是网易公司专门针对手游优化的UI测试方案，使用Python语言进行编写脚本。IDE、中文文档等配套完善易用，同时也是免费的。缺点在于，**单就iOS而言**，核心框架 `iOS-Tagent`的代码并不开源，且在新版Xcode适配，bug修复等问题上都较慢。使用人员碰到问题往往比较被动，只能Github报issue然后等待官方修复。

综合考虑后，笔者选择了appium作为iOS手游UI测试方案。

## appium 环境配置
appium官方文档完善，请先行阅读[开始文档](http://appium.io/docs/en/about-appium/getting-started/)，本小节只补充一些关键点。

* * *

安装命令行版appium使用NPM即可，因为appium是基于node开发的。
`npm install -g appium`

* * *

此外，appium还有一个桌面版，使用Electron编写。从[Github官方仓库](https://github.com/appium/appium-desktop/releases/)下载安装包，直接安装即可。桌面版既可用于**启动appium命令行版**服务器，也可用于**控件查找调试**以及**自动录制生成脚本**，推荐新手安装。但桌面版本身并不是一个IDE，需要使用另外的IDE编写UI测试脚本并运行。
{% asset_img appium_desktop.png appium_桌面版 %}
* * *

安装完appium，重要的一步是确保`WebDriverAgent`工程的`WebDriverAgentRunner` target编译通过。找到工程的位置后，调整对应target的证书以及描述文件即可。
```shell
# 命令行版appium WebDriverAgent工程默认位置
/usr/local/lib/node_modules/appium/node_modules/appium-webdriveragent/WebDriverAgent.xcodeproj

# 桌面版appium WebDriverAgent工程默认位置
/Applications/Appium.app/Contents/Resources/app/node_modules/appium/node_modules/appium-webdriveragent/WebDriverAgent.xcodeproj
```
其他问题，[开始文档](http://appium.io/docs/en/about-appium/getting-started/)已经写得较清晰，不再赘述。

## appium 原生控件查找
下文所提`控件`一般都特指`iOS原生控件`，如`UIButton`、`UITextField`等。
控件查找主要应用于原生SDK界面的自动化操作，如输入账号密码、点击SDK的登录按钮等。进行自动化操作，重点就在控件查找；而找到控件以后与其交互，只需调用对应的click之类API，这个是相对直接简单的。

appium的原生控件查找的策略，笔者根据是否需要在iOS端提前适配，将其分为两种：**侵入式**与**非侵入式**。

### 侵入式查找策略
侵入式查找策略原理是iOS端的对应控件提前适配一个标识符`accessibility_id`，然后UI测试脚本凭借该标识符查找到对应控件。代码示例如下：

```Swift
// iOS端：Swift
loginBtn.accessibilityLabel = "login_vc_login_btn"
```
```Python
# appium脚本代码端：Python
driver.find_element_by_accessibility_id('login_vc_login_btn')
```
侵入式方案优势在于测试端的自动化脚本容易编写。而劣势显而易见是，得为众多的UI控件添加**唯一**标识符（如果两个控件标识符相同，情况会变得复杂），对开发人员来说比较麻烦，尤其是在同时开发维护多套SDK的时候。
此外，侵入式方案的查找效率往往会比下面介绍的非侵入式方案**更慢**，可以使用桌面版appium进行控件查找时间测试。
{% asset_img appium_time.png appium_time %}

### 非侵入式查找策略
非侵入式查找策略原理是通过**规则匹配**的方式查找控件，无需iOS端提前适配，且识别速度会**更快**。

#### ios-class-chain 上手使用
非侵入式查找策略有多种，但此处只介绍其中的集大成者`ios-class-chain`查找策略。先看代码：
```Python
# appium脚本代码
driver.find_element_by_ios_class_chain('**/XCUIElementTypeButton[`label == "登录"`]')
```
这是典型的使用场景，作用是：查找label属性等于登录的按钮元素，别的元素也能以类似的方式进行查找。`find_element_by_ios_class_chain`是appium提供的API。函数需要的参数， 本文称之为`selector`。

#### ios-class-chain selector分析
使用`ios-class-chain`查找策略的关键就在于编写`selector`，我们还是用上面的例子对`selector`进行拆解分析。
```Python
# selector
'**/XCUIElementTypeButton[`label == "登录"`]'
```
* * *
`**/` 符号放在了`selector`开头，作用是避免在根层级直接查找元素。此外，正如`ios-class-chain`名称本身所示，我们能**链式组合**出更精确更复杂的`selector`去满足查找需求，`**/`符号此时可能出现在`selector`中间。含义指后面的元素不是当前层级的直接子代（child），是间接子代（子代的子代，descendant）。
```
**/XCUIElementTypeCell[`name BEGINSWITH "D"`]/**/XCUIElementTypeButton
```

* * *

`XCUIElementTypeButton`是按钮元素在iOS的XCTest框架的枚举名称表示。查看[XCTest文档](https://developer.apple.com/documentation/xctest/xcuielementtype?language=objc)可以查看更多其他可用控件元素名称。

* * *

```
[`label == "登录"`]
```
方括号里面的表达式叫**谓词表达式**，是被查找控件的**约束条件**。此谓词表达式的含义是：**label属性等于登录**。除了表示相等的`==`运算符，表达式能用的运算符，还有逻辑表达式的 `AND`,字符串比较的`BEGINSWITH`等等。appium的[iOS谓词指南](http://appium.io/docs/en/writing-running-appium/ios/ios-predicate/)以及苹果的[谓词编程指南](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Predicates/AdditionalChapters/Introduction.html)作了更详细的介绍。

* * *

表达式左边使用了`label`属性。label一般是用户能**直接看到**的内容。如一个**文字按钮**文本写的是立即登录，那按钮label属性值就是`立即登录`。但如果这个按钮的内容不是文字，而是一张**图片**，那按钮label属性值是由**图片的文件名**经过命名规则的转换而来的。编写脚本的人员往往不知道图片文件名，这时候可以用桌面版appium调试工具查看最终的`selector`结果。

* * *

表达式左边还有一个属性也很常见：`name`。假如在 **iOS** 端已经给控件**提前适配**了标识符(参考侵入式查找策略一节)，名为：`ctrl_access_id`，谓词表达式就可写成：
```
[`name == "ctrl_access_id"`]
```
* * *


#### ios-class-chain 总结
appium的`ios-class-chain`查找策略会将`selector`转换成一系列苹果原生API（`XCUITest`）的直接调用，而不是递归地构建整个UI树，所以往往会比其他策略更高效。再加上此策略无需iOS端提前适配，综合下来，是笔者首选使用的查找策略。
它的劣势在于`selector`编写会比直接的标识符复杂一些。所幸我们只要掌握要一些常用规则，足以应付多数情况，而且桌面版的appium也能作为参考工具。

## appium 图片识别
appium能实现原生控件查找，是因为底层的`WebDriverAgent`框架把脚本命令转换了成原生`XCTest`框架的调用。而游戏引擎生成的游戏画面内容不能像原生控件一样去查找，因此就需要用到图片识别的技术。
appium的图片识别依赖于`openCV`，请先根据命令行引导安装好环境，否者在调用`find_element_by_image`API时会抛出异常。
图片识别值得讲的点不如原生控件查找的多。比较常用的一些配置和笔者碰到的问题例举如下，供大家参考：

* getMatchedImageResult 设置为True能保留查找到的图片的base64字符串表示，从而进行图片识别的debug。
* imageMatchThreshold 控制图片准确度阈值，默认值是0.4。实际使用发现经常会误识别，所以笔者一般会调高此值。
* fixImageTemplateScale 调整基准图片的比例。图片识别最终是转换成了屏幕坐标点。一般手动截图往往是是2倍或者3倍图，因此需要先调整图片分辨率比例，才能转换成正确的坐标点。这个配置理论上是用来自动调整比例的。但在编写本文的此时，这个配置实测有bug，笔者只能用自己的脚本另行处理。

## 总结
文章讲述了自动化测试方案对比、appium的环境配置、原生控件查找、图片识别的基础内容。原生控件查找一节重点分析了`ios-class-chain`策略的`selector`的结构。文章目的在于帮助读者更快上手使用appium，避免重复踩坑。而对于什么是自动化测试的最佳方案，控件查找的最佳实践等问题，不能一概而论，读者可结合实际情况与本文的优缺点分析进行选择。

## 参考链接

* https://appium.io/docs/en/about-appium/getting-started/
* https://appiumpro.com/editions/8-how-to-find-elements-in-ios-not-by-xpath
* https://github.com/facebookarchive/WebDriverAgent/wiki/Class-Chain-Queries-Construction-Rules
