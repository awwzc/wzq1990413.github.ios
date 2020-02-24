---
layout: post
title: docker搭建redis集群
categories: [docker,redis-cluster]
description: centos7下docker搭建redis集群
keywords: docker redis-cluster, docker搭建redis集群
---


### 硬件环境

1. mac 下centos虚拟机,版本号CentOS Linux release 7.2.1511.
2. docker 版本号 19.03.
3. docker compose 版本号 1.25.4.
4. 本次不考虑设置redis集群密码
### 安装docker与docker-compose

#### 1.安装docker

##### 卸载旧版本

```
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

##### 安装 Docker Engine-Community

```
# 1.设置仓库
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

# 2.设置稳定的仓库
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# 3.安装最新版本的 Docker Engine-Community 和 containerd
sudo yum install docker-ce docker-ce-cli containerd.io

# 4.或者安装特定版本的 Docker Engine-Community，请在存储库中列出可用版本，然后选择并安装
 #列出并排序您存储库中可用的版本。此示例按版本号（从高到低）对结果进行排序
  yum list docker-ce --showduplicates | sort -r
  docker-ce.x86_64  3:18.09.1-3.el7                     docker-ce-stable
  docker-ce.x86_64  3:18.09.0-3.el7                     docker-ce-stable
  docker-ce.x86_64  18.06.1.ce-3.el7                    docker-ce-stable
  docker-ce.x86_64  18.06.0.ce-3.el7                    docker-ce-stable
 #通过其完整的软件包名称安装特定版本，该软件包名称是软件包名称（docker-ce）加上版本字符串（第二列），从第一个冒号（:）一直到第一个连字符，并用连字符（-）分隔。例如：docker-ce-18.09.1
 sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io

 eg:sudo yum install docker-ce-<19.03> docker-ce-cli-<19.03> containerd.io

```

##### 启动docker

```
sudo systemctl start docker

#验证是否安装成功
# 执行docker --version，出现下面提示表示安装成功
Docker version 19.03.6, build 369ce74a3c
```

#### 2.安装docker-compose

##### 下载安装包

```
#Linux 上我们可以从 Github 上下载它的二进制包来使用，最新发行的版本地址：#https://github.com/docker/compose/releases。
#运行以下命令以下载 Docker Compose 的当前稳定版本,要安装其他版本的 Compose，请替换 1.24.1

sudo curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

##### 将可执行权限应用于二进制文件

```
sudo chmod +x /usr/local/bin/docker-compose
```

##### 创建软链

```
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

##### 测试是否安装成功

```
#执行docker-compose --version，出现下面提示代表安装成功
docker-compose version 1.25.4, build 8d51620a
```



### 搭建redis集群

#### 1.redis配置配置文件配置

##### github上面下载redis.conf文件

```
# 新建配置目录
sudo mkdir docker-test-redis-cluster
# 下载地址https://github.com/antirez/redis/blob/unstable/redis.conf,把下载文件复制到docker-test-redis-cluster

```

##### 配置3主3从redis配置

```
# 以redis.conf为模本复制出6份配置文件
#nodes-6391.conf，nodes-6392.conf，nodes-6393.conf，nodes-6394.conf
#nodes-6395.conf，nodes-6396.conf
cd  docker-test-redis-cluster
ll

-rw-r--r--. 1 root root 80013 2月  23 22:10 nodes-6391.conf
-rw-r--r--. 1 root root 80014 2月  23 22:10 nodes-6392.conf
-rw-r--r--. 1 root root 80014 2月  23 22:10 nodes-6393.conf
-rw-r--r--. 1 root root 80015 2月  23 22:10 nodes-6394.conf
-rw-r--r--. 1 root root 79940 2月  23 22:10 nodes-6395.conf
-rw-r--r--. 1 root root 80014 2月  23 22:10 nodes-6396.conf

# 分别修改6个配置文件中的一下项：

#注释此项，容许任意ip连接
#bind 127.0.0.1

#开启集群功能
cluster-enabled yes

#设置节点端口，与配置文件名称数字一致，如当前文件为nodes-6391.conf，则为6391
port 6391

#集群内部配置文件，与配置文件名称数字一致，如当前文件为nodes-6391.conf，则为6391.conf
cluster-config-file "6391.conf"

#节点超时时间，单位毫秒
cluster-node-timeout 15000

#容许无密码连接
protected-mode no

```

##### docker-compose.yml文件配置

```
# 创建docker-compose.yml文件
cd docker-test-redis-cluster

sudo mkdir rediscluster-compose

cd rediscluster-compose

touch  docker-compose.yml

# 往docker-compose.yml文件中添加以下配置:

version: "3.6"
services:
  redis-master1:
     image: redis:5.0 # 基础镜像
     container_name: redis-master1 # 容器服务名
     working_dir: /config # 工作目录
     environment: # 环境变量
       - PORT=6391 # 跟 config/nodes-6391.conf 里的配置一样的端口
     network_mode: host # 使用docker宿主计IP，在我自己Mac docker下无效...
     ports: # 映射端口，对外提供服务
       - "6391:6391" # redis 的服务端口
       - "16391:16391" # redis 集群监控端口
     stdin_open: true # 标准输入打开
     tty: true
     privileged: true # 拥有容器内命令执行的权限
     volumes: ["/usr/wzq/docker-test-redis-cluster:/config"] # 映射数据卷，配置目录
     entrypoint: # 设置服务默认的启动程序
       - /bin/bash
       - redis.sh
  redis-master2:
       image: redis:5.0
       working_dir: /config
       container_name: redis-master2
       environment:
              - PORT=6392
       network_mode: host
       ports:
         - "6392:6392"
         - "16392:16392"
       stdin_open: true
       tty: true
       privileged: true
       volumes: ["/usr/wzq/docker-test-redis-cluster:/config"]
       entrypoint:
         - /bin/bash
         - redis.sh
  redis-master3:
       image: redis:5.0
       container_name: redis-master3
       working_dir: /config
       environment:
              - PORT=6393
       network_mode: host
       ports:
         - "6393:6393"
         - "16393:16393"
       stdin_open: true
       tty: true
       privileged: true
       volumes: ["/usr/wzq/docker-test-redis-cluster:/config"]
       entrypoint:
         - /bin/bash
         - redis.sh
  redis-slave1:
       image: redis:5.0
       container_name: redis-slave1
       working_dir: /config
       environment:
            - PORT=6394
       network_mode: host
       ports:
         - "6394:6394"
         - "16394:16394"
       stdin_open: true
       tty: true
       privileged: true
       volumes: ["/usr/wzq/docker-test-redis-cluster:/config"]
       entrypoint:
         - /bin/bash
         - redis.sh
  redis-salve2:
       image: redis:5.0
       working_dir: /config
       container_name: redis-salve2
       environment:
             - PORT=6395
       network_mode: host
       ports:
         - "6395:6395"
         - "16395:16395"
       stdin_open: true
       tty: true
       privileged: true
       volumes: ["/usr/wzq/docker-test-redis-cluster:/config"]
       entrypoint:
         - /bin/bash
         - redis.sh
  redis-salve3:
       image: redis:5.0
       container_name: redis-slave3
       working_dir: /config
       environment:
          - PORT=6396
       network_mode: host
       ports:
         - "6396:6396"
         - "16396:16396"
       stdin_open: true
       tty: true
       privileged: true
       volumes: ["/usr/wzq/docker-test-redis-cluster:/config"]
       entrypoint:
         - /bin/bash
         - redis.sh



```

##### 编写 redis 默认的启动脚本

```
# redis配置文件同级目录下添加redis启动脚本
cd docker-test-redis-cluster

sudo  touch  redis.sh

#添加下面内容，compose文件指定了docker容器的工作目录为/config
redis-server  /config/nodes-${PORT}.conf
```



##### docker-compose 启动

```
#启动
docker-compose up -d
#出现下面提示
Creating redis-master1 ... done
Creating redis-slave3  ... done
Creating redis-salve2  ... done
Creating redis-master2 ... done
Creating redis-slave1  ... done
Creating redis-master3 ... done

#测试是否启动成功
docker ps -a

# 出现下面提示表示启动成功
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                      PORTS               NAMES
ac5e4158a965        redis:5.0           "/bin/bash redis.sh"     9 seconds ago       Up 6 seconds                                    redis-salve2
1ad975cc9e27        redis:5.0           "/bin/bash redis.sh"     9 seconds ago       Up 6 seconds                                    redis-master2
792aa7ac1344        redis:5.0           "/bin/bash redis.sh"     9 seconds ago       Up 6 seconds                                    redis-slave3
cd9dded0cd07        redis:5.0           "/bin/bash redis.sh"     9 seconds ago       Up 6 seconds                                    redis-master3
3140f27fdcae        redis:5.0           "/bin/bash redis.sh"     9 seconds ago       Up 7 seconds                                    redis-slave1
7349f6dba594        redis:5.0           "/bin/bash redis.sh"     9 seconds ago       Up 7 seconds                                    redis-master1



```

##### 初始化redis集群

```
#创建 3 主 3 从的 redis 集群
#进入其中一个redis容器，docker宿主计IP为192.168.0.105

docker exec -it redis-master1 /bin/bash

#执行初始化命令，这一步开始命令须在 redis5.0 及以上版本运行

redis-cli --cluster create  192.168.0.105:6391 192.168.0.105:6392 192.168.0.105:6393 192.168.0.105:6394 192.168.0.105:6395 192.168.0.105:6396 --cluster-replicas 1

#出现以下提示：
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 192.168.0.105:6395 to 192.168.0.105:6391
Adding replica 192.168.0.105:6396 to 192.168.0.105:6392
Adding replica 192.168.0.105:6394 to 192.168.0.105:6393
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 8488b09fc576379b11cd692a0016a8a4ab0cdf42 192.168.0.105:6391
   slots:[0-5460] (5461 slots) master
M: 704dc6e634c183b9368c868b75686fe9b765648b 192.168.0.105:6392
   slots:[5461-10922] (5462 slots) master
M: d845dc1b0f1354ce77b8ae671698e80eddfcce6f 192.168.0.105:6393
   slots:[10923-16383] (5461 slots) master
S: f5b2baca3411d8b7c86ca2c73d50c2bbc3e9ee3c 192.168.0.105:6394
   replicates 704dc6e634c183b9368c868b75686fe9b765648b
S: b7d03824ae4c23ed6706323465e8263face24b2f 192.168.0.105:6395
   replicates d845dc1b0f1354ce77b8ae671698e80eddfcce6f
S: 85fe1b81c15917d776cd868c971d60812847db8b 192.168.0.105:6396
   replicates 8488b09fc576379b11cd692a0016a8a4ab0cdf42
Can I set the above configuration? (type 'yes' to accept):

#输入yes,出现下面提示，代表初始化成功
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
......
>>> Performing Cluster Check (using node 192.168.0.105:6391)
M: 8488b09fc576379b11cd692a0016a8a4ab0cdf42 192.168.0.105:6391
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: b7d03824ae4c23ed6706323465e8263face24b2f 192.168.0.105:6395
   slots: (0 slots) slave
   replicates d845dc1b0f1354ce77b8ae671698e80eddfcce6f
M: 704dc6e634c183b9368c868b75686fe9b765648b 192.168.0.105:6392
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: f5b2baca3411d8b7c86ca2c73d50c2bbc3e9ee3c 192.168.0.105:6394
   slots: (0 slots) slave
   replicates 704dc6e634c183b9368c868b75686fe9b765648b
S: 85fe1b81c15917d776cd868c971d60812847db8b 192.168.0.105:6396
   slots: (0 slots) slave
   replicates 8488b09fc576379b11cd692a0016a8a4ab0cdf42
M: d845dc1b0f1354ce77b8ae671698e80eddfcce6f 192.168.0.105:6393
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

##### 测试是否搭建成功

```
#docker宿主计上也已安装redis-cli
#执行下面名令，注意要加-c 代表集群模式，否则重定向时会出现 (error) MOVED 15588错误
redis-cli -c  -h 192.168.0.105 -p 6391
#连接成功则出现：
192.168.0.105:6391>
#输入cluster nodes,出现下面信息则代表搭建成功
192.168.0.105:6391>cluster nodes

b7d03824ae4c23ed6706323465e8263face24b2f 192.168.0.105:6395@16395 slave d845dc1b0f1354ce77b8ae671698e80eddfcce6f 0 1582518782773 5 connected
704dc6e634c183b9368c868b75686fe9b765648b 192.168.0.105:6392@16392 master - 0 1582518783000 2 connected 5461-10922
8488b09fc576379b11cd692a0016a8a4ab0cdf42 192.168.0.105:6391@16391 myself,master - 0 1582518782000 1 connected 0-5460
f5b2baca3411d8b7c86ca2c73d50c2bbc3e9ee3c 192.168.0.105:6394@16394 slave 704dc6e634c183b9368c868b75686fe9b765648b 0 1582518781000 4 connected
85fe1b81c15917d776cd868c971d60812847db8b 192.168.0.105:6396@16396 slave 8488b09fc576379b11cd692a0016a8a4ab0cdf42 0 1582518783778 6 connected
d845dc1b0f1354ce77b8ae671698e80eddfcce6f 192.168.0.105:6393@16393 master - 0 1582518781767 3 connected 10923-16383

#set 一个key为wl，你会发现会从定向到其他节点，自此大功告成
192.168.0.105:6391> set wl wzq
-> Redirected to slot [15588] located at 192.168.0.105:6393

```


