
## mysql innodb lock 测试实验

### 前置 `performance_schema.data_locks` 表认知
`lock_mode` 字段组合结果表示：
`lock_type`|`lock_mode` | 锁
-----------|------------|-----
table|IX|意向锁
record|X|nextkey lock(gaplock + 记录锁)
record|X,GAP|gaplock
record|`X,REC_NOT_GAP`|记录锁


### 测试表:
```mysql
CREATE TABLE `lock_test` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `ut` int NOT NULL,
  `it` int NOT NULL,
  `t` int NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ukt` (`ut`),
  KEY `nit` (`it`)
) ENGINE=InnoDB AUTO_INCREMENT=72 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

### 表数据:
```mysql
mysql> select * from lock_test;    
+----+-----+-----+-----+
| id | ut  | it  | t   |
+----+-----+-----+-----+
|  4 |   3 |   2 |   3 |
|  5 |   4 |   3 |   2 |
|  6 |   5 |   3 |   2 |
|  7 |   6 |   5 |   4 |
|  8 |   7 |   5 |   4 |
|  9 |   8 |   6 |   4 |
| 10 |  20 |   7 |   9 |
| 11 |  21 |   7 |   9 |
| 12 |  22 |   7 |   9 |
| 16 |  33 |   9 |  11 |
| 17 |  34 |   9 |  10 |
| 18 |  35 |  15 |  10 |
| 37 |  55 |  16 |   3 |
| 38 |  56 |  17 |   3 |
| 39 |  59 |  18 |   3 |
| 40 |  60 |  30 |  15 |
| 41 |  61 |  30 |  16 |
| 42 |  62 |  31 |  30 |
| 43 |  63 |  31 |  30 |
| 44 |  64 |  32 |  33 |
| 45 | 123 | 100 | 100 |
| 50 |  25 | 100 | 100 |
| 66 | 172 | 100 | 100 |
| 67 | 124 | 100 | 100 |
| 68 | 125 | 100 | 100 |
| 70 | 170 | 100 | 100 |
| 71 | 171 | 100 | 100 |
+----+-----+-----+-----+
```

### 语句锁情况

* 锁主键用 where id = 11;
```mysql
mysql> begin;
mysql> select * from lock_test where id=11 for update;
mysql> select * from data_locks;         
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| ENGINE | ........ | OBJECT_SCHEMA | OBJECT_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| INNODB |          | my_test       | lock_test   | NULL       |       139862120425264 | TABLE     | IX            | GRANTED     | NULL      |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X,REC_NOT_GAP | GRANTED     | 11        |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+

mysql> rollback;
```

* 锁id, where id > 20, 结果应该是gapLock, 主键锁记录中，如果不是gapLock应该`lock_mode`字段有标记:
``` mysql
mysql> begin;
mysql> select * from lock_test where id>20 for update;
mysql> select * from data_locks;
+--------+----------+---------------+-------------+------------+-----------------------+-----------+-----------+-------------+------------------------+
| ENGINE | ........ | OBJECT_SCHEMA | OBJECT_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE | LOCK_STATUS | LOCK_DATA              |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+-----------+-------------+------------------------+
| INNODB |          | my_test       | lock_test   | NULL       |       139862120425264 | TABLE     | IX        | GRANTED     | NULL                   |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | supremum pseudo-record |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 66                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 50                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 37                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 38                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 39                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 40                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 41                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 42                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 43                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 44                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 45                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 67                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 68                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 70                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 71                     |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+-----------+-------------+------------------------+
mysql> rollback;
```

* 用唯一索引查询, where ut = 8, 可以大概看出是通过聚族索引先锁索引后锁主键索引
```mysql
mysql> begin;
mysql> select * from lock_test where ut=8 for update;
mysql> select * from data_locks;
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| ENGINE | ........ | OBJECT_SCHEMA | OBJECT_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| INNODB |          | my_test       | lock_test   | NULL       |       139862120425264 | TABLE     | IX            | GRANTED     | NULL      |
| INNODB |          | my_test       | lock_test   | ukt        |       139862120422304 | RECORD    | X,REC_NOT_GAP | GRANTED     | 8, 9      |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422648 | RECORD    | X,REC_NOT_GAP | GRANTED     | 9         |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+

mysql> rollback;

```

* 唯一索引范围条件, `where ut > 40` vs. `where ut > 40 and ut < 63`
``` mysql
mysql> begin;
mysql> select * from lock_test where ut>40 for update;
mysql> select * from data_locks;
+--------+----------+---------------+-------------+------------+-----------------------+-----------+-----------+-------------+------------------------+
| ENGINE | ........ | OBJECT_SCHEMA | OBJECT_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE | LOCK_STATUS | LOCK_DATA              |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+-----------+-------------+------------------------+
| INNODB |          | my_test       | lock_test   | NULL       |       139862120425264 | TABLE     | IX        | GRANTED     | NULL                   |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | supremum pseudo-record |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 66                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 50                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 4                      |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 5                      |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 6                      |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 7                      |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 8                      |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 9                      |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 10                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 11                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 12                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 16                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 17                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 18                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 37                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 38                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 39                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 40                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 41                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 42                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 43                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 44                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 45                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 67                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 68                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 70                     |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422304 | RECORD    | X         | GRANTED     | 71                     |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+-----------+-------------+------------------------+
29 rows in set (0.00 sec)

mysql> rollback;




mysql> begin;
mysql> select * from lock_test where ut > 40 and ut < 63 for update;
mysql> select * from data_locks;
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| ENGINE | ........ | OBJECT_SCHEMA | OBJECT_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| INNODB |          | my_test       | lock_test   | NULL       |       139862120425264 | TABLE     | IX            | GRANTED     | NULL      |
| INNODB |          | my_test       | lock_test   | ukt        |       139862120422304 | RECORD    | X             | GRANTED     | 55, 37    |
| INNODB |          | my_test       | lock_test   | ukt        |       139862120422304 | RECORD    | X             | GRANTED     | 56, 38    |
| INNODB |          | my_test       | lock_test   | ukt        |       139862120422304 | RECORD    | X             | GRANTED     | 59, 39    |
| INNODB |          | my_test       | lock_test   | ukt        |       139862120422304 | RECORD    | X             | GRANTED     | 60, 40    |
| INNODB |          | my_test       | lock_test   | ukt        |       139862120422304 | RECORD    | X             | GRANTED     | 61, 41    |
| INNODB |          | my_test       | lock_test   | ukt        |       139862120422304 | RECORD    | X             | GRANTED     | 62, 42    |
| INNODB |          | my_test       | lock_test   | ukt        |       139862120422304 | RECORD    | X             | GRANTED     | 63, 43    |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422648 | RECORD    | X,REC_NOT_GAP | GRANTED     | 37        |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422648 | RECORD    | X,REC_NOT_GAP | GRANTED     | 38        |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422648 | RECORD    | X,REC_NOT_GAP | GRANTED     | 39        |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422648 | RECORD    | X,REC_NOT_GAP | GRANTED     | 40        |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422648 | RECORD    | X,REC_NOT_GAP | GRANTED     | 41        |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422648 | RECORD    | X,REC_NOT_GAP | GRANTED     | 42        |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+
14 rows in set (0.00 sec)

mysql> rollback;

```

上面两个不同范围的条件，锁的结果不同，第一个条件`where ut > 40`，直接全表主键索引gap lock了，大概是因为 >40 的条件范围太广，锁 ut 索引和 主键索引对应gap的记录数太多了不划算;而后一个条件 `where ut > 40 and ut < 63`，则只锁了ut 索引的gaplock 范围，以及主键的记录锁.


* 非唯一索引条件, `where it = 15`
```mysql
mysql> begin;
mysql> select * from lock_test where it = 15 for update;
mysql> select * from data_locks;
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| ENGINE | ........ | OBJECT_SCHEMA | OBJECT_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| INNODB |          | my_test       | lock_test   | NULL       |       139862120425264 | TABLE     | IX            | GRANTED     | NULL      |
| INNODB |          | my_test       | lock_test   | nit        |       139862120422304 | RECORD    | X             | GRANTED     | 15, 18    |
| INNODB |          | my_test       | lock_test   | PRIMARY    |       139862120422648 | RECORD    | X,REC_NOT_GAP | GRANTED     | 18        |
| INNODB |          | my_test       | lock_test   | nit        |       139862120422992 | RECORD    | X,GAP         | GRANTED     | 16, 37    |
+--------+----------+---------------+-------------+------------+-----------------------+-----------+---------------+-------------+-----------+

mysql> rollback;
```

锁了3个锁：
1. it索引的 `15 -> 18` nextkey lock(it索引->主键索引,包括gap lock 和 record lock),由于这条nextkey lock, 别的事务无法插入 it=11 的记录， 因为it索引里,it=15的前一个值是9,所以锁住了(915]的gap;
2. 锁了it索引的gaplock 16(防止插入it=15的记录);
3. 最后锁了主键 id 18 的记录锁(防止现存it=15的数据被修改).


* 非索引字段where, `where t = 11`
结果直接锁住了整个表的主键索引的 nextkey lock, 



### lock tables xx read/write

锁表的情况用 `show open tables where in_use > 0` 查看
