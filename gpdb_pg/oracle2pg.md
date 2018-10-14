## 从oracle迁移到postgresql

为省去一些基础软件的安装，方便数据迁移和验证，采用docker完成迁移工作，具体步骤如下：



## 一、准备迁移数据

### 导出oracle数据库
```
expdp dbuser directory=DATA_PUMP_DIR dumpfile=dbxxxx.dmp logfile=dbxxxx.log SCHEMAS=dbxxxx
```



## 二、准备镜像

### 获取oracle镜像
```
docker pull alexeiled/docker-oracle-xe-11g
```

### 获取postgresql镜像
```
docker pull centos/postgresql-96-centos7
```

### 准备制作ora2pg镜像

下载oracle client安装包  http://www.oracle.com/technetwork/topics/linuxx86-64soft-092277.html
oracle-instantclient11.2-basic-11.2.0.1.0-1.x86_64.rpm
oracle-instantclient11.2-devel-11.2.0.1.0-1.x86_64.rpm
oracle-instantclient11.2-sqlplus-11.2.0.1.0-1.x86_64.rpm

下载: ora2pg 安装包
https://github.com/darold/ora2pg/archive/v18.2.zip

下载DBD-Oracle-1.74.tar.gz
http://www.perl.org/CPAN/authors/id/P/PY/PYTHIAN/DBD-Oracle-1.74.tar.gz



### 编写Dockerfile
```
FROM fedora

MAINTAINER kk <jinjh007@gmail.com>

RUN dnf update -y && dnf install unzip gcc perl perl-DBI perl-CPAN tar perl-DBD-Pg postgresql-devel perl-open -y
ADD ./pkgs/ora2pg-18.2.zip /tmp
RUN cd /tmp && mv /tmp/ora2pg* /tmp/ora2pg && (cd /tmp/ora2pg && perl Makefile.PL && make && make install)
RUN mkdir /usr/lib/oracle/11.2/client64/network/admin -p
ADD ./pkgs/oracle-instantclient11.2*-11.2.0.1.0-1.x86_64.rpm /tmp/
ADD ./pkgs/DBD-Oracle-1.74.tar.gz /tmp/
RUN dnf install /tmp/oracle-instantclient11.2-basic-11.2.0.1.0-1.x86_64.rpm --nogpgcheck -y && \
    dnf install /tmp/oracle-instantclient11.2-devel-11.2.0.1.0-1.x86_64.rpm --nogpgcheck -y && \
    dnf install /tmp/oracle-instantclient11.2-sqlplus-11.2.0.1.0-1.x86_64.rpm --nogpgcheck -y && \
    yum install libaio -y

ENV ORACLE_HOME=/usr/lib/oracle/11.2/client64
ENV TNS_ADMIN=/usr/lib/oracle/11.2/client64/network/admin
ENV LD_LIBRARY_PATH=/usr/lib/oracle/11.2/client64/lib
ENV PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/usr/lib/oracle/11.2/client64/bin
RUN cd /tmp && mv /tmp/DBD-Ora* /tmp/DBD-Oracle
RUN cd /tmp/DBD-Oracle && perl Makefile.PL -l && make && make install
RUN mkdir /data
RUN rm /tmp/*
```

### 编译镜像

```
docker build . -t ora2pg
```



## 三、迁移

### 准备oracle导出文件的备份文件

```
mkdir /home/ora2pg
cp dbxxxx.*  /home/ora2pg/
```

### 运行一个oracle容器

```
docker run -d -p 1521:1521 -p 8080:8080 --name oracle -v /home/ora2pg/oradata:/data  --privileged alexeiled/docker-oracle-xe-11g  
```

### 进入oracle容器，创建用户和导入数据

```
docker exec -it oracle /bin/bash
mkdir /data/oradata && chown oracle:dba /data/oradata
sqlplus system/oracle

create tablespace ts_demo DATAFILE  '/data/oradata/ts_demo.dbf' size 500m;
create user dbuser identified by dbpasswd default tablespace ts_demo;
grant dba to dbuser;
grant create table to dbuser;
```

### 导入数据库dbxxxx 

```
cd /data
cp dbxxxx.* /u01/app/oracle/admin/XE/dpdump/
chown oracle:dba /u01/app/oracle/admin/XE/dpdump/dbxxxx.*
impdp dbuser directory=DATA_PUMP_DIR dumpfile=dbxxxx.dmp logfile=dbxxxx.log SCHEMAS=dbxxxx
#用户名和表空间如果不一致，可以映射: remap_schema=olduser:newuser remap_tablespace=old_ts:new_ts 
```

### 退出oracle容器,准备迁移配置文件

```
#oracle container ip: 192.168.1.78
vi cat ora2pg_obj.conf 
ORACLE_HOME /u01/app/oracle/product/11.2.0/xe
ORACLE_DSN  dbi:Oracle:host=192.168.1.78;sid=xe
ORACLE_HOME /u01/app/oracle/product/11.2.0/xe
ORACLE_DSN  dbi:Oracle:host=192.168.1.78;sid=xe
ORACLE_USER dbuser
ORACLE_PWD  dbpasswd
SCHEMA      dbxxxx
USER_GRANTS     0
DEBUG       0
ORA_INITIAL_COMMAND
EXPORT_SCHEMA   0
CREATE_SCHEMA   1
COMPILE_SCHEMA  0
TYPE        TABLE, SEQUENCE, FUNCTION, PROCEDURE, TRIGGER, VIEW
OUTPUT      /data/dbxxxx_obj.sql
```

```
vi cat ora2pg_data.conf 
ORACLE_HOME /u01/app/oracle/product/11.2.0/xe
ORACLE_DSN  dbi:Oracle:host=192.168.1.78;sid=xe
ORACLE_HOME /u01/app/oracle/product/11.2.0/xe
ORACLE_DSN  dbi:Oracle:host=192.168.1.78;sid=xe
ORACLE_USER dbuser
ORACLE_PWD  dbpasswd
SCHEMA      dbxxxx
USER_GRANTS     0
DEBUG       0
ORA_INITIAL_COMMAND
EXPORT_SCHEMA   0
CREATE_SCHEMA   1
COMPILE_SCHEMA  0
TYPE        INSERT
OUTPUT      /data/dbxxxx_data.sql
```

### 生成创建TABLE, SEQUENCE, FUNCTION, PROCEDURE, TRIGGER, VIEW脚本

```
docker run -it --rm --network host -v /home/ora2pg:/data ora2pg  ora2pg -c /data/ora2pg_obj.conf
```

### 生成创建迁移数据INSERT脚本

```
docker run -it --rm --network host -v /home/ora2pg:/data ora2pg  ora2pg -c /data/ora2pg_data.conf
```

### 启动postgresql容器

```
cd /home/ora2pg && mkdir pgdata && cp dbxxxx_*.sql /home/ora2pg/pgdata
docker run -d --name pg -e POSTGRESQL_USER=dbuser -e POSTGRESQL_PASSWORD=dbpasswd \
       -e POSTGRESQL_DATABASE=dbxxxx -p 5432:5432 -v /home/ora2pg/pgdata:/data \
       centos/postgresql-96-centos7 
```

### 进入pq容器,导入数据

```
docker exec -it pg /bin/bash

#创建用户和导入数据库对象和数据
psql -U dbuser -d dbxxxx
#dbxxxx=> \c dbxxxx
dbxxxx=> \i /data/dbxxxx_obj.sql
dbxxxx=> \i /data/dbxxxx_data.sql
```

到此数据库迁移完成，应用迁移也可以通过docker容器来验证,例如tomcat



## 四、修改应用

各个应用系统相差很大，具体问题具体解决。例如：

1. Error committing transaction.  Cause: org.postgresql.util.PSQLException: Cannot commit when autoCommit is enabled
   数据源配置：        <property name="defaultAutoCommit" value="false"></property>  

1. oracle中字段名是大写的，迁移后变成小写  

2. oracle序列访问方式为 SEQ_XXXX.nextval postgresql需要修改为 nextval(seq_xxxx);

3. 当前时间: sysdate 修改为 now()

4. nvl 修改为 coalesce

5. 分页:  offset idx  limit pagesize

   

## 五、安装oracle兼容插件，减少应用sql修改

```
#下载：orafce 3.6.1 
https://pgxn.org/dist/orafce/

#编译 放到postgresql源码编译环境
unzip orafce-3.6.1.zip
cd orafce-3.6.1
make -j 2
make install

cd $PG_BIN

#打包- pg10
tar czvf orafce-3.6.1-bin.tar.gz lib/postgresql/orafce.so share/postgresql/extension/orafce* doc/postgresql/extension/*.orafce
lib/postgresql/orafce.so
share/postgresql/extension/orafce--3.2--3.3.sql
share/postgresql/extension/orafce--3.3--3.4.sql
share/postgresql/extension/orafce--3.4--3.5.sql
share/postgresql/extension/orafce--3.5--3.6.sql
share/postgresql/extension/orafce--3.6.sql
share/postgresql/extension/orafce.control
doc/postgresql/extension/COPYRIGHT.orafce
doc/postgresql/extension/INSTALL.orafce

#安装
到目标数据库: 例如 pg10
cp lib/postgresql/orafce.so /usr/lib/postgresql/10/lib/
cp share/postgresql/extension/orafce* /usr/share/postgresql/10/extension/

#登录目标数据库
dbxxxx=> create extension orafce;

#修改权限
dbxxxx=>grant all schema oracle to public;
dbxxxx=>alter user xxxx set search_path="$user", public, oracle;

```

使用兼容函数集后， 原应用的sql修改量会大大减少，比如 to_char, nvl,  日期函数等都可以继续使用。 注意点:

```
1. postgresql大版本差异比较大，所以 编译环境的pg版本和目标数据库的版本最好一致
2. orafce实现了很大一部分函数，但不是全部，比如round函数没有。
```

