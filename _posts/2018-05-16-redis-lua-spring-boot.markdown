---
layout: post
title:  "spring-boot 2.0使用redis+lua出错"
date:   2018-05-16 17:14:00 +0800
categories: redis spring
---
在使用spring-boot 2.0.3与spring data redis时，加载lua脚本：
{% highlight kotlin %}
private val script1 = """
        local keyName = KEYS[1]
        local indexes = {}
        for i = 2, #KEYS do
            indexes[i - 1] = KEYS[i]
        end
        local scores = ARGV
        local res = -1
        for i, v in ipairs(indexes) do
            res = redis.call('zadd', keyName, scores[i], v)
        end
        return res
    """
    @GetMapping("/test1")
    fun test1(): Long {
        val keys = mutableListOf<String>()
        val values = mutableListOf<Double>()
        for (i in 1..1200) {
            keys.add(i.toString())
            values.add(i.toDouble())
        }
        return mObjectRedisTemplate.execute(DefaultRedisScript(script1, Long::class.java), mutableListOf("KEYS_ZSET").plus(keys), (values.toTypedArray()))
    }
{% endhighlight %}
运行时报错：
{% highlight java %}
java.lang.IllegalStateException: null
	at io.lettuce.core.output.CommandOutput.set(CommandOutput.java:75) ~[lettuce-core-5.0.3.RELEASE.jar:na]
	at io.lettuce.core.protocol.RedisStateMachine.safeSet(RedisStateMachine.java:357) ~[lettuce-core-5.0.3.RELEASE.jar:na]
	at io.lettuce.core.protocol.RedisStateMachine.decode(RedisStateMachine.java:138) ~[lettuce-core-5.0.3.RELEASE.jar:na]
	at io.lettuce.core.protocol.CommandHandler.decode(CommandHandler.java:663) ~[lettuce-core-5.0.3.RELEASE.jar:na]
	at io.lettuce.core.protocol.CommandHandler.decode0(CommandHandler.java:627) ~[lettuce-core-5.0.3.RELEASE.jar:na]
	at io.lettuce.core.protocol.CommandHandler.decode(CommandHandler.java:622) ~[lettuce-core-5.0.3.RELEASE.jar:na]
	at io.lettuce.core.protocol.CommandHandler.decode(CommandHandler.java:542) ~[lettuce-core-5.0.3.RELEASE.jar:na]
	at io.lettuce.core.protocol.CommandHandler.channelRead(CommandHandler.java:511) ~[lettuce-core-5.0.3.RELEASE.jar:na]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:340) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.ChannelInboundHandlerAdapter.channelRead(ChannelInboundHandlerAdapter.java:86) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:340) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.ChannelInboundHandlerAdapter.channelRead(ChannelInboundHandlerAdapter.java:86) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:340) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1414) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:945) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:146) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:645) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:580) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:497) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:459) ~[netty-transport-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:886) ~[netty-common-4.1.22.Final.jar:4.1.22.Final]
	at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30) ~[netty-common-4.1.22.Final.jar:4.1.22.Final]
	at java.lang.Thread.run(Thread.java:748) [na:1.8.0_144]
{% endhighlight %}
由于spring data redis 2.0之后redis client默认使用lettuce，查看lettuce 5.0代码，发现
{% highlight java %}
public void set(long integer) {
    throw new IllegalStateException();
}
{% endhighlight %}
貌似没有实现CommandOutPut.set。所以可以暂时先改用jedis作为redis client。
首先在gradle里配置：
{% highlight gradle %}
compile group: 'redis.clients', name: 'jedis', version: '2.9.0'
compile group: 'org.apache.commons', name: 'commons-pool2', version: '2.5.0'
{% endhighlight %}
然后再redis config里配置：
{% highlight kotlin %}
@Bean
fun jedisConnectionFactory() = JedisConnectionFactory()
@Bean
fun stringRedisTemplate(jedisConnectionFactory: JedisConnectionFactory) = StringRedisTemplate(jedisConnectionFactory)
{% endhighlight %}
恢复正常。
