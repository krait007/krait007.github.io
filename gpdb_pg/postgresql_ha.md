# Postgresql HA配置

### 1. 安装数据库(master  slave 服务器)

```bash
#demo服务器:  master-193.71.0.88  slave-193.71.0.51

#把二进制程序解压到 
/opt/pg11
#创建postgres用户
useradd -m postgres
#创建数据库数据目录
mkdir /opt/pgdata
chown postgres:postgres /opt/pgdata
chmod 0700 /opt/pgdata

#切换到postgres用户，编辑环境变量
vi .bash_profile

export PG_HOME=/opt/pg11
export PATH=$PG_HOME/bin:$PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PG_HOME/lib
export PGDATA=/opt/pgdata
export PGPORT=51234

#防火墙添加51234端口
firewall-cmd --permanent --zone=public --add-port=51234/tcp
```



### 2. master服务器

#### 初始化数据库

```bash
#切换到postgres用户
initdb -D $PGDATA
#启动数据库
pg_ctl start -D $PGDATA
```

#### psql创建用户

```sql
create user repuser replication LOGIN CONNECTION limit 3 encrypted password 'repuser';
```

#### 编辑 vi $PGDATA/pg_hba.conf

```c
host    replication     repuser         193.71.0.51/32           md5
```

#### 编辑 vi $PGDATA/postgresql.conf

```
listen_addresses = '*'
port = 51234
max_wal_senders = 10
wal_level = hot_standby
archive_mode = on 
archive_command = 'cd ./'
wal_keep_segments = 64
full_page_writes = on
wal_log_hints = on
```

重启数据库

```
pg_ctl restart -D $PGDATA
```



### 3. slave服务器

####  pg_basebackup创建备库

```bash
pg_basebackup -U repuser -D $PGDATA -Fp -Xs -v -P -h 193.71.0.88 -p 51234

#输出结果
Password: 
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/2000028 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_1442"
23827/23827 kB (100%), 1/1 tablespace                                         
pg_basebackup: write-ahead log end point: 0/2000130
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: base backup completed
```

编辑 vi $PGDATA/pg_hba.conf

```
host    replication     repuser         193.71.0.88/32           md5
```



### 4.配置recovery

主库: recovery.done 从库: recovery.conf

#### 配置master

```bash
cp /opt/pg11/share/recovery.conf.sample $PGDATA/recovery.done
vi $PGDATA/recovery.done
--------------------------------------------------
recovery_target_timeline = 'latest'
standby_mode = on
primary_conninfo = 'host=193.71.0.51 port=51234 user=repuser password=repuser'
trigger_file = '/opt/pgdata/trigger_file'
```

#### 配置slave

```bash
cp /opt/pg11/share/recovery.conf.sample $PGDATA/recovery.conf
vi $PGDATA/recovery.conf
--------------------------------------------------
recovery_target_timeline = 'latest'
standby_mode = on
primary_conninfo = 'host=193.71.0.88 port=51234 user=repuser password=repuser'
trigger_file = '/opt/pgdata/trigger_file'
```



### 5.查看状态

```sql
#psql
#主库
select sent_lsn,write_lsn,pid,state,client_addr,sync_state from pg_stat_replication;
#从库
select pg_last_wal_receive_lsn();
#主从库
select pg_is_in_recovery();
```

```bash
######################################################
#主库服务器
pg_controldata |grep 'Database cluster'
#Database cluster state:               in production
#从库服务器
pg_controldata |grep 'Database cluster'
#Database cluster state:               in archive recovery

######################################################
#主库服务器
ps -ef | grep postgres | grep walsender
postgres  1863  1840  0 10:51 ?        00:00:00 postgres: walsender repuser 193.71.0.51(47216) streaming 0/D001180
#从库服务器
[postgres@localhost]$ ps -ef | grep postgres | grep walreceiver
postgres  7602  7597  0 10:51 ?        00:00:14 postgres: walreceiver   streaming 0/D001180
```



### 6. 主从切换

```bash
#1. 关闭数据库
#登录主库服务器(postgres)
pg_ctl stop -m fast
#登录从库服务器(postgres)
pg_ctl stop -m fast

#2. 切换
#登录从库服务器postgres用户 
pg_ctl promote

---------------------------------------
[postgres@localhost]$ pg_ctl promote
waiting for server to promote....2019-04-25 16:18:39.775 CST [7598] LOG:  received promote request
2019-04-25 16:18:39.775 CST [7602] FATAL:  terminating walreceiver process due to administrator command
2019-04-25 16:18:39.775 CST [7598] LOG:  invalid record length at 0/D001260: wanted 24, got 0
2019-04-25 16:18:39.775 CST [7598] LOG:  redo done at 0/D001228
2019-04-25 16:18:39.776 CST [7598] LOG:  last completed transaction was at log time 2019-04-25 16:08:42.859152+08
2019-04-25 16:18:39.779 CST [7598] LOG:  selected new timeline ID: 2
2019-04-25 16:18:39.933 CST [7598] LOG:  archive recovery complete
2019-04-25 16:18:39.941 CST [7597] LOG:  database system is ready to accept connections
 done
server promoted
-----------------------------------------------------
#切换后，从库的recovery.conf文件名字变成了recovery.done

#切换原主库为从库(登录postgres用户 )
pg_ctl stop -m fast
cd $PGDATA
mv recovery.done recovery.conf


```

 

### 7. 验证测试

```sql
------------------------------------------------------
--验证是否自动同步

--分别登录主从库psql
--主库创建数据库、表和插入测试数据
create database dbtest;
\c dbtest
create table t_test ( id int);
insert into t_test select * from generate_series(1,1000);
--从库确认是否自动同步
\l
\c dbtest
select count(*) from t_test ;
--查看lsn
--主
select sent_lsn,write_lsn from pg_stat_replication;
--从
select pg_last_wal_receive_lsn();

--验证从数据库关闭再开启是否能自动同步
--从库 关闭
pg_ctl stop
--主库插入1000条记录
insert into t_test select * from generate_series(1001,2000);
--从库 启动
pg_ctl start
--确认数据库是否同步
dbtest=# select count(*) from t_test ;
 count 
-------
  2000
(1 row)
```



### 8.  常见问题及解决方法

```bash
#############################################################
#主从切换后出现数据不能同步
2019-04-25 16:43:16.703 CST [8576] ERROR:  requested starting point 0/E000000 on timeline 1 is not in this server's history
2019-04-25 16:43:16.703 CST [8576] DETAIL:  This server's history forked from timeline 1 at 0/D001260.
#解决方法
#登陆新从库服务器
pg_ctl stop
pg_rewind --target-pgdata=$PGDATA --source-server='host=193.71.0.51 port=51234 user=postgres' -P
#修改$PGDATA/recovery.done
vi $PGDATA/recovery.done
#修改IP地址
mv  $PGDATA/recovery.done $PGDATA/recovery.conf
#启动从服务
pg_ctl start

##############################################################################
#如果数据库较小，可以考虑直接用 pg_basebackup，然后再修改配置
```





