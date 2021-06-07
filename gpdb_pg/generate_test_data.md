## Pg快速构建测试库



#### Seriales values

```sql
select id from generate_series(1,10) t(id);
```

generate_series 可以指定最大值，最小值，递增值。也可以生成时间等类型

```sql
\df generate_series
   Schema   |      Name       |         Result data type          |                        Argument data types                         |  Type
------------+-----------------+-----------------------------------+--------------------------------------------------------------------+--------
 pg_catalog | generate_series | SETOF bigint                      | bigint, bigint                                                     | normal
 pg_catalog | generate_series | SETOF bigint                      | bigint, bigint, bigint                                             | normal
 pg_catalog | generate_series | SETOF integer                     | integer, integer                                                   | normal
 pg_catalog | generate_series | SETOF integer                     | integer, integer, integer                                          | normal
 pg_catalog | generate_series | SETOF timestamp without time zone | timestamp without time zone, timestamp without time zone, interval | normal
 pg_catalog | generate_series | SETOF timestamp with time zone    | timestamp with time zone, timestamp with time zone, interval       | normal
```



#### 随机数

```sql
select random()  from generate_series(1,10);

#生成指定范围的整数：min+(random()*(max-min))::integer

select 5+(random()*(7-5))::integer from generate_series(1,10);
```



#### 随机字符串

```sql
# 生成指定长度的字符串
create or replace function f_random_str(length INTEGER) 
returns character varying AS $$
DECLARE
    result varchar(50);
BEGIN
    SELECT array_to_string(ARRAY(SELECT chr((65 + round(random() * 25)) :: integer)
    FROM generate_series(1,length)), '') INTO result;
    
    return result;
END;
$$ LANGUAGE plpgsql;

select f_random_str(10);

#利用md5函数

select md5(random()::text);

select md5(random()::text),f_random_str(5) from generate_series(1,10);
```



#### 重复字符串

```sql
repeat('abc', 10)  
```



#### 随机中文

```sql
create or replace function gen_hanzi(int) returns text as $$    
declare    
  res text;    
begin    
  if $1 >=1 then    
    select string_agg(chr(19968+(random()*20901)::int), '') into res from generate_series(1,$1);    
    return res;    
  end if;    
  return null;    
end;    
$$ language plpgsql strict;

select gen_hanzi(10) from generate_series(1,10);
```



#### 构建测试表

```
create table t_demo( 
    id integer,
    name varchar(20),
    course int,
    grade numeric(4,2),
    testtime date,
    note text);

insert into t_demo 
  select id,
         f_random_str(3+(random()*5)::integer) as name,
         (random()*100)::integer as course,
         (random()*99)::numeric(4,2) as grade,
         now() - ((random()*1000)::integer||' day')::interval as testtime,
         gen_hanzi(3+(random()*5)::integer) as note
         from generate_series(1,10000) t(id);
         
```



