## Extensions

Extensions are add-ons that you can install in a PostgreSQL database to extend functionality beyond the base offerings. People collaborating, building, and freely sharing new features.

不需要为所有的数据库安装所有的Extension, 按需安装即可, 如果需要所有的数据库安装一些Extension，可以创建一个模板数据库，对这个模板数据库安装需要的Extension集，新建数据库可以继承这个模板库

## Extensions installed in a database
```sql
SELECT name, default_version, installed_version, comment
FROM pg_available_extensions
WHERE installed_version IS NOT NULL
ORDER BY name;
```

Get details about a particular extension already installed in your database
```sql
\dx+ extension_name

or

#only for pg
SELECT pg_describe_object(D.classid,D.objid,0) AS description
FROM pg_catalog.pg_depend AS D INNER JOIN pg_catalog.pg_extension AS E
ON D.refobjid = E.oid
WHERE D.refclassid = 'pg_catalog.pg_extension'::pg_catalog.regclass AND
deptype = 'e' AND E.extname = 'extension_name';
```

Installing Extensions
```
CREATE EXTENSION extension_name;
CREATE EXTENSION extension_name SCHEMA my_extensions;
```

## Create Extensions

> local page
>
> [Writing Postgres Extensions - the Basics](/gpdb_pg/extensions/WritingPostgresExtensions-i-TheBasics)

> [Writing Postgres Extensions - Types and Operators](/gpdb_pg/extensions/WritingPostgresExtensions-ii-TypesAndOperators)

> [Writing Postgres Extensions - Debugging](/gpdb_pg/extensions/WritingPostgresExtensions-iii-Debugging)

> [Writing Postgres Extensions - Testing](/gpdb_pg/extensions/WritingPostgresExtensions-iv-Testing)

> [Writing Postgres Extensions Code Organization and Versioning](/gpdb_pg/extensions/WritingPostgresExtensions-v-CodeOrganizationAndVersioning)
>
> #source origin

> [Writing Postgres Extensions - the Basics](http://big-elephants.com/2015-10/writing-postgres-extensions-part-i/)

> [Writing Postgres Extensions - Types and Operators](http://big-elephants.com/2015-10/writing-postgres-extensions-part-ii/)

> [Writing Postgres Extensions - Debugging](http://big-elephants.com/2015-10/writing-postgres-extensions-part-iii/)

> [Writing Postgres Extensions - Testing](http://big-elephants.com/2015-11/writing-postgres-extensions-part-iv/)

> [Writing Postgres Extensions Code Organization and Versioning](http://big-elephants.com/2015-11/writing-postgres-extensions-part-v/ )

> [pl-swift](https://pl-swift.github.io/)

> [pl-go](https://github.com/microo8/plgo)

> [PostgreSQL HeteroDB GPU 加速 - pl/cuda , pg-strom , heterodb](http://heterodb.github.io/pg-strom/)

```sql
create extension extension_name;   #dblink
create extension extension_name with schema pg_catalog;

#old version
cd $GPHOME/share/postgresql/contrib/
psql -d database -U gpadmin -f dblink.sql

#query current extensions
select * from pg_extension;
\dx

#alter
alter extension extension_name set schema pg_catalog;  

```
