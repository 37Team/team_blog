---
title: Bugless异常监控系统 （iOS端）
date: 2020-11-05 18:59:06
category: iOS
tags: 
- 业务异常 
- 崩溃
- Bugless
- 闪退捕获
---
# 一、引言
目前部分线上的业务，缺少客户端的异常错误的线上监控、告警与异常数据聚合并沉淀的平台。也无法在多维度进行异常数据的对比，使得收集应用信息和收集崩溃日志变得日益迫切。
Bugless定位于从线上问题追踪的视角出发，检测代码异常，通过回溯问题，从而解决代码本身问题。它的作用如下：
> 实时监控SDK业务异常
汇总包体崩溃排重与聚合后的数据
统计影响设备数
上报崩溃日志
收集iOS系统向上兼容性问题
监控客户端请求的网络问题
<!-- more -->
上线Bugless意义重大，它支持不同项目不同端的异常上报告警，及时通知相关人员处理，减少损失。也支持后台聚合错误信息数据，分析历史异常数据，协助开发人员优化对应项目进行优化。
# 二、认识崩溃和异常
> 认识Bugless之前，让我们从三个层面来认识为什么会出现崩溃和异常，以及如何应对。
## 1、应用层面
应用崩溃（crash）出现原因，即违反iOS系统规则产生crash的三种类型
### 1) 内存引发闪退。
有以下几个方面，无效的内存访问，内存访问越界，运行时方法调用不存在，解引用指向无效内存地址的指针，跳转到无效地址的指令等
### 2) 响应超时
启动、挂起、恢复、结束等事件响应不及时
### 3）触发Watchdog机制
Watchdog是为了防止一个应用占用过多系统资源，如果超出了该场景规定的运行时间，“看门狗”就会强制kill掉这个应用，在crashlog 会看到 “0x8badf00d”的错误代码。
## 2、Bugless崩溃捕获流程原理
跟应用紧密相关的异常莫过于Objective-C抛出异常，也是我们最容易捕获到的一种异常。捕获此异常方法如下：

{% asset_img e7f821ad.png 获取崩溃异常的代码实现 %}
<center>获取崩溃异常的代码实现</center>
                                                             
{% asset_img 0bde2ef2.png 注册异常捕获函数 %}
<center>注册异常捕获函数</center>

> 接下来整理了三种iOS系统产生异常，并被系统捕获。只要监听系统开放的API即可监听。以下是其捕获流程图如下，

{% asset_img daf897a4.png daf897a4 %}

> Objective-C产生异常的表现形式如下，表前5列中的Invalid类型异常。除了Objective-C异常以外，还有两种异常。分别由Mach Exception Handler和POSIX signer handler捕获到，崩溃表现形式形如表中的SEGV_ACCERR类型。

{% asset_img 9241e1cf.png SEGV_ACCERR类型 %}

### 1）Bugless上报闪退堆栈
从数据全量收集出发，获取闪退的日志时机有两个：
* 第一时机
    闪退立即上报，但第一次可能因为进程被杀死而发送不成功。
* 第二时机
    是重新启动发现上次有闪退日志，进行上报。但如果用户不再次启动，可能就无法上传。
### 2) Bugless异常分析流程
拿到一份闪退日志，按如下步骤可初步定位出异常的类型。如下图所示：

{% asset_img aabb4f84.png 定位出异常的类型 %}
### 3）Bugless堆栈解析
>    按流程初略分析异常产生原因之后，如何定位问题所在位置呢？我们这时就需要用到崩溃堆栈解析工具。 
对比两款符号化工具Symbolicatecrash（命令行工具）和SymbolicateX（UI工具），
总的来看，两个工具都使用了相同解析关键工具atos。

解析过程为，首先遍历出属于‘cheng’这个主程序的全部内存地址，存储为addresses数组，再通过symbolicationCommand函数传入符号表dsym文件，架构armv7或arm64，以及loadAddress进行符号化，如以下代码示例：

{% asset_img d4b67857.png 符号化 %}
* Symbolicatecrash  
    使用到Xcode自带内存地址转函数堆栈命令atos。
* SymbolicateX
    SymbolicateX是第三方开源工具，基于它进行二次开发为的命令行解析工具XcheckSymb，可使用atosl替代atos工具，实现跨平台的日志解析，以达到不再依赖macOS系统及Xcode的xcrun，为将堆栈符号化作成通用的在线服务作铺垫。
    后续对解析工具的优化，将朝着解决堆栈解析效率低的问题出发：
* 一方面缩短解析时长；
* 另一方面引入批量异步解析和缓存重复堆栈机制。
## 3、聚合
崩溃标题
    主要根据偏移量进行区分。
    苹果官方聚合方案：使用AppBundleName 加内存地址，再加偏移量。
    例如 ：syios: 0f100afc000 + 8691804
  			新方案：Exception Codes 做标题，结合闪退线程中第一个有效偏移量, 

如下图所示日志中二进制文件名cheng所对应的第一个偏移量：+ 4437668

{% asset_img 167c14c1.png 偏移量位置 %}
<center>偏移量位置</center>


{% asset_img c2e11129.png 新方案标题 %}
<center>新方案标题</center>

* 堆栈聚合
根据去除堆栈变量后的hash值聚合。
聚合先过滤掉崩溃线程的内存地址、偏移量，再将文本做hash标签，按标签进行聚合，再按设备标示进行排重。以此种方法聚合堆栈由于iOS系统版本的不同堆栈md5值会有出入。（具体原因是，不同系统当前崩溃堆栈依赖库行数可能不同。）
过滤方法如下，
{% asset_img fbb03b7e.png 过滤方法 %}

正则过滤排除内存地址和偏移量正则条件如下：

{% asset_img 749353db.png 正则条件 %}
# 二、网络层面异常
> 1）能按分钟报告诸如找不到页面（状态码404）、服务不可用（503）网络异常等。
> 2）详细统计出，客户端请求超时次数，计算出超时请求设备的占比。
> 3）通过检查返回的数据是不是预期的JSON格式，监测是否出现域名劫持的情况。
# 三、服务器业务层面异常
> 通过对客户端网络请求的错误上报，实时上报SDK业务异常，可以方便的监测账号认证异常、下单应用内购买异常及发货异常。
# 四、告警
## 实时告警
> Bugless提供按分钟、每小时或按天进行错误累计并告警，一旦超过阀值就会通过企业微信进行告警

### 告警系统的结构图如下,
{% asset_img 9d2888c7.png 告警系统的结构图 %}
### 小助手告警消息示例如下，
{% asset_img 69141c9a.png 小助手告警 %}

# 五、Bugless
> 以上告警系统上线后，只能获得零散的告警信息，借助了bugless后台，可以满足我们对多维度进行异常数据的对比需求。
下面具体介绍Bugless后台做了些什么。
## 1、Bugless后台
Bugless后台统计出了业务异常：

{% asset_img 34e60542.png 业务异常 %}
<center>表 1自动生成账号密码错误</center>


## 2、Bugless接入应用案例
> 目前为止Bugless接入上线4款游戏，共接到有效告警三次。包括：
1)   研发下单商品ID错误
2)   苹果应用内购买服务异常
3)   手机注册重复请求率高
## 3、准确性
> 与苹果iTunes Connect的崩溃日志做统计数值对比基本吻合。
Bugless崩溃上报正确性验证（Bugless VS Xcode Organizer Crashes） 仅漏报2台设备，评估是闪退后没有再启动，没上报上来。同一处崩溃，苹果iTunes后台收集到61台设备闪退，Bugless收集到59台设备受影响。
如下图所示，

{% asset_img 11fea4bf.png XcodeOrganizer Crashes %}

{% asset_img 82b6498c.png XcodeOrganizer Crashes 2 %}

<center>表 2 XcodeOrganizer Crashes</center>

{% asset_img 0174d081.png Bugless后台日志详情 %}

{% asset_img 2469ca70.png Bugless后台日志详情2 %}

<center>表 3 Bugless后台日志详情</center>

{% asset_img d6689de7.png d6689de7 %}

<center>表 4 Bugless解析日志</center>

3、Bugless应用过程中存在的问题
> 在使用过程中也发现了几个问题，其中告警误报的情况时有发生。由于先期对阈值把握不足，阈值就调到足够低，这样不会放过绝大多数的有效数据样本。随着数据样本增加，告警的阈值逐步精确起来，误报情况将得到改善。
# 结束语
> 本次对Bugless项目的技术关键点的设计、开发和上线，可以看出该项目能持续有效的对苹果平台发行业务问题排查提供数据支撑。当然该项目仍有一个自身不断完善的过程。比如二次开发的符号解析工具，缺少了系统库函数堆栈信息，有待改进；另一方面崩溃日志解析性能有待进一步提升，减少用户等待时间。
随着业务的拓宽，Bugless也有了更多服务用户的机会。将Bugless推广到更多的业务领域，诸如联运SDK、海外业务等。同时提供埋点上报供研发使用，让游戏可以通过自建平台（非第三方平台）统计到用户的使用习惯，如有定制报表需求可提供一对一的技术支持，给更多用户带来便利。

# 附录

* 参考链接 - 异常堆栈字段说明
https://developer.apple.com/documentation/foundation/nsexception
https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/examining_the_fields_in_a_crash_report
https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/understanding_the_exception_types_in_a_crash_report
* SymbolicateX iOS/Mac 项目崩溃文件自动符号化工具
https://github.com/tomlee130/SymbolicatorX 