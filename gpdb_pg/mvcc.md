### 并发控制

#### ACID

Atomicity: 原子性

Consistency: 一致性

Isolation: 隔离性

Durability: 持久性



#### MVCC

Multi-Version Concurrency Contronl



#### 三种并发控制技术

mvcc

S2PL（Strict Two-phase Locking): 严格两阶段锁定

OCC(Optimistic Concurrency Control): 乐观并发控制

**postgresql使用一种MVCC的变体，叫做 快照隔离( Snapshot Isolation) SI**

SI不会出现ANSI SQL-92:  脏读、不可重读读、幻读， SI无法实现真正的可串行化。

PG9.1以后添加了 可串行化快照隔离(Serializable Snapshot Isolation) **SSI**

Postgresql 对 DML(select update insert delete等)使用SSI，对DDL( create table等) 使用2PL



#### 事务标识txid (transaction id)

txid:  32位无符号整数，取值空间42亿

在事务启动后执行内置函数 **txid_current**() 获取当前事务的txid

pg保留三个特殊txid:

 0 表示无效的txid

 1 表示初始化启动的txid，仅用于数据库集群的初始化过程

 2 表示冻结的txid



#### 元组结构

表页中的堆元组分为:**普通数据元组** 与 **TOAST**元组

堆元组由三部分组成: **HeapTupleHeaderData结构**、**空值位图** 及 **用户数据**

​    HeapTupleHeaderData                                                             空值位图                用户数据

t_xmin t_xmax t_cid t_ctid t_informask2 t_informask t_hoff | NULL bitmap   | User data              |

xmin: 插入此元组的事务txid

xmax: 保存删除或更新此元组的事务txid

cid: 保存命令标识(command id, cid) 意思是当前事务中，执行当前命令之前执行了多少sql命令，从零开始计数



```sql
create extension pageinspect;
\dx
select * from page_header(get_raw_page('t_demo', 0));
select lp as tuple, t_xmin, t_xmax, t_field3 as t_cid, t_ctid 
  from heap_page_items(get_raw_page(t_demo, 0));
  
#demo
dbdemo=# begin;
BEGIN
dbdemo=*# insert into t_demo values('22');
INSERT 0 1
dbdemo=*# insert into t_demo values('33');
INSERT 0 1
dbdemo=*# insert into t_demo values('44');
INSERT 0 1
dbdemo=*# commit;
COMMIT
dbdemo=# select lp as tuple, t_xmin, t_xmax, t_field3 as t_cid, t_ctid
  from heap_page_items(get_raw_page('t_demo', 0));
 tuple | t_xmin | t_xmax | t_cid | t_ctid
-------+--------+--------+-------+--------
     1 |    489 |      0 |     0 | (0,1)
     2 |    505 |      0 |     0 | (0,2)
     3 |    508 |      0 |     0 | (0,3)
     4 |    508 |      0 |     1 | (0,4)
     5 |    508 |      0 |     2 | (0,5)
(5 rows)


dbdemo=# begin;
BEGIN
dbdemo=*# delete from t_demo where id='33';
DELETE 1
dbdemo=*# select lp as tuple, t_xmin, t_xmax, t_field3 as t_cid, t_ctid
  from heap_page_items(get_raw_page('t_demo', 0));
 tuple | t_xmin | t_xmax | t_cid | t_ctid
-------+--------+--------+-------+--------
     1 |    489 |      0 |     0 | (0,1)
     2 |    505 |      0 |     0 | (0,2)
     3 |    508 |      0 |     0 | (0,3)
     4 |    508 |    509 |     0 | (0,4)
     5 |    508 |      0 |     2 | (0,5)
(5 rows)

dbdemo=*# select txid_current();
 txid_current
--------------
          509
(1 row)

dbdemo=*# commit;
COMMIT
dbdemo=# select lp as tuple, t_xmin, t_xmax, t_field3 as t_cid, t_ctid
  from heap_page_items(get_raw_page('t_demo', 0));
 tuple | t_xmin | t_xmax | t_cid | t_ctid
-------+--------+--------+-------+--------
     1 |    489 |      0 |     0 | (0,1)
     2 |    505 |      0 |     0 | (0,2)
     3 |    508 |      0 |     0 | (0,3)
     4 |    508 |    509 |     0 | (0,4)
     5 |    508 |      0 |     2 | (0,5)
(5 rows)
```



#### 自由空间映射 FSM(Free Space Map)

插入堆或索引元组时，pg使用表与索引相应的FSM来选择可供插入的页面

表和索引有各自的FSM

**pg_freespacemap**

```
dbdemo=# create extension pg_freespacemap ;
CREATE EXTENSION

dbdemo=# select *, round(100*avail/8192,2) as "freespace ratio" from pg_freespace('t_demo');
 blkno | avail | freespace ratio
-------+-------+-----------------
     0 |     0 |            0.00
(1 row)
```



#### 提交日志CLOG(commit log)

Commit log中保存事务的状态，提交日志分配于共享内存，并用于事务处理过程的全过程

PG定义了4种事务状态: IN_PROGRESS、COMMITTED、ABORTED、SUB_COMMITTED

CLOG在逻辑上是一个数组，由共享内存中一系列8KB页面组成，数组的序号索引对应着相应的事务的标识，其内容则是相应的事务的状态。

当PG关机或执行归档过程时,CLOG数据会写入p'g_clog子目录， pg10以后重名为 pg_xact



**事务快照**

事务快照在pg内部的文本格式定义为.  100 : 100 :  

内置函数 txid_current_snapshot 显示当前快照

```
dbdemo=# select txid_current_snapshot();
 txid_current_snapshot
-----------------------
 511:511:
(1 row)

###########################################
dbdemo=# begin;
BEGIN
dbdemo=*# select txid_current_snapshot();
 txid_current_snapshot
-----------------------
 511:511:
(1 row)

dbdemo=*# select txid_current();
 txid_current
--------------
          511
(1 row)

dbdemo=*# insert into t_demo values('55');
INSERT 0 1
dbdemo=*# select txid_current_snapshot();
 txid_current_snapshot
-----------------------
 511:511:
(1 row)

dbdemo=*# \!
/ # psql -U postgres
psql (13.0)
Type "help" for help.

postgres=# \c dbdemo
You are now connected to database "dbdemo" as user "postgres".
dbdemo=# select txid_current_snapshot();
 txid_current_snapshot
-----------------------
 511:511:
(1 row)

dbdemo=# begin;
BEGIN
dbdemo=*# select txid_current_snapshot();
 txid_current_snapshot
-----------------------
 511:511:
(1 row)

dbdemo=*# update t_demo set id='66' where id='22';
UPDATE 1
dbdemo=*# select txid_current_snapshot();
 txid_current_snapshot
-----------------------
 511:511:
(1 row)

dbdemo=*# select txid_current();
 txid_current
--------------
          512
(1 row)

dbdemo=*# commit;
COMMIT
dbdemo=# select txid_current();
 txid_current
--------------
          513
(1 row)

dbdemo=# select txid_current_snapshot();
 txid_current_snapshot
-----------------------
 511:514:511
(1 row)

dbdemo=# exit
/ # exit
dbdemo=*# select txid_current_snapshot();
 txid_current_snapshot
-----------------------
 511:514:
(1 row)

dbdemo=*# select txid_current();
 txid_current
--------------
          511
(1 row)

dbdemo=*# commit;
COMMIT
dbdemo=# select txid_current_snapshot();
 txid_current_snapshot
-----------------------
 514:514:
(1 row)

dbdemo=#
```

txid_current_snapshot的文本表示:  xmin : xmax : xip_list          例如   511:514:511

xmin 最早仍然活跃的事务的txid。

xmax 第一个尚未分配的txid

xip_list 获取快照时活跃事务的txid列表



**可见性规则检查**

提示位(Hint Bits)

pg内部提供了三个函数   TransactionIdIsInProgress、TransActionIdDidCommit和 T'ransactionIdDidAbort

提示位 设置到元组 t_informask 字段中