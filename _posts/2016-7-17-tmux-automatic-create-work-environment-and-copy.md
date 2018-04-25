---
layout: post
title: tmux进阶
categories: tools
description: 介绍tmux脚本化
keywords: tmux script
---

## 前言
tmux是一个进阶终端复用软件很多强大的功能。

## 问题
* 部署时，有两台机器，一般需要同时对两台机器上的文件进行编辑，同时执行一些相同的命令。为了保险起见，并且可能有时候要执行不能一开始就可以预知的命令，需要看到命令的执行过程。
* 在终端需要复制一段文字时，经常需要用鼠标，从键盘切换到鼠标会降低效率。

对于多台服务器部署，现在一般的方法，就是如果命令比较少，就一台服务器一台来操作；如果命令比较多，就打开一个tmux会话，将会话分割成4个面板，每个面板都ssh到不同机器，然后使用tmux的同步发送功能，4个面板一起操作。这样就出现一个问题，就是每次为了搭建出这样一个环境得执行好多命令，都是重复工作。身为一个程序员，就应该将重复工作脚本化，减少人肉。

对于经常鼠标和键盘切换，为了提高效率，必须克服，用全键盘操作。

## 解决部署问题
### tmux命令组成的准备环境的脚本

以下命令创建一个命名的tmux会话：

```tmux new-session -s work```

以下命令可以水平分割一个窗口，用`-v`代替`-h`就可以垂直分割，`-t`参数来指定一个会话：

```tmux split-window -h -t development```

下面这个命令是最重要的，它可以向某个会话发送shell命令。我们在这行配置的最后添加了一个`C-m（Control-M）`，这样就向 tmux 里发送了一个回车符：

```tmux send-keys -t development 'cd ~/code' C-m```

但是以上命令只能发送到当前的窗口中的当前面板，如果我们需要发送到一个特定面板，就需要使用`[session]:[window].[pane]`, `window`可以是序号或者名字，如以下发送命令到第一个窗口中的第二个面板：

```tmux send-keys -t development:1.2 'cd ~/code' C-m```

以下命令创建一个新窗口：

```tmux new-window -n code -t development```

有了以上命令，我们就可以完成一个脚本，如下:

```
NAME=deploy_test
tmux has-session -t $NAME
if [ $? != 0 ]
then
    tmux new-session -s $NAME -d
    tmux split-window -v -t $NAME
    tmux split-window -h -t $NAME:1.1
    tmux split-window -h -t $NAME:1.2
    tmux send-keys -t $NAME:1.1 'ssh test100' C-m
    tmux send-keys -t $NAME:1.2 'ssh test101' C-m
    tmux send-keys -t $NAME:1.3 'ssh test132' C-m
    tmux send-keys -t $NAME:1.4 'ssh test133' C-m
    tmux set synchronize-panes
    tmux send-keys -t $NAME 'cd /home/hzw/git' C-m
fi
tmux a -t $NAME
```

运行上面的脚本，我们就打开了一个tmux会话，里面有一个窗口，这个窗口被分割成4个面板，每个面板都ssh到一台不同的服务器，然后设定了同步发送功能，对一个窗口键入的命令，也会被发往其它窗口。这样我们就可以交互式的同时和4台机器操作。而且操作结果也一目了然。

### 工作开发环境
由上面的方式如法炮制，我就可以搭建出一个开发环境，所有窗口都准备好了。这个环境打开了5个窗口，第1个窗口切换到代码目录，第2个连接到mongo，第3个连接到MySQL，第4个查看日志文件，第5个进入虚拟环境，可以执行脚本。

```
NAME=wtest
tmux has-session -t $NAME
if [ $? != 0 ]
then
    tmux new-session -s $NAME -n code -d
    tmux new-window -n mongo -t $NAME
    tmux new-window -n mysql -t $NAME
    tmux new-window -n log -t $NAME
    tmux new-window -n stat -t $NAME
    tmux send-keys -t $NAME:code 'cd code/test' C-m
    tmux send-keys -t $NAME:mongo 'test' C-m
    tmux send-keys -t $NAME:mysql 'db_reader' C-m
    tmux send-keys -t $NAME:log 'cd ~/project/test/var/log; tail -f root.log' C-m
    tmux send-keys -t $NAME:stat 'cd ~/project/test; source ~/project/test/virtualenv/bin/activate' C-m
    tmux select-window -t development:code
fi
tmux a -t $NAME
```

### 更进一步
线上的操作是要切换到一个admin账号上去的。一般这种输入`su admin`，然后输入密码的操作都是要手动交互式的进行的。我一直在想能不能将这个过程能够自动化。

直到发现except这个Linux命令行工具，expect是一个用来处理交互的命令，借助expect，我们可以将交互过程写在一个脚本上，使之自动化完成。
了解了except大体的用法后在目标服务器上写下如下脚本:
```
#! /usr/bin/expect

set timeout 5
spawn su admin
expect "Password: "
send "******\r"
send "cp /home/hzw/source/test/*.jar /home/admin/project/test/jar/\r"
send "sudo -E /home/admin/project/test/init.d/test_initd.sh restart\r"
send "exit\r"
expect eof
```

* 其中`spawn`表示开始执行一个交互式命令。
* `expect "Password: "`表示这个交互式命令期望的输出。
* `expect "Password: "`表示发送密码。
* 接下来都是发送执行的命令。
* 最后退出。

执行以上脚本，我们就可以切换到admin账户，执行一些命令，然后退出。

配合tmux，我们可以把这条命令的执行写到tmux的启动脚本里，这样当我们打开这个tmux环境时，它会切出4个窗口，每个窗口都会ssh到一台机器，切换到一个账户，一般我们可以选择不退出，停留到切换后的界面，然后让我们操作。当然我们也可以直接执行命令后退出，达到部署的自动化，不过当然部署的自动化的话，就不需要tmux了。


## 解决复制问题
我们很多人都是用vim来进行编程，所以都会比较熟悉vim的复制方式，要是tmux也能这样复制就好了。

好在tmux就可以支持vim类似方式的复制，只需要将一下一句话放在配置文件中:
```
setw -g mode-keys vi
```

然后我们按下`Ctrl-a [`进入复制模式，接下来整个终端就好像是在一个vim环境中。我们可以使用vim常见的`hjkl`进行移动，使用`/?`进行搜索。

现在就可以复制了，但是tmux里复制的快捷键和vim的不一样，我们可能不熟悉，没关系，做一下配置，映射一下就可以了，如下:
```
unbind p
bind p paste-buffer
bind -t vi-copy 'v' begin-selection
bind -t vi-copy 'y' copy-selection
```

* 第一句解除`p`的绑定。
* 将`p`映射为黏贴。
* 将`v`映射为开始复制。
* 将`y`映射为拷贝。

有了以上配置，我们就可以使用vim的方式来进行复制了，告别鼠标，生产效率大大提高。

## 小结
* 可以在 shell 里使用 tmux 的每个命令，这就意味着你可以编写脚本来自动化 tmux 的几乎所有功能。
* 缓存区和剪贴板之间来移动文本，用键盘代替鼠标来复制，在一个很长的控制台的输出历史之间来回滚动，这些用键盘代替鼠标的方式，都可以大大提高生产效率。
* 程序员在日常生活中总是能碰到很多重复性的工作，可能我们懒惰一下，就过去了，每次就花那么点时间，但是利用脚本让这些重复工作脚本化可以大大节约我们的时间。
