---
layout: post
title: 基于大山游戏的直播弹幕互动点歌系统
categories: project
tags: [livestream, mountain, autohotkeys, script, windows]
image: /assets/mountain-the-game-notation-player/feature.png
description: 
---

## TL;DR

TODO copy description

## 引言

[大山(Mountain)][wiki-mountain]是一款2014年发行的模拟类游戏。作者制作这款游戏旨在表现事物演变和变迁更迭的过程，没错的，这款游戏是真的在模拟一座大山的四季的演变。
除了通常将其作为屏幕保护程序外，我们还能发现在游戏内进行击键，能弹奏出各色音阶。

在网络上能找到不少广为好评的乐谱，这需要流畅、有节奏的键入对应按键演奏出曲调。排除[音游][wiki-music-game]大拿们能自在[享受沉浸式游戏体验][bili-music-game-guru]外，
如果能通过某种方式将乐谱自动带有节奏的键入游戏，相信会是不错的体验。

现在已经是2017年末，已经自“[2016直播元年][sina-china-livestream-year]”开始的第2年。后知后觉的博主直到最近才切身体验了直播产业，
某天进入一个24小时宠物猫直播间，第一次尝试直播间的点歌系统，感觉是一种很奇妙的互动体验，就会希望能上手制作一个弹幕互动的项目。

在项目初期，简单考虑到[游戏运行效率][linux-unity-perf]等因素后，最终决定在Windows平台下完成这个项目。

## 目录： 迫切需要解决如下几个问题

1. 将击键输入大山

2. 乐谱的存储

3. 读取弹幕并过滤

4. 对于歌单的处理

5. 更新点歌器界面

6. 流程的粘合

7. 从零开始的直播部署

## 一、将击键输入大山

AutoHotKey是Windows操作系统下的一款自动化类开源软件。通过其后缀名为`.ahk`的脚本文件，我们能进行编程得到提升日常办公效率的小工具，
如：[键盘实时击键窗口][ahk-script-onscreenkbd]，[组合键使当前窗口始终在最前方][ahk-script-alwaystop]，[可控制的快速鼠标连击][ahk-script-autoclick]。
本文的稍后篇幅，将使用**AHK**作为此工具以及AHK脚本语言的简称。

阅读相关[AHK官方文档][ahk-doc-winactivate]后，发现激活一个已知进程的窗口可以使用

> WinActivate ahk_exe Mountain.exe

之后只要[使用`Send {KeyName}`指令][ahk-doc-send]，就能把某一按键输入到游戏中。

### 解决这一问题的思路

{% highlight plain text %}
-- 使用AHK脚本激活大山窗口 -->-- 读取乐谱文件内容并发送击键

{% endhighlight %}

## 二、乐谱的存储

在找到把击键输入到游戏的方法后，我们自然会想到将乐谱以某种形式存储，并在需要的时候能进行调用。
项目中选择将乐谱存储到**满足特殊格式规定的**文本文件中。

#### 乐谱文件格式规定

1. 文件命名格式为`乐谱名.编曲人.mnt.txt`

2. 任何使用'#'字符作为第一个字符的行，作为注释使用（哪怕是第二个字符是'#'都不算注释行）。

3. 任何使用'%'字符作为第一个字符的行，作为节拍信息使用。'%'后可含有空格，文本内容必须为数字。

因为项目内的乐谱都是自网上摘抄所得，所以使用文件内的注释行表示乐谱的来源是非常有必要的。
同时每首乐谱其本身的节奏拍子大都不同，缺省的节拍参数自然是必要，可这并不能很好的还原歌曲原来的感觉，所以在文件中使用特殊符号标记了节拍信息，
在进行乐谱播放时作参考。

## 三、读取弹幕并过滤

因为个人主观的讨厌斗鱼tv，对其竞争对手们也不报有过多期待。所以使用了已有[Bilibili帐号的直播房间][bililive-room]。

在搜索引擎中使用关键词“**b站 弹幕 读取**”寻找一段时间后，找到几个Python写的直播弹幕获取方案，粗略看了下应该是解析[WebSocket][wiki-ws]的包装脚本。
正当决定安装python环境之前，找了其中几个方案试了下（比如[bilibili_danmu][github-bili-danmu]），结果无一例外的失败。于是打开直播网页，使用网页控制台，查看Network标签页，
发现[脚本中预设][github-search-livecmt]（写死）的弹幕通讯服务器(livecmt-1.bilibili.com)不在站点列表内（实验于2017年12月20日前后）。后多方查询，
应该是B站直播更改了弹幕服务方案。虽然尝试的几个方案都算是稳定开发中（2、3个月前更新），但**计划赶不上变化，变化改不了Dead line**，至此我们必须另寻出路。

B站直播有一个官方的工具，叫做[直播姬(LiveHime)][bili-livehime]。这个工具在开源圈内曾引起过[小风波][zhihu-livehime-gpl]，现在能从官方下载到1.12.xx版本。
目前国内直播平台都支持[RTMP推流][adobe-rtmp]这个方案，虽说我们有开源的[OBS(Opensource Broadcast Software)][link-obs]同样能满足直播推流，
但LiveHime工具有能记录房间弹幕并存储为[SQLite][wiki-sqlite]**未加密**数据库文本。不用说，无论B站直播服务架构怎么改，LiveHime官方工具总能通过本地数据库文件读取弹幕。 

分析弹幕内容，过滤出符合命令规范的指令的过程必不可少。我们在设计弹幕命令规范时最重要考量两点，

一是规范的**人性化**，尽力通俗易懂不容易输入错误。另一面是**高容错性**，在筛选关键词后还能保持命令形态的多元性。

项目的弹幕命令规范如是

> 敬语 + 标点或空格 + 指令 + 标点或空格 + 乐谱关键词 [ @ + 空格或没有空格 + 节拍数字 ]

代码中使用了遵守[POSIX RegExp][wikibook-posix-regexp]规范的正则表达式过滤弹幕指令。

此外，点歌器一般会有一个指令结果回显，用来通知用户弹幕指令已经被成功识别执行。在项目设立之初，第一反应是将脚本的控制台直接投射到直播画面。一方面是最为简单，
不需要额外的代码就能满足需要。可是在真正实施时，发现[mintty.exe][github-mintty]画面无法被正常捕获，被捕获的内容除了变成1x1的像素点。
于是在多方尝试后，决定使用最为基础的纯文本解决方案。

#### 纯文本解决方案

我们知道常用的直播工具都会用“场景”这个概念。通过预设的场景，我们能将多媒体元素，如图片、视频、文字、抓屏内容，布置到一系列的场景中，这样用户能看到一个结构化的展示画面。

如果遇到“抓屏”不能解决的元素，最简单的一种方法，是将格式化文本存储在一个文件中。由脚本程序改变文本内容后，直播工具自动检测到文件变化，实现场景元素的动态化。

项目中使用了名为`status.txt`的格式化文本作为对弹幕指令的“反馈窗口”。文本元素对比控制台方案还有一些杂项需要手动处理，如回显行长度的控制、文本行数的控制、
对于存在时间久远的命令进行淡出窗口的“动画”。这些额外的问题大都是文字处理工具能轻易处理的，值得一提的是对“文本行数限制”的处理中，项目中引用了另一个文本文件`backlog.txt`，
用来记录过程中所有的弹幕指令。现在想来这并不是最优雅、最节省资源的方案，但不能忽视在开发过程中**调错更容易**、**节省IO操作编写时消耗的脑细胞**、**方便后期审计弹幕指令**的优点。

### 解决这一问题的思路

{% highlight plain text %}
-- 检查sqlite3数据库是否被更新
      |
      |-> 读取弹幕内容（SQL查询）-->-- 检查内容是否符合弹幕命令规范（正则表达式匹配）
                                      |
                                      |-> 更新status.txt -->-- 调用其他模块 ...

{% endhighlight %}

## 四、对歌单的处理

虽然对弹幕的读取使用了sqlite3工具，但这并不意味着项目一定要用数据库抽象出一个歌曲列表。本项目使用一个纯文本文件`playlist.txt`简易实现存储读取歌单。

在项目设想时遇到了对于处理歌单方式的困扰，一方面我们能将这部分功能拆分成一些独立的函数，以函数定义方式在处理弹幕指令后调用。但如果这么考虑，那**处理歌单**和**处理弹幕**
两部分将会不可避免变成紧耦合的状态（共用函数，函数间调用），虽说这个项目不大，但是这是一个设计时的大忌讳。最终在项目中将其设计为一个独立于弹幕处理的常驻进程（或叫守护进程TODO)。

那么独立运行的歌单处理进程，如何能确保自身只有一个实例在运行呢（不能同时播放两个乐谱）。我们用到了最简单、最容易实现的资源锁（TODO wiki）机制。在锁存在时，
进程将认为自身正在播放歌曲，而在锁不存在时，将从睡眠中唤醒，开始处理歌单。


### 解决这一问题的思路

{% highlight plain text %}

 -- 检测激活信号
       |
       |-> 检测资源锁
              |
              |-> 更新player.txt相关内容 -->-- 调用其他模块 ...

{% endhighlight %}

## 五、更新点歌器界面

点歌器的界面设计为四行文本，也使用了上文所述的**纯文本场景元素**方案

#### 点歌器界面文件内容（player.txt)

第一行 - **当前播放**歌曲信息

第二行 - 当前**歌曲**进度，歌曲的节拍**参数**

第三行 - **实时**演奏的**乐谱**数据，以乐谱文件行作为最小单位

第四行 - **下一首**歌曲信息

点歌器界面文件的操作难度更高，主要原因并不在于每行都有特定内容，而是在对于*下一首*内容的更新。

这个项目设计的原则是尽可能的“优雅”，也就是*低耦合*、*高容错*、*高易读*，但这在处理界面时遇到了史无前例的困难。究其原因，主要因为弹幕点歌是一个**异步更新操作**，
而处理歌单是一个**不可中断的顺序操作**。

详细来说，*下一首*的部分可由异步的*弹幕读取*进程更新，同时又必须能被顺序执行的*歌单处理*进程改动。同一个功能实现在两个进程代码中，是一大忌讳，一方面后期对函数的维护
会变的十分复杂，另一方面，这时常会造成死锁的情况发生（在本项目中不太可能发生）。

至此我们看到一个**高度格式化**的文本文件，将经由三个不同的脚本（甚至是不同语言的脚本）进行更新，如果将更新文本的函数分别实现，那么无疑会对代码的更迭维护造成影响。
所以引入一个单独的工具脚本`make_txt_player`，进程都能调用这个工具，完成格式化文本`player.txt`的结构更新操作。


### 解决这一问题的思路

{% highlight plain text %}
              -- 更新player.txt当前曲目信息
             /
 -- 接受参数  -- 更新player.txt实时乐谱
             \
              -- 更新player.txt下一首信息

{% endhighlight %}


## 六、流程的粘合和具体实现

在解决上述问题的过程中，刻意的回避了几个模块间交互。现在让我们整理下思路，整个互动点歌系统的流程，可以大致解释成一个[用户故事][wiki-userstory]

![A valid danmaku's travelling, cute user story diagram -- inspired by steins;gate](/assets/mountain-the-game-notation-player/user-story.png)

综合上述同时考虑到运行环境为Windows系统。项目中选用已有22年开发历程的[Cygwin平台][wiki-cygwin]并结合[BASH脚本][wiki-bash]作为脚本语言，究其原因，请参考这则[无根的根寓言故事][faqs-rr-tenthousand]

读取弹幕进程和歌单处理进程之间的通信，我们使用[POSIX信号规范][wiki-signal-posix]中的`SIGUSR1`信号。因为点歌器界面是通过调用`make_txt_player.sh`更新的，
而AHK脚本编写的乐谱处理脚本也需要调用这个bash语言工具脚本，所以就诞生了`helper_make_txt_palyer.bat`Windows批处理脚本，用以将更新界面的逻辑和乐谱处理脚本解耦合。

## 七、从零开始的直播部署

以上篇幅大都讲的是项目解决问题的思路，这个章节我们将介绍如何使这个项目运行起来。

首先，需要安装`AutoHotKey`, `Cygwin`, `livehime` 和 Cygwin平台上的`sqlite3`包。依赖工具的安装请自行参考网络或官方教程。

此外还需要将脚本代码下载到本地，推荐将文件解压至cygwin家目录，或一个熟悉的地方。


## 后记：

三种语言600行代码。

BiliHive 读文本文件有Glitch，同一个函数的显示结果能不一样。

开脚本就不能用电脑，等于被远程协助了

为什么开始直播，游戏公司面试时，前台妹子讨论晚上直播

从来没有在Windows下通过cygwin完成过项目，因为遇到过BUG。

## FYI

- AHK script guide

- Wikipedia - [DMZ][wiki-dmz]

- Bookmarks


[wiki-mountain]: https://en.wikipedia.org/wiki/Mountain_(video_game)
[wiki-music-game]: https://zh.wikipedia.org/zh-hans/%E9%9F%B3%E6%A8%82%E9%81%8A%E6%88%B2
[sina-china-livestream-year]: http://tech.sina.com.cn/zl/post/detail/i/2017-04-01/pid_8510390.htm
[linux-unity-perf]: https://www.youtube.com/watch?v=mISF1pRtjWU
[bili-music-game-guru]: https://www.bilibili.com/video/av975843
[ahk-script-alwaystop]: https://github.com/delight09/gadgets/blob/master/code-snippets/AutoHotKey/alwaysTop.ahk
[ahk-script-autoclick]: https://github.com/delight09/gadgets/blob/master/code-snippets/AutoHotKey/auto_click.ahk
[ahk-script-onscreenkbd]: https://www.autohotkey.com/docs/scripts/KeyboardOnScreen.htm
[ahk-doc-winactivate]: https://autohotkey.com/docs/commands/WinActivate.htm
[ahk-doc-send]: https://autohotkey.com/docs/commands/Send.htm
[bililive-room]: http://live.bilibili.com/344428
[wiki-ws]: https://en.wikipedia.org/wiki/WebSocket
[github-bili-danmu]: https://github.com/lyyyuna/bilibili_danmu
[github-search-livecmt]: https://github.com/search?q=livecmt.bilibili.com&type=Code&utf8=%E2%9C%93
[zhihu-livehime-gpl]: https://www.zhihu.com/question/47335914#answer-37660782
[adobe-rtmp]: http://www.adobe.com/devnet/rtmp.html
[link-obs]: https://obsproject.com/
[bili-livehime]: https://live.bilibili.com/liveHime
[wiki-sqlite]: https://en.wikipedia.org/wiki/SQLite
[wikibook-posix-regexp]: https://en.wikibooks.org/wiki/Regular_Expressions/POSIX_Basic_Regular_Expressions
[github-mintty]: https://github.com/mintty/mintty/wiki/Tips
[wiki-userstory]: http://localhost:4000/project/mountain-the-game-notation-player/
[faqs-rr-tenthousand]: http://www.faqs.org/docs/artu/ten-thousand.html
[wiki-cygwin]: https://en.wikipedia.org/wiki/Cygwin
[wiki-bash]: https://en.wikipedia.org/wiki/Bash_(Unix_shell)
[wiki-signal-posix]: https://en.wikipedia.org/wiki/Signal_(IPC)#POSIX_signals