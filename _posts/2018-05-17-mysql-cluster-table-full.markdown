---
layout: post
title:  "线上mysql cluster出现table full error"
date:   2018-05-17 20:29:00 +0800
categories: mysql
tags: mysql bug
---
线上生产环境mysql cluster某一个table插入新数据时，java业务层报错：
{% highlight java %}
Caused by: java.sql.SQLException: The table 'user_relation' is full
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:965) ~[mysql-connector-java-5.1.45.jar!/:5.1.45]
	at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3973) ~[mysql-connector-java-5.1.45.jar!/:5.1.45]
	at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3909) ~[mysql-connector-java-5.1.45.jar!/:5.1.45]
	at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:2527) ~[mysql-connector-java-5.1.45.jar!/:5.1.45]
	at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2680) ~[mysql-connector-java-5.1.45.jar!/:5.1.45]
	at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2484) ~[mysql-connector-java-5.1.45.jar!/:5.1.45]
	at com.mysql.jdbc.PreparedStatement.executeInternal(PreparedStatement.java:1858) ~[mysql-connector-java-5.1.45.jar!/:5.1.45]
	at com.mysql.jdbc.PreparedStatement.executeUpdateInternal(PreparedStatement.java:2079) ~[mysql-connector-java-5.1.45.jar!/:5.1.45]
	at com.mysql.jdbc.PreparedStatement.executeUpdateInternal(PreparedStatement.java:2013) ~[mysql-connector-java-5.1.45.jar!/:5.1.45]
	at com.mysql.jdbc.PreparedStatement.executeLargeUpdate(PreparedStatement.java:5104) ~[mysql-connector-java-5.1.45.jar!/:5.1.45]
	at com.mysql.jdbc.PreparedStatement.executeUpdate(PreparedStatement.java:1998) ~[mysql-connector-java-5.1.45.jar!/:5.1.45]
	at sun.reflect.GeneratedMethodAccessor209.invoke(Unknown Source) ~[na:na]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_162]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_162]
	at org.apache.tomcat.jdbc.pool.StatementFacade$StatementProxy.invoke(StatementFacade.java:114) ~[tomcat-jdbc-8.5.27.jar!/:na]
	at com.sun.proxy.$Proxy142.executeUpdate(Unknown Source) ~[na:na]
	at org.hibernate.engine.jdbc.internal.ResultSetReturnImpl.executeUpdate(ResultSetReturnImpl.java:204) ~[hibernate-core-5.0.12.Final.jar!/:5.0.12.Final]
	... 150 common frames omitted
{% endhighlight %}
表存储已满。但是有些表可以插入，有些不行。

先登录data node节点，看看是不是硬盘空间不足。
如果空间充足，登录mysql cluster manager机器，执行mgm查看集群情况：
{% highlight shell %}
/opt/mysql/server-5.7/bin/ndb_mgm
all report memory
{% endhighlight %}
输出：
{% highlight shell %}
Node 2: Data usage is 95%(93389 32K pages of total 98304)
Node 2: Index usage is 40%(38778 8K pages of total 96032)
Node 3: Data usage is 95%(93389 32K pages of total 98304)
Node 3: Index usage is 40%(38779 8K pages of total 96032)
{% endhighlight %}
data node存储页快满了。
在manager节点上查看集群存储配置：
{% highlight shell %}
cat /var/lib/mysql-cluster/config.ini

[ndb_mgmd]
hostname=172.17.29.55
datadir=/var/lib/mysql-cluster

[ndbd]
hostname=172.17.29.47
datadir=/var/lib/mysql-cluster/data
TcpBind_INADDR_ANY=TRUE
MaxNoOfTables=4096
MaxNoOfAttributes=5000000
MaxNoOfOrderedIndexes=10000
DataMemory=3G
IndexMemory=750M
[ndbd]
hostname=172.17.29.48
datadir=/var/lib/mysql-cluster/data
TcpBind_INADDR_ANY=TRUE
MaxNoOfTables=4096
MaxNoOfAttributes=5000000
MaxNoOfOrderedIndexes=10000
DataMemory=3G
IndexMemory=750M

[mysqld]
hostname=172.17.29.56
[mysqld]
hostname=172.17.29.57
{% endhighlight %}
可以把DataMemory和IndexMemory改大一些（根据节点实际内存改大），分别设置为32G和6G。

整个步骤如下：
> 1.关闭manager和两台data node上的supervisor
> 2.在manager上关闭集群，/opt/mysql/server-5.7/bin/ndb_mgm -e shutdown
> 3.修改/var/lib/mysql-cluster/config.ini中[ndbd]下的DataMemory与IndexMemory
> 4.重启manager节点：sudo /opt/mysql/server-5.7/bin/ndb_mgmd -f /var/lib/mysql-cluster/config.ini --configdir=/var/lib/mysql-cluster  --reload
> 5.重启manager和data node节点上的supervisor

重启过程比较慢，要等两个data node数据索引全部建好才行，api node在data node启动后会自动连接。
完成之后，再次查看：
{% highlight shell %}
Node 2: Data usage is 8%(93549 32K pages of total 1048576)
Node 2: Index usage is 4%(38825 8K pages of total 786464)
Node 3: Data usage is 8%(93549 32K pages of total 1048576)
Node 3: Index usage is 4%(38825 8K pages of total 786464)
{% endhighlight %}
1048576 * 32 / 1024 = 32768
32G的设置生效了。

7月5日更新：
修改这个配置其实不需要先关闭整个集群，这样数据服务会停止很久。
由于我们的集群结构是一个manager，两个data node，data node互为备份，所以可以rolling retart。
官网链接：https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster-rolling-restart.html
先关闭manager和data node上的supervisor。
然后进入mgm，show一下：
{% highlight shell %}
ndb_mgm> show
Connected to Management Server at: 172.17.31.20:1186
Cluster Configuration
---------------------
[ndbd(NDB)]	2 node(s)
id=2	@172.17.31.22  (mysql-5.7.17 ndb-7.5.5, Nodegroup: 0)
id=3	@172.17.31.21  (mysql-5.7.17 ndb-7.5.5, Nodegroup: 0)

[ndb_mgmd(MGM)]	1 node(s)
id=1	@172.17.31.20  (mysql-5.7.17 ndb-7.5.5)

[mysqld(API)]	2 node(s)
id=4	@172.17.31.25  (mysql-5.7.17 ndb-7.5.5)
id=5	@172.17.31.24  (mysql-5.7.17 ndb-7.5.5)
{% endhighlight %}
在mgm里关闭manager：1 stop。 或者直接杀死进程。
修改完config.ini的配置之后，reload启动manager：sudo /opt/mysql/server-5.7/bin/ndb_mgmd -f /var/lib/mysql-cluster/config.ini --configdir=/var/lib/mysql-cluster --reload
然后再进入mgm，分别重新启动两个数据节点：
2 restart
3 restart
然后all report memory一下，看到total值有变化了。
