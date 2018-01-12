---
layout: post
title: 无需cid的下载bilibili视频方案
categories: howto
tags: [bash, chrome, curl, archive]
image: /assets/always-working-bilibili-video-download/feature.gif
description: 不借助第三方工具和浏览器插件，无需提交个人帐号密码，借鉴传统的视频缓存思路。使用脚本分析浏览器的访问日志，并用常见工具下载视频资源。文末介绍了简单的视频合并方法，并提供演示视频作为整个下载流程的参考。
---

## TL;DR

不借助第三方工具和浏览器插件，无需提交个人帐号密码，借鉴传统的视频缓存思路。[使用定制脚本][github-gadgets-script]分析浏览器的访问日志，并用常见工具下载视频资源。
文末介绍了简单的视频合并方法，并提供[演示视频][ytb-demo]作为整个下载流程的参考。

## 引子

2010年创办的Bilibili网站，经过多年探索发展，在中国的视频网站圈已占有举足轻重的地位。据彭博社报道，Bilibili所属公司预计于2018年在美IPO。
此外，在近期发生的[某网红煽动群众事件][sina-douyu-news]中，Bilibili视频网站官方的[公关表态][sina-bili-weibo]，也让人不禁赞许。本文下方将用**B站**指代Bilibili视频网站。

![Bilibili weibo post](/assets/always-working-bilibili-video-download/weibo-bili.jpeg)

## 问题描述

网间曾流传过“Bilibili黑科技”、“Acfun里区”等“神秘链接”。究其技术本身大都是通过访问**视频网站播放器**附加指定**视频源参数**构建而成。
其中，B站使用了`cid`和`aid`两个参数决定视频源和弹幕池。而经过多年的技术迭代，B站将cid的获取方式变得非常困难。

究其技术发展的轨迹，能从各个时期的博客文章进行推断。从HTML页面源码就能得到（例如[此篇网络找到的新浪博客][sina-blog-howtocid]），演化到集中帖收集cid数据（例如此[百度贴吧帖子][tieba-cid-list]），转变为通过私有API访问得到（例如此[已死亡的开源项目][github-bili-deadproj]），直到如今的使用第三方API得到（例如此[唧唧网][bilijj-api-page]）。

视频下载的“挑战”不仅于此。出于访问量的蹭蹭上涨，B站使用CDN（分布式资源网络）将视频资源投放（推送）到全国本地节点服务器。一来用户能享受低延迟的服务，二来还能将整体流量作去瓶颈化。

可是对CDN资源的**访问验证**又成为一座大山。至此可见，通过第三方API得到`cid`，再由`cid`计算出的CDN资源链接，也不足以让我们成功下载视频。因为CDN的访问验证过程是一个[黑箱系统][wiki-blackbox]，
对于满足一个黑盒系统的验证需求，除了代码本身要随着黑盒的更新不断尝试外，唯有用社会工程学给出稳定方案。

不过别忘了，**一个网站**再怎么牛，也是**需要浏览器作为媒介**的。本文将介绍的是，借鉴**传统的视频缓存思路**，无需任何第三方软件介入下，获取**获取B站视频下载链接**的方法。

## 解决步骤

#### 1. 使用谷歌浏览器，打开网页控制台，访问某一B站投稿

第一步是在打开的谷歌浏览器（基于Chromium的浏览器），开启网页控制台选项卡，快捷键是`F12`或`Ctrl-Shift-I`。
然后再打开某一视频投稿链接，如&nbsp;[*https://www.bilibili.com/video/av367901*](https://www.bilibili.com/video/av367901)

#### 2. 等待视频缓冲完毕，将网络选项卡内的链接“全部复制为cURL”指令

点击播放视频。你可以选择等待视频缓冲自己完成，或用鼠标点击快进视频的缓冲进度。我们的想法是，确保投稿视频的每一分段都正确进入下载流程。

> 什么是视频分段    
我们知道，在B站投稿的视频有的很短才几分钟，有的很长达到几小时。
但如果所有视频都是用单一文件进行存储，势必会造成视频文件大小的不同。这不利于视频文件的底层存储的同时，还不利于视频文件的点播效率。
所以，一般视频网站都会采取视频分段策略。例如将一个600兆的文件，分成6份存储。B站的一个视频分片文件约为*120MiB*左右（文章撰写时）。

待视频缓冲完毕，我们需要使用谷歌浏览器的*全部复制为cURL*指令获得链接数据。

#### 3. 将复制的指令保存到文件，将文件输入到分析脚本，得到获取各视频各分段的下载链接

由于B站投稿网页不单纯是视频数据，还有例如图片，文字，JS脚本等资源存在。所以我们需要[下载分析脚本][github-gadgets-script]。
将上一步获得的cURL指令保存至一个文件，然后执行并获得如下

{% highlight shell %}
delight09 % bilivideo_resource_filter.sh /path/to/your-saved-content.txt
curl 'https://cn-hbcd2-cu-v-01.acgvideo.com/vg8/2/c8/564111-1.flv?expires=.....' -H 'Pragma: no-cache' -H ...  --compressed -o 564111-1.flv ;
curl 'https://cn-hbcd2-cu-v-01.acgvideo.com/vg8/2/c8/564111-2.flv?expires=.....' -H 'Pragma: no-cache' -H ...  --compressed -o 564111-2.flv ;
...

{% endhighlight %}

我们可以看到，分析程序回显了所有视频分段的*下载代码*。我们需要做的就是执行这些代码，开始下载过程(需要本地安装有cURL工具）。

#### 4. 使用FFMPEG工具，将各分段视频整合为完整视频

由于视频分段的封装都一样，所以我们能使用FFmpeg的[*concat*协议][ffmpeg-wiki-concat]，将视频分段整合成完整视频。

首先创建视频分段文件列表，格式如下

{% highlight plain text %}
file '/path/to/your-video-part1.flv'
file '/path/to/your-video-part2.flv'
file '/path/to/your-video-part3.flv'
...
{% endhighlight %}

执行如下指令进行视频分段合并

{% highlight plain text %}
delight09 % ffmpeg -f concat -safe 0 -i /path/to/mylist.txt -c copy bilivideo.flv
{% endhighlight %}

上述解决问题的步骤，在[这个演示视频][ytb-demo]中得以重现，可以观看作为整个下载流程的参考。

## 后记

基于浏览器的网页流量分析其实推荐使用HAR(Http ARchive)格式存档，再对HAR存档进行视频源解析。

同时这方法不是所有浏览器都适用，比如火狐58就不支持将网页流量全部复制为cURL指令。

另外的一个不足之处是，如果B站日后发展成当下优酷网站的技术架构，即用浏览器XHR请求获取纯流格式的视频文件（MPEG-2），那么这个脚本就要改动下了。


## FYI

- 彭博社 -- [China's Top Anime Streaming Site Is Said to Plan IPO in U.S.][bloomberg-bili-ipo]

- Wikipedia -- [Alexa Internet][wiki-alexa]

- FFmpeg wiki -- [Concatenating media files][ffmpeg-wiki-concat]

下方摘录有撰写本文时的站点排名情况

### Alexa全球站点排行摘录

- Bilibili -- 全球256, 中国47（[archive.is网页存档][archiveis-rank-bili]）

- 爱奇异 -- 全球381, 中国53

- “搜狐视频” -- 全球17, 中国5

- 优酷网 -- 全球1394, 中国151

- 斗鱼直播 -- 全球170, 中国26

虽说看上去搜狐网站的排名很靠前，但搜狐域名下还有新闻、邮箱等服务。如果还不能很好理解的话，可以自行搜下`qq.com`的Alexa域名排行，应该就能
直观了解被称为“腾讯帝国”的恐怖了。

### Alexa中国站点排行摘录（[archive.is网页存档][archiveis-rank-cn]）

#### 按人均每天停留时间排行

1. 搜狗 -- 18:56

2. 1688网 -- 14:38

3. Bilibili -- 11:05

4. 阿里云 -- 8:50

Hmm，无法理解为什么搜狗会是第一？很简单，请看这个知乎提问[`Chrome主页被搜狗劫持，应该怎么办？`][zhihu-sogou-hijack]。1688是批发行的淘宝，也隶属于阿里巴巴集团（中国贸易大王实至名归）。

#### 按人均页面浏览量排行

1. Google.co.jp -- 11.44

2. Google.com -- 8.82

3. 1688 -- 8.81

4. Bilibili -- 8.80

博主对PV（页面浏览量）排行的不想作过多解释，给出另一个数据，留给读者自行思考。

> CSDN.net -- 提供业界新闻，技术文章，讨论社区和软件下载；出版《程序员》杂志    
Alexa排名 全球56,中国15


[wiki-alexa]: https://en.wikipedia.org/wiki/Alexa_Internet
[bloomberg-bili-ipo]: https://www.bloomberg.com/news/articles/2017-10-10/china-s-top-anime-streaming-hub-is-said-to-plan-initial-offering
[archiveis-rank-cn]: http://archive.is/r4Al1
[archiveis-rank-bili]: http://archive.is/oYmM2
[zhihu-sogou-hijack]: https://www.zhihu.com/question/36528457https://www.zhihu.com/question/36528457
[sina-bili-weibo]: http://games.sina.com.cn/g/g/2017-12-05/fypnqvn0313564.shtml
[github-bili-deadproj]: https://github.com/Vespa314/bilibili-api/blob/1464bb3ae9adefbc85deba8bb16da106cc080e36/bilibili-video/bilibili.py#L201
[bilijj-api-page]: http://www.jijidown.com/apiword/
[tieba-cid-list]: https://tieba.baidu.com/p/4360653977
[sina-blog-howtocid]: http://blog.sina.com.cn/s/blog_58c506600100utap.html
[sina-douyu-news]: http://games.sina.com.cn/o/n/2017-12-10/fyppemf6185807.shtml
[ffmpeg-wiki-concat]: https://trac.ffmpeg.org/wiki/Concatenate#protocol
[wiki-blackbox]: https://zh.wikipedia.org/wiki/黑箱
[github-gadgets-script]: https://github.com/delight09/gadgets/blob/master/filter/bilivideo_resource_filter.sh
[ytb-demo]: https://youtu.be/nBEQPGKNKss