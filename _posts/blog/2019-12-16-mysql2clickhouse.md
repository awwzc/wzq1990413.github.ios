---
layout: post
title: canal同步mysql数据到clickhouse
categories: [mysql,clickhouse,OLAP]
description: mysql同步数据到clickhouse
keywords: mysql sync, clickhouse
---

### mysql配置

本次不考虑接入kafka等mq

####  开启binlog

```
# mysql配置文件/etc/my.cnf添加下面配置
vi  /etc/my.cnf

#插入下面内容
server-id        = 1
log_bin          = /var/lib/mysql/bin.log
binlog-format    = row # very important if you want to receive write, update and delete row events
# optional
expire_logs_days = 30
max_binlog_size  = 768M
# setup listen address
bind-address     = 0.0.0.0
```

#### 新增同步账号

```
 #登陆mysql,执行下面命令，创建账号maxwell，密码为123456

 CREATE USER 'maxwell'@'%' IDENTIFIED BY '123456';
 GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'maxwell'@'%';
 flush privileges;

```



###  canal安装配置

####  安装jdk1.8

google安装下jdk1.8即可



#### canal-server安装配置

下载 canal, 访问 [release](https://github.com/alibaba/canal/releases)页面 , 选择需要的包下载, 如以 1.1.4 版本为例

```
#下载canalan安装包
wget https://github.com/alibaba/canal/releases/download/canal-1.1.4/canal.deployer-1.1.4.tar.gz

#解压到指定目录
mkdir -p /usr/tool/canal
tar zxvf  canal.deployer-1.1.4.tar.gz  -C  /usr/tool/canal

#解压后进入目录，结构如下
drwxr-xr-x   7 awwzc  staff   238 12 14 23:34 bin
drwxr-xr-x   9 awwzc  staff   306 12 14 23:32 conf
drwxr-xr-x  83 awwzc  staff  2822 12 14 23:30 lib
drwxr-xr-x   4 awwzc  staff   136 12 14 23:34 logs

# canal启动时会读取conf目录下面的文件夹，当作instance,进入conf目录下复制example文件夹
cp -R  example／  maxwell/

#移除example文件夹
mv -rf example

#修改maxwell文件夹下面的instance.properties文件下面几项为你自己的数据库配置即可
vi conf/maxwell/instance.properties

# position info
canal.instance.master.address=192.168.0.102:3306

# username/password
canal.instance.dbUsername=maxwell
canal.instance.dbPassword=123456

# 启动，安装目录下执行以下命令,server,instance出现下面日记说明启动成功
bin/startup.sh
# 查看server日记，会出现以下日记
tail -200f  logs/canal/canal.log

2019-12-14 23:34:47.247 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## set default uncaught exception handler
2019-12-14 23:34:47.312 [main] INFO  com.alibaba.otter.canal.deployer.CanalLauncher - ## load canal configurations
2019-12-14 23:34:47.334 [main] INFO  com.alibaba.otter.canal.deployer.CanalStarter - ## start the canal server.
2019-12-14 23:34:47.406 [main] INFO  com.alibaba.otter.canal.deployer.CanalController - ## start the canal server[192.168.0.111(192.168.0.111):11111]
2019-12-14 23:34:49.026 [main] INFO  com.alibaba.otter.canal.deployer.CanalStarter - ## the canal server is running now ......

# 查看instance日记，会出现以下日记
tail -200f  logs/maxwell/maxwell.log

2019-12-15 17:59:12.908 [destination = example , address = /192.168.0.102:3306 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - ---> begin to find start position, it will be long time for reset or first position
2019-12-15 17:59:12.913 [destination = example , address = /192.168.0.102:3306 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - prepare to find start position just last position
 {"identity":{"slaveId":-1,"sourceAddress":{"address":"192.168.0.102","port":3306}},"postion":{"gtid":"","included":false,"journalName":"bin.000002","position":249315,"serverId":1,"timestamp":1576282583000}}
2019-12-15 17:59:13.015 [destination = example , address = /192.168.0.102:3306 , EventParser] WARN  c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - ---> find start position successfully, EntryPosition[included=false,journalName=bin.000002,position=249315,serverId=1,gtid=,timestamp=1576282583000] cost : 105ms , the next step is binlog dump


#关闭
sh bin/stop.sh

```

#### canal-client安装配置

##### 下载 canal-adapter, 访问 [release](https://github.com/alibaba/canal/releases)页面 , 选择需要的包下载, 如以 1.1.4 版本为例

```
#下载
wget https://github.com/alibaba/canal/releases/download/canal-1.1.4/canal.adapter-1.1.4.tar.gz

#解压
mkdir -p /usr/tool/canal-adapter
tar zxvf canal.adapter-1.1.4.tar.gz  -C /usr/tool/canal-adapter
#解压后目录如下
drwxr-xr-x   7 awwzc  staff   238 12 15 13:19 bin
drwxr-xr-x   9 awwzc  staff   306 12 15 13:18 conf
drwxr-xr-x  87 awwzc  staff  2958 12 15 13:18 lib
drwxr-xr-x   3 awwzc  staff   102 12 15 13:19 logs
drwxr-xr-x   6 awwzc  staff   204 12 15 13:09 plugin

#在lib目录下面添加clickhouse连接驱动
clickhouse-jdbc-0.2.jar
httpclient-4.2.5.jar
httpcore-4.4.5.jar

```

######  修改conf/application.yml

```
#修改conf/application.yml以下项
vi conf/application.yml

#canal-server地址
canalServerHost: 127.0.0.1:11111

#同步数据源配置
srcDataSources
defaultDS:
#mysql连接信息
      url: jdbc:mysql://192.168.0.102:3306/maxwell?useUnicode=true
      username: root
      password: 123456
  canalAdapters:
  - instance: maxwell # canal instance Name or mq topic name
    groups:
    - groupId: g1
      outerAdapters:
      - name: logger
      - name: rdb   #rdb类型
        key: mysql
        properties:
        #clickhouse数据看配置
          jdbc.driverClassName: ru.yandex.clickhouse.ClickHouseDriver
          jdbc.url: jdbc:clickhouse://127.0.0.1:8123/maxwell
          jdbc.username: default
          jdbc.password:
```

##### 修改rdb文件夹中配置

```
#1.同步增个库配置，前提条件是mysql数据库名，跟clickhouse数据库名一致
vi  rdb/all.xml
#添加以下内容
## Mirror schema synchronize config
dataSourceKey: defaultDS #application.yml中一致
destination: maxwell  #跟application.yml中instance值一样
groupId: g1 #跟application.yml中groupId值一样
outerAdapterKey: mysql #跟application.yml中outerAdapters中key值一样
concurrent: true
dbMapping:
  mirrorDb: true
  database: maxwell #要同步的mysql库

#2.如果mysql,clickhouse库名不一致，则要同步的表分别在rdb中新增一个配置文件，以t_download表为列
vi  rdb/all.xml
#添加以下内容
dataSourceKey: defaultDS   #application.yml中一致
destination: maxwell    #跟application.yml中instance值一样
groupId: g1             #跟application.yml中groupId值一样
outerAdapterKey: mysql  #跟application.yml中outerAdapters中key值一样
concurrent: true
dbMapping:
  database: clickhouse  #mysql数据库名
  table: t_download   #mysql要同步的表
  targetTable: maxwell.t_download    #clickhouse中对应的表
    id: id                      # 如果是复合主键可以换行映射多个
  mapAll: true                 # 是否整表映射, 要求源表和目标表字段名一模一样 (如果targetColumns也配置了映射,则以targetColumns配置为准)
  #targetColumns:                # 字段映射, 格式: 目标表字段: 源表字段, 如果字段名一样源表字段名可不填
  #  id:
   # name:
   # role_id:
   #c_time:
   #test1:

```

##### 启动

```
bin/startup.sh

#查看日记，出现以下日记说名启动成功
tail -200f logs/adapter/adapter.log

2019-12-15 13:19:40.457 [main] INFO  c.a.o.canal.adapter.launcher.loader.CanalAdapterService - ## start the canal client adapters.
2019-12-15 13:19:40.464 [main] INFO  c.a.otter.canal.client.adapter.support.ExtensionLoader - extension classpath dir: /Users/awwzc/Documents/my_soft/tool/canal-adpater/plugin
2019-12-15 13:19:40.542 [main] INFO  c.a.o.canal.adapter.launcher.loader.CanalAdapterLoader - Load canal adapter: logger succeed
2019-12-15 13:19:40.546 [main] INFO  c.a.otter.canal.client.adapter.rdb.config.ConfigLoader - ## Start loading rdb mapping config ...
2019-12-15 13:19:40.637 [main] INFO  c.a.otter.canal.client.adapter.rdb.config.ConfigLoader - ## Rdb mapping config loaded
2019-12-15 13:19:40.640 [main] ERROR com.alibaba.druid.pool.DruidDataSource - testWhileIdle is true, validationQuery not set
2019-12-15 13:19:40.951 [main] INFO  com.alibaba.druid.pool.DruidDataSource - {dataSource-2} inited
2019-12-15 13:19:40.959 [main] INFO  c.a.o.canal.adapter.launcher.loader.CanalAdapterLoader - Load canal adapter: rdb succeed
2019-12-15 13:19:40.986 [main] INFO  c.a.o.canal.adapter.launcher.loader.CanalAdapterLoader - Start adapter for canal instance: example succeed
2019-12-15 13:19:40.986 [Thread-4] INFO  c.a.o.canal.adapter.launcher.loader.CanalAdapterWorker - =============> Start to connect destination: example <=============
2019-12-15 13:19:40.986 [main] INFO  c.a.o.canal.adapter.launcher.loader.CanalAdapterService - ## the canal client adapters are running now ......
2019-12-15 13:19:40.995 [main] INFO  org.apache.coyote.http11.Http11NioProtocol - Starting ProtocolHandler ["http-nio-8081"]
2019-12-15 13:19:41.021 [main] INFO  org.apache.tomcat.util.net.NioSelectorPool - Using a shared selector for servlet write/read
2019-12-15 13:19:41.048 [main] INFO  o.s.boot.web.embedded.tomcat.TomcatWebServer - Tomcat started on port(s): 8081 (http) with context path ''
2019-12-15 13:19:41.053 [main] INFO  c.a.otter.canal.adapter.launcher.CanalAdapterApplication - Started CanalAdapterApplication in 7.099 seconds (JVM running for 7.832)
2019-12-15 13:19:41.122 [Thread-4] INFO  c.a.o.canal.adapter.launcher.loader.CanalAdapterWorker - =============> Start to subscribe destination: example <=============
2019-12-15 13:19:41.128 [Thread-4] INFO  c.a.o.canal.adapter.launcher.loader.CanalAdapterWorker - =============> Subscribe destination: example succeed <=============
```



##### 关闭

```
sh  bin/stop.sh

```



#####  支持clickhouse,update,delete语句，基于阿里官网1.1.4版本

下载页面：[下载地址](https://github.com/wzq1990413/canal/releases)


```
#阿里官网的canal-adapter,rdb同步目前不支持clickhouse update,delete,自己做了些修改让其支持这两个操作
#直接下载adapter-clickhouse下面canal.adapter-1.1.4.tar.gz包安装即可，已经集成clickhouse驱动
wget https://github.com/wzq1990413/canal/releases/download/adapter-clickhouse/canal.adapter-1.1.4.tar.gz

#配置步骤同上
```




