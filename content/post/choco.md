---
title: "Windows上的包管理工具"
date: 2020-10-21T11:20:35+08:00
draft: false
---

# Windows上的巧克力味Chocolatey

Zypher的windows开发环境搭建需要安装Chocolatey。于是看了下这个巧克力味。

Chocolatey是一款专为Windows系统开发的、基于NuGet的包管理器工具，类似于Node.js的npm，MacOS的brew，Ubuntu的apt-get，它简称为choco。Chocolatey的设计目标是成为一个去中心化的框架，便于开发者按需快速安装应用程序和工具。

Chocolatey的官网： https://chocolatey.org/

1. Chocolatey安装
   [Install chocolatey](https://chocolatey.org/install)
2. Chocolatey用法
+ search  - 搜索包 choco search something
+ list  - 列出包 choco list -lo
+ install  - 安装 choco install baretail
+ pin  - 固定包的版本，防止包被升级 choco pin windirstat
+ upgrade  - 安装包的升级 choco upgrade baretail
+ uninstall  - 安装包的卸载 choco uninstall baretail
+ 安装Ruby Gem  - choco install compass -source ruby
+ 安装Python Egg  - choco install sphynx -source python
+ 安装IIS服务器特性  - choco install IIS -source windowsfeatures
+ 安装Webpi特性  - choco install IIS7.5Express -source webpi