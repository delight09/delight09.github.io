---
layout: post
title: 基于大山游戏的直播弹幕互动点歌系统
categories: project
tags: [livestream, mountain, autohotkeys, script, windows]
image: /assets/mountain-the-game-notation-player/feature.png
description: 讨论了首次接触游戏类直播项目时遇到的问题和对应解决思路。绘制了一张用户故事概念图。在文末介绍了项目的部署步骤和部署完成时的效果。
---

## TL;DR

讨论了首次接触游戏类直播项目时遇到的问题和对应解决思路。绘制了一张用户故事概念图。在文末介绍了项目的部署步骤和部署完成时的效果。

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
-- 检查SQLite数据库是否被更新
      |
      |-> 读取弹幕内容（SQL查询）-->-- 检查内容是否符合弹幕命令规范（正则表达式匹配）
                                      |
                                      |-> 更新status.txt -->-- 调用其他模块 ...

{% endhighlight %}

## 四、对歌单的处理

虽然对弹幕的读取使用了SQLite数据库，但这并不意味着要用数据库抽象出播放队列。项目中使用一个文本文件`playlist.txt`实现简易的歌单操作。

在项目设想时，对于处理歌单的方式实则让人困扰。一方面，我们能将这部分功能拆分成一些独立函数，以调用的方式在处理弹幕模块中处理。但如果这样实施，**处理歌单**和**过滤弹幕**
两模块将会不可避免变成紧耦合的状态（共用函数，函数间调用）。虽说这个项目不大，但是这是一个设计时的大忌讳。最终项目中，我们将处理歌单设计为一个独立于弹幕处理的[守护进程(daemon)][wiki-daemon]。

那么对于独立运行的处理歌单进程，如何确保自身只有一个实例在运行呢（不能同时播放两个乐谱）。我们用到了最简单、最容易实现的[资源锁][wiki-mutex]机制。当锁文件存在时，
进程将认为自身正在播放歌曲，不进行操作。在锁不存在时，将从睡眠中唤醒，开始处理歌单。


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
而AHK脚本编写的乐谱处理脚本也需要调用这个bash语言工具脚本，所以就诞生了`helper_make_txt_palyer.bat`辅助批处理脚本，用以解耦更新界面与乐谱处理逻辑。

而守护进程的引导，采用“弹幕分析”进程派生出“歌单处理”子进程的策略。一方面方便控制台直接中断进程执行，另一方面使得在杀死弹幕分析进程时能收割项目所有进程。

## 七、从零开始的直播部署

通过以上篇幅，我们讨论了项目中解决问题的思路，这个章节将介绍如何安装到运行这个互动系统。

首先，安装依赖工具`AutoHotKey`，`Cygwin`，`B站直播姬(livehime)`，Cygwin平台上的[`sqlite3`][cygwin-package-sqlite3]包。工具的安装请自行参考网络或各官方教程。
此外，项目中用到了开源内存盘工具[`ImDisk`][link-imdisk]，因为项目大量用道纯文本动态元素，将文本文件存储在内存盘，将大大加快元素更新效率，同时还能减缓磁盘消亡速度。

0. 下载项目代码

因为项目具有实验性质，并**没有打算长期维护**。代码存放于[`delight09/gadgets`][github-delight09-gadgets]仓库下，`livestream/mountain_the_game/client`文件夹内。

> 文件夹名为client的原因，是因为立项之前曾做过一些场景动态元素的实验。身为client的直播推流主机会向另一个API服务器请求获取元数据。究其原因，
主要是在Windows下部署ruby环境太痛苦所致。详细说明可参见父目录的*metadata.yml*文件内容。

将整个**delight09/gadgets**仓库下载并解压到本地。如下说明中将默认Cygwin平台的安装路径为`C:\cygwin64\`，同时用户名为`dummy`。

使用鼠标拖拽操作，将`client`文件夹粘贴至`C:\cygwin64\home\dummy\`目录下（或是一个熟悉的地方，方便以后启动进程）。

1. 挂载内存盘

启动ImDisk工具的**RamDisk Configuration**，将大小设置为最小的`64MB`，盘符设置为`R:`，其余选项保持默认，点击确定。

2. “导入”直播姬场景配置

我们知道OBS有它的场景配置scenes.xconfig。作为前职业运维，对于能导出配置的程序，自然得会在使用时带有三分敬意。

很可惜的，虽然直播姬(LiveHime)运用了OBS作为组件，可当前1.12.xx版本并不能导出配置。在进行了若干次网络搜索无果后，决定主动出击。
一个能在退出后仍“记得”上次退出时配置的工具，一般都会在本地留有某种格式的配置文件。果不其然，其配置的位置是
`%localappdata%\bililive\User Data\<bili-userid>\untitled.json`。

> 请注意，在1.12.xx版本的LiveHime使用过程中，场景配置文件仅在退出时进行持久化存储。换句话说，如果LiveHime不正常退出，那么你在启动后
对场景做出的更改都会消失。

项目中附带了一个演示用场景配置文件`livestream/mountain_the_game/untitled.json`，在直播姬(LiveHime)没有运行时，将项目中的配置覆盖到
`%localappdata%\bililive\User Data\<bili-userid>\untitled.json`。

> 这里我们推荐首先启动一次LiveHime，让其自动生成一个默认配置，完全退出LiveHime。再进行覆盖操作，使得导入过程更明了。

#### 导入后的场景布局如是

![Project scenes template](/assets/mountain-the-game-notation-player/scenes-template.png)

3. 初始化环境

首先启动大山游戏，直播姬(LiveHime)

打开Cygwin控制台，需要输入的指令和对应解释如下

{% highlight shell %}
cd client  # 进入项目目录
mkdir /cygdrive/r/txt # 在内存盘创建目标文件夹

./make_txt_librarylist.sh  # 生成曲库列表文件

# 生成点歌系统使用说明
echo -e '发送弹幕: \n 大山大山，演奏 + 关键词\n进行点歌' >/cygdrive/r/txt/usage_explain.txt
{% endhighlight %}


4. 启动互动点歌系统进程

首先需要修改配置。项目中对于LiveHime本地的数据库路径是[硬编码][wiki-hardwired]的。请在启动脚本前修正`env.conf`配置文件中的对应内容。

> 项目中脚本文件均使用能被Cygwin正常读取的[LineFeed][wiki-newline]换行符，不建议使用Windows记事本打开编辑。

然后就是“一键”启动点歌系统

{% highlight shell %}
./daemon_danmaku_checker.sh

{% endhighlight %}

> 此脚本不会退出，换句话说这个打开的控制台就处于被占用，不能接受输入的状态，这是正常状况。

#### 正常运行的点歌系统

![working shot](/assets/mountain-the-game-notation-player/working-shot.jpg)

## 后记：

这是第一次在Windows下完成项目，一共用了三种语言（AHK脚本，批处理辅助脚本，bash脚本），合计约600行代码。也是第一次针对直播内容的项目。

可能因为出于震惊的原因，促使我急于着手一个直播项目的缘由，是在参加某游戏公司的面试时，前台的姑娘在讨论下班后直播内容云云，那一刻我仿佛听到了晴天霹雳。
什么!?!?**居然**连“可能会成为同事”的前台姑娘都在做直播了，我**居然**连直播都没看过没体验过，我可**不是**从山洞里刚踏入文明的野人。
甚至于面试后给的Offer，也不及这一事实带来的强烈感官冲击。

经过这个项目，不难发现直播姬(LiveHime)的当前版本(1.12.xx)在处理文本文件内容更新时会有错误发生。要么不能即使发现更新（两次以上对文件的操作后才展现实际内容），
换句话说同一个函数的执行结果能展现的不一样。

此外，在展示场景文本元素时，还遇到了没有设想到的“*现实世界的文字长度*”问题。简单来说就是*nix平台下，那套文字对齐的方案在这里不适用，因为一个中文字抵得上4个空格元素宽度
。项目中通过在`make_txt_librarylist.sh`中使用了定制函数，在一定程度上真正实现了中英文混合下的文字长度计算。有兴趣的朋友可以查看对应源代码。

从来没有在Windows下通过Cygwin完成过项目，因为cygwin64位BASH环境有个[“知名”的变量溢出BUG][so-cygwin64-glitch]一直没有得到解决（自2017年上半年被发现）。
不过再怎么说，对于文字内容的处理，我对`sed`类*nix工具的信任要远远超过**任何**语言的字符处理函数。

项目至此是告一段落了，除了在真正实施过程中被砍掉的无数设想，这个项目依旧有很多不足的地方。
脚本必须在文件所在目录启动，因为脚本间工具的调用路径都是写死的。在使整个系统正常工作后，直接就开始了这篇总结文章。至于现在也不想再改了，还有太多事排队等着处理。
比如目前没有想到解决方案的：一旦打开互动点歌进程，整个电脑等于是被互联网绑架了（参考[生活大爆炸S01E09的Remote Control片段][bili-tbbt-cut]）！LOL！！1

## FYI

- AHK Tutorial -- [AutoHotkey Beginner Tutorial][ahk-tut]

- 知乎 -- [如何获取斗鱼直播间的弹幕信息？][zhihu-danmaku]

- AHK Forum -- [Switch to or open application][ahk-forum-focuswindow]

- Steam Community -- [优秀的钢琴软件《山》的乐谱分享][steamcomm-notations-share]

[ahk-tut]: https://autohotkey.com/docs/Tutorial.htm
[zhihu-danmaku]: https://www.zhihu.com/question/29027665
[ahk-forum-focuswindow]: https://autohotkey.com/board/topic/107450-switch-to-or-open-application/
[steamcomm-notations-share]: https://steamcommunity.com/sharedfiles/filedetails/?id=724185952
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
[wiki-daemon]: https://en.wikipedia.org/wiki/Daemon_(computing)
[wiki-mutex]: https://zh.wikipedia.org/wiki/互斥锁
[cygwin-package-sqlite3]: https://cygwin.com/cgi-bin2/package-cat.cgi?file=x86_64%2Fsqlite3%2Fsqlite3-3.21.0-1&grep=sqlite
[link-imdisk]: https://sourceforge.net/projects/imdisk-toolkit/
[github-delight09-gadgets]: https://github.com/delight09/gadgets
[so-cygwin64-glitch]: https://stackoverflow.com/questions/44606606/cygwin-bash-script-goes-wrong-on-pipes-when-sourced
[bili-tbbt-cut]: https://www.bilibili.com/video/av13786919/
[wiki-newline]: https://en.wikipedia.org/wiki/Newline
[wiki-hardwired]: https://en.wikipedia.org/wiki/Hardwire