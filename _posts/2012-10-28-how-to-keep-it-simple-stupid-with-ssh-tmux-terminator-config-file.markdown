---
layout: post
title: 改善工具配置简单操作运维
date: 2012-10-28 00:07
comments: true
---

记录分享下自己日常工作中使用的到一些工具如ssh、tmux、terminator等按个人习惯对其做了相应配置以保持简洁傻瓜式方便。

其中涉及到的配置文件主要有

* ssh的用户配置文件$HOME/.ssh/config
* tmux的$HOME/.tmux.conf
* terminator终端工具配置文件$HOME/.config/terminator/config

以及可能涉及到$HOME/.bashrc等

##### 修改配置$HOME/.ssh/config让你生活更简单（省心省事）

通常默认情况我们使用以下方式ssh连接：

<pre>$ssh username@hostname/ip -p port</pre>

机器一多的话每次都要输入一串便会令人感觉繁琐，如果再加上实现没有使用ssh-copy-id功能采用公密钥认证方式登陆的话，每次都还要手动输入密码的话就更浪费时间。

改善后，很多人或许会想到结合$HOME/.bashrc的别名alias功能，在此配置文件添加相应的别名如下：

<pre>alias name='ssh username@hostname/ip -p port'</pre>

其实这里如果知道`ssh_config`这东西的话，可以直接修改$HOME/.ssh/config如下:
<pre>
host name
hostname hostip
user username
port port
</pre>

如此一来，你只需要使用命令`ssh name`连接管理。或许你也发现这样一来ssh还是需要每次重复输入，是的，所以可以结合前面的$HOME/.bashrc别名进一步处理。

单独创建一个ssh连接的别名文件为：$HOME/.ssh/bashrc 内容类似于：

<pre>alias name='ssh name'</pre>

然后将此文件加载到用户目录下的bashrc配置文件结尾，编辑$HOME/.bashrc后，执行source $HOME/.bashrc使其生效：

<pre>source $HOME/.ssh/bashrc</pre>

这样以后就只需输入相应的name（如个人这里取地区名称简写加ip末尾为name）回车执行即可。

##### 再结合一些小工具如tmux/screen的可以更方便管理操作

关于这些小工具想必都比较清楚，这里记录个人下使用的配置文件：[$HOME/.tmux.conf](https://gist.github.com/3950948)

<script src="https://gist.github.com/Yousri/3950948.js"></script>

##### 终端建议改用Terminator，至于原因个人觉得其支持的功能更强大，特别其group功能相当的方便。

这个是自己对Terminator做了写改动后的配置文件:[$HOME/.config/terminator/config](https://gist.github.com/3950895)

<script src="https://gist.github.com/Yousri/3950895.js"></script>
