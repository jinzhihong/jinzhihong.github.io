---
layout:     post
title:      Mac OS Redis 安装启动停止设置密码
subtitle:   Mac OS Redis 安装启动停止设置密码
date:       2019-01-02
author:     金志宏
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Mac Os
    - Redis
    - 安装配置
---
# Mac OS Redis 安装启动停止设置密码

## 1. 下载和安装

``` shell
# 我安装在 /usr/local/redis 目录下
cd /usr/local/
# 下载
wget http://download.redis.io/releases/redis-5.0.3.tar.gz
# 解压 输入密码
sudo tar -xzvf redis-5.0.3.tar.gz
# 编译安装Redis
cd redis-5.0.3
# 编译
sudo make
# 安装
sudo make install 
```

然后即可启动 redis-server

## 2. 自定义配置

1. 首先在redis 的目录下新建三个文件夹bin，etc，db，log。 mkdir bin etc db log
2. 给log和db 添加权限

```shell
sudo chmod 777 db/
sudo chmod 777 log/
```

3. 在将redis/src目录下的mkreleasehdr.sh，redis-benchmark.c， redis-check-rdb.c， redis-cli.c， redis-server(如果没有执行上面的make等操作回没有这个文件，所以一定要执行)拷贝到刚刚新建的bin目录下，命令为

```properties
cp mkreleasehdr.sh redis-benchmark redis-check-rdb redis-cli redis-server ../bin
```

4. 在etc下，参考原redis目录下的redis.conf，新建一个redis.conf ,并修改redis.conf，也是copy过去即可

```shell
vim redis.conf

#修改为守护模式
daemonize yes
#设置进程锁文件
pidfile /usr/local/redis-5.0.3/redis_6379.pid #根据自己的路径进行相关配置
#端口
port 6379
#客户端超时时间
timeout 300
#日志级别
loglevel debug
#日志文件位置
logfile /usr/local/redis-5.0.3/log/log-redis.log #根据自己的路径进行相关配置
#设置数据库的数量，默认数据库为16，可以使用SELECT 命令在连接上指定数据库id
databases 16
##指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合
#save
#Redis默认配置文件中提供了三个条件：
save 900 1
save 300 10
save 60 10000
#指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，
#可以关闭该#选项，但会导致数据库文件变的巨大
rdbcompression yes
#指定本地数据库文件名
dbfilename dump.rdb
#指定本地数据库路径
dir /usr/local/redis-5.0.3/db/ #根据自己的路径进行相关配置
#指定是否在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能
#会在断电时导致一段时间内的数据丢失。因为 redis本身同步数据文件是按上面save条件来同步的，所以有
#的数据会在一段时间内只存在于内存中
appendonly no
#指定更新日志条件，共有3个可选值：
#no：表示等操作系统进行数据缓存同步到磁盘（快）
#always：表示每次更新操作后手动调用fsync()将数据写到磁盘（慢，安全）
#everysec：表示每秒同步一次（折衷，默认值）
appendfsync everysec
#requirepass 放开注释则配置密码
requirepass redispassword
```

## 3. 启动和停止

```shell
# 启动
redis-server /usr/local/redis-5.0.3/etc/redis.config
# 停止
# 连接客户端 如果配置密码 则输入密码
redis-cli -p 6379 -a redispassword
# 停止 这里必须给log和db添加777不然会报错，REDIS停止服务时会进行日志记录和本地数据库同步
127.0.0.1:6379> SHUTDOWN

# 测试
127.0.0.1:6379> set mykey 222
OK
127.0.0.1:6379> get mykey
"222"

# 退出客户端
127.0.0.1:6379> quit
```

>[Mac 安装redis并进行配置](https://blog.csdn.net/huxiaodong1994/article/details/80607726)

