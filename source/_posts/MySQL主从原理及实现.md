---
title: MySQL 主从原理、读写分离及实现
urlname: mysql-master-slave-introduction
date: 2019-07-10 14:28:02
tags: MySQL
---

&emsp;&emsp;MySQL最重要的功能之一。指一台服务器充当主数据库服务器，另一台或多台服务器充当从数据库服务器，主服务器中的数据自动复制到从服务器之中。

# 主从作用
+ 数据备份。
+ 高可用性（负载均衡，容灾）。
+ 在MySQL主从复制架构中，读操作可以在所有的服务器上面进行，而写操作只能在主服务器上面进行，给读操作提供了扩展。

# 工作原理
方式：基于日志与基于GTID全局事务标志符，这里主要介绍第一种。

+ 基础是主服务器对数据库修改记录二进制日志(以下简称 binlog)，从服务器通过主服务器的二进制日志自动执行更新。
+ 主服务器上面的任何修改都会保存在 binlog (mysql- bin.xxxxxx)里面。
+ 从服务器上面启动一个 I/O 线程（实际上就是一个主服务器的客户端进程），连接到主服务器读取指定 binlog 指定位置之后信息，然后返回给 slave 的 I/O 线程。返回的信息中除了 binlog 内容外，还有本次返回日志内容后在 master 服务器端的新的 binlog 文件名称以及在 binlog 中的下一个指定更新位置。
+ 当 slave 服务器的 I/O 线程获取到来自 master 服务器上 I/O 线程发送日志内容以及日志文件以及位置点后，将 binlog 内容一次写入到 slave 端自身的 relaylog (中继日志)文件(mysql-relay-bin.xxxxxx)的末尾，并将新的 binlog 文件名和位置记录到 master-info 文件中，以便下次读取 master 端新 binlog 日志时，能够告诉 master 服务器需要从新 binlog 日志的哪个文件哪个位置开始请求新的 binlog 内容。
+ slave 服务器的 SQL 线程会实时的检测本地 relaylog 中新增加的日志内容，然后及时的把 relaylog 文件中的内容解析成在 master 端曾经执行的 SQL 语句的内容，并在自身按语句的顺序执行应用这些 SQL 语句，应用完毕后清理用过的日志。
+ 当复制状态正常的情况下，在 master 和 slave 执行了同样的SQL语句，主从的数据是完全一样的。
+ 在老版本的 MySQL 主从复制中 slave 端并不是两个进程完成的，而是由一个进程完成。复制 binlog 和解析日志并在自身执行的过程成为一个串行的过程，性能受到了一定的限制，异步复制的延迟也会比较长。

# 主从复制模式
+ 异步模式
    + MySQL 默认的复制是异步模式，主库在执行完客户端提交的事务后会立即将结果返给给客户端，并不关心从库是否已接收并处理。
+ 同步模式
    + 主库执行完一个事务，所有的从库都执行了该事务才返回给客户端。
    + 由于需要等待所有从库执行完事务，性能会受到极大影响。
+ 半同步模式
    + 主库在执行完客户端提交的事务后不是立刻返回给客户端，而是等待至少一个从库接收到并写到 relaylog 中才返回给客户端。
    + 相对于异步复制，半同步复制提高了数据的安全性，同时它也造成了一定程度的延迟。

# 主从复制类型
+ 基于语句的复制(STATEMENT)
    + MySQL 默认使用基于语句的复制。
    + 主服务器上面执行的语句在从服务器上面再执行一遍。
    + 存在的问题：时间上可能不完全同步造成偏差，执行语句的用户也可能是不同一个用户。
    + 使用 MySQL 的功能相对少（存储过程、触发器、函数），选择基于语句的复制。
+ 混合类型的复制(MIXED)
    + 当基于语句的复制会引发问题的时候就会使用基于行的复制，MySQL会自动进行选择。
    + 使用MySQL的特殊功能（存储过程、触发器、函数），选择混合类型的复制。
+ 基于行的复制(ROW)
+ 把主服务器上面更新后的内容直接复制过去，而不关心到底改变该内容是由哪条语句引发的。
+ 存在的问题：表t有10000行数据，某字段+100，基于行的复制需制10000行的内容，开销较大；基于语句的复制只需要一条语句就可以了。
+ 使用 MySQL 的特殊功能（存储过程、触发器、函数）又希望数据最大化一致，选择基于行的复制。

# MySQL的复制过滤(Replication Filters)
主要用途：复制特定的数据表或库到从库
+ 在 master 上过滤二进制日志中的事件。
+ 在 slave 上过滤中继日志中的事件。

# 主从复制优化方案
+ 若一主多从，master 既需要负责写操作，又要负责为几个从库提供二进制日志。此时可以稍做调整，将 binlog 只给某一 slave，该 slave 再开启 binlog 并将自己的 binlog 再分发给其他从库。或者是干脆这个从库不记录数据，只负责转发 binlog。
+ And so on ...
  
# 主从数据一致性检测
TODO...
  
# 读写分离
+ 优点
    + 高可用
    + 负载均衡
    + 业务模块化（索引 | 存储引擎）
    + 数据备份
    + 便于拓展
+ 存在的问题
    + 成本增加（增加机器 | 额外开启 binlog 占用存储空间）
    + slave 数据延迟
    + 写入更慢
+ 主流实现方式
    + 程序代码内部实现
    + 中间代理层实现

# 数据库中间件
+ 读写分离方案
+ 负载均衡的简单实现
+ 分库分表主流实现思路
    + 业务起步初始，为了加快应用上线和快速迭代，很多应用都采用集中式的架构。随着业务系统的扩大，系统变得越来越复杂，通过硬件提高系统性能的方式带来的成本也越来越高。
  
# Just do it !
## 配置主库
```shell
[mysqld]
server-id   = 1
log_bin     = /var/log/mysql/mysql-bin.log

# binlog_format = "STATEMENT"
# binlog_format = "ROW"
# binlog_format = "MIXED"

# 被忽略的数据库
# binlog-ignore-db=mysql

# 需要同步的数据库，不配置则表示同步所有
# binlog-do-db=DB-XXX
```

```sql
# 查看binlog模式
SHOW GLOBAL VARIABLES LIKE '%binlog_format%';

# 查看binlog
SHOW MASTER STATUS;
```

## 配置从库
```shell
[mysqld]
server-id = 2

# 忽略的数据库
# replicate-ignore-db=mysql

# 需要同步的数据库，不配置则表示同步所有
# replicate-do-db=DB-XXX
```

```sql
STOP SLAVE;

# 连接master
# 其中repl是master上拥有 Replication Slave 权限的账户
CHANGE MASTER TO 
master_host = '127.0.0.1',
master_port = 3306,
master_user = 'repl',
master_password = 'passw0rd'
master_log_file = 'master-bin.000001',
master_log_pos = 0;

START SLAVE;
```

## 查看binlog文件
+ binlog文件一般为mysql-bin.+序号
+ 重启mysql或运行flush logs,都会生成一个新的二进制日志，并且序号加1
+ 索引文件mysql-bin.index

```sql
SHOW BINARY LOGS;
```
```shell
mysqlbinlog --base64-output="decode-rows" mysql-bin.000001
```