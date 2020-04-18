---
title: linux debain 操作系统制作和添加服务
layout: post
categories: 解决问题
tags: debain 操作系统 服务
excerpt:  纯手工制作和添加服务 
---

### 问题
今天为自家的树莓派安装了[frp](https://github.com/fatedier/frp/releases), 是家里的树莓派可以被外网访问。因为这个服务要开机自启动，所以就学习了一下。怎么制作服务，并设置为开机自启动。

### 制作服务
不废话直接上代码
```shell
#!/bin/sh
### BEGIN INIT INFO
# Provides:          app_run
# Required-Start:    $local_fs $remote_fs $network $syslog $named  #此处为依赖服务，即该服务脚本会在这些服务启动后运行
# Required-Stop:     $local_fs $remote_fs $network $syslog $named
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: app_run service
# Description:       Start the app_run service and associated helpers
### END INIT INFO
do_start()
{
    nohup /usr/local/frp_0.32.1_linux_arm/frpc -c /usr/local/frp_0.32.1_linux_arm/frpc.ini 2>&1 &  #这里是你要在服务启动时执行的动作
}
 
do_stop()
{
    pkill -f /usr/local/frp_0.32.1_linux_arm/frpc   #这里是你要在关闭服务时执行的动作
}

#
# Function that sends a SIGHUP to the daemon/service
#
do_restart() {
    do_stop
    sleep 1
    do_start
}
 
case "$1" in
  start)
    do_start
    ;;
  stop)
    do_stop
    ;;
  status)
    exit $?
    ;;
  reload)
    echo "reload"
    ;;
  restart)
    do_restart
    ;;
  *)
    echo "Usage: {start|stop|restart|reload}" >&2
    exit 3
    ;;
esac
 
exit 0



```
保存为frpc，并放到/etc/init.d 文件夹中，用下面的语句让文件可执行
```shell

sudo chmod +x /etc/init.d/frpc

```

到这里服务的脚本就写完了，可以用下面的语句检查脚本是否可以正常执行
```shell
sudo service frpc start
sudo service frpc stop
```

此处因为我是用windows 编辑的脚本， 然后上传到linux上的， 还报了一个错，“-bash: ./test.sh: /bin/sh^M: bad interpreter: No such file or directory  ”
主要原因是frpc是我在windows下编辑然后上传到linux系统里执行的。.sh文件的格式为dos格式。而linux只能执行格式为unix格式的脚本。

- 修改文件格式为unix
1. 方法一：使用vi修改文件format
```shell
    # vim frpc
    :set ff
    :set ff=unix
```
2. 方法二： 使用dos2unix
```shell
    sudo apt-get install dos2unix
    dos2unix frpc
```
3. 方法三： windows 修改sublime text3配置
Preferences --> Settings, 加入下面的配置
```
    "default_encoding": "UTF-8",
    "enable_hexadecimal_encoding": true,
    "default_line_ending": "unix",
```

### 添加开机启动
update-rc.d 是用管理服务的，具体用法有很多，这里不详细记录。只用最简单的用法
```shell
    
    update-rc.d frpc defaults #注册服务（添加为开机自启动）

    update-rc.d -f frpc remove #移除服务

```

### 其它说明
若更改了/etc/init.d/frpc的内容，可以运行 
```
    systemctl daemon-reload 
```
这个命令会重新装载所有守护进程的unit文件，然后重新生成依赖关系树

