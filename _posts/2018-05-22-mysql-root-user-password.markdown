---
layout: post
title:  "mysql cluster root用户密码修改"
date:   2018-05-22 14:18:00 +0800
categories: mysql
tags: mysql
---
在mysqld节点上登录root时，发现密码总是不正确，或者忘记密码，需要在无密码登录情况下修改密码。
首先修改mysqld上配置文件，允许无密码登录。
{% highlight shell %}
sudo vi /etc/my.cnf
{% endhighlight %}
输出配置内容为：
{% highlight shell %}
[mysqld]
ndbcluster
socket=/var/run/mysqld/mysqld.sock

[mysql_cluster]
ndb-connectstring=172.17.29.209
[client]
socket=/var/run/mysqld/mysqld.sock
{% endhighlight %}
在myqld下面增加一句：skip-grant-tables
然后重启mysqld服务：
{% highlight shell %}
sudo /opt/mysql/server-5.7/support-files/mysql.server restart
{% endhighlight %}
之后，无密码登录mysql，并修改root密码
{% highlight shell %}
sudo /opt/mysql/server-5.7/bin/mysql
use mysql
flush privileges;
set password for 'root'@'localhost' = PASSWORD('2LRZ9LmT5GDFHAs3');
flush privileges;
{% endhighlight %}
然后再进入/etc/my.cnf把刚才加的那句删掉，再重启mysqld服务：
{% highlight shell %}
sudo /opt/mysql/server-5.7/support-files/mysql.server restart
{% endhighlight %}
现在密码应该生效了。
