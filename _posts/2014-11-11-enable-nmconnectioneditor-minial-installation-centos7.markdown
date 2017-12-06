---
layout: post
title: 用nm-connection-editor配置最小化安装的CentOS7
category: howto
tags: [linux,centos,networkmanager,nm-connection-editor]
image: /assets/enable-nmconnectioneditor-minial-installation-centos7/feature.png
description: 最小化安装的CentOS7是没有X环境的，我们需要安装组包`X Window System`和`Fonts`。使用本地SSH客户端的X11forwarding功能即可图形化配置服务器网络。
---

## 安装必要组包

最小化安装的CentOS7是没有X环境的，我们需要安装组包`X Window System`和`Fonts`。

{% highlight shell %}
sudo yum -y group install "X Window System"
sudo yum -y group install "Fonts"
sudo yum -y install nm-connection-editor
{% endhighlight %}

安装完成后，在本地使用`ssh -X -Y 10.x.y.z nm-connection-editor`链接服务器IP，即可看到nm-connection-editor图形配置界面。

## FYI

搜索组包是否存在

{% highlight shell %}
yum grouplist hidden | grep X
yum grouplist hidden | grep Font 
yum grouplist env | grep Minimal
{% endhighlight %}
