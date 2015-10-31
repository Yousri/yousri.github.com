---
layout: post
title: "Nginx无缝升级及其自带信号处理"
date: 2013-09-27 19:30
comments: true
---

任何一种服务版本或插件升级都需要考虑最佳状态的无缝过渡升级（不影响任何请求服务的中断且无需人员的干涉）

对于 Nginx 其进程管理模式及其自带各种信号处理可以很好的控制管理 master 和 work 进程。之前用得相对比较多的基本只是修改 Nginx 配置文件后使用到 ‘HUP’ 信号处理方式来重新加载配置文件，但其实 Nginx 还有很多其他信号可以用于控制对其进程（不管是 master 或 work）做相应不同的处理的，详见官方 wiki：[NginxCommandLine](http://wiki.nginx.org/NginxCommandLine/)


##### 对于 Master 管理进程而言：

* ERM, INT（暴力结束进程，当前的请求不执行完成就退出）
* QUIT （优雅退出，执行完当前的请求后退出）
* HUP （重新加载配置文件，用新的配置文件启动新worker进程，并优雅的关闭旧的worker进程，可作为 reload 生效新配置）
* USR1 （重新打开日志文件，可作为日志切割处理）
* USR2 （平滑的升级nginx二进制文件，可作为无缝平滑升级过渡）
* WINCH （优雅的关闭worker进程，master进程仍在监听服务）

##### 对于 work 进程而言：

其实 master 进程即可对 work 进行管理，但这里指的是使用信号单独对 work 进程做处理,主要包括：

* TERM / INT
* QUIT
* USR1

---

这里主要测试说明两种之前使用较少的 无缝平滑升级二进制文件 和 升级后如有异常的回滚 操作：

##### 无缝平滑升级二进制文件方式

不管对于升级Nginx到一个新的版本，或新增 / 更新 nginx module的时候，都难免需要替换Nginx 的二进制文件，所谓想无缝更新文件而不想对用户或对服务有任何请求失败或丢失的影响，这里可以借助 Nginx 的 USR2 信号处理。
  
首先，编译生成新的二进制文件（只需 configure / make 即可），通常编译后默认会在 objs/nginx 直接覆盖旧的二进制文件后，对现有 master 进程发送 USR2 信号，此时现有 master 进程会将自己之前的 pid 文件重命名为 pid.oldbin
  
  ![](http://lifecycles.b0.upaiyun.com/nginx-pid.png!fw600)
  
同时执行新的 nginx 二进制文件并启动新的 master 进程，此时会同时存在运行 2 个 Nginx 实例即 2 组的新旧 master / work 进程同时处理请求。截图如下:

  ![](http://lifecycles.b0.upaiyun.com/nginx-upgrade.png!fw600)

这时只需将旧的 Nginx 实例的 master 直接优雅退出便可完成版本升级，即：kill -quit `ca/usr/local/nginx/logs/nginx.pid.oldbin`
  
但建议暂且还是先不要将旧的 master 进程，以防新升级的版本有异常问题可以快速的回滚旧实例上，详见下面的回滚操作

##### 新服务异常回滚操作

接上，建议采用 kill -winch `ca/usr/local/nginx/logs/nginx.pid.oldbin` 的方式只先将旧的 Nginx 实例的 work 进程优雅关闭退出，而非直接 quit 连旧的 Nginx 实例的 master 进程也优雅退出。

因为这样此时旧的 Nginx 实例的 master 进程还在运行着也意味着仍在监听着 socket 服务，这样一旦新版本二进制服务有异常问题时，仍可以快速的回滚旧的 Nginx 实例来提供服务，只需对旧实例的 master 进程发送 HUP 信号，此时它就会再次唤起启动 work 进程来处理请求服务，同时发送 QUIT 信号到新 Nginx 实例的 master 进程关闭退出，一切就又回滚恢复到没有更新升级前的状态。

当然一切顺利的话，在 1 的情况下便可直接将 旧实例的 master 进程 QUIT 优雅的退出关闭，完全由新实例来提供服务处理请求。
