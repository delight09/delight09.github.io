---
layout: post
title: 使用树梅派组建低成本网络存储
categories: project
tags: [rpi, low-end, nas, management, linux]
image: /assets/diy-nas-with-rpi-openmediavault/feature.jpg
description: 使用两条U盘组成RAID1作为存储介质，树梅派1代b型作为主机。使用qshell工具从AliOSS获得bucket所有资源URL。通过vmstat检测分析FTP和SMB两种常见NAS服务的性能数据，结果性能相近，可根据实际需要开启或关闭。讨论了RAID1自建NAS的一般灾备操作。最后使用第三方服务和一些辅助脚本，验证成功NAS的内网穿透。
lightbox_enable: true
---

## TL;DR

使用两条U盘组成[RAID1][wiki-raid1]作为存储介质，安装有[OpenMediaVault][omv-install-guide]2.x系统的树梅派1代b型作为主机。使用[qshell工具][qshell-link]从AliOSS获得bucket所有资源URL。通过vmstat检测分析FTP和SMB两种常见NAS服务的性能数据，结果性能相近，可根据实际需要开启或关闭。讨论了RAID1自建NAS的一般灾备操作。最后使用[第三方服务][natapp-register]和[辅助脚本][natapp-checker-script]，实现NAS的内网穿透并成功验证可行性。

## 引言： 再见AliOSS，信息寡头

要放在半年前，“*信息寡头*”类的论述是无法令我信服的，天真以为信息公开是所有互联网人都自觉遵守的最低底线。 直到某天，闲逛某购物网站时，发现搜索的结果明显有用户画像的介入，在“*诱导购买*”为主旨的购物网站中暗藏这类搜索条目，可见阿X巴巴早对用户做了完美的取向勾画，更别说工程师会对用户数据做哪些“*[穷比训练][xiami-code-comments]*”。别的例子还有很多，从**合法购买**第三方电商平台销售统计、定向广告投放公司，到手机APP[用户操作采集][insight2-jike-userlog]。拒绝寡头的用户画像，从拒绝隐私泄漏、尽力避免使用产品开始。

曾经“*很理性*”算过一笔账，结果明显是使用AliOSS类似的云端存储，相比个人NAS更为划算（零硬盘硬件费用、零维护费用、高带宽）。可如今[某些公司（X易XX有态度，呵呵）][netease-data-leak]对用户数据态度的现状令人担忧。那只有自己[*撸起袖子加油干*][just-get-stuff-done]，走自建NAS这条道路了。

由于处于待业的状态，手头能调用的资金很紧张。并且没有相关的硬件存货，于是只能自己“*开脑洞*”。

## 目录： 迫切需要解决如下几个问题

1. 存储介质和一些连接介质哪来

2. NAS主机用什么

3. 如何从ALIOSS下载数据

4. NAS服务性能测试

5. 容错性方案和灾难预备工作

6. 穿透内网的NAS

## 一、存储介质哪来

半年前，家里台式机突然无法点亮，由于这台机器之前挖矿把主板烧了（换了一块tb淘的主板，使用多年后坏下来也就懒得再一次追究原因）。闲置了两块成色还行的3.5寸机械硬盘，可没有**有源**SATA转USB接口转换工具(或叫3.5硬盘盒、3.5易驱线)，无奈放弃。家里的笔记本换过固态硬盘，空下来一块2.5寸机械硬盘，无奈**无源**SATA转USB工具都没，放弃。于是把目光转向“*闲置*”着用来应急装系统的两条8G大小U盘，也是我们接下来组建NAS的存储设备：

![NAS Storage NO.1 SPEC](/assets/diy-nas-with-rpi-openmediavault/upan1.jpg)

![NAS Storage NO.2 SPEC](/assets/diy-nas-with-rpi-openmediavault/upan2.jpg)

因为这两条U盘已近4年的高龄，不知什么时候会“*[自行了断][electronic-components-fails]*”。于是两条U盘做RAID1数据冗余，也就是每一份数据存储两份，NAS总大小是两条U盘中更小那条的容量。解决最根本的存储介质，接着的是些作为链接中介存在的设备。很有幸得在上学期间买过某品牌**有源**4口USB2.0集线器（之所以强调“*品牌*”，是要确保各USB2.0输出口能达到标称的500mA电量），一个USB无线接入器（或叫做便携AP，无线上网棒）。BOOM！就有了如示意图的**单电**NAS解决方案。

![NAS design SPEC](/assets/diy-nas-with-rpi-openmediavault/feature.jpg)

## 二、NAS主机用什么

由于经历过挖矿烧坏台式机（简单说家用机满负载跑了几天后自动断电了，各种测试无果，结果换了主板就好了）。于是自然不敢再用家庭PC折腾7×24服务。想到13年那会儿跟大潮买过树梅派1代b型，接下来我们简称为RPI。记得当时跑过XBMC（早已更名为[kodi][kodi-link]）卡得[让人作呕][xbmc-lag]，单独跑个X也很吃力更别提用浏览器上网，于是闲置了近3年。由于树梅派是ARM架构的SoC（板载系统）发热量不会很大，打算让它7×24跑着发挥机生最后的光芒。相信它跑几个**不涉及图像操作的**NAS服务（SMB/FTP）和服务框架（Nginx、PHP）应该还是轻轻松松的吧？其实自己也没底气，只能说做好实验失败的最坏打算。

选择完硬件接下来就轮到操作系统。谷歌搜索了下，市面上NAS解决方案一般是FreeNAS、OpenMediaVault、基于openwrt搭建的NAS，还有使用类似owncloud/nextcould等个人网盘服务的webdav构成的“*不健全NAS*”。因为FreeNAS基于BSD，所以RPI平台直接PASS掉。好吧，[NetBSD当然能在RPI上跑][ofcuz-it-runs-netbsd]。可FreeBSD官网写着的[配置要求4GRAM][freenas-hw-requirements]，RPI1b只有其八分之一，用脚趾思考了下调优的工作量果断放弃了。openwrt从没上手过，其适用于众多路由器，虽然能装在RPI上，可如果有更好的方案，为什么要用这个特殊用途OS呢？PASS！最终我把宝压在了同样没有接触过的OpenMeidaVault和ArchLinux搭建nextcould网盘服务。你可能会奇怪为什么不用树梅派官方的raspbian系统？用过raspbian装出来的X图形桌面，把我都卡吐了，对这个系统不作过多期待，用个人更熟悉的ArchLinux做备用方案。 OpenMediaVault的安装和raspbian的安装方式如出一辙，都是把带扇区信息的img磁盘映像直接烧录到sd卡。对比之下ArchLinux的安装要复杂得多，[官方指南在这里][archlinux-arm-guide]。

首发用OpenMediaVault尝试搭建，安装过程就不赘述了，官方提供的[教程简单易懂][omv-install-guide]。 首次启动过程中，RPI会自动重启几遍，主动更新SD卡扇区信息和创建新分区。接着，如果你使用**网线链接**的RPI，那没什么好担心的。将RPI链接上显示器，启动完成后就会在屏幕上显示获取的IP地址（或者看路由器控制界面中DHCP服务租赁表单中RPI的IP），我们在这里假设获取的IP为`192.168.1.111`。使用浏览器登录`http://192.168.1.111/`进行NAS服务配置。但如果你使用了USB型无线接入器（或叫USB无线上网棒，随身AP），那还要操作额外几个步骤。因为OpenMediaVault 2.x版本的系统对于USB型无线接入器的自动适配有问题，驱动正常可是IP拿不到，这里将会演示的是使用CLI手动配置IP，使RPI能够联网，后进入OpenMediaVault后再用网页配置“*固定*”配置，那么先将RPI链接到显示器键盘，着手开干。

> OpenMediaVault**操作系统级别**用户密码默认为：root/openmediavault

链接上键盘登陆后，假设我们要指定树梅派的IP为`192.168.1.111`，内网网关为`192.168.1.1`，掩码**24**位，则指令和指令对应解释如下：

{% highlight shell %}
ip link set wlan0 down     #关闭初始化失败的便携AP
pkill -9 dhc               #杀掉所有可能存在的dhcp客户端进程
ip address flush wlan0     #将可能存在的配置地址清空
ip address add 192.168.1.9/24 dev wlan0        #将指定指定的IP配置
ip link set wlan0 up       #将便携AP启动
ip route add default via 192.168.1.1 dev wlan0 #添加路由表，我们要上外网更新文件的
echo 'nameserver 114.114.114.114' > /etc/resolv.conf #更新dns服务器为114公开dns
{% endhighlight %}

使用浏览器登陆`http://192.168.1.111`。好的，既然操作系统已经成功启动，让我们登陆网页进行NAS配置吧。什么情况，[502 Bad Gateway][httpstatusdogs-502]?这是一个已知问题，[官方答复在此][omv-2-fix-502]。 解决起来也容易，在接入显示器键盘，输入操作系统级账号后（已在上文提及），按照官方提供的方案，键入如下指令（更新*php5-pam*包）：

> 据称RPI2代以及之后的型号不会有此错误，直接跳过此段配置即可

{% highlight shell %}
wget http://omv-extras.org/debian/pool/main/p/php5-pam/php5-pam_1.0.3-2_armhf.deb
dpkg -i php5-pam_1.0.3-2_armhf.deb
reboot
{% endhighlight %}

之后是在网页配置RAID1的过程，也就是在页面上点点按键，应该已经很直观了，在这里就简述下

> **/!\\** 以下的配置操作，均会在网页界面中产生一个**肉色警告条**，记得每次点击**Apply**生效配置 **/!\\**

> OpenMediaVault**网页配置端默认用户名密码**：admin/openmediavault

### 1. 建立软RAID1

左侧导航栏`Storage`->`RAID Management`->`Create`->`Level` 选*RAID1*,再钩上sda和sdb点击`save`。

> 注意看*Capacity*、*Vendor*参数别选错了设备，不然之后开始RAID1块同步操作后，就算神仙都救不回原有数据

### 2. 建立文件系统

在md设备上（multi devices也就是虚拟出的RAID）建立文件系统`Storage`->`File Systems`->`Create`->设备选择md，*Label*随意，是文件系统的标签名。文件系统的选择，如果是U盘推荐用ext4相对更快，机械硬盘推荐xfs。

### 3. 等待块同步完成

之后RAID1会开始同步磁道数据，8G的U盘来说1小时就好了(对比320G硬盘需要8小时)，进度能在`RAID Management`中*State*单元格中看到详细。

### 4. 创建文件共享区域

创建完文件系统，接下来需要的是在文件系统上创建文件共享区域（或者干脆认为是文件夹其实没多大区别）

1) 先创建内容读写账号，`Access Rights Management`->`User`，创建一个就是了

2) 再创建共享区域，`Access Rights Management` ->`Shared Folders`->`Add`，在*Voume*处选择你的创建的md设备就是了，*Name*随意只要之后启用服务的时候认得出就行，*Path*就是在md设备上文件系统上的一个文件夹，*Permissions*不必改动

3) 配置内容读写账号对共享区域的权限，要将内容读写账号赋予对共享区域的读写权限，`Access Rights Management`->`User`->单击你的用户名->`Privileges`，将*Read/Write*权限打开

### 5. 启动NAS服务

最后就是启动服务了，因为相当简单，我就用一个FTP服务当作例子了

1) 启动FTP服务，`Services`->`FTP` 将*enable*那边一个开关打开

2) 点击`Shares`标签页，点击`Add`，在*Shared folder*栏选中刚才创建的共享区域，别的不必填，点击`Save`

## 三、如何从AliOSS下载数据

### 1. 使用七牛qshell工具导出OSS文件列表

1）从[qshell项目][qshell-link]下载二进制文件

![Qiniu qshell download hint](/assets/diy-nas-with-rpi-openmediavault/qshell_download_hint.jpg)

2）从aliyun控制台中得到*accesskeyId*和*accesskeySecret*

3）使用`qshell`命令，将OSS某一*bucket*中所有文件资源地址导出至文件*files.txt*

{% highlight shell %}
/path/to/qshell-linux-x64 alilistbucket oss-cn-shanghai.aliyuncs.com your-bucket-name LxAIXxXxd33qxXxX KxXedRUxXxXxNrWJKJqAIx38xXxX files.txt
{% endhighlight %}

4）使用任意下载工具下载`files.txt`中的URL资源

### 2. 使用官方提供的OSS客户端

1）下载[客户端][oss-client-link]（其实是0元购买，需要阿里云帐号）

2）根据界面填入*accesskeyId*和*accesskeySecret*，点击`登录`

3）根据GUI操作

## 四、NAS服务性能测试

性能测试着重还原现实中最常发生的情形。设计了三个用例，分别是用户上传/下载大量中等大小文件（定期将手机中的照片备份到NAS，用户上传/下载单个大型文件（点播720p电影，很抱歉由于本文NAS总共只有8G空间，存放不了4k电影作为测试），用户上传/下载大量细小文件（将个人git项目上传NAS）。性能测试采用了操作系统内置的工具vmstat，每间隔10秒采集到内存盘中（系统内置的*/run/shm*文件夹），以最小化采集对系统本身的性能影响。希望读者从性能测试的报告中能选出适合自己的NAS服务。

### 0. NAS参数回顾

1) RPI 1代b型，ARM架构双核心ARMv7架构，512MB内存，配备8Gclass10SD卡装载操作系统。搭载2个USB2.0接口和广为诟病的USB芯片控制的有线网口（只是提一下，没用到有线网络）。系统自动分配了100MiB的swap交换空间。

2） USB无线AP，仅支持2.4Ghz频段，实验环境中距离无线路由器6米距离，路径上需要穿越一扇空心木质门和大厅玻璃门，时不时还有博主穿越直线路径。

3）4口USB2.0有源集线器，各口最大500mA足标称。

4）两个U盘都是垃圾品相，间断使用过4年左右时间，格式化/烧写过的系统两只手数不过来（有时候忘记一个丢在哪了，就烧了另一个用）。接口是2.0协议，u盘因为用得少所以哪怕3.0U盘出了也没再买过（也没人送）。

5）FTP服务默认标准配置，除了：单客户端最大连接数翻倍（变为4），最大客户端承载量翻倍（变为10）。测试客户端为Filezilla最新版本。

6）SMB服务为默认标准配置。测试客户端为系统内置资源浏览器。

7）测试平台Windows 7，2012年的标准配置。位置距离无线路由器6米左右，路径需要绕过1堵水泥墙，1空心木门，1博主。

### 1. 下载测试

1）FTP方式

大量细小文件 -- 个人项目仓库，总`400MiB`，`16212个文件`。用时`48分钟`，平均速度`138.9kB/s`

![ftp-download speedtest1 vm chart](/assets/diy-nas-with-rpi-openmediavault/ftp-download-mon-t1.png)

![ftp-download speedtest1 disk chart](/assets/diy-nas-with-rpi-openmediavault/ftp-download-disk-t1.png)

大量中等文件 -- 年度照片存档，总`800MiB`，`200个文件`。用时`24分钟`，平均速度`555.6kB/s`

![ftp-download speedtest2 vm chart](/assets/diy-nas-with-rpi-openmediavault/ftp-download-mon-t2.png)

![ftp-download speedtest2 disk chart](/assets/diy-nas-with-rpi-openmediavault/ftp-download-disk-t2.png)

单个大文件 -- 720p视频，总`924MiB`，`1个文件`。用时`15分钟`，平均速度`1.0MB/s`

![ftp-download speedtest3 vm chart](/assets/diy-nas-with-rpi-openmediavault/ftp-download-mon-t3.png)

![ftp-download speedtest3 disk chart](/assets/diy-nas-with-rpi-openmediavault/ftp-download-disk-t3.png)

2）SMB方式

大量细小文件 -- linux内核源代码，总`845MiB`，`64859个文件`。用时`112分钟`，平均速度`125.7kB/s`

![smb-download1 speedtest3 vm chart](/assets/diy-nas-with-rpi-openmediavault/smb-download-mon-t1.png)

![smb-download1 speedtest3 disk chart](/assets/diy-nas-with-rpi-openmediavault/smb-download-disk-t1.png)

大量中等文件 -- 年度照片存档，总`800MiB`，`200个文件`。用时`18分钟`，平均速度`740.7kB/s`

![smb-download2 speedtest3 vm chart](/assets/diy-nas-with-rpi-openmediavault/smb-download-mon-t2.png)

![smb-download3 speedtest3 disk chart](/assets/diy-nas-with-rpi-openmediavault/smb-download-disk-t2.png)

单个大文件 -- 720p视频，总`924MiB`，`1个文件`。用时`18分钟`，平均速度`855.6kB/s`

![smb-download3 speedtest3 vm chart](/assets/diy-nas-with-rpi-openmediavault/smb-download-mon-t3.png)

![smb-download3 speedtest3 disk chart](/assets/diy-nas-with-rpi-openmediavault/smb-download-disk-t3.png)

### 2. 上传测试

1）FTP方式

大量细小文件 -- linux内核源代码，总`845MiB`，`64859个文件`。用时`73分钟`，平均速度`192.9kB/s`

![ftp-upload speedtest1 vm chart](/assets/diy-nas-with-rpi-openmediavault/ftp-upload-mon-t1.png)

![ftp-upload speedtest1 disk chart](/assets/diy-nas-with-rpi-openmediavault/ftp-upload-disk-t1.png)

大量中等文件 -- 游戏原声合集，总`321MiB`， `50个文件`。用时`25分钟`，平均速度`214kB/s`

![ftp-upload speedtest2 vm chart](/assets/diy-nas-with-rpi-openmediavault/ftp-upload-mon-t2.png)

![ftp-upload speedtest2 disk chart](/assets/diy-nas-with-rpi-openmediavault/ftp-upload-disk-t2.png)

单个大文件 -- 720p视频，总`1096MiB`，`1个文件`。用时`25分钟`，平均速度`730.7kB/s`

![ftp-upload speedtest3 vm chart](/assets/diy-nas-with-rpi-openmediavault/ftp-upload-mon-t3.png)

![ftp-upload speedtest3 disk chart](/assets/diy-nas-with-rpi-openmediavault/ftp-upload-disk-t3.png)

2）SMB方式

大量细小文件 -- 个人项目仓库，总`400MiB`，`16212个文件`。用时`77分钟`，平均速度`86.6kB/s`

![smb-upload speedtest1 vm chart](/assets/diy-nas-with-rpi-openmediavault/smb-upload-mon-t1.png)

![smb-upload speedtest1 disk chart](/assets/diy-nas-with-rpi-openmediavault/smb-upload-disk-t1.png)

大量中等文件 -- 游戏原声合集，总`321MiB`，`50个文件`。用时`18分钟`，平均速度`297.2kB/s`

![smb-upload speedtest2 vm chart](/assets/diy-nas-with-rpi-openmediavault/smb-upload-mon-t2.png)

![smb-upload speedtest2 disk chart](/assets/diy-nas-with-rpi-openmediavault/smb-upload-disk-t2.png)

单个大文件 -- 720p视频，总`924MiB`，`1个文件`。用时`13分钟`，平均速度`1.2mB/s`

![smb-upload speedtest3 vm chart](/assets/diy-nas-with-rpi-openmediavault/smb-upload-mon-t3.png)

![smb-upload speedtest3 disk chart](/assets/diy-nas-with-rpi-openmediavault/smb-upload-disk-t3.png)

## 五、灾难备份方案

哪怕是上万的EMC Isilon方案也有宕机的可能，灾备永远是NAS不可逃避的课题之一。这边我们来详细说下几个常见的NAS灾难场景，以及在灾难发生后获取数据的方法。

### 1. RPI坏了（NAS主机损坏）

将两条U盘插入一台Linux系统的主机（比如用knoppix启动盘启动后的环境），执行如下指令（附带指令解释）。

{% highlight shell %}
sudo fdisk -l /dev/sd? #查看插入U盘的设备名，如sdb, sdc, sdd... 下文我们使用sdz作为一条u盘的设备名
sudo mdadm --examine /dev/sdz #查看u盘的RAID信息，注意sdb要根据自己情况更改。只需要填写一个u盘设备名作为参数。
sudo mdadm -A -R /dev/md10 /dev/sdz #创建md10虚拟RAID设备
sudo mount /dev/md10 /mnt #将md10虚拟RAID设备挂载，读取NAS数
{% endhighlight %}

> 注意：如果在执行`sudo mdadm -A -R /dev/md10 /dev/sdz`后，发生了诸如**/dev/sdz device busy**的报错，说明系统已经自动将插入的u盘加入默认md设备。跳过如上步骤，无需手工挂载。

查看系统中已经存在的md设备指令：

{% highlight shell %}
ls /dev/md* #显示系统中已经存在的md设备，如下我们用md127代指。
sudo mdadm --detail /dev/md127 #显示系统自动生成虚拟RAID设备的状态。
{% endhighlight %}

### 2.一条U盘坏了（NAS存储介质损坏）

同RPI坏了步骤

### 3. 无法ping通NAS（NAS网络故障）

重新插拔USB型无线接入器，等1分钟，系统会自动重连上wifi。或使用[networkchecker_daemon.sh][network-helper-daemon]等网络健康监控脚本自动控制。

## 六、穿透内网的NAS

由于全球使用网络人数不断上涨，未被分配的IPv4区段早已见底。国内IPv6网络尚未全面使用。大部分运营商用NAT(地域级别**100.0.x.y/16**）套NAT（片区级别**10.x.y.z/8**）的方式供用户上网。这里提供两个常见的主机穿透内网方案。

### 1. DMZ主机 / 路由器端口映射

> 此方法仅适用于路由器获得公网IP的情形。如何得知路由器是否有公网IP，你可以打开路由器的网页管理界面。如果诸如WAN地址是一个合法IP的话，就应该是了。分配有公网IP的路由器常见于建成时间较久的运营商片区。

DMZ和路由器端口映射的教程在网络上非常多(换句话说，你用百度都能轻松找到教程)，这里不再赘述。请自行查阅。

### 2. 使用某品牌的内网穿透工具

博主居住在一个近两年才完工的片区，看了路由器网页控制台，如下图所示没有分配到公网地址（得到了**10.6x.y.z/32**的内部地址）。所以DMZ方案无效。这里展开说一下，有网友发帖称可以联系客服将家里网络“工单修理”成公网IP的状态，博主自己没有试过、持中立态度，在此仅提及这种可能性。

![demo public IP from ISP](/assets/diy-nas-with-rpi-openmediavault/no-ip.jpg)

对于内网穿透工具的选择，我们着重看这3点

1）服务稳定。**不是指**服务接口会不会变动，**特指**穿透内网服务本身的稳定性。关键时刻不会掉链子，靠得还是服务本身的*uptime*。

2）客户端轻巧。花生壳直接PASS。客户端做得和IM软件一样，真不知道这家公司有多大野心。

3）收费合理。额，其实我是指**我根本不愿意为此付哪怕一分钱** 。

> 这里我选择了natapp产品，无赞助非广告，个人、主观、暂时的选择。

操作流程可参考如下：

1）服务[注册帐号][natapp-register]。帐号根据国家规定需要实名制，过程不再赘述。之后使用官方的1分钟快速[新手图文教程][natapp-guide]，购买/建立免费**TCP隧道到本地22端口**，得到并记录*authtoken*。

server酱[注册帐号][serverchan-register]。使用github帐号登陆即可，过程不再赘述。得到并记录[SCKEY][serverchan-sckey]，扫描[二维码绑定微信推送][serverchan-bind]。用作服务端口变化的收信平台。

2）[下载服务客户端][natapp-download-client]到本地，再上传至树梅派（方式随意scp或者NAS提供的FTP/SMB均可)

3）开启SFTP服务

也许你会问为什么不用NAS提供的FTP或者SMB呢，因为这两个协议涉及到多个端口的使用。出于**简单就是美**考虑，我们在内网穿透NAS时，使用更现代更安全的SFTP。换句话说，内网的FTP/SMB服务就算有漏洞，也不能被外网所触及。

> 将SFTP服务暴露到外部网络将不可避免的使ssh服务公开。所以，请**一定修改默认的操作系统级别root密码**（如下操作指令将会涉及使用CLI修改root密码）。否则NAS毫无安全性可言。

通过CLI修改root密码（当然也可以用网页配置修改root密码），将键盘、显示器链接到RPI（如果之前一切顺利，也可以用ssh客户端登陆）。指令和对应解释如下：

{% highlight shell %}
# 输入如下指令，注意在行未输入回车键，修改root密码为Sup3rStr0ngP@ss..
passwd <<EOI
Sup3rStr0ngP@ss..
Sup3rStr0ngP@ss..
EOI
usermod -a -G ssh dummyred #将NAS用户dummyred加入到SSH用户组，使其能登陆SFTP
usermod -d /media/xxxx-uuiduuid/ dummyred #假设NAS用户叫dummyred，登陆后即能看到文件（设置家目录），注意将uuid修改正确
{% endhighlight %}

4）下载穿透内网控制脚本，安装开机启动和计划任务

参考内网穿透[nat_endpoint_checker.sh][natapp-checker-script]辅助脚本，请自行下载。将*SCKEY*和*AUTHKEY*对应修改正确。复制控制脚本到文件夹*/usr/local/bin/*下。

安装开机启动和计划任务，可以使用CLI控制台操作，命令和指令解释如下（当然也能用网页配置计划任务）。

{% highlight shell %}
chmod +x /usr/local/bin/nat_endpoint_checker.sh #使控制脚本能够执行
sed -i 's%^exit.*$%/usr/local/bin/nat_endpoint_checker.sh;exit 0%' /etc/rc.local #将控制脚本粘贴到启动脚本
echo '*/10 * * * * /usr/local/bin/nat_endpoint_checker.sh' >/tmp/zola #新建crontab计划任务，10分钟检查一次natapp终端地址
crontab /tmp/zola #安装计划任务
{% endhighlight %}


5）验证

保存并退出编辑的文件。重启树梅派（可以使用OpenMediaVault网页端或直接拔RPI电源）。

等待几分钟，准备在微信Server酱公众号中收经穿透的公网访问地址。

使用支持SFTP的软件/应用验证登陆。完成！

![NAS verification shot1](/assets/diy-nas-with-rpi-openmediavault/nas_verify1.png)
![NAS verification shot2](/assets/diy-nas-with-rpi-openmediavault/nas_verify2.png)
![NAS verification shot3](/assets/diy-nas-with-rpi-openmediavault/nas_verify3.png)

## 后记：

用着如此**寒酸**的设备，不禁令我回想到之前工作中EMC品牌Isilon系列中Onefs的爽快大气（当然它也会OOM，任何机器都会OOM）。2.5寸配置的SAS磁盘阵列柜，插满单价上千价格的10k转硬盘。磁盘柜之间由Intel万兆光纤交换机链接。呵，那光纤缆线，是和一个成年男人的小指一般粗的。真是“*风水轮流转*”，当初指尖连接的是总价达百万的企业级套件，潇洒闲适的大数据运维沦落为如今依靠拼凑零件，勉强苟活的“*普通人*”。

虽说搭建出的NAS写入速度慢(为了安全性考虑，组成的RAID1阵列会对数据写入速度造成很大影响)，在网络不稳定时可能会直接掉线。可是**数据本身的可靠**和**个人隐私数据的全部权利**是真正看重的部分。

其实如今（2017年）早已是没有个人隐私的时代了。不信的话可以看下本博客下insight2分类下的文章。如果我们仅看手机App这单一市场，无处不在的应用错误收集服务（我说的非常好听了，有些完全就是打着好听的旗号手机用户操作数据），再有就是广告投放服务。如果说真的想要做到个人隐私，除**不听**（不被安利）**不看**（不顾潮流）**不说**（不发意见）这类极端的做法，也是非常困难的一个过程，甚至还有国外出版社上市[相关书籍(Smart Girl’s Guide to Privacy)][privacy-guide-book]。

手机App更底层的手机操作系统，也是近年各寡头厮杀的战场（我随口就能说出已经夭折的Baidu OS, 已经改变重心的Tentent OS和已经壮大的Aliyun OS）。但一般可以分为三派：谷歌安卓手机、第三方定制安卓手机、苹果手机。谷歌安卓手机在国内的应用场景非常有限，除非*全程开代理*（至少我做不到），否则谷歌服务框架**只是个累赘**。第三方定制安卓手机，请[看这篇文章][horrified-cn-android]。苹果，苹果手机需要的是信仰。我个人用的是苹果手机，主要是受不了使用时的延迟，并且更能接受它的界面设计。可这也带来了更为封闭的手机系统，换句话说厨子（Cook）说的算。听说最近苹果在郑州造icloud国内服务器。乌呼！**终究逃离不了的国服**。[苹果的个人隐私][cook-on-apple-privacy]也算有个很搞笑的收场了。

最后附上国内某二手平台上，对本篇涉及所有硬件设备的采购价格作参考。

![Price showbox shot1](/assets/diy-nas-with-rpi-openmediavault/buy1.png)
![Price showbox shot2](/assets/diy-nas-with-rpi-openmediavault/buy2.png)
![Price showbox shot3](/assets/diy-nas-with-rpi-openmediavault/buy3.png)

> 100（树梅派）+ 40（usb hub）+ 40（usb AP）+ 2×闲置U盘 ~= 180元 (低成本NAS）

![Price showbox shot3](/assets/diy-nas-with-rpi-openmediavault/feature2.jpg)

------
## FYI

- 单电NAS， 即使用单一电源插头的NAS，我自己定义的名词。主要是为了节省排插位置。

- 知乎 - [用户画像][zhihu-personas]

- Quora - [Why does computer RAM have a lifetime warranty?][why-ram-lifetime-warranty]

- Wikipedia - [RAID1][wiki-raid1]

- Wikipedia - [NAS][wiki-nas]

- Wikipedia - [SMB][wiki-smb]

- Simple wikipedia - [FTP][sw-ftp]

- Wikipedia - [DMZ][wiki-dmz]


[wiki-dmz]: https://en.wikipedia.org/wiki/DMZ_(computing)
[cook-on-apple-privacy]: http://time.com/4262480/tim-cook-apple-fbi-2/
[wiki-raid1]: https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_1
[zhihu-personas]: https://www.zhihu.com/question/19853605
[wiki-nas]: https://en.wikipedia.org/wiki/Network-attached_storage
[wiki-smb]: https://en.wikipedia.org/wiki/Server_Message_Block
[sw-ftp]: https://simple.wikipedia.org/wiki/FTP
[xiami-code-comments]: https://web.archive.org/web/20171203072758/https://lax.v2ex.com/t/407653?p=1
[insight2-jike-userlog]: https://example.com/todo
[netease-data-leak]: https://www.zhihu.com/question/36648049
[just-get-stuff-done]: http://opinion-people-com-cn/n1/2017/0119/c1003-29035910.html
[electronic-components-fails]: https://en.wikipedia.org/wiki/Failure_of_electronic_components
[kodi-link]: https://kodi.tv/
[why-ram-lifetime-warranty]: https://www.quora.com/Why-does-computer-RAM-have-a-lifetime-warranty
[xbmc-lag]: https://www.youtube.com/watch?v=yGN0cYZvgl8
[ofcuz-it-runs-netbsd]: https://wiki.netbsd.org/ports/evbarm/raspberry_pi/
[freenas-hw-requirements]: https://www.freenas.org/hardware-requirements/
[archlinux-arm-guide]: https://archlinuxarm.org/platforms/armv6/raspberry-pi
[omv-install-guide]: https://wiki.openmediavault.org/index.php?title=Setup_Guide
[httpstatusdogs-502]: https://httpstatusdogs.com/502-bad-gateway
[omv-2-fix-502]: https://forum.openmediavault.org/index.php/Thread/7383-502-bad-gateway/?postID=69165#post69165
[qshell-link]: https://github.com/qiniu/qshell
[oss-client-link]: https://market.aliyun.com/products/53690006/cmgj000281.html
[network-helper-daemon]: https://github.com/delight09/gadgets/blob/master/network/networkchecker_daemon.sh
[natapp-register]: https://natapp.cn/
[natapp-guide]: https://natapp.cn/article/natapp_newbie
[serverchan-sckey]: http://sc.ftqq.com/?c=code
[serverchan-register]: http://sc.ftqq.com/3.version
[serverchan-bind]: http://sc.ftqq.com/?c=wechat&a=bind
[natapp-download-client]: https://natapp.cn/#download
[natapp-checker-script]: https://github.com/delight09/gadgets/blob/master/network/nat_endpoint_checker.sh
[privacy-guide-book]: https://www.nostarch.com/smartgirlsguide
[horrified-cn-android]: https://web.archive.org/web/20171030030154/https://commondatastorage.googleapis.com/letscorp_archive/archives/124165
