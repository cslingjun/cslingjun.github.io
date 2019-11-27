---
layout:     post   				    # 使用的布局（不需要改）
title:      PostgreSQL高可用集群repmgr 				# 标题 
subtitle:   postgres|repmgr    #副标题
date:       2019-11-27 				# 时间
author:     BY lingjun						# 作者
header-img: img/post-bg-miui6.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - postgres
    - repmgr
---

## PostgreSQL高可用集群repmgr 
> repmgr是一套开源工具，用于管理PostgreSQL服务器集群中的复制管理和故障转移，它扩展了PostgresSQL内建的hot-standby能力，可以监控复制和执行管理任务（自动故障转移，手工切换）。

#### repmgr提供两个主要工具
- **repmgr** 是一个用于执行管理任务的命令行工具，主要用来设置备用服务器，切换主服务器和备服务器，显示复制群集中的服务器状态。
- **repmgrd** 是一个守护程序，它主动监视复制集群中的服务器并监控和记录复制性能，通过检测主服务器的故障并选择最合适的备用服务器来执行自动主备库切换。

#### Version Info
PostgresSQL 11.6
Repmgr 4.2

|host|node|
|--|--|
|192.168.111.133|master|
|192.168.111.170|slave|

#### Repmgr架构图
![](https://i.loli.net/2019/11/27/CQGjVRlEsTqDP9A.jpg)

#### 安装配置
1、postgresql安装
- yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-6-x86_64/pgdg-redhat-repo-latest.noarch.rpm
- yum install -y postgresql11 
- yum install -y postgresql11-server 

2、postgresql初始化
- service postgresql-11 initdb

3、repmgr安装
- curl https://dl.2ndquadrant.com/default/release/get/11/rpm \| sudo bash
- yum install repmgr11-4.2-2.el6 -y

4、配置
1. 两台主机配置免密互信
2. 修改主库配置文件postgresql.conf（主备配置的必要参数）
3. 修改pg_hba.conf配置文件（赋权远程登录）
4. 主库创建repmgr用户和repmgr数据库
- create user repmgr superuser login;
- alter user repmgr with password 'repmgr';
- create database repmgr;
- alter database repmgr owner to repmgr;
- alter user repmgr set search_path to repmgr, "$user", public;
5. 主库创建repmgr.conf配置文件（repmgr节点信息及连接pg信息）
6. 主库节点初始化
- repmgr -f /etc/repmgr.conf primary register
7. 验证初始化
- repmgr -f /etc/repmgr.conf cluster show
8. 备库节点创建repmgr.conf
9. 克隆备库
- repmgr -h 192.168.111.170 -U repmgr -d repmgr -f /etc/repmgr.conf standby clone 
10. 备库节点初始化
- repmgr -F -f /etc/repmgr.conf standby register
11. 验证初始化
- repmgr -f /etc/repmgr.conf cluster show
![](https://i.loli.net/2019/11/27/lnC78QHWuqRye3S.jpg)
12. 验证同步状态


#### repmgr高可用测试


#### 备注
1. 开启repmgrd守护进程前， postgres.conf配置文件该shared_preload_libraries = 'repmgr'需配置好
2. rejoin节点时执行命令报错如下，postgres.conf配置文件该wal_log_hints = on需开启
![](https://i.loli.net/2019/11/27/UxyYqGSuWw6gJnj.jpg)


#### 附录
1. **常用命令**
- 利用repmgr克隆备库
```
repmgr -h 192.168.111.170 -U repmgr -d repmgr -f /etc/repmgr.conf standby clone
```
- 利用repmgr注册主备库
```
repmgr -f /etc/repmgr.conf primary register
repmgr -f /etc/repmgr.conf standby register
```
- 利用repmgr switch over数据库
```
repmgr standby switchover -f /etc/repmgr.conf --siblings-follow
```
- 利用repmgr promote备库
```
repmgr -f /etc/repmgr.conf standby promote
```
- 利用rpmgr rejoin备用数据库
```
repmgr -f /etc/repmgr.conf node rejoin -d 'host=192.168.111.133 user=repmgr dbname=repmgr connect_timeout=2' --force-rewind --verbose
```
- repmgr常用显示主备库状态命令
```
repmgr -f /etc/repmgr.conf cluster show
repmgr -f /etc/repmgr.conf cluster event
repmgr -f /etc/repmgr.conf cluster crosscheck
```

2. **配置文件**