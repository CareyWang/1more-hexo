---
title: MySQL 事务嵌套解决方案小结
urlname: mysql-transaction-nesting
date: 2019-07-12 14:27:34
tags: MySQL
---

&emsp;&emsp;MySQL 事务机制被广泛用于处理操作量大、复杂度高的数据，日常业务中也很常见。在默认的 MySQL 配置下，事务默认是自动提交的，因此要显式地开启一个事务务须使用命令 BEGIN 或 START TRANSACTION，或者执行命令 SET AUTOCOMMIT=0，用来禁止使用当前会话的自动提交。

--- 

&emsp;&emsp;MySQL [官方文档](http://dev.mysql.com/doc/refman/5.0/en/implicit-commit.html)中，明确的说明了不支持嵌套事务嵌套。但是在我们开发一个复杂的系统时，由于思维的局限性，函数之间相互调用中，事务的嵌套很难彻底避免。

```sql
mysql> SELECT `value` FROM tmptable.transaction_test;
+-------+
| value |
+-------+
| 1     |
+-------+
1 row in set (0.00 sec)

mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO tmptable.transaction_test (`value`) VALUES ( '2' );
Query OK, 1 row affected (0.01 sec)

mysql> START TRANSACTION;
Query OK, 0 rows affected (0.03 sec)

mysql> INSERT INTO tmptable.transaction_test (`value`) VALUES ( '3' );
Query OK, 1 row affected (0.02 sec)

mysql> COMMIT;
Query OK, 0 rows affected (0.04 sec)

mysql> ROLLBACK;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT `value` FROM tmptable.transaction_test;
+-------+
| value |
+-------+
| 1     |
| 2     |
| 3     |
+-------+
3 rows in set (0.00 sec)
```

&emsp;&emsp;这段 SQL 的执行结果，按常规思维理解应该是 commit 操作提交了 2 的insert，rollback 操作回滚了 3 的 insert，最终结果应该是 1, 2。然而事实上，看上去 MySQL 只是执行了其中的 commit 操作，后续的 rollback 并没有带来数据上的改变。当 SQL 解释器遇到 START TRANSACTION 时候会触发 commit，也就是说 2 的 insert 操作在第二次开启事务时，已经被隐式地提交了。

# ORM 中常见的解决思路
- [Doctrine](https://www.doctrine-project.org/)
TODO
- Laravel
TODO