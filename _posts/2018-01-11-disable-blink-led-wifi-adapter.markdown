---
layout: post
title: Linux内核4.x关闭USB网卡灯
categories: howto
tags: [nas, linux, usb]
image: /assets/disable-blink-led-wifi-adapter/feature.png
description: 下载关灯脚本，配置udev规则。实现插入指定USB网卡设备后不会亮灯、闪烁的效果。
---

## TL;DR

下载[关灯脚本][github-gadgets-script]，配置udev规则。实现插入指定USB网卡设备后不会亮灯、闪烁的效果。


## 问题描述

由[*使用树梅派组建低成本网络存储*][self-proj-rpinas]项目搭建成的个人NAS是使用USB网卡进行互联的，这造成了USB网卡上的链接状态灯在夜间无数据访问时也会不断闪烁。
势必会影响周遭的正常休息。

在上网查找后，发现有基于Linux老版本内核的关灯教程。但是并不适用于Linux4.x的新子系统。故实验成功后记录。

解决的思路是直接将网卡灯在任何情况下的亮度设置为**0**，并配置内核udev设备管理器，在插入USB网卡设备后当即运行关闭网卡灯脚本。

## 解决方法

#### 1. [下载`disable_led_usb_wifi.sh`脚本][github-gadgets-script]，设置其可被执行的权限，并存储在`/usr/local/bin`目录中。

{% highlight shell %}
#!/bin/sh
# Wait for associate then disable led on USB wifi adapter with led sub-system

disable_led() {
local DEVICE_NAME=rt2800usb-phy0

sleep 60
for i in /sys/class/leds/${DEVICE_NAME}*
do
    echo 0 > ${i}/brightness
done
}

(disable_led ) &
{% endhighlight %}

如上所示，关灯脚本使用了后台执行的方案。这是因为udev的**策略执行是阻塞性的**，同时在插入设备后**立即执行脚本不能实现关灯**效果。
所以我们选择等待一个魔法数字（MAGIC NUMBER）的时长，待此之后再执行关灯脚本。

同时，你需要根据自身网卡设备，更改变量`DEVICE_NAME`的内容。

#### 2. 添加udev策略

在目录`/etc/udev/rules.d`中添加规则**100-usbwifi.rules**，内容如下

{% highlight plain text %}
ACTION=="add", ATTRS{idVendor}=="your-idvendor-here", ATTRS{idProduct}=="your-idproduct-here", RUN+="/usr/local/bin/disable_led_usb_wifi.sh"

{% endhighlight %}

请根据实际`lsusb`指令的输出，更换上述文本中的**your-idvendor-here**与**your-idproduct-here**。

举个例子，

{% highlight plain text %}
root@OpenMediaVault:~# lsusb
Bus 001 Device 002: ID 0424:9512 Standard Microsystems Corp.
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp.
Bus 001 Device 004: ID 1a40:0101 Terminus Technology Inc. 4-Port HUB
Bus 001 Device 005: ID 0080:a001
Bus 001 Device 006: ID 2001:3c1b D-Link Corp.
Bus 001 Device 007: ID 2537:1068

{% endhighlight %}

在上述的命令回显中，我们能得知USB网卡设备为

> Bus 001 Device 006: ID 2001:3c1b D-Link Corp.

其中，idVendor为**2001**,idProduct为**3c1b**，规则**100-usbwifi.rules**的内容应为

> ACTION=="add", ATTRS{idVendor}=="2001", ATTRS{idProduct}=="3c1b", RUN+="/usr/local/bin/disable_led_usb_wifi.sh"


## FYI

- topbug.net -- [Control the LED on a USB WiFi Adapter on Linux][topbug-orig-post]

- Wikipedia -- [udev][wiki-udev]

- debian.org -- [HowToIdentifyADevice > USB][debian-wiki-idusb]

- Wikipedia -- [Magic number (programming)][wiki-magic-number]

[self-proj-rpinas]: /project/diy-nas-with-rpi-openmediavault/
[topbug-orig-post]: https://www.topbug.net/blog/2015/01/13/control-the-led-on-a-usb-wifi-adapter-on-linux/
[wiki-udev]: https://en.wikipedia.org/wiki/Udev
[github-gadgets-script]: https://github.com/delight09/gadgets/blob/master/display/disable_led_usb_wifi.sh
[debian-wiki-idusb]: https://wiki.debian.org/HowToIdentifyADevice/USB
[wiki-magic-number]: https://en.wikipedia.org/wiki/Magic_number_%28programming%29