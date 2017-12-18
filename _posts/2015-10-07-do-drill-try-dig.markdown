---
layout: post
title: 先用drill之后再考虑dig
category: toolchain
tags: [network, migrate, dns]
image: /assets/do-drill-try-dig/feature.png
description: 两者工具在使用时会有一些差别。对于运维人员来说，dig无疑还是更好的选择，资料更多、对脚本更加友善、支持的发行版也更全。可如果考虑到dig工具背负的软件库更多，对比drill优雅的ldns。相信读者能作出自己的选择。
---

## TL;DR

两者工具在使用时会有一些差别。对于运维人员来说，dig无疑还是更好的选择，资料更多、对脚本更加友善、支持的发行版也更全。可如果考虑到dig工具“**背负**”的`libbind9.so``libisc.so`等等，对比drill优雅的`libldns.so`。相信你能作出自己的选择。

## 引子

随着`dig`工具与`bind`服务器的[项目内高度耦合][archlinux-ldns-migration]，使得dig工具的二进制包变得**非常臃肿**，拿ArchLinux发行版来讲，约[6MB][archlinux-package-bindtools]：[1MB][archlinux-package-ldns]的区别。
但是4年过去了，网上铺天盖地的仍是dig教程，所以就有了这篇关于drill工具的使用简介。

## 简单说明

由NLnetlabs发起的[ldns项目][ldns-link]，自1.0.0版本起，drill工具与ldns代码库一并分发。drill工具继承了ldns库的特点，是对于新RFC标准能更快的支持。

在drill工具的手册(man page)中还有一个[有趣的说明][man-drill]：

>  The name drill is a *pun* on dig. With drill you should be able to get even more information than with dig.

## 一般使用

### 1. 查A记录

> USAGE: dig \<domain\>

{% highlight Plain Text %}
dig djh.im

; <<>> DiG 9.11.2 <<>> djh.im
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43247
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;djh.im.				IN	A

;; ANSWER SECTION:
djh.im.			15	IN	A	a.b.c.d

;; Query time: 1 msec
;; SERVER: 192.168.1.1#53(192.168.1.1)
;; WHEN: Wed Dec 06 21:27:43 CST 2017
;; MSG SIZE  rcvd: 40


{% endhighlight %}

> USAGE: drill \<domain\>

{% highlight Plain Text %}
drill djh.im
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 51351
;; flags: qr rd ra ; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0 
;; QUESTION SECTION:
;; djh.im.	IN	A

;; ANSWER SECTION:
djh.im.	282	IN	A	a.b.c.d

;; AUTHORITY SECTION:

;; ADDITIONAL SECTION:

;; Query time: 1 msec
;; SERVER: 192.168.1.1
;; WHEN: Wed Dec  6 21:23:17 2017
;; MSG SIZE  rcvd: 40
{% endhighlight %}

查A记录是dns工具最常用的功能之一，两个工具的使用方式相同。值得注意的是`dig v9.11.2`在回显的第一行和最后一行分别插入了空行，
同时会在查询记录时自动加上`[ad 查询标识位][dig-info]`，用以查询通过DNSSEC验证的记录(后将详细演示)，同时dig工具的回显更罗嗦(verbose)默认包含工具版本号。

### 2. 查其他域名记录

> USAGE: drill \<record-type\> \<domain\> # dig \<record-type\> \<domain\>

两个工具使用方法一致，这里不作演示。

### 3. 追踪DNS查询过程

> USAGE: dig +trace \<domain\>

{% highlight Plain Text %}
dig +trace djh.im

; <<>> DiG 9.11.2 <<>> +trace djh.im
;; global options: +cmd
.			82727	IN	NS	c.root-servers.net.
.			82727	IN	NS	k.root-servers.net.
.			82727	IN	NS	d.root-servers.net.
.			82727	IN	NS	l.root-servers.net.
.			82727	IN	NS	i.root-servers.net.
.			82727	IN	NS	g.root-servers.net.
.			82727	IN	NS	m.root-servers.net.
.			82727	IN	NS	b.root-servers.net.
.			82727	IN	NS	h.root-servers.net.
.			82727	IN	NS	e.root-servers.net.
.			82727	IN	NS	f.root-servers.net.
.			82727	IN	NS	a.root-servers.net.
.			82727	IN	NS	j.root-servers.net.
;; Received 251 bytes from 192.168.1.1#53(192.168.1.1) in 6 ms

im.			172800	IN	NS	barney.advsys.co.uk.
im.			172800	IN	NS	pebbles.iom.com.
im.			172800	IN	NS	hoppy.iom.com.
im.			172800	IN	NS	ns4.ja.net.
im.			86400	IN	NSEC	imamat. NS RRSIG NSEC
im.			86400	IN	RRSIG	NSEC 8 1 86400 20171219050000 20171206040000 46809 . HcAi5q1FYZpWJUC+LXe12k6QPLAkBM9h9sKvrxrfWKulDUKu6ijXgQn+ 4X9mmBAidGxvVEm+Oad8DIfz20+3mOfHaixXsxqpSGvtK7xYSv9WbMN+ uDloz33vGMz1qO06SfuBYORjV0VWZ7HK1aVSuvKTJLRAV8ZbITlJpBsI NgIqBuibodM60DubQ+i9vMI7cqiZdop0UirFJglg9m9SL4QAGySAdOZk i1klEu8r5UAc2KvqRsOS15U2uTBFXbFueeB8XmmVx2q0XOtEO639abn+ G9cDjwTVcMGH33gavhjbNA0tV5QfWflzylbEj0cQ3OFpiVU7kpLWKcb9 EGMX4w==
;; Received 548 bytes from 192.36.148.17#53(i.root-servers.net) in 37 ms

;; expected opt record in response
djh.im.			259200	IN	NS	ns1.domain-resolution.net.
djh.im.			259200	IN	NS	ns2.domain-resolution.net.
djh.im.			259200	IN	NS	ns3.domain-resolution.net.
djh.im.			259200	IN	NS	ns4.domain-resolution.net.
;; Received 117 bytes from 217.23.163.140#53(hoppy.iom.com) in 429 ms

djh.im.			300	IN	A	a.b.c.d
;; Received 51 bytes from 162.88.60.47#53(ns2.domain-resolution.net) in 301 ms

{% endhighlight %}

> USAGE: drill -T \<domain\>

{% highlight Plain Text %}
drill -T djh.im
.	518400	IN	NS	c.root-servers.net.
.	518400	IN	NS	f.root-servers.net.
.	518400	IN	NS	l.root-servers.net.
.	518400	IN	NS	a.root-servers.net.
.	518400	IN	NS	k.root-servers.net.
.	518400	IN	NS	d.root-servers.net.
.	518400	IN	NS	g.root-servers.net.
.	518400	IN	NS	h.root-servers.net.
.	518400	IN	NS	j.root-servers.net.
.	518400	IN	NS	m.root-servers.net.
.	518400	IN	NS	b.root-servers.net.
.	518400	IN	NS	e.root-servers.net.
.	518400	IN	NS	i.root-servers.net.
im.	172800	IN	NS	barney.advsys.co.uk.
im.	172800	IN	NS	hoppy.iom.com.
im.	172800	IN	NS	pebbles.iom.com.
im.	172800	IN	NS	ns4.ja.net.
djh.im.	259200	IN	NS	ns1.domain-resolution.net.
djh.im.	259200	IN	NS	ns2.domain-resolution.net.
djh.im.	259200	IN	NS	ns3.domain-resolution.net.
djh.im.	259200	IN	NS	ns4.domain-resolution.net.
djh.im.	300	IN	A	a.b.c.d
{% endhighlight %}

`dig v9.11.2`在回显的第一行和最后一行分别插入了空行，dig工具的回显更罗嗦(verbose)并且默认返回了DNSSEC验证相关的NSEC和RRSIG记录。
同时我们在使用中能明显感受到`drill v1.7.0`工具在traceDns记录时返回结果**更慢**，原因不明。

### 4. 用TCP协议请求并指定DNS服务器端口

> USAGE: dig +tcp -p \<port\> \<domain\> @dns-server-ip

{% highlight Plain Text %}
dig +tcp -p 443 djh.im @208.67.220.220

; <<>> DiG 9.11.2 <<>> +tcp -p 443 djh.im @208.67.220.220
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1718
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 16384
;; QUESTION SECTION:
;djh.im.				IN	A

;; ANSWER SECTION:
djh.im.			300	IN	A	a.b.c.d

;; Query time: 247 msec
;; SERVER: 208.67.220.220#443(208.67.220.220)
;; WHEN: Wed Dec 06 23:04:20 CST 2017
;; MSG SIZE  rcvd: 51

{% endhighlight %}

> USAGE: drill -t -p \<port\> \<domain\> @dns-server-ip

{% highlight Plain Text %}
drill -t -p 443 djh.im @208.67.220.220
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 7686
;; flags: qr rd ra ; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0 
;; QUESTION SECTION:
;; djh.im.	IN	A

;; ANSWER SECTION:
djh.im.	300	IN	A	a.b.c.d

;; AUTHORITY SECTION:

;; ADDITIONAL SECTION:

;; Query time: 421 msec
;; SERVER: 208.67.220.220
;; WHEN: Wed Dec  6 23:03:12 2017
;; MSG SIZE  rcvd: 40
{% endhighlight %}

我们能从两个工具的手册(man page)中得知，drill工具的DNS查询选项(queryopt)是形如`-t`，区别于dig工具的形如`+tcp`。

> FYI:
SYNOPSIS versus
 - dig [@server] [-b address] [-c class] [-f filename] [-k filename] [-m] [-p port#] [-q name] [-t type] [-v] [-x addr]
           [-y [hmac:]name:key] [-4] [-6] [name] [type] [class] [queryopt...]
 - drill [ OPTIONS ] name [ @server ] [ type ] [ class ]


### 5. 查询IP的PTR记录(域名的反向查询)

> USAGE: dig -x x.y.z.o

{% highlight Plain Text %}
dig +short -x a.b.c.d
domain.tld
{% endhighlight %}

> USAGE: drill -x # drill[仍没有][bugzilla-link]适配于脚本的简短输出参数)

{% highlight Plain Text %}
drill -x a.b.c.d | grep PTR | tail -n 1 | awk '{print $5}'
{% endhighlight %}

### 6. 查询DNSSEC验证的记录

> USAGE: dig \<domain\> @dnssec-enabled-server

{% highlight Plain Text %}
# 查询DNSSEC配置错误的域名
dig sigfail.verteiltesysteme.net @8.8.8.8

; <<>> DiG 9.11.2 <<>> sigfail.verteiltesysteme.net @8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 59526
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;sigfail.verteiltesysteme.net.	IN	A

;; Query time: 634 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Wed Dec 06 23:29:15 CST 2017
;; MSG SIZE  rcvd: 57

# 查询DNSSEC正确配置的域名
dig sigok.verteiltesysteme.net @8.8.8.8  
                                                                                
; <<>> DiG 9.11.2 <<>> sigok.verteiltesysteme.net @8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56950
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;sigok.verteiltesysteme.net.	IN	A

;; ANSWER SECTION:
sigok.verteiltesysteme.net. 59	IN	A	134.91.78.139

;; Query time: 648 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Wed Dec 06 23:31:40 CST 2017
;; MSG SIZE  rcvd: 71

{% endhighlight %}

> USAGE: drill \<domain\> @dnssec-enabled-server

{% highlight Plain Text %}
# 查询DNSSEC配置错误的域名
drill sigfail.verteiltesysteme.net @8.8.8.8  
;; ->>HEADER<<- opcode: QUERY, rcode: SERVFAIL, id: 17663
;; flags: qr rd ra ; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0 
;; QUESTION SECTION:
;; sigfail.verteiltesysteme.net.	IN	A

;; ANSWER SECTION:

;; AUTHORITY SECTION:

;; ADDITIONAL SECTION:

;; Query time: 637 msec
;; SERVER: 8.8.8.8
;; WHEN: Wed Dec  6 23:32:21 2017
;; MSG SIZE  rcvd: 46
# 查询DNSSEC正确配置的域名
drill sigok.verteiltesysteme.net @8.8.8.8  
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 18051
;; flags: qr rd ra ; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0 
;; QUESTION SECTION:
;; sigok.verteiltesysteme.net.	IN	A

;; ANSWER SECTION:
sigok.verteiltesysteme.net.	59	IN	A	134.91.78.139

;; AUTHORITY SECTION:

;; ADDITIONAL SECTION:

;; Query time: 632 msec
;; SERVER: 8.8.8.8
;; WHEN: Wed Dec  6 23:34:06 2017
;; MSG SIZE  rcvd: 60
{% endhighlight %}

`dig v9.11.2`在回显的第一行和最后一行分别插入了空行，回显更罗嗦(verbose)并且会在*了解到*DNS服务器支持DNSSEC时*自动*加入`ad 请求标识位`。两工具都在没有额外参数的情况下正确反应了服务器的DNSSEC应答。
值得注意的是`dig v9.11.2`并不能很好的判断DNS服务器是否支持DNSSEC功能，至少博主路由器自带的DNS缓存服务器成功欺骗了dig，让它用`ad 请求标识位`请求后续DNS记录。

## FYI

- Debian官方软件库索引 - 提供dig工具的[dnsutils][debian-package-dnsutils]

- ArchLinux官方软件库索引 - 提供dig工具的[bind-tools][archlinux-package-bindtools]

- Debian官方软件库索引 - 提供drill工具的[ldnsutils][debian-package-ldnsutils]

- ArchLinux官方软件库索引 - 提供drill工具的[ldns][archlinux-package-ldns]

- StackExchange - [Why do Unix-heads say “minus”?][so-why-unix-options]

[bind homeurl]: https://www.isc.org/downloads/bind/
[ldns-link]: https://www.nlnetlabs.nl/projects/ldns/
[archlinux-ldns-migration]: https://www.archlinux.org/todo/dnsutils-to-ldns-migration/
[bugzilla-link]: https://www.nlnetlabs.nl/bugs-script/show_bug.cgi?id=550#c1
[man-drill]: https://www.freebsd.org/cgi/man.cgi?query=drill&sektion=1
[dig-info]: http://www.perdisci.com/useful-links/dig-info
[archlinux-package-bindtools]: https://www.archlinux.org/packages/extra/x86_64/bind-tools/
[debian-package-dnsutils]: https://packages.debian.org/stretch/dnsutils
[so-why-unix-options]: https://unix.stackexchange.com/questions/2212/why-do-unix-heads-say-minus
[archlinux-package-ldns]: https://www.archlinux.org/packages/core/x86_64/ldns/
[debian-package-ldnsutils]: https://packages.debian.org/stretch/ldnsutils