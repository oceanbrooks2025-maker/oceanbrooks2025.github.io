---
title: Ubuntu 系统上常用后台部署方案
date: 2026-04-02 18:00
categories: 'skills'
tags: 
  - 'Ubuntu'
  - 'corn'
  - 'tmux'
description: 介绍了在ubuntu系统上后台运行一个进程的常见方法，包括注册为系统服务、corn定时执行、窗口化工具tmux等
---



本文将介绍在ubuntu系统上后台运行一个程序的方法，主要包括注册为系统服务、窗口化运行、定时执行等。全文按照简单到复杂、低级到高级的逻辑顺序



## 临时执行脚本

顾名思义，让程序脱离当前终端，这样即使关闭当前终端也不影响程序运行

### nohup+&

`&`表示放到后台，`nohup`表述忽略挂断信号

需要在执行脚本之前加上，用法示例：

~~~
nohup python scripts.py &
~~~

默认会将输入重定向到 `nohup.out` 也可用输出流 `>` 重定向到指定文件

~~~
nohup python scripts.py > scripts.log 2>&1 &
~~~

再找出来并停止：

~~~
ps -ef | grep scripts.py
kill <pid>
~~~



优劣势

### disown

对于已经在运行的程序，可以先`ctrl+z`暂停，输入`bg`使其后台运行，再输入`disown`将其从当前终端移除。

优劣势

### tmux/screen (推荐)

终端复用器，作用是创建一个虚拟会话，允许用户在这个虚拟回话里执行程序，并且随时可以脱离（detach)和重新连接(attach)

类似于在 Windows 系统上开多个命令行窗口，而不会互相影响

tmux 常见用法

新建、脱离、连接、查看所有

优劣势



## 用户级进程管理

部署后台程序。带有自动重启、日志管理、状态监控

### supervisor

工业级进程管理工具，简单配置 `.conf` 文件即可将任何程序部署为后台服务

使用方法

优劣势

### pm2

使用方法

优劣势



## 系统级服务

### systemd （推荐）

最正统、标准，几乎所有 Linux 发行版均支持

工作方式：编写一个 `.service` 文件，定义程序的启动路径、运行用户、环境变量以及重启策略，随后将其放在 `/etc/systemd/system/` 目录下。

最高级别的系统集成度。支持开机自启、精确的资源限制（限制 CPU/内存）、依赖关系管理、以及统一的日志管理



## 容器化部署

### docker (推荐)

将你的程序及其所有依赖项（比如特定版本的 Python、特定的系统库）打包成一个镜像

通过 `docker run -d --restart=always`，Docker 守护进程会像管家一样让你的服务在后台运行，并保证服务器重启或程序崩溃时自动拉起

### docker compose

通过一个 `docker-compose.yml` 文件定义和运行多个容器（比如你的应用容器 + Redis 容器 + MySQL 容器）。只需一句 `docker-compose up -d` 即可在后台启动整套服务。



## 定时任务

### cron

通过 `crontab -e` 配置，基于分钟/小时/日/月/星期的表达式，定时唤起程序

基本用法

优劣势



### systemd timers

`systemd` 提供的定时器，被视为 `cron` 的现代替代品

比 `cron` 强大得多。可以精确到秒，支持更复杂的依赖逻辑，并且由于它本质上是触发 systemd service，所以**日志记录极其完善**（`cron` 出错时排查日志通常很痛苦）

基本用法

优劣势



### at

适用于单次执行



























