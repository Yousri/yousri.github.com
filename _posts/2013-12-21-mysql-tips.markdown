---
layout: post
title: 关于近阶段对MySQL维护中学习小结
date: 2013-12-21 22:30
comments: true
---

## 前奏感触
---

最近一段时间无证上岗充当 DBA 临时工接触 MySQL 一些维护操作，总体感觉挺好，时常能够遇到'新'问题'挑战'，所以边尝试边学习边查手册，边请教专业的同事朋友边和小伙伴们闲聊分享新发现，有种自己刚毕业入社会那会入门运维的感觉。DBA确实感觉挺好，有意思有挑战，唯一不好的一点就是：个人感觉很多都得在夜里才能操作维护。说来也惭愧自己不是真正计算机类毕业的如今却实打实的干着IT基础服务这行，所以就自己而言，更多涉及广度缺少深度（系统的学习深入的研究），遇到问题时排查原因分析定位基本都还不成问题，至于解决方案措施有时即便有思路有想法却也很容易比较没了底气自信，所以没优势，这样通常也不易被看好看重。

在维护 MySQL 过程中，前前后后遇到几次坑及个人困惑，也此处理过程了解些使用 MySQL 数据库过程可能需要留意的几个地方，简单记录下希望对后来人或像自己这样的临时工有所帮助吧。至于那些关于 MySQL 主从、主主、还是主主各带从的关系这里不多记录，基本就都那样。

## MySQL 使用考虑点 
--- 

### 一、MySQL平台环境选择
---

MySQL平台环境：硬件、系统等

#### 硬件：

1. CPU
2. 内存：尽可能的大，主要是InnoDB中有个内存池 innodb\_buffer\_pool ，尽量将热数据预加载保存在内存里以便尽量多的从内存里读取数据。
3. 磁盘：提供磁盘 I/O 读写速度性能，SSD现在很普遍，有钱的都上FusionIO，SSD可使用Raid 如：系统分区可选Raid1，数据分区则用Raid10的。
4. d网卡：低延迟，传输/同步

#### 系统环境：

1. EXT4 / XFS / Btrfs（这个目前应该还不很少用线上吧）
2. I/O调度算法：deadline

---
### 二、MySQL 自身配置
---

存储引擎选择，目前基本都是InnoDB引擎，或者各大公司基于InnoDB基础按需二次开发，所以就以个人这段时间在InnoDB引擎下遇到的几个问题来描述

#### 1、共享表空间与独享表空间的选择（最好事先选择好，建议独立表空间性能且不说就方便维护）

{% highlight bash %}
	innodb_file_per_table=1
{% endhighlight %}

---
#### 2、InnoDB 内存池：将数据尽量多的保存在内存里，更多的减少磁盘读写操作，提供性能。

{% highlight bash %}
	innodb_buffer_pool_size = 80% * Memory
{% endhighlight %}

InnoDB 内存池这东西有个需要注意就是：数据库新启动或重启，内存会被自动释放，所以都需要重新进行数据预热，将磁盘上的所有数据缓存到内存中，这个时候数据库压力负载都会异常的飙升，特别如果是在访问请求量高峰时，可能直接引发崩溃，这个自己在最后一次维护没留意就采坑中招了。详见下面两台分别是被重启过和线上的innodb\_buffer\_pool相关的配置信息区别： Innodb\_buffer\_pool\_pages\_free 值便是内存池空闲页面数，第一个是刚重启起来所以很多都尚未预热，第二个是线上服务已经为0即所有的innodb\_buffer\_pool都已经使用上。

{% highlight bash %}
	mysql> SHOW GLOBAL STATUS LIKE '%innodb_buffer_pool_pages%';
	+---------------------------------------+-------------+
	| Variable_name                         | Value       |
	+---------------------------------------+-------------+
	| Innodb_buffer_pool_pages_data         | 2209585     |
	| Innodb_buffer_pool_pages_dirty        | 147528      |
	| Innodb_buffer_pool_pages_flushed      | 1953689     |
	| Innodb_buffer_pool_pages_free         | 2484453     |
	| Innodb_buffer_pool_pages_misc         | 24553       |
	| Innodb_buffer_pool_pages_total        | 4718591     |
	+---------------------------------------+-------------+
	6 rows in set (0.00 sec)
	
	mysql> SHOW GLOBAL STATUS LIKE '%innodb_buffer_pool_pages%';
	+---------------------------------------+---------------+
	| Variable_name                         | Value         |
	+---------------------------------------+---------------+
	| Innodb_buffer_pool_pages_data         | 4545938       |
	| Innodb_buffer_pool_pages_dirty        | 96512         |
	| Innodb_buffer_pool_pages_flushed      | 1609426350    |
	| Innodb_buffer_pool_pages_free         | 0             |
	| Innodb_buffer_pool_pages_misc         | 172653        |
	| Innodb_buffer_pool_pages_total        | 4718591       |
	+---------------------------------------+---------------+
	6 rows in set (0.00 sec)
{% endhighlight %}

至于如何加快预热速度，Google了下有各种方法，要么是限制于XtraDB要么是要基于二次开发的引擎上，文章可见[快速预热Innodb Buffer Pool的方法](http://www.mysqlsystems.com/2011/07/quickly-warm-up-innodb-buffer-pool.html)。个人愚见感觉即便模拟访问也不是很靠谱，因为模拟的东西本身和用户访问不太一致，不可能把磁盘数据都写入到内存里，因为这个内存池大小有限。

据说 MySQL5.6 版本支持在正常关闭服务的情况下支持事先做好预热备份的配置，相应配置（摘自网络,个人暂未测试）

{% highlight bash %}
	innodb_buffer_pool_dump_at_shutdown = 1	#在关闭时把热数据dump到本地磁盘。
	innodb_buffer_pool_dump_now = 1		#采用手工方式把热数据dump到本地磁盘。
	innodb_buffer_pool_load_at_startup = 1	#在启动时把热数据加载到内存。
	innodb_buffer_pool_load_now = 1		#采用手工方式把热数据加载到内存。
{% endhighlight %}

在关闭MySQL时，会把内存中的热数据保存在磁盘里ib\_buffer\_pool文件中，位于数据目录下。

---
#### 3、磁盘写入操作调节

问题：临时从线上配置从库准备作为导数据用的时遇到同步严重延迟的现象

定位：同步追了一晚上发现 Seconds\_Behind\_Master 数值还是有点出乎的大，事先对MySQL 不熟所以只能怀疑延迟，网上查了下发现这个值是：SQL thread 与 I/O thread 之间的差值(摘自[Seconds\_Behind\_Master 解析](http://hi.baidu.com/ytjwt/item/02ebd0c0811f3952bcef690a))，有两种情况可能会引起这个大差值：网络延迟或执行写入I/O压力；看了下 master 的 binlog 都是很快传递到 slave 上且加上从库服务器磁盘I/O确实超负荷（虽然写入数据不大却过于频繁原因，留一个疑问：不知当时的读写频率次数是多少），为了更确切的确定，在[@nettedfish](http://weibo.com/nettedfish)的指点帮助下"在主库的对应db上，临时创建一个heartbeat 表，插入时间戳，在从库上，过一段时间取出来，与当前的时间戳想比较下"，确实如show slave status\G;显示的延时那时长。

解决：反馈给老大，很快指点自己试着将 innodb\_flush\_log\_at\_trx\_commit 由线上的配置为 1 临时改为 0，因为考虑是临时作为迁移导数据用的从库，不是线上生产环境影响不大，果然让其自动定期写入磁盘，效果明显：很快便追上同步且磁盘负荷恢复正常。

查证：这个参数原来对于磁盘读写性能影响甚大啊

{% highlight bash %}
	innodb_flush_log_at_trx_commit = 0	#不write()，也不fsync()。每秒同时执行一次write()和fsync();
	innodb_flush_log_at_trx_commit = 1	#同步IO,确定将redo log同步写入磁盘，即write()，又fsync().
	innodb_flush_log_at_trx_commit = 2	#只write()，不fsync()，即不确认写入磁盘，可能还在OS page cache
{% endhighlight %}

注：
* write():用户态进程mysqld内存空间log buffer pool的数据复制到内核态线程内存空间中(即page cache),此时,日志数据并没有写至磁盘设备,
* fsync():才会把页面缓存中的日志数据同步到磁盘设备上

由此可见：
1. 设置为1,易产生大量同步I/O操作。每次同步I/O操作，都会阻止I/O调用，直到数据被真正写回磁盘(写磁盘较写内存慢很多)，这样就会显著地降低InnoDB每秒可以提交的事务数。
2. 设置为0或2,意味着更少的调用fsync(),最多有可能会丢失1s的事务，所以对事务要求不高业务环境下，其实完全可以设置值2,甚至值0来减少事务引起的磁盘I/O。

Ps:后来意识到那些监控 MySQL 主从同步是否延时是不是就是根据查看 Seconds\_Behind\_Master 这个值来确定的呢?!

其实关于磁盘读写操作相关的还有个参数是：sync\_binlog --- InnoDB同步bin log至磁盘的频率

{% highlight bash %}
	sync_binlog = 0		#默认值是0。不调用fsync()，依赖于OS调度。
	sync_binlog = 1		#每次commit，都要求进行一次fsync()同步I/O操作。
{% endhighlight %}

---
#### 4、定期分析表，回收空间，间接优化表性能

InnoDB 引擎数据即便删除也不会被自动释放，所以如果用共享表空间那似乎意味需要定期重导数据操作?！对于独立表空间虽然delete操作同样不会回收已用空间，但至少还可以通过定期分析表，实现手动回收。

{% highlight bash %}
	optimize NO_WRITE_TO_BINLOG table tablename;
{% endhighlight %}

需要注意的是：
1. optimize执行过程会锁表，最好在slave上执行;
2. optimize表分析实质是先将所有数据导入到新的innodb文件中，所以数据分区空余空间必须要大于该表文件
3. optimize命令默认会写入binlog同步到其从库，所以需要加参数 NO\_WRITE\_TO\_BINLOG

---
#### 5、剩下的估计开发比较清楚

其他注意事项：
1. a定期分析慢查询日志;
2. b必要选择建立索引;



参考文档：

[What to tune in MySQL Server after installation](http://www.mysqlperformanceblog.com/2006/09/29/what-to-tune-in-mysql-server-after-installation/)

[Choosing innodb\_buffer\_pool\_size](http://www.mysqlperformanceblog.com/2007/11/03/choosing-innodb_buffer_pool_size/)

