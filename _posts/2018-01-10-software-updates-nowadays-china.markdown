---
layout: post
title: 由长虹电视浅析中国软件业升级迷思
category: thoughts
tags: [software-update, feedback, production]
image: /assets/software-updates-nowadays-china/feature.jpg
description: 博主升级了家中的长虹智能电视，没想到用户体验变得极差。从各角度探讨中国现如今软件升级带来的不便之处，简单分析原因。最终给用户一些应对升级迷思的思路。
---

## 引子

近来充实本地NAS资源时，捣鼓了些*基于NAS服务的流媒体点播*实验。平台有二，KindleFire2015特惠版和长虹电视42Q1N。以下文字将分别用**ford**和**q1n**指代这两种设备。

ford上跑的是由XDA社区某大牛维护*LineageOS*第三方ROM，基于安卓5.1。q1n上跑的是官方ROM，基于安卓4.2。

由于q1n平台并没有使用[虚拟SD卡机制][android-virtual-sdcard]，导致Kodi16会出现[`Waiting for external storage`][kodi-forum-wfes]启动错误。

瞥到电视机右上角有一个系统版本号**V1.00080**的升级提示，单纯想着“说不定新版本把问题解决了”，就开启了长达一天的灾难之旅...


## 长虹电视怎么了

#### 升级后基本功能残缺

升级后无法读取U盘，无法连接无线网络。根据[官方社区人员的解答][wayback-ch-offical-resp]，重置为出厂状态后问题解决了。

#### 外部存储不可用

任何关于外部存储的操作都变的极慢。从U盘中安装Kodi16居然历时**30分钟**仍处于**正在安装**的状态。这里提前提一下，最后成功降级了系统。
同一个U盘，Kodi16安装包仅用少于1分钟的时间就完成了安装。

真不知长虹官方是怎么厚着脸皮在[更新通告][wayback-ch-changelog]中写上

> 深度优化系统内存管理，提升您的操控体验;

这种**欺骗性词汇**的。

简析升级后卡顿起因，说起来也很简单。

新系统内置的捆绑软件无法移除，在成功连接无线网后，受到[`互联网状态变更`][so-android-internet-event]的事件通知后，统一开始了后台更新。
操作的路径是新插入的外部存储器--U盘。所以U盘本身的IO被吃满，这时再去安装U盘中的软件包无疑是痴人说梦，安装时长要等到天荒地老。
如果你不相信我的一面之词，那么读者可以自己实验下。在更新系统并连接网络后，观察一个小时内的U盘占用空间、文件的变化。

#### 内置软件无法删除

中国工信部于2016年制定《移动智能终端应用软件预置和分发管理暂行规定》，于2017年7月规定实施。

> 生产企业和互联网信息服务提供者应确保除基本功能软件外的移动智能终端应用软件可卸载。    
同时还要确保已被卸载的预置软件在移动智能终端操作系统升级时不被强行恢复。

新系统于2016年11月14日发布，很好的规避了违规的风险，同时让老机型充斥不可卸载的广告/合作软件，从老用户那儿赚上最后一笔。

其实在内置软件这方面，长虹还是嫩了些。应该向小米等前辈学习下，通过各种途径[让用户神不知鬼不觉得装回应用][archiveis-xiaomi-bbs]。

或者学阿里，[把**可选的**用户协议默认勾上][zhihu-zhifubao-anapy]，再把字体设置成芝麻大小。

## 这仅仅是安卓的问题吗

固然安卓平台有着“越用越卡”的历史遗留性迷思，但究其原因，无外乎硬件老化和[老油漆工问题][joelon-painter-problem]。
所以更新迷思不仅存在于安卓平台。

在问题被更细致的划分下，如产品的定位变化和社会浪潮也会影响到软件本身。

QQ、TIM与微信同属于腾讯旗下，针对不同人群需要进行产品迭代。最终使得QQ拥有了视频美颜、QQ秀等潮流属性。
微信显得更个人化，其朋友圈子产品可所谓中国的SNS（只能说是封闭的、残缺的Socal Network System，因为朋友圈没有陌生人的存在）。
TIM自QQ而来，接下TM的商务大旗，包括了所有的商务功能，还能贴心将长期没有联系的用户归类（请自行思考腾讯是如何区分**长期没有联系的**联系人）

又比如现在的视频站点，为了确保正版资源能留住用户，在新版本的*优酷客户端*中[“阉割”了原先版本中功能良好的Kux格式转码功能][zhihu-youku-kux]。
使得部分用户必须在网上寻找特定版本的老客户端，实现视频文件的解密。

而这些外部原因使**“软件升级”**带来的**恶果**，无外乎都是**由最终用户**来**承担**。

## 是升级软件这件事做错了吗

如果抛开现如今常见的“热升级”（或叫*强奸型*升级）不谈，软件升级本身并没有过错。

升级能帮助修正已有的程序错误（BUG）、提升软件的安全性，定期升级软件能更好的享受软件公司提供的新服务，形成一个良好的用户生态系统。

但同样的，软件升级不可避免的涉及到人力的付出。这也是为什么你会发现大多数单机、主机游戏的版本号通常很小，因为在多重QA的严格把关后，升级的意义较小。
同时的，格斗或是MOBA等竞技游戏会有定期的更新。这不仅是在设计时某角色属性失衡，从而需要调整。更因为其长期经营的需要，我们能预见在这类游戏更新后，关注度、用户活跃水平的提高。

## 到底谁错了

我们拿iOS平台举个例子。

iOS平台是一个较为封闭的移动系统平台。集中统一、不可替换的软件中心*App Store*造成了应用软件不可回归老旧版本的事实（抛开越狱不谈）。

**既然**软件升级其本身并**不存在问题**，**加以**iOS被设计为不能安装指定版本的**事实**，**那么**iOS平台为人诟病的`应用软件“强制”升级`**不能作为弊端**。

可事实真的如此吗？

如果读者放眼各活跃的社交平台，会发现*iOS不能安装指定版本*成为不选择购买苹果手机的一大原因。
那么这种用户心理到底是怎么造就的？

这里我们给出几张**V1.00080**系统q1n智能电视的使用界面照片。

![changhong sowrong 1](/assets/software-updates-nowadays-china/sowrong1.JPG)

![changhong sowrong 2](/assets/software-updates-nowadays-china/sowrong2.JPG)

![changhong sowrong 3](/assets/software-updates-nowadays-china/sowrong3.JPG)

我们能在统一的“Windows Metro“风格中，观察到“**VIP**”、“**钱**”、“**会员**”、“**买**”、“**优惠**/**赠送**”等重复出现的关键词。
按照一般的理解来说，这些“引导性”词汇不该出现在电视平台。但事实既是如此，我们只能认为是软件公司（别忘了和投资软件公司的股东企业）被金钱蒙蔽了双眼。

错只在软件公司，由利益导向的思维全然不顾[用户的反馈][wayback-ch-00080sucks]。员工冲着KPI、企业看着年度报表总数。
说难听一点，*产品卖一台是一台*这种机械性逻辑，是不能做出好产品、拉拢用户群的（除非你的用户都拥有[“爵士人生”][sohu-huawei-ecc]般的信仰）。

## 我们该怎么办

对于安卓平台，能Root就用“钛备份专业版”等应用冻结捆绑安装的应用。

对每个频繁使用的软件，记录软件版本，备份安装包（或输出APK）到NAS存储空间。
主动、手动关闭每一个频繁使用的应用内更新设置。

对于智能电视的选择，经历了这些坑人的旅途，总结个人经验如下

1. **避免使用蓝牙技术的遥控器**，虽然红外线需要“对准”才能收到信号，可别忘了红外信号有*廉价的*转录设备，而蓝牙遥控器没有。q1n使用蓝牙遥控器。

2. 如果可以的话，请带上U盘，**在商场内尝试安装必要的应用**（如Kodi），并记录当时系统的版本号。q1n越升级越卡、捆绑的软件也越多。

3. 如果使用无线方式接入互联网，请确保电视**支持5G频段无线网络**。5G的实际体验比2.4G频段快约4倍。q1n只支持2.4G网络。

4. 请**不要选择长虹智能电视**。参考长虹论坛的用户反馈，[来源1][wayback-ch-noroot],[来源2][wayback-ch-noroot4chiq],[来源3][wayback-ch-alios-sucks],来源4即本文。

## FYI

- HN -- [iOS APP STORE 禁止热更新][hn-apple-reject-rollout]

- Quora -- [Why are software updates Important?][quora-why-updatesw]

- Wikia -- [War of Warcraft-Blue post][wikia-wow-bluepost]

长虹ChiQ智能电视42Q1N，机芯型号`ZLM50HiS`。网上可下载到的最老完整刷机包为`V1.00020`，刷机方式为格式化U盘为FAT32格式，复制刷机包到U盘。
将U盘插入任意USB插口，插上电源，长按机体侧面电源键，直到BL自动识别U盘进入刷机模式（期间会自动重启一次）。

[hn-apple-reject-rollout]: https://news.ycombinator.com/item?id=13817557
[wayback-ch-offical-resp]: https://web.archive.org/web/20180111140814/http://bbs.changhong.com/thread-152596-1-1.html
[wayback-ch-noroot]: https://web.archive.org/web/20180111135733/http://bbs.changhong.com/thread-87979-1-1.html
[wayback-ch-noroot4chiq]: https://web.archive.org/web/20180111135837/http://bbs.changhong.com/forum.php?mod=viewthread&tid=81219&page=1
[wayback-ch-alios-sucks]: https://web.archive.org/web/20180111140046/http://bbs.changhong.com/thread-88786-1-1.html
[wayback-ch-00080sucks]: https://web.archive.org/web/20180110160937/http://bbs.changhong.com/thread-135280-1-1.html
[wayback-ch-changelog]: https://web.archive.org/web/20180110143551/http://bbs.changhong.com/forum.php?mod=viewthread&tid=3855&page=1
[archiveis-xiaomi-bbs]: http://archive.is/C6bwg
[zhihu-zhifubao-anapy]: https://www.zhihu.com/question/265015710
[joelon-painter-problem]: https://www.joelonsoftware.com/2001/12/11/back-to-basics/
[zhihu-youku-kux]: https://www.zhihu.com/question/51792949/answer/128682253
[quora-why-updatesw]: https://www.quora.com/Why-are-software-updates-Important
[wikia-wow-bluepost]: http://wowwiki.wikia.com/wiki/Blue_post
[sohu-huawei-ecc]: http://www.sohu.com/a/135254641_483686
[android-virtual-sdcard]: http://android-revolution-hd.blogspot.ca/2013/03/virtual-sd-card-on-android.html
[kodi-forum-wfes]: https://forum.kodi.tv/showthread.php?tid=208089
[so-android-internet-event]: https://stackoverflow.com/questions/3767591/check-intent-internet-connection