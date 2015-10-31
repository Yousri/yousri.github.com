---
layout: post
title: Weekly-notes-at-2014-01-06
date: 2014-01-06 22:13
comments: true
---

## 2014 年第一周
上月底在豆瓣上某篇评论最后看到这么一句话---“把要做的事情想清楚，把想好的事情做出来，把做过的事情写下来”，让自己产生一个想法接下来一年让自己尝试能坚持做到这样。争取每周或每月记录日常所遇所学问题。

## LVS 工作模式 

### 翻阅[@吴佳明\_普空](http://weibo.com/benjiaming1981) 分享的[关于LVS在淘宝环境中的应用](http://adc.alibabatech.org/ppts/up-1341918098-0.pdf)笔记

如今最常用的基本是LVS-DR模式，但其存在一定安&局限性问题，全部对外暴露：集群vip不仅配置在LVS机器，而且后端RealServer也需要绑定集群vip；而且网络要求还需要在同一个Vlan局域网内。LVS-NAT模式虽说可以解决这问题，但其自身存在需配置路由复杂且维护成本高及性能易受影响

淘宝在原生基础上综合新增了LVS-FULLNAT模式，在NAT模式的基础做了些相应改进，将客户端源包全部转化成LVS local源包与RealServer通信。具体原理可见来自其PPT：（这里存在一个问题：RealServer看不到实际客户端来源信息，全部都是变成来自LVS local的'客户端'源，所以要查看到还需要RealServer内核上打淘宝提供的补丁，而且内核要求相对较新2.6.32这2点比较麻烦）

![LVS-FULLNAT工作原理](http://yousri-pic.b0.upaiyun.com/lvs-fullnat.png)


具体安装使用结合官网说明对于  [IPVS FULLNAT and SYNPROXY](http://kb.linuxvirtualserver.org/wiki/IPVS_FULLNAT_and_SYNPROXY) 的介绍 ，后来回头再看了遍之前在小米运维部的NoOPS博客见到那篇[LVS-ospf集群](http://noops.me/?p=974)（可以说是LVS NAT Cluster的实现吧），总算理解。

通过细细翻阅分享的这话题让自己算是重新对整个LVS各工作模式情况更清晰理解，同时只想说各种工作模式好多，各大公司真是量大牛逼啊！（其实普通公司通常基本模式就够用或单FULLNAT模式足矣，基本不需动用到交换机ospf，配置什么Quagga这东西啊），没想到单单一个LVS的工作模式自己如今了解到就有：原版的 NAT/DR/TUN，淘宝新增的FULLNAT / SYNPROXY，小米的DSNAT，百度?的BigNAT，腾讯的TWG?! 其他未知的

---
### By the way

其中这PPT里对于 FULLNAT 设计考虑部分中提及到 RealServer kernel 开启 tcp_tw_recycle 用户A和B，timestamp 大癿访问成功，timestamp 小癿访问失败的问题中，曾两次帮我顺利快速的解决来自开发同事@lfeng 和@belltoy 反馈的两个问题：关于公司访问CRM慢疑似被'重置'现象；垮IDC机房访问建立连接TCP数据包传输异常。其实两次共同点：通过NAT网关（1个出口ip）访问同一服务（前者出口都是公司公网IP；后者都是走同一台机器路口出去），都是因TCP时间戳及所谓系统内核优化参数配置有关引起，详细分析解释 --- [tcp\_tw\_recycle和tcp\_timestamps导致connect失败问题](http://blog.sina.com.cn/s/blog_781b0c850100znjd.html)

所以其实未搞清楚搞懂TCP和所谓Linux内核，确实不能随便网上轻易随意copy什么内核优化参数啊，不一定是对的，不一定是适合你的。同样道理，不是安装使用上随随便便出个分析报告，说自己研究过，熟悉熟练啥的。

## TCPCopy工具使用
---
### 个人感触

刚开始每次要使用TCPCopy时，都总为找适合像自己这样新手菜鸟操作使用帮助文档而烦恼（可能自身对其深藏的功能/参数尚不够了然），第一感觉牛人都有喜欢只谈架构原理设计说明的习惯么，而不屑站在菜鸟使用立场考虑下实际基本操作帮助文档说明么？

后来发现是自己土鳖错了，应该首选看英文文档，比中文版来得更详细清晰易理解，看来技术这东西还确认应该看英文版，自以为是国人开源的产品中文文档可能会更易懂，没想到作者可谓达到高大上档次的国际范（可能技术的东西英文描述起来更容易理解的缘故吧）

---
### 使用方法

支持多种测试模式
{% highlight bash %}
    --enable-debug      #compile TCPCopy with debug support (saved in a log file)
    --enable-mysqlsgt   #run TCPCopy at mysql skip-grant-tables mode
    --enable-mysql      #run TCPCopy at mysql mode
    --enable-offline    #run TCPCopy at offline mode
    --enable-thread     #run TCPCopy server (intercept) at multi-threading mode
    --enable-nfqueue    #run TCPCopy server (intercept) at nfqueue mode 
{% endhighlight %}

这里有个麻烦就是，要不同测试模式，都需要重新编译一次TCPCopy二进制支持，这个以用户的角度应该是一次编译多次多种使用，具体由程序内部控制区分开发选择的不同参数不同测试模式。就如@timebug 所言：“其实如果非要这样的话，应该选择让用户安装一次，然后通过 bash 或者其他封装一下不同二进制，这样用户使用起来就方便了 ./tcpcopy -type xxx 类似的”

---
#### 测试服务器Target Server 配置

基本步骤：

  - 加载模块
{% highlight bash %}
	modprobe ip_queue
{% endhighlight %}
  - 设置iptables拦截TCP包
{% highlight bash %}
	iptables -I OUTPUT -p tcp --sport 3130 -j QUEUE 
{% endhighlight %}
  - 启动接收包服务端
{% highlight bash %}
	/usr/local/sbin/intercept -d
{% endhighlight %}

其中 intercept 常用参数使用说明：

{% highlight bash %}
	-x ip,ip 授权可直接访问测试服务器自身服务（不然前面设置被拦截掉）
	-b < ip_addr > 指定服务监听的IP地址，默认监听所有 0.0.0.0
	-d  后台运行
{% endhighlight %}

---
#### 线上服务器Source Server 配置

基本使用：

{% highlight bash %}
	/usr/local/sbin/tcpcopy &
{% endhighlight %}

tcpcopy 常用参数使用说明：

{% highlight bash %}
	-x sourceIP:sourcePort-targetIP:targetPort,...  #服务器ip:端口-测试服务器ip:端口,...支持多服务copy
	-i file  #保留TCP数据包，离线copy，enable-offline模式下
	-n number #放大线上流量多少倍到测试服务器，最大 1023
	-r number #引线上多少比例的流量到测试服务器上(数值范围:1~100)
{% endhighlight %}

---
#### TCPCoy 间接功能

在查找TCPCopy使用帮助过程，无意间看到这篇文章 --- [Migrating memcached](http://globaldev.co.uk/2013/01/migrating-memcached/) 其一方面借助tcpcopy引流测试外，另一方面还间接起到预热 memcached 的效果。自己不由产生之前琢磨怎么有效的对数据库做预热的困惑可谓有解 --- 借助tcpcopy引流功能；支持 MySQL 服务，来对数据库 MySQL 的InnoDB内存池做线下预热操作，相比其他预热方式看似来得更靠谱些，因为其是引用线上真实生产环境请求过来的相当于用户直接的访问。

看来TCPcopy不仅让测试更真实，让开发更大胆调试，让运维更方便操作，看来后续还有可能尝试着拿来作为类似 Cache 功能的预热工具啊。

## 运维工具

13年9月份自己曾在微博感慨过点愚见：工具终究是辅助，自动化快速轻松的前提是先需要将服务标准化规范化服务化，及对业务服务了然于心后积累成型的。同时面对实际问题的定位分析解决才是更关键吧，否则最终只能说是个操作员吧。

个人分别体验使用过salt和ansible两者工具后，最大的感受未来基本趋势是：Salt其强大确实更偏向走系统化路线，是作为服务状态/配置管理维护的首选，加上其拥有齐全的api接口更易按需定制成平台化吧（同样具备后者实时管理等功能）；而相对而言ansible比较适用于日常实时操作管理/维护发布部署。

---
## 兴趣与态度

"如果一个人对所做的事情感兴趣，那么自然愿意投入时间，与是否工作无关，与是否IT行业无关。"

说到心坎上，只有兴趣的东西才是长久的，技术能力其实并非是最关键，态度/责任更胜一切，最近半年最大的感受是：有心学习搞懂搞明白其实技术也就那么回事而已，遇到问题用心弄清楚后便会发现并不是想象中那么神秘，而兴趣/态度/责任完全不是靠培养学习能成就的。

## Want To Do
 1. 重拾iptables
 2. TCP协议/网络规划
 3. MySQL参数整理/监控

