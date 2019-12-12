---
layout: post
title: clickhouse单机安装及配置
categories: [clickhosue, OLAP]
description: 安装，配置优化
keywords: clickhosue, 安装，配置优化
---

### 安装

####  centOS在线安装，此时版本 19.17.2 revision 5442

```
# 添加官方存储库
sudo yum install yum-utils
sudo rpm --import https://repo.yandex.ru/clickhouse/CLICKHOUSE-KEY.GPG
sudo yum-config-manager --add-repo https://repo.yandex.ru/clickhouse/rpm/stable/x86_64

# 安装稳定版本
sudo yum install clickhouse-server clickhouse-client

# 启动
service clickhosue-server start

# 停止
service clickhouse-server stop

```



####  docker安装
```
# 下载clickhouse镜像
docker pull yandex/clickhouse-server

# 启动镜像,映射出端口8123，9000，并且把clickhouse数据挂载到本地目录/Users/awwzc/dockerdata/clickhouse

docker run -d --name dev-clickhouse  -p 8123:8123 -p 9000:9000 -v /Users/awwzc/dockerdata/clickhouse:/var/lib/clickhouse  yandex/clickhouse-server

# 进入容器
docker exec -it dev-clickhouse  /bin/bash

# 安装vim，ifconfig clickhouse镜像使用ubuntu
apt-get update
apt-get install vim

#安装ifconfig
apt-get install  net-tools
```

#### 容许远程访问

```
# centos中安装需按下面配置
vi /etc/clickhouse-server/config.xml

#放开下面这项注释
<!-- Listen specified host. use :: (wildcard IPv6 address), if you want to accept connections both with IPv4 and IPv6 from everywhere. -->
 <listen_host>::</listen_host>

 # 重启服务
 service clickhosue-server restart

#如果放开上面那一项配置，重启后提示UNKNOW，则改成放开下面这一项，重启即可
<!-- Same for hosts with disabled ipv6: -->
<listen_host>0.0.0.0</listen_host>

```

#### 设置默认账户密码

```
#用户相关配置在/etc/clickhouse-server/users.xml文件中设置
vi /etc/clickhouse-server/users.xml

#找到users >>default标签下面的password项，配置密码为123456
<password>123456</password>

# 重启服务
service clickhouse-server restart

# 验证 ，替换具体IP
clickhouse-client -h 172.17.0.2 --port 9000  -u default --password "123456"
#出现一下日记说明配置成功
ClickHouse client version 19.17.4.11 (official build).
Connecting to 172.17.0.2:9000 as user default.
Connected to ClickHouse server version 19.17.2 revision 54428.
4d1d1c488e32 :)

```

#### 新增账户密码

```
vi /etc/clickhouse-server/users.xml

#找到users标签，在下面增加一下内容，用户名为test，配置密码为123456
<test>
            <password>123456</password>
            <networks incl="networks" replace="replace">
                <ip>::/0</ip>
            </networks>
            <profile>default</profile>
            <quota>default</quota>
</test>

# 重启服务
service clickhouse-server restart

# 验证 ，替换具体IP
clickhouse-client -h 172.17.0.2 --port 9000  -u test --password "123456"
#出现一下日记说明配置成功
ClickHouse client version 19.17.4.11 (official build).
Connecting to 172.17.0.2:9000 as user default.
Connected to ClickHouse server version 19.17.2 revision 54428.
4d1d1c488e32 :)


```



###  配置优化

#### clickhouse-server 配置优化

```
#增大clickhouse查询使用最大内存，默认为10G
vi /etc/clickhouse-server/users.xml

#找到此节点配置最大内存
<max_memory_usage>10000000000</max_memory_usage>
```



####  服务器配置优化
```
#关闭大页
echo 'performance' | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
#调整内存使用
echo 0 > /proc/sys/vm/overcommit_memory
#关闭cpu节能模式
echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled
```
