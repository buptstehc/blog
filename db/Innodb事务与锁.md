# Innodb事务隔离级别
- 未提交读(Read Uncommitted)：事务A读到事务B中未提交的数据，也就是脏读
- 提交读(Read Committed)：事务A只能读到事务B中已提交的数据
- 可重复读(Repeated Read)：事务在其生命周期中读到的数据不会发生变化，也是Innodb的默认级别
- 串行读(Serializable)：完全串行化的读，每次读都需要获得表级共享锁，读写相互都会阻塞

## 示例
### 测试表

```
mysql> show create table task;
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                    |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| task  | CREATE TABLE `task` (
  `id` int(11) NOT NULL,
  `status` int(11) DEFAULT NULL,
  `name` varchar(255) COLLATE utf8_bin DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select * from task;
+----+--------+------+
| id | status | name |
+----+--------+------+
|  1 |      0 | foo  |
|  2 |      0 | bar  |
+----+--------+------+
2 rows in set (0.00 sec)
```

### Read Committed
Innodb的默认事务级别为RR：

```
mysql> select @@global.tx_isolation,@@tx_isolation;
+-----------------------+-----------------+
| @@global.tx_isolation | @@tx_isolation  |
+-----------------------+-----------------+
| REPEATABLE-READ       | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.00 sec)
```
修改session的事务级别为RC：

```
mysql> set session transaction isolation level read committed;
Query OK, 0 rows affected (0.00 sec)
```
|时间|事务A|事务B|
|---|---|---|
|1|begin;|begin;|
|2|select * from task where id = 1;||
|3||update task set name = 'baz' where id = 1;|
|4||commit;|
|5|select * from task where id = 1;||
|6|commit;||

事务A在t2时刻查到的数据如下：

```
mysql> select * from task where id = 1;
+----+--------+------+
| id | status | name |
+----+--------+------+
|  1 |      0 | foo  |
+----+--------+------+
1 row in set (0.00 sec)
```

事务A在t5时刻，查到了事务B提交的数据：

```
mysql> select * from task where id = 1;
+----+--------+------+
| id | status | name |
+----+--------+------+
|  1 |      0 | baz  |
+----+--------+------+
1 row in set (0.00 sec)
```

### Repeated Read
将session的事务级别改回为RR：

```
mysql> set session transaction isolation level repeatable read;
Query OK, 0 rows affected (0.00 sec)
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)
```

|时间|事务A|事务B|事务C|
|---|---|---|---|
|1|begin;|begin;|begin;|
|2|select * from task where name = 'bar';|||
|3||update task set name = 'bar' where id = 1;||
|4|||insert into task (id, status, name) values (3, 0, 'bar');|
|5||commit;||
|6|||commit;|
|7|select * from task where name = 'bar';|||
|8|commit;|||

事务A在t2时刻查到的数据如下：

```
mysql> select * from task where name = 'bar';
+----+--------+------+
| id | status | name |
+----+--------+------+
|  2 |      0 | bar  |
+----+--------+------+
1 row in set (0.00 sec)
```

事务B和C相继提交后，在t7时刻，事务A读到的数据跟t2时刻是一致的：

```
mysql> select * from task where name = 'bar';
+----+--------+------+
| id | status | name |
+----+--------+------+
|  2 |      0 | bar  |
+----+--------+------+
1 row in set (0.00 sec)
```

事务A并没有读到事务C新增的数据，也就是不存在幻读的问题。

# MVCC
# GAP锁