## TPC-H Tool for Postgresql 

### 环境

OS：Linux（Centos 7.4） 
TPC-H：2.17.3 
PostgreSQL：10.1
TPC-H Download：<http://www.tpc.org/tpch/>



### 一.编译生成: dbgen、qgen

**1.Download tpc-h-tool.zip**

**2.unzip tpc-h-tool.zip  && cd 2.17.3/dbgen**

**3.vi makefile.suite**

```makefile
CC      =  gcc
# Current values for DATABASE are: INFORMIX, DB2, TDAT (Teradata)
#                                  SQLSERVER, SYBASE, ORACLE, VECTORWISE
# Current values for MACHINE are:  ATT, DOS, HP, IBM, ICL, MVS,
#                                  SGI, SUN, U2200, VMS, LINUX, WIN32
# Current values for WORKLOAD are:  TPCH
DATABASE= ORACLE    #use oracle template for pg
MACHINE = LINUX
WORKLOAD = TPCH
```

**4. vi print.c**

```c
#生成的tbl数据每一行的末尾会有一个“|”
#for pg, 去掉每行末尾的 ‘|’
#注释145和147行，如下所示

//#ifdef EOL_HANDLING
        if (sep)
//#endif /* EOL_HANDLING */
        fprintf(target, "%c", SEPARATOR);
        return(0);
}
```

**4. make -f makefile.suite**



### 二.运行dbgen生成.tbl数据

```bash
#-s 1  1G
./dbgen -s 1 -f
#生成八个.tbl文件
```



### 三. 导入数据

```bash
#在dbgen目录下启动pg
psql
#创建数据库
postgres=# create database tpcd;
postgres=# \c tpcd

#创建表
tpcd=# \i dss.ddl

#在psql里tpcd库中导入表数据
Copy region FROM '/tpc/tpc-h-2.17.3/dbgen/dbgen/region.tbl' WITH DELIMITER AS '|';
Copy nation FROM '/tpc/tpc-h-2.17.3/dbgen/dbgen/nation.tbl' WITH DELIMITER AS '|';
Copy part FROM '/tpc/tpc-h-2.17.3/dbgen/dbgen/part.tbl' WITH DELIMITER AS '|';
Copy supplier FROM '/tpc/tpc-h-2.17.3/dbgen/dbgen/supplier.tbl' WITH DELIMITER AS '|';
Copy customer FROM '/tpc/tpc-h-2.17.3/dbgen/dbgen/customer.tbl' WITH DELIMITER AS '|';
Copy lineitem FROM '/tpc/tpc-h-2.17.3/dbgen/dbgen/lineitem.tbl' WITH DELIMITER AS '|';
Copy partsupp FROM '/tpc/tpc-h-2.17.3/dbgen/dbgen/partsupp.tbl' WITH DELIMITER AS '|';
Copy orders FROM '/tpc/tpc-h-2.17.3/dbgen/dbgen/orders.tbl' WITH DELIMITER AS '|';


```

### 四. 创建表约束主键和外键

```sql
--cat dss.ri
--copy 如下语句，并删除schema "TPCD." 修改foreign key 写法

-- For table REGION
ALTER TABLE REGION
ADD PRIMARY KEY (R_REGIONKEY);

-- For table NATION
ALTER TABLE NATION
ADD PRIMARY KEY (N_NATIONKEY);

ALTER TABLE NATION
--ADD FOREIGN KEY NATION_FK1 (N_REGIONKEY) references REGION;
ADD CONSTRAINT NATION_FK1 FOREIGN KEY (N_REGIONKEY) REFERENCES REGION (r_regionkey);

-- For table PART
ALTER TABLE PART
ADD PRIMARY KEY (P_PARTKEY);

-- For table SUPPLIER
ALTER TABLE SUPPLIER
ADD PRIMARY KEY (S_SUPPKEY);

ALTER TABLE SUPPLIER
--ADD FOREIGN KEY SUPPLIER_FK1 (S_NATIONKEY) references NATION;
ADD CONSTRAINT SUPPLIER_FK1 FOREIGN KEY (S_NATIONKEY) references NATION(n_nationkey);

-- For table PARTSUPP
ALTER TABLE PARTSUPP
ADD PRIMARY KEY (PS_PARTKEY,PS_SUPPKEY);

-- For table CUSTOMER
ALTER TABLE CUSTOMER
ADD PRIMARY KEY (C_CUSTKEY);

ALTER TABLE CUSTOMER
--ADD FOREIGN KEY CUSTOMER_FK1 (C_NATIONKEY) references NATION;
ADD CONSTRAINT CUSTOMER_FK1 FOREIGN KEY (C_NATIONKEY) references NATION(n_nationkey);

-- For table LINEITEM
ALTER TABLE LINEITEM
ADD PRIMARY KEY (L_ORDERKEY,L_LINENUMBER);

-- For table ORDERS
ALTER TABLE ORDERS
ADD PRIMARY KEY (O_ORDERKEY);

-- For table PARTSUPP
ALTER TABLE PARTSUPP
--ADD FOREIGN KEY PARTSUPP_FK1 (PS_SUPPKEY) references SUPPLIER;
ADD CONSTRAINT PARTSUPP_FK1 FOREIGN KEY (PS_SUPPKEY) references SUPPLIER(s_suppkey);

ALTER TABLE PARTSUPP
--ADD FOREIGN KEY PARTSUPP_FK2 (PS_PARTKEY) references PART;
ADD CONSTRAINT PARTSUPP_FK2 FOREIGN KEY (PS_PARTKEY) references PART(ps_partkey);

-- For table ORDERS
ALTER TABLE ORDERS
--ADD FOREIGN KEY ORDERS_FK1 (O_CUSTKEY) references CUSTOMER;
ADD CONSTRAINT ORDERS_FK1 FOREIGN KEY (O_CUSTKEY) references CUSTOMER(c_custkey);

-- For table LINEITEM
ALTER TABLE LINEITEM
--ADD FOREIGN KEY LINEITEM_FK1 (L_ORDERKEY)  references ORDERS;
ADD CONSTRAINT LINEITEM_FK1 FOREIGN KEY (L_ORDERKEY) references ORDERS(o_orderkey);

ALTER TABLE LINEITEM
--ADD FOREIGN KEY LINEITEM_FK2 (L_PARTKEY,L_SUPPKEY) references PARTSUPP;
ADD CONSTRAINT LINEITEM_FK2 FOREIGN KEY (L_PARTKEY,L_SUPPKEY) references 
   PARTSUPP(ps_partkey, ps_suppkey) ;

```

### 五. 生成查询sql

```bash
# -d 表示默认参数，1表示按照模板一生成sql语句
# queries_pg download url: 
# https://github.com/krait007/tpch_pg/tree/master/dbgen/queries_pg
cp qgen dists.dss queries_pg
cd queries_pg
#./qgen -d 1 >d1.sql  
rm workload.sql
for q in `seq 1 22`;do ./qgen -d $q -r $((`cat /dev/urandom|od -N3 -An -i` % 10000)) >> workload.sql;done;

#替换^M   ^M输入方法:  ctrl+V+M, 注意大写
sed -i "s/^M//g" workload.sql 

```

### 六. 执行workload

```bash
#start the processes 
for c in `seq 1 4`; do /usr/bin/time -f "total=%e" -o result-$c.log \
psql tpcd < workload.sql >/dev/null 2>&1 & done; 

# wait for the processes 
for p in `jobs -p`; do wait $p; done;
```

