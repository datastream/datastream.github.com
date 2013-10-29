**Time-stamp: <2013-10-29 20:18:19 datastream>**

# 监控系统的权衡

---

最近更新了一下自己的业余项目[metrictools](github.com/datastream/metrictools)。相比上次写博客时的设计，新的框架基本做到了分布式和高可用了。

程序大的框架基本没有什么变动。比较明显的改进是：

1. 数据存储使用了twemproxy+redis
1. 同时metricstatistic程序抛弃了以前的阈值判断模式，改为使用skyline算法智能检测异常

metrictools在设计的时候被划分为好几个子程序：

1. metricproccessor 用于数据插入和trigger扫瞄
1. metricstatistic 用于trigger计算。如果是表达式，那么会将计算结果存储起来
1. metricarchive 用于数据压缩和清理
1. metricnotify 用于trigger异常状态的通知。目前只支持邮件通知
1. metricwebservice 是用户交互的web api

目前为止所有的处理部分都可以是分布式，通过额外的配置，所有子系统可以做到高可用了。

## 设计的取舍

最近重新对比了一下`collectd`和`statsd`。
感觉statsd处理数据的理念相当不错，非常适合程序嵌入监控。相对于拿到监控数据，被监控程序本身的高速运行更加重要。
collectd对数据的描述更加细致，比较适合后期挖掘处理。

* udp vs tcp

可能在处理的时候，udp丢包会导致统计不对。但是udp丢包本身就是被监控的情况。
如果是tcp模式，那么最坏的情况是因为监控程序挂了，可能会导致整个业务流程堵住。当然http模式会好一些，但是依旧消耗太多。
在这里我更加欣赏Statsd的处理方式。
Statsd的client端不进行数据聚合统计，只是单纯的将数据用udp方式推送出来。

* 数据处理节点的无状态性 vs 处理性能

> 程序的无状态可能会带来额外的数据获取调用消耗，但是额外带来的好处是可以分布式部署。

* 使用消息队列 vs 程序直接处理

> 使用消息队列带来了额外的可用性和扩展性。如果使用程序直接接受处理，那么扩展能力无法做到线性。

* redis vs mongodb/mysql/postgresql

> 使用redis最大的问题在于数据的多少，如果数据很多那么redis可能无法存储。所以在使用redis做完数据存储的时候，需要额外的程序去处理老旧数据。
> 使用database（包含NoSQL），最大的问题是磁盘处理速度会拖慢整个系统的处理效率。

* 阈值报警 vs 算法报警

> 常见的监控体系种很重要的一点是报警。一般使用阈值报警，但是阈值的设置可能需要额外的维护和经验值调整。
> 算法报警的好处是能自动检测异常，但是对于处理磁盘容量缓慢变满的情况可能不是非常灵敏。

* 自定义数据agent vs 开源现成工具

> 自定义编写的agent在处理数据上可能会有优势，但是会引入额外的开发成本。而使用现成的开源工具例如collectd，statsd会在部署上会有优势。