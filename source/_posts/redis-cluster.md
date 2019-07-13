---
title: Redis 集群之 Redis-Cluster
urlname: redis-cluster-introduction
date: 2019-07-13 14:27:10
tags: Redis
---

# 业务背景
&emsp;&emsp;组内大约是去年引入 redis 作为缓存服务，由于当时经验的欠缺，只是局限于 key-value 存储极少部分配置，因此只是在某台机器上起了一个 docker，一台 redis 服务供几个系统调用。
&emsp;&emsp;由于业务需求、组内成员缓存方面知识加深以及我这个『始作俑者』对缓存的好感，缓存使用频度日益增长。虽然性能上暂时还未出现瓶颈，在新系统上线后，基于 redis 消息队列的加入、各类非关系型数据的存储逐渐增加，加上缓存的手动刷新机制的不完善，redis 业务分离与灾备刻不容缓。

--- 

# 方案

## Redis-Sentinel
高可用集群，各节点同时只有一个 master，各实例数据保持一致。

![avatar](http://pugk0v3np.bkt.clouddn.com/redis-sentinel.png)

## Redis-Cluster
分布式集群，同时有多个 master，数据分片部署在各个 master 上（不一定均匀），也是本次实践的目标。

![avatar](http://pugk0v3np.bkt.clouddn.com/redis-cluster-model.png)

- 特点
    - 无中心节点，客户端与 redis节点直连，不需要中间代理层。
    - 数据可以被分片存储，集群数据加起来就是全量数据。
    - 可以通过任意一个节点，读写不属于本节点的数据。
    - 管理方便，后续可自行增加或删除节点。
- 结构特点
    - 所有的 redis 节点彼此互联(PING-PONG机制)，内部使用二进制协议优化传输速度和带宽。
    - 节点的 fail 是通过集群中超过半数的节点检测失效时才生效。
    - Redis 集群预分好 16384 个哈希槽(hash slot)，当需要在 Redis 集群中放置一个 key-value 时，根据 CRC16(key) mod 16384 的值，决定将一个 key 放到哪个哈希槽中。
    ![avatar](http://pugk0v3np.bkt.clouddn.com/redis-cluster.png)
- 由于采用无中心结构，每个节点保存数据和整个集群状态，每个节点都和其他所有节点连接。
- 由于分片存储，当集群中一个节点故障时，会损失该节点数据。为了实现高可用，需要对每个节点各添加一个从节点，形成主从同步。
- 为了保证数据的高可用性，加入了主从模式，一个 master 对应一个或多个 slave ，master 提供数据存取，slave 则是从 master 拉取数据备份。当 master 挂了，就会从 slave 节点中选取一个来充当 master，从而保证集群不会挂掉。

---

# Just do it !
## 依赖
本次 Redis-Cluster 在 Ubuntu 下，基于 docker + docker-compose + 官方 redis 镜像实现，后续可能会自己打包镜像，加入现有的 kibana, zabbix 等服务的集成。

| Program | Version  |
| ---- | ---- |
| Ubuntu | 16.04 |
| docker | 18.09.1, build 4c52b90 |
| docker-compose | 1.17.1, build 6d101fb |
| redis | 5.0.5 |

## 开工
> [Github](https://github.com/CareyWang/gaia/tree/master/redis) 可以找到下面 docker-compose.yml 以及其他配置。学习之路才刚刚开始，欢迎大佬们指正错误。

- 建立集群

```bash
> cd /var/www/http/docker/redis/cluster

# 由于 Redis-Cluster fail 机制的要求，需要奇数台服务，即最少开启三台 master，也就是三主三从，六个服务
> docker-compose up -d
Creating network "redis_cluster-net" with the default driver
Creating redis-node-6 ... done
Creating redis-node-3 ... done
Creating redis-node-4 ... done
Creating redis-node-2 ... done
Creating redis-node-5 ... done
Creating redis-node-1 ... done

> docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                               NAMES
a4c9fc13b94b        redis:5.0.5         "docker-entrypoint.s…"   34 minutes ago      Up 34 minutes       0.0.0.0:46378->6379/tcp, 0.0.0.0:56378->56379/tcp   redis-node-2
ae35b2c82512        redis:5.0.5         "docker-entrypoint.s…"   34 minutes ago      Up 34 minutes       0.0.0.0:46377->6379/tcp, 0.0.0.0:56377->56379/tcp   redis-node-3
b3d3dd2c5c18        redis:5.0.5         "docker-entrypoint.s…"   34 minutes ago      Up 34 minutes       0.0.0.0:46376->6379/tcp, 0.0.0.0:56376->56379/tcp   redis-node-4
fbc91dfe0274        redis:5.0.5         "docker-entrypoint.s…"   34 minutes ago      Up 34 minutes       0.0.0.0:56379->56379/tcp, 0.0.0.0:46379->6379/tcp   redis-node-1
ddd307094e91        redis:5.0.5         "docker-entrypoint.s…"   34 minutes ago      Up 34 minutes       0.0.0.0:46375->6379/tcp, 0.0.0.0:56375->56379/tcp   redis-node-5
125ba63e2d62        redis:5.0.5         "docker-entrypoint.s…"   34 minutes ago      Up 34 minutes       0.0.0.0:46374->6379/tcp, 0.0.0.0:56374->56379/tcp   redis-node-6

# 随意进入刚才创建的某个 docker 
> docker exec -it redis-node-1 bash

# 建立 redis 集群，--cluster-replicas 表明每台实例有一台 slave 
> redis-cli --cluster create 172.10.15.11:6379 172.10.15.12:6379 172.10.15.13:6379 172.10.15.14:6379 172.10.15.15:6379 172.10.15.16:6379 --cluster-replicas 1
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.10.15.15:6379 to 172.10.15.11:6379
Adding replica 172.10.15.16:6379 to 172.10.15.12:6379
Adding replica 172.10.15.14:6379 to 172.10.15.13:6379
M: 5f5d5d64f02f5b7b9df21c9e58aa5ea860749a15 172.10.15.11:6379
   slots:[0-5460] (5461 slots) master
M: 61d23b6fe5f5a7d2ff97cc9d33c192c6fc8dcd58 172.10.15.12:6379
   slots:[5461-10922] (5462 slots) master
M: 011d9587f7d2b11890bce67ea70f614fd726a881 172.10.15.13:6379
   slots:[10923-16383] (5461 slots) master
S: 472b162a3dd50ee8f805780be576cd21a292d3e2 172.10.15.14:6379
   replicates 011d9587f7d2b11890bce67ea70f614fd726a881
S: 81d44f046103351041bf0dd8cae8643269029354 172.10.15.15:6379
   replicates 5f5d5d64f02f5b7b9df21c9e58aa5ea860749a15
S: ef57b7c1b1378ad43fe396c5780a653a6989594f 172.10.15.16:6379
   replicates 61d23b6fe5f5a7d2ff97cc9d33c192c6fc8dcd58
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.....
>>> Performing Cluster Check (using node 172.10.15.11:6379)
M: 5f5d5d64f02f5b7b9df21c9e58aa5ea860749a15 172.10.15.11:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: ef57b7c1b1378ad43fe396c5780a653a6989594f 172.10.15.16:6379
   slots: (0 slots) slave
   replicates 61d23b6fe5f5a7d2ff97cc9d33c192c6fc8dcd58
S: 81d44f046103351041bf0dd8cae8643269029354 172.10.15.15:6379
   slots: (0 slots) slave
   replicates 5f5d5d64f02f5b7b9df21c9e58aa5ea860749a15
M: 011d9587f7d2b11890bce67ea70f614fd726a881 172.10.15.13:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 472b162a3dd50ee8f805780be576cd21a292d3e2 172.10.15.14:6379
   slots: (0 slots) slave
   replicates 011d9587f7d2b11890bce67ea70f614fd726a881
M: 61d23b6fe5f5a7d2ff97cc9d33c192c6fc8dcd58 172.10.15.12:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

## 集群测试
```bash
# -c 表示连接的 Redis-Cluster 集群，这里必须加上，否则各台服务之间通信会有问题
> redis-cli -h 172.10.15.11 -p 6379 -c

# 查看集群信息
172.10.15.11:6379> cluster info 
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:3
cluster_stats_messages_ping_sent:343
cluster_stats_messages_pong_sent:371
cluster_stats_messages_meet_sent:1
cluster_stats_messages_sent:715
cluster_stats_messages_ping_received:367
cluster_stats_messages_pong_received:344
cluster_stats_messages_meet_received:4
cluster_stats_messages_received:715

172.10.15.11:6379> set user admin
-> Redirected to slot [5474] located at 172.10.15.12:6379
OK
```

# 业务使用
> 由于目前作为一名 phper ，目前没有涉猎其他语言的扩展研究，这里只以 Predis 为例。原理都是相近，应该都有各自成熟的解决方案。
```php
# 这里 server 可以填写上面六台中任意一台的 ip（当然不嫌麻烦也可以都填上），集群内部会自动重定向
$servers = [
    'tcp://172.10.15.11:6379',
];
$options = ['cluster' => 'redis'];

$client = new Predis\Client($servers, $options);
$client->set('user', 'admin');
```

# 小结
这里只是很简单地介绍了个人 Redis-Cluster 服务的搭建的经历，对于内部例如主从复制机制、通信机制等更深入的知识，个人也在学习之中。有遗漏之处，还望海涵。
