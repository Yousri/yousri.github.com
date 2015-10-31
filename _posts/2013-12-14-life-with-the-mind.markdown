---
layout: post
title: 工作随记
date: 2013-12-14 19:30
comments: true
---

## 身心体会
---

确实啊，这近一个月来就自己而言跟打战似的，各种忙碌、通宵熬夜导数据。前要盯流量看监控，后要顾集群服务存储兼干DBA活；左边答疑技术支持，右边请教开发确认机制；对外细到服务销售/客户上传数据，对内还需受命服务于两边项目；说白基本是“白天作战UpYUN，晚上转移Huaban”、“白天要跑机房，晚上得理配置，半夜上岗无证DBA临时工”的节奏一点不够分。其实领导会上所言的关于老大事情交接，其实不过是岗位的交接罢了，某种程度并不包括工作内容任务。

这样也不算什么坏事…而且也不是说仅仅是忙累而才不爽。小公司就是这么点优势---只要主动多去参与你就可以学习更多积累更多。其实不爽的地方关键在于：那些整天喊着“共同创业，共事”的领导们就只会口头忽悠说着些多好听的话，而实际上对于员工而言基本是看不到自己所创造的价值所在（最基本的业务销售情况无人知晓）；做基础服务不舍得投入基础设施，一心只想考虑看要在技术人/程序服务上榨干所有，倒是很舍得投入找来各种摆饰好看撑场面的“展示品”，需要有这样的阶段没错，但关键至少目前还没到吧！如今一定程度上必备基本的容量规划预估是值得考虑，多少次都是被动的等项目迫在眉睫才到处要资源的，适当的冗余备置批资源想必还抵不上一个“展示品”一个月的费用吧?！

身为领导们很正常的一贯都只喜好那些好被自己降服、唯命是从、时刻追随自己等方面的人。可惜自己情商低下，属比较不正常的另类，言论随心、直言不讳、单刀直入、不降服于任何人，不想压抑内心不爽、不想虚假处事，只想做真实自己，哪怕自己知晓这样最终受伤的是自己。

其实就公司目前而言，管理方式及氛围而言都相当令人不敢苟同啊。

最后，只能用朋友曾给的一条共勉评论---“要么忍，要么滚”来终结身心体会，以这样方式记录下自己人生的点滴就好。

## 无证上岗DBA临时工
---
毕业至今多年，经历过的几家公司部门貌似都没有专职的DBA这岗位角色，可能自己处过都是小公司吧，这样的好处就是运维可以负责兼顾DBA维护的角色，学习到更多实战东西，之前在厦门就是如此，可惜自从来杭州后新接触更多是CDN的东西，所以后端DB基本就没再碰过，没想到这近一个月来能再次捡起几次熬夜都和数据库维护有关啊。

主要包括是 MySQL 维护及版本升级以及 Redis 服务实例的迁移分离，都是由于业务规模的速度增长而做的完善或扩充：MySQL 之前配置考虑不够周到及业务量增长触发到版本的某个BUG；花瓣 Redis 集群服务内存告警做相应的扩容迁移分离。

### MySQL维护记录：共享表空间-->独享表空间;版本升级5.5.19-->5.5.31
---

#### 共享表空间--->独享表空间
---
上次迁移更新数据库还是一年多前，当时自己就只负责安装配置数据库所需的平台环境（回想起那时都是泪啊！），至于 MySQL 的参数配置还是老大来，当时也没有考虑太周到，采取的是共享表空间方式。（时间的推移发现磁盘空间回收释放成一个问题），故决定重整调整配置开启设置使用独享表空间，方便后续对单表做表分析回收释放空间。（采用重新导出导入数据方式来做变更），主要配置修改：

{% highlight bash %}
	vi /etc/my.cnf
	innodb_file_per_table =1
{% endhighlight %}

至于共享表空间和独享表空间的比较，个人无权评论，因为自己是个无证上岗的DBA临时工，建议可以请教相关专职DBA或也可以看看[MySQL 独立表空间 VS 共享表空间](http://www.iamcjd.com/?p=1294)

#### 升级 MySQL 数据库版本：5.5.19--->5.5.31
---

当时主要有2个疑问需事先确认：

1. a数据需不需要重新导出导入
2. b升级后同现有的相对较低版本与升级后高版本的同步兼容性问题

为了确认验证上面这两个问题，找了2台之前下线的服务器（事先已经有部署有 MySQL5.5.19版本）做了下基本逻辑的测试：对于第一个问题，测试大致

* 安装部署新版本 MySQL5.5.31
{% highlight bash %}
	tar zxf mysql-5.5.31-linux2.6-x86_64.tar.gz
	/usrl/local/mysql/bin/mysqladmin -uroot -p shutdown #关闭旧版本MySQL服务
	mv /usr/local/mysql /usr/local/mysql5.5.19
	mkdir /usr/local/mysql
	mv mysql-5.5.31-linux2.6-x86_64/* /usr/local/mysql/
	mkdir /usr/local/mysql/log
	chown -R mysql.mysql /usr/local/mysql
{% endhighlight %}
* 启动新版本 MySQL 服务
{% highlight bash %}
	/usr/local/mysql/bin/mysqld_safe --user=mysql&

	mysql> select version();
	+------------+
	| version()  |
	+------------+
	| 5.5.31-log |
	+------------+
	1 row in set (0.00 sec)
{% endhighlight %}

* 执行 mysql\_upgrade 调整修复数据兼容性 （按规范流程来做事）

{% highlight bash %}
	[root@hadoop01 local]# /usr/local/mysql/bin/mysql_upgrade -uroot -p
	Enter password:
	Looking for 'mysql' as: /usr/local/mysql/bin/mysql
	Looking for 'mysqlcheck' as: /usr/local/mysql/bin/mysqlcheck
	Running 'mysqlcheck' with connection arguments: '--port=3306' '--socket=/tmp/mysql.sock'
	Running 'mysqlcheck' with connection arguments: '--port=3306' '--socket=/tmp/mysql.sock'
	mysql.columns_priv                                 OK
	mysql.db                                           OK
	mysql.event                                        OK
	mysql.func                                         OK
	mysql.general_log                                  OK
	mysql.help_category                                OK
	mysql.help_keyword                                 OK
	mysql.help_relation                                OK
	mysql.help_topic                                   OK
	mysql.host                                         OK
	mysql.ndb_binlog_index                             OK
	mysql.plugin                                       OK
	mysql.proc                                         OK
	mysql.procs_priv                                   OK
	mysql.proxies_priv                                 OK
	mysql.servers                                      OK
	mysql.slow_log                                     OK
	mysql.tables_priv                                  OK
	mysql.time_zone                                    OK
	mysql.time_zone_leap_second                        OK
	mysql.time_zone_name                               OK
	mysql.time_zone_transition                         OK
	mysql.time_zone_transition_type                    OK
	mysql.user                                         OK
	Running 'mysql_fix_privilege_tables'...
	OK
{% endhighlight %}

后来了解到 mysql\_upgrade 这命令所做的事情主要是包括：

1. mysqlcheck –check-upgrade –all-databases –auto-repair
2. mysql\_fix\_privilege\_tables
3. mysqlcheck –all-databases –check-upgrade –fix-db-names –fix-table-names

故是建议每一次的升级过程中，mysql\_upgrade这个命令最好也应该去执行，通过mysqlcheck命令检查表是否兼容新版本的数据库同时作出修复，还有个很重要的作用就是使mysql\_fix\_privilege\_tables命令去升级权限表。

更多详细升级帮助说明详见：[MySQL 官网升级文档](http://dev.mysql.com/doc/refman/5.5/en/upgrading.html)

对于第二个疑问：升级后同现有的相对较低版本间的同步兼容性问题，做如下测试：

先在刚刚已经升级的 MySQL 版本为5.5.31 server1 和先前下架但未升级版本对应 MySQL 版本为5.5.19 的server2，这 2 台配置为互为主主。测试如下操作：

1. 在 server1 的 test 库创建一张表 t1 字段为 id / name ，查看 server2 上的 test 库表情况，正常；
2. 在 server1 刚创建的 t1 表 插入一条记录 ，到 server2 上查询，结果正常；
3. 在 server2 上删除刚刚在 server1 上插入的记录，到 server1 上查看，结果正常；

查看前后 slave status 变化是否正常，MySQL自身log日志记录情况，可以判断确定同步疑问也是没问题的。

注：随手分享篇关于 [Linux performance tuning tips for MySQL](http://www.mysqlperformanceblog.com/2013/12/07/linux-performance-tuning-tips-mysql/)

---
### Redis 实例分离迁移
---
Redis目前还没有成熟的集群方案，虽说2.8版本已经支持且标记为Stable版本，但一直还是处于更新迭代阶段，拿来线上环境使用还是显得不是很靠谱，个人的看法是：适当追新是应该的没错，但感觉还是不要太够于激进追新，那样只会是更冒险且是小白鼠吧。这次迁移分离主要是随Huaban项目redis服务内存占用增长使得之前每台分配的实例数显得紧张，到了必要的扩容分离处理（不大不小阶段吧）。主要备录几点刚接触过程了解到的点：

1. 迁移分离方式--复制功能，一定要用到内存快照，所以需要充当master库所在服务器相等量的内存容量闲置;
2. master和slave断开后重新建立连接都得重新完整的将整个快照做一次同步传输，没有类似MySQL的位置标记概念（增量备份）;
3. 从2.6版本后，slave角色默认是开启只读的，这个如果没有事先使用 config set slave-read-only no 关闭的话，在做分离迁移完后应用端切换到新的上面后，会遇到 redis slave read only 的提示，以致新的写入失败丢失;
4. 确保本地磁盘空间足够（基本不会遇到但是不好说），因为内存快照过程redis进程是会dump一份临时的temp.rdb文件到磁盘再传输到slave服务器上加载进内存中，如果master磁盘不足的话会出现快照恶性循环失败启动新的slave进程;
5. so,用前需要事先做好业务的预估容量规划及后期的扩容问题。

还是那句话，个人始终只是临时工就只了解些外表的一些皮毛而已，对于看得懂源码的人想了解可以读读源码去，哈哈。这里随手分享一两篇个人觉得写得不错的牛人微博或文章大家可以看看---[田琪的微博](http://weibo.com/1779195673/zF0tmG4fT8)和[Redis复制与可扩展集群搭建](http://www.cnblogs.com/chenying99/archive/2012/06/14/2548730.html)

最近还尝试接触了个新老玩意---[TCPCopy](https://code.google.com/p/tcpcopy/)

---
## 不忘生活
---
工作在忙在累，也不能忘周末的休闲骑行生活，工作最终都是为了享受生活，故不能忘。就如自己的一个愿望是：哪一天可以申请回老家乡下远程办公，看似很简单的愿望却至今未能遇到有一家公司肯批准（很多都觉得不放心、会偷懒；成为特例影响他人，但自己要求想法就是如此另类，因为个人觉得完全没任何问题。）所以一直坚持着和小伙伴们一起踩着脚踏车到周边游逛。有骑行随记图文有证据分享，如：

1. [萧山无人村挑战盘山而上](http://www.omoway.com/2013/11/riding-life-to-wurencun/)
2. [奔杨家赏银杏](http://www.omoway.com/2013/11/riding-life-to-yangjiacun-for-yinxing/)
3. [甘岭水库，长乐林场](http://www.omoway.com/2013/11/riding-life-to-ganling-and-changle/)

近年来各地污染指数各种爆表，搞得本来要去徒步爬山都不能……

