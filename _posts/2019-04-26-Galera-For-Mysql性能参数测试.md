---
layout:     post   				    # 使用的布局（不需要改）
title:      Galera For Mysql重要性能参数测试 				# 标题 
subtitle:   Sysbench压测    #副标题
date:       2019-04-26 				# 时间
author:     BY lingjun						# 作者
header-img: img/post-bg-coffee.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Mysql
    - Galera
    - Sysbench
---

## Galera For Mysql重要性能参数测试

>Galera Cluster for MySQL是一套基于同步复制的多主MySQL集群解决方案，使用简单，没有单点故障，可用性高，能很好保证业务不断增长时数据的安全和随时的扩展。

**Galera多主模型的主要特点**  
- 基于同步复制
- 多主服务器的拓扑结构
- 可以在任意节点上进行读写
- 自动剔除故障节点
- 自动加入新节点
- 真正行级别的并发复制
- 客户端连接跟操作单台MySQL数据库的体验一致
