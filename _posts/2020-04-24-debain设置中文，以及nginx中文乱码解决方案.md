---
title: debain设置中文支持和nginx中文乱码解决方案
layout: post
categories: linux 问题记录
tags: linux设置中文 nginx中文乱码
excerpt: 
---

### 写在前面
在windows安装了debain子系统，发现无法显示中文。之前安装linux都是带中文安装的，所以都不用考虑这个问题。在网上找了一些，解决方案，这里写下来总结一下

### 操作步骤
1. debain 中文设置
```shell
    # 安装locales 
    sudo apt-get locales
    
    # 配置locales
    sudo dpkg-reconfigure locales
    # 空格可以选择，选择 en_GB.UTF-8 > OK > 选择en_GB.UTF-8回车
    
    # 重启系统
    # 如果没有修改成功， 查看locale 文件, 把配置改为 'LANG=en_GB.UTF-8'
    sudo vim /etc/default/locale 
```
2. nginx 中文乱码
在配置中加入 charset utf-8;


