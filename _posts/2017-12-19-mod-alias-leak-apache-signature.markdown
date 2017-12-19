---
layout: post
title: 阻止mod_alias泄露服务器指纹
category: howto
tags: [harden, apache, signature]
image: /assets/mod-alias-leak-apache-signature/feature.png
description: debian分发的apache2包，默认开启alias模块，其模块配置将网站/icons路径引用到一个已知文件夹。所以爬虫可以根据是否能找到这个资源得到www服务器的指纹，间接判断出服务器的型号、版本。
---

## 引子

无聊看Piwik日志报告，发现了一个不常见的200报告。后在`access.log`缺省日志中找到

> aa.bb.cc.dd - [09/Dec/2017:17:51:49 +0000] "HEAD /icons/apache_pb.gif HTTP/1.0" 200 0 "-" "Mozilla/5.0 (compatible; NetcraftSurveyAgent/1.0; +info@netcraft.com)"

因为网站并没有搭建完，也没有很多PV，日志中多的是网际爬虫带来的[404没有找到][httpstatusdogs-404]，反复低频率的出现200就显得十分突兀。

## 事出原因

Apache2.4默认启用了mod_alias模块，由[apache2二进制包][debian-package-apache2]分发的模块配置文件`/etc/apache2/mods-available/alias.conf`，
默认包含了对网站`/icons/apache_pb.gif`URL的引用。

{% highlight shell %}
sha1sum ./etc/apache2/mods-available/alias.conf
4384d95541236cd27083cb56a3cfe1c8ea277197  ./etc/apache2/mods-available/alias.conf

grep -vE '^\s*#' ./etc/apache2/mods-available/alias.conf
<IfModule alias_module>

        Alias /icons/ "/usr/share/apache2/icons/"

        <Directory "/usr/share/apache2/icons">
                Options FollowSymlinks
                AllowOverride None
                Require all granted
        </Directory>

</IfModule>
{% endhighlight %}

## 解决方法

将`alias.conf`中，涉及`/icons/`路径的配置项注释即可。

## FYI

- Apache Documentation - [Apache Module mod_alias][apache-doc-mod-alias]

[httpstatusdogs-404]: https://httpstatusdogs.com/404-not-found
[debian-package-apache2]: https://packages.debian.org/stretch/apache2
[apache-doc-mod-alias]: https://httpd.apache.org/docs/2.4/mod/mod_alias.html
