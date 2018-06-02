---
layout: post
title:  "redis cluster 搭建（单台机器多节点）"
date:   2018-06-02 11:17:00 +0800
categories: redis
tags: redis
---
下载最新的redis版本，编译，安装。
如果没有安装ruby、zlib和openssl，先安装他们仨。
ruby和gem安装按照官方安装步骤就行，安装完gem之后要gem install redis。
安装zlib和openssl可以进入ruby源码目录下的ext/zlib和ext/openssl安装。直接make安装。
可能需要修改Makefile里的$(top_srcdir)为../..
完成之后，进入redis目录，新建一个cluster目录，来存放cluster各节点配置文件。
进入cluster目录，新建六个目录，分别为7001、7002、7003、7004、7005、7006。
复制redis目录下的redis.conf文件到cluster目录中，修改如下行：
{% highlight shell %}
daemonize    yes                          //redis后台运行
pidfile  /var/run/redis_7000.pid          //pidfile文件对应7000,7002,7003
port  7000                                //端口7000,7002,7003
cluster-enabled  yes                      //开启集群  把注释#去掉
cluster-config-file  nodes_7000.conf      //集群的配置  配置文件首次启动自动生成 7000,7001,7002
cluster-node-timeout  5000                //请求超时  设置5秒够了
appendonly  yes                           //aof日志开启  有需要就开启，它会每次写操作都记录一条日志
protected-mode no
{% endhighlight %}
找到bind 127.0.0.1这一行，注释掉。
复制六份到各个节点的目录下，并修改对应端口数字的地方（上面对应的是7000的地方）。
然后执行：
{% highlight shell %}
src/redis-server cluster/7001/redis.conf
src/redis-server cluster/7002/redis.conf
src/redis-server cluster/7003/redis.conf
src/redis-server cluster/7004/redis.conf
src/redis-server cluster/7005/redis.conf
src/redis-server cluster/7006/redis.conf
/data1/redis/redis-3.2.1/src/redis-cli -p 7001 //进入某个client看看启动没
src/redis-trib.rb create --replicas 1 172.17.31.23:7001 172.17.31.23:7002 172.17.31.23:7003 172.17.31.23:7004 172.17.31.23:7005 172.17.31.23:7006
{% endhighlight %}
注意，最后一句一定不能用127.0.0.1，不然其他服务器连不上。
