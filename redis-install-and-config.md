title: 在Ubuntu系统中安装/卸载Redis
date: 2015-06-16 23:41:59
tags: NoSQL
---

##安装Redis服务器端
    sudo apt-get install redis-server
安装完成后，Redis服务器会自动启动，我们可以检查Redis服务器程序：
### 检查Redis服务器系统进程

    ps -aux|grep redis
redis     4162  0.1  0.0  10676  1420 ?        Ss   23:24   0:00 /usr/bin/

	redis-server /etc/redis/redis.conf

conan     4172  0.0  0.0  11064   924 pts/0    S+   23:26   0:00 grep --color=auto redis

#### 通过启动命令检查Redis服务器状态
    netstat -nlt|grep 6379

tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN

#### 通过启动命令检查Redis服务器状态
	
	sudo /etc/init.d/redis-server status

redis-server is running

##卸载Redis

	sudo apt-get purge --auto-remove redis-server