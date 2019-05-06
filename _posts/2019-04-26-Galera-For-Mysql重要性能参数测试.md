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

#### 测试背景
- 公司在把数据迁移到Galera集群时，会触发`流控`，导致整个集群不可对外提供服务
- 慢查询日志中大量insert语句
- 大表DDL耗时长

#### Galera多主模型的主要特点 
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

#### 流控场景一
wsrep_slave_threads参数设置为`1`,开启50个线程并发脚本写入数据,然后再手动导入`270W+`数据到Galera集群

- 步骤一：执行python脚本,开启50线程往`test_limit`表写入数据,可以看到写入速度很快,如下图
![](https://i.loli.net/2019/05/06/5ccfd9307cf34.jpg)
执行一段时间后,观察指标,如下图
![](https://i.loli.net/2019/05/06/5ccfdc4559532.jpg)
平缓写入数据，所有节点基本上都能处理过来，没有触发流控

- 步骤二：load数据270W+,如下图
![](https://i.loli.net/2019/04/30/5cc7f2384f236.jpg)
此时再看执行速度,如下图所示，执行速度慢了很多，但是还是能处理完成
![](https://i.loli.net/2019/05/06/5ccfddc25f537.jpg)
再观察性能指标，如下图
![](https://i.loli.net/2019/05/06/5ccfde5a8e5fc.jpg)
可以看出触发了流控,但是没有完全堵塞,还是能持续写入数据，只是影响了部分性能

- 步骤三：load结束,如下图
观察性能指标,如下图
![](https://i.loli.net/2019/05/06/5ccfe0bc0d0cd.jpg)
此时再查看执行速度，如下图，此时写入速度恢复
![](https://i.loli.net/2019/05/06/5ccfe11de6dd3.jpg)


**结论**
流控起始时间:15:08:47
流控结束时间:15:20:17
流控持续时长:12分钟
整个过程虽然触发了流控，但是没有完全堵塞，影响了写的性能，读的性能没有受影响
整个测试阶段服务器性能都在正常范围内


#### 流控场景二
wsrep_slave_threads参数设置为`32`,开启50个线程并发脚本写入数据,然后再手动导入`270W+`数据到Galera集群






#### 流控场景三 
开启50线程并发脚本写入数据,`开启事务`，手动导入270W+数据到Galera集群










#### 附录

**Galera常用端口及其作用**
- 3306-数据库对外服务的端口号。
- 4444-请求SST的端口（SST是指数据库一个备份全量文件的传输。）
- 4567-组成员之间进行沟通的一个端口号
- 4568-用于传输IST（相对于SST来说的一个增量）

**python并发脚本**
```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# Date: 2018/11/30/030
# Created by 灵均
import sys
reload(sys)
sys.setdefaultencoding('utf8')

import time
import threading
from time import ctime
import pymysql

row_count=10000000 #共计插入数据令
threads_count = 50 #并发线程数

def time_me(fn):
    '''
        记录方法执行时间
        :param args:
        :param kwargs:
        :return:
        '''
    def _wrapper(*args, **kwargs):
        start = time.time()
        fn(*args, **kwargs)
        seconds = time.time() - start
        print("{func}函数每{count}条数数据写入耗时{sec}秒".format(func='ordinary_insert', count=args[0], sec=seconds))
    return _wrapper

@time_me
def ordinary_insert(count):
    db= pymysql.connect(host='host',
                        user='user',
                        passwd='passwd',
                        port=3306,
                        db='db',
                        charset="utf8")
    cur = db.cursor()
    for i in range(count):
        #具体sql
        sql = '''INSERT INTO `test_limit`( `tno`) select round(rand()*10000000)'''.format(i)
        cur.execute(sql)
        db.commit() #每次都提交
    db.close() #关闭连接


local_var=threading.local()
def Clean(args):
    local_var.name =args
    ordinary_insert(int(row_count/threads_count))



threads=[]
for i in range(threads_count):
    t = threading.Thread(target=Clean, args=(i,))
    threads.append(t)


print ('start:', ctime())
start = time.time()

if __name__ == '__main__':
    for i in threads:
        i.start()
    for i in threads:
        i.join()
seconds = time.time() - start
print ('end:', ctime())
print("{func}函数每{count}条数数据写入耗时{sec}秒".format(func='ordinary_insert', count=row_count, sec=seconds))
```




