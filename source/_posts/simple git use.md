---
title: 简单的 Git 同步方法
date: 2026-02-04 12:00
categories: 'test'
tags: 
  - 'test'
description: 
---

1. 安装 Git

   如果是安装的 portable 版本，右键没有 Git Bash 选项，手动添加一个

   （估计没有管理员选项，不然为什么安装 portable 版本）

   - 创建启动脚本

   就在 Git 安装目录创建一个 OpenGitBash.bat 的脚本，内容如下：

   ~~~
   @echo off
   :: 1. 切换到目标目录
   cd /d %1
   
   :: 2. 设置临时代理环境变量
   set http_proxy=http://127.0.0.1:7897
   set https_proxy=http://127.0.0.1:7897
   
   :: 3. 启动 Git Bash
   start "" "...\git-bash.exe"
   ~~~

   注意这里的第二步设置了代理，顺便解决了 Git 的网络问题

   再创建一个快捷方式 OpenGitBash 指向这个脚本

   - 添加到“发送到”

     WIN + R ，打开 shell:sendto （“发送到”目录），将刚才创建的快捷方式 OpenGitBash 放入"发送到"目录

   **如何使用？** 使用方法与原生 Git Bash 有所不同，不能在目录内右键空白的菜单中找到“发送到”选项；而要返回上一级，右键选中目录->"显示更多选项"->"发送到"->"OpenGitBash"

2. 拉取仓库

   第一次拉取

   ~~~
   git clone https://github.com/user/repo.git
   cd repo
   ~~~

   平时拉取

   ~~~
   git pull 
   ~~~

3. 推送仓库

   ~~~
   git add .
   git commit -m "description"
   git push
   ~~~

   

