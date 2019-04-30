---
layout:     post   				    # 使用的布局（不需要改）
title:      Galera For Mysql性能参数测试 				# 标题 
subtitle:   Sysbench压测    #副标题
date:       2019-04-26 				# 时间
author:     BY lingjun						# 作者
header-img: img/post-bg-rwd.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Mysql
    - Galera
    - Sysbench
---

## Galera For Mysql性能参数测试

>Galera Cluster for MySQL是一套基于同步复制的多主MySQL集群解决方案，使用简单，没有单点故障，可用性高，能很好保证业务不断增长时数据的安全和随时的扩展。

**测试背景**
- 公司在把数据迁移到Galera集群时，会触发`流控`，导致整个集群不可对外提供服务
- 慢查询日志中大量insert语句
- 大表DDL耗时长

**Galera多主模型的主要特点**  
- 基于同步复制
- 多主服务器的拓扑结构
- 可以在任意节点上进行读写
- 自动剔除故障节点
- 自动加入新节点
- 真正行级别的并发复制
- 客户端连接跟操作单台MySQL数据库的体验一致

通过Galera集群特点，我们可以知道，Galera是基于同步复制的，数据传输到各个节点是通过一个类似队列形式，当从节点处理不过来时，就会发生堵塞，这也就是Galera集群的`限流`。很显然从节点可以开启多个线程来提高处理速度，这就涉及参数`wsrep_slave_threads`.  
`wsrep_slave_threads`： 本地的执行队列的线程数量，一般为CPU线程数的1-1.5倍  
`gcs.fc_limit`:此参数确定Flow Control参与的点。当从队列超过此限制时，节点将暂停复制。默认值为16  
`gcs.fc_factor`:此参数用于确定节点何时可以脱离流控制。默认值为0.5
>官方文档提示：注意 警告：不要将wsrep_slave_threads的值用于高于wsrep_cert_deps_distance状态变量给出的平均值。

**流控场景**  
导入270W+数据到Galera集群
![](https://i.loli.net/2019/04/30/5cc7c7999ca93.jpg)

|参数|值|
|--|--|
|wsrep_slave_threads|1|
|wsrep_slave_threads|32|

![](https://i.loli.net/2019/04/30/5cc7c8f8a9d24.jpg)
- 结论  
当调大wsrep_slave_threads参数值为32时，在第二次Load数据时，没有触发流控

![](img/post-bg-rwd.jpg)



![](https://i.loli.net/2019/04/30/5cc7c8f8a9d24.jpg)


**Galera常用端口及其作用**
- 3306-数据库对外服务的端口号。
- 4444-请求SST的端口（SST是指数据库一个备份全量文件的传输。）
- 4567-组成员之间进行沟通的一个端口号
- 4568-用于传输IST（相对于SST来说的一个增量）

