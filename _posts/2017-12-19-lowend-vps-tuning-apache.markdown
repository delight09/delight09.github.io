---
layout: post
title: 在低内存VPS上配置使用Apache2.4
category: howto
tags: [tuning, apache, low-end]
image: /assets/lowend-vps-tuning-apache/feature.png
description: 介绍了更先进的响应式多线程mod_event多进程模块，揭开Apache服务器占用内存多的迷思。
---

## TL;DR

介绍了更先进的响应式多线程mod_event多进程模块，揭开Apache服务器占用内存多的迷思。

## 引子

所谓低配虚拟主机(VPS)，首要问题就是**内存**小。除了某厂商[坚持][v2ex-ali-201602]_[使用][v2ex-ali-201609]_[石头][v2ex-ali-2017]作为磁盘，和某些痴迷挖矿的群体对XEN虚拟CPU斤斤计较外，内存的大小*或多或少地*[限制了我们的想象空间][zhihu-poor-restrict]。

## 遇到的问题

自Apache2.4开始mod_event多进程模块(MPM)正式被列入稳定状态（区分实验状态）。
然而包括debian9在内的大多数发行版仍然默认使用mod_prefork配置。

简单来说prefork方案使得Apache派生出非常多进程，每个进程包含一个线程，并且每个进程同时只处理一个请求。
虽然这么做能让不是“线程安全”的软件库正常工作，也正是每个请求一个进程的原因，
使得网间对Apache的印象就是吃很多内存。

## 解决方法

#### 拒绝`mod_php`

很可惜，博主读过的大部分文章、官方教程都会使用这个模块。出厂即用的感觉确实不错，可惜它并不优雅。

这里引用官方对于`mod_php`的[描述][httpd-wiki-php]

>Why you shouldn't use mod_php with the prefork mpm anymore    
- mod_php is loaded into every httpd process all the time. Even when httpd is serving static/non php content, that memory is in use.    
- mod_php is not thread safe and forces you to stick with the prefork mpm (multi process, no threads), which is the slowest possible configuration

简单来说，就是指`mod_php`不是线程安全的，用了它就只能使用prefork模块。
同时这个模块会常驻所有接受请求的进程，无论请求内容是否和PHP解析有关。

#### 使用mod_event

{% highlight shell %}
sudo a2dismod mpm_prefork
sudo a2enmod mpm_event
sudo systemctl restart apache2

{% endhighlight %}

开启mod_event是很简单的，复杂的是对依赖于prefork的模块的处理。比如将mod_php迁移成php-fpm，再对后者进行调优的操作。

#### 限制连接数

{% highlight plain text %}
<IfModule event.c>
ServerLimit            8
MaxRequestWorkers    512
StartServers           2
ThreadsPerChild       64
ThreadLimit          256
</IfModule>
{% endhighlight %}

将上述内容保存到Apache配置目录的`conf-available`下，命名为`mod_event.conf`。

上述配置讲述了一个Apache服务器，启动时派生`StartServers`个进程，每个进程`ThreadsPerChild`个线程，用线程处理请求。
服务高峰期，最多将会派生`ServerLimit`个工作进程，同样每个`ThreadsPerChild`线程。也就是说服务器最多能同时接受`8 * 64 = 512`个请求。

如果你好奇在调优完成后，Apache内存的占用，那么如下便是一个闲置Apache2.4服务器的占用 -- **`0.0%`**（对比同样是www-data用户启动的nginx,闲置时0.4%）

{% highlight plain text %}
Tasks: 173 total,   2 running, 171 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  1.0 sy,  0.0 ni, 96.1 id,  0.0 wa,  0.0 hi,  0.0 si,  2.6 st
KiB Mem :   504396 total,    31880 free,   297720 used,   174796 buff/cache
KiB Swap:  2097148 total,  1418200 free,   678948 used.   166144 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
 1191 www-data  20   0  114916    192    160 S  0.0  0.0   0:00.02 apache2
 1238 www-data  20   0 1182604     88     24 S  0.0  0.0   0:18.80 apache2
25921 www-data  20   0   82020   2200   2092 S  0.0  0.4   0:01.08 nginx
31253 www-data  20   0  450168  29084  19852 S  0.0  5.8 264:46.30 php-fpm7.0
31980 www-data  20   0  452636  22816  14608 S  0.0  4.5 264:34.56 php-fpm7.0

{% endhighlight %}


## FYI

- Apache Documents -- [Apache MPM event][apache-docs-mod-event]

- Apache Wiki -- [Running PHP on Apache httpd][httpd-wiki-php]

- wiki.mikejung.biz -- [Apache Tunning][mikejung-apache-tunning]

[apache-docs-mod-event]: https://httpd.apache.org/docs/2.4/mod/event.html
[v2ex-ali-201602]: https://www.v2ex.com/t/258716
[v2ex-ali-201609]: https://www.v2ex.com/t/300804
[v2ex-ali-2017]: https://www.v2ex.com/t/405510
[zhihu-poor-restrict]: https://www.zhihu.com/question/67080789
[httpd-wiki-php]: https://wiki.apache.org/httpd/php
[mikejung-apache-tunning]: https://wiki.mikejung.biz/Apache