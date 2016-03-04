---
layout: post
title:  "ICMP报文限速"
date:   2014-11-21
categories: linux
---

### Linux对ICMP报文的限速可配置

操作系统对于ICMP报文有一个速率的限制,具体的限制参数都在
`/proc/sys/net/ipv4`文件夹.  

---

### 配置说明

以线上服务器为例,有以下两个重要参数

* icmp_ratelimit
* icmp_ratemask

这两个参数的意义可以参考 [man 7 icmp](http://man7.org/linux/man-pages/man7/icmp.7.html)

#### icmp_ratelimit  

默认是1000  
用于限制ICMP报文的速率,至于限制哪些报文的速率在icmp_ratemask设置.(至于速率是指所有的还是单独的需要再确认)  
此数值表示的意思是每次发送完ICMP报文以后延迟多长时间,单位是1ms  
例如1000的意思是:每发一个ICMP报文,1s之后才能再次发送.  
0表示没有限制  

#### icmp_ratemask

需要限制的报文类型  
此数值每个二进制位代表一种类型的报文.  
ICMP类型的定义见icmp.h,通常位置为`/usr/include/linux/icmp.h`  

```
#define ICMP_ECHOREPLY          0       /* Echo Reply                   */
#define ICMP_DEST_UNREACH       3       /* Destination Unreachable      */
#define ICMP_SOURCE_QUENCH      4       /* Source Quench                */
#define ICMP_REDIRECT           5       /* Redirect (change route)      */
#define ICMP_ECHO               8       /* Echo Request                 */
#define ICMP_TIME_EXCEEDED      11      /* Time Exceeded                */
#define ICMP_PARAMETERPROB      12      /* Parameter Problem            */
#define ICMP_TIMESTAMP          13      /* Timestamp Request            */
#define ICMP_TIMESTAMPREPLY     14      /* Timestamp Reply              */
#define ICMP_INFO_REQUEST       15      /* Information Request          */
#define ICMP_INFO_REPLY         16      /* Information Reply            */
#define ICMP_ADDRESS            17      /* Address Mask Request         */
#define ICMP_ADDRESSREPLY       18      /* Address Mask Reply           */
```

线上的一台服务器的值为6168, 二进制的第 3,4,11,12位为1,表示对

* Destination Unreachable
* Source Quench
* Time Exceeded
* Parameter Problem

进行限速

sysctl -p 生效

---

### 测试

```log
15:59:23.407311 IP 202.108.7.153.domain > 10.209.1.7.61060:  44159*- 1/6/6 A 123.125.104.197 (263)
15:59:23.416546 IP 10.209.1.7.56533 > 202.108.7.153.domain:  2090+ A? weibo.com. (27)
15:59:23.417309 IP 202.108.7.153.domain > 10.209.1.7.56533:  2090*- 1/6/6 A 123.125.104.197 (263)
15:59:23.451576 IP 10.209.1.7.60053 > 202.108.7.153.domain:  6264+ A? weibo.com. (27)
15:59:23.451587 IP 202.108.7.153 > 10.209.1.7: ICMP 202.108.7.153 udp port domain unreachable, length 63
15:59:23.453310 IP 202.108.7.153.domain > 10.209.1.7.60053:  6264*- 1/6/6 A 123.125.104.197 (263)
15:59:23.462692 IP 10.209.1.7.59476 > 202.108.7.153.domain:  64782+ A? weibo.com. (27)

省略若干行

15:59:24.435307 IP 202.108.7.153.domain > 10.209.1.7.54583:  936*- 1/6/6 A 123.125.104.197 (263)
15:59:24.459080 IP 10.209.1.7.64006 > 202.108.7.153.domain:  61537+ A? weibo.com. (27)
15:59:24.459090 IP 202.108.7.153 > 10.209.1.7: ICMP 202.108.7.153 udp port domain unreachable, length 63
15:59:24.459308 IP 202.108.7.153.domain > 10.209.1.7.64006:  61537*- 1/6/6 A 123.125.104.197 (263)
15:59:24.470375 IP 10.209.1.7.57920 > 202.108.7.153.domain:  32846+ A? weibo.com. (27)
15:59:24.471308 IP 202.108.7.153.domain > 10.209.1.7.57920:  32846*- 1/6/6 A 123.125.104.197 (263)
15:59:24.482516 IP 10.209.1.7.60647 > 202.108.7.153.domain:  8390+ A? weibo.com. (27)
```
