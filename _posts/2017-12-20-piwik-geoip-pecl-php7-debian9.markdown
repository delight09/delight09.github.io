---
layout: post
title: php7下使用PECL安装Piwik的GeoIP插件
category: howto
tags: [piwik, php7, signature]
image: /assets/piwik-geoip-pecl-php7-debian9/feature.png
description: 
---

## 引子

Piwik默认使用*来访者语言*判断地理位置，官方也同意这种判断方式不准确（但出厂既可用）。官方推荐了几种方式来增加定位的可靠性，常见的有`mod_geoip`和使用PECL编译`GeoIP插件`。
引用官方对于*mod_geoip*的描述，

> It(mod_geoip) uses a GeoIP database and MaxMind's **PHP API** to accurately determine the location of your visitors.
If your website gets a lot of traffic, you may find that this location provider is too slow. In this case, you should install the PECL extension or a server module.

mod_geoip因为调用的是**PHP API**，不如PECL编译出的**geoip.so**软件库**快**。所以毫无疑问我们**用GeoIP插件**。

## 事出原因

查阅[官方FAQ][pk-faq-geoip]后，发现其针对的是PHP5环境，不是*甚至*debian9都在用的PHP7。测试后发现原先命令会有错误发生，故实验并作此文。

## 解决方法

{% highlight shell %}
sudo apt-get install php-geoip php7.0-dev libgeoip-dev #安装PECL，以及依赖库
sudo pecl install geoip-1.1.1 #编译安装插件
sudo vi /etc/php/7.0/mods-available/geoip.ini #将geoip.so载入到php.ini
# geoip.ini文件内容应为 '>>>'标识后的
#>>> extension=geoip.so
#>>> geoip.custom_directory=/path/to/piwik/misc
sudo systemctl restart php7.0-fpm #重启PHP服务，如果使用的是mod_php就重启apache

{% endhighlight %}

使用的指令和解释如上。

## 后记

风风火火装完插件，发现GeoIP插件的配置有不少坑。

#### 1. GeoLiteCity数据库下载错了

博主用免费提供的GeoLiteCity数据库，但是错误的下载了名为`GeoLiteCityv6.dat`的文件。后发现文件名中包含的**v6**不是数据库版本，而是指数据库的内容是IPv6的对应地理信息。
同时PECL安装所得的`geoip.so`不支持IPv6型GeoIP库。强行使用会导致Piwik相关管理页面500报错。

#### 2. GeoIP数据库的文件名错误

数据库文件名必须为`GeoIPCity.dat`，同时其路径在`geoip.ini`中由`geoip.custom_directory`规定。否则GeoIP插件将无法读取数据库。

#### 3. 不推荐将GeoIP数据库放在Piwik文件夹中

因为Piwik会定期自动检查安装文件的一致性，所以不能将GeoIP数据库随便扔到诸如`/var/lib/piwik/`中，会有警报弹出。所以推荐将GeoIP数据库放入`/path/to/piwik/misc`。

## FYI

- Piwik FAQ - [(outdated, php5)How do I install the GeoIP Geo location PECL extension?][pk-faq-geoip]

[pk-faq-geoip]: https://piwik.org/faq/how-to/faq_164/
