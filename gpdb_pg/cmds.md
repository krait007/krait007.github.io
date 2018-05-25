#extension cmds
```
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

#function
```
SELECT pg_proc.proname, pg_type.typname, pg_proc.pronargs 
FROM pg_proc JOIN pg_type ON (pg_proc.prorettype = pg_type.oid)
WHERE pg_type.typname != 'void'
  AND pronamespace = (SELECT pg_namespace.oid FROM pg_namespace WHERE nspname = 'public');
```