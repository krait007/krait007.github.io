# Writing Postgres Extensions Code Organization and Versioning

In the last four posts of our series on writing Postgres Extensions, we got the [basics](http://big-elephants.com/2015-11/writing-postgres-extensions-part-v/2015-10/writing-postgres-extensions-part-i/) [covered types and operators](http://big-elephants.com/2015-11/writing-postgres-extensions-part-v/2015-10/writing-postgres-extensions-part-ii/), [introduced a debugger](http://big-elephants.com/2015-11/writing-postgres-extensions-part-v/2015-10/writing-postgres-extensions-part-iii/) and [completed the test suite](http://big-elephants.com/2015-11/writing-postgres-extensions-part-v/2015-11/writing-postgres-extensions-part-iv/).

Now let’s add another type and see how we can organize the code base when it grows.

You can find the last post’s code base [on the github branch part_iv](https://github.com/adjust/postgresql_extension_demo/tree/part_iv) Today’s changes can be followed on [branch part_v](https://github.com/adjust/postgresql_extension_demo/tree/part_v)

## Versioning

We might be happy with our Extension and use it in production for a while without any issues. Now that our business succeed, the range for `integer` might no longer be enough. That means we’ll need another `bigint` based type `bigbase36`, which can have up to 13 characters.

The problem here is that we can’t simply drop the extension and re-install the new version.

```sql
`test=# drop extension base36 ; ERROR:  cannot drop extension base36 because other objects depend on it DETAIL:  table important_data column token depends on type base36 HINT:  Use DROP ... CASCADE to drop the dependent objects too. `
```

If we `DROP ... CASCADE` here, all our data would be lost. Also, dumping and recreating is not an option for a terabyte-sized database. What we want is to `ALTER EXTENSION UPDATE TO '0.0.2'`. Luckily, Postgres has Versioning for Extensions built in. Remember in the `base36.control` file we defined:

base36.control

```
`# base36 extension comment = 'base36 datatype' default_version = '0.0.1' relocatable = true `
```

Version ‘0.0.1’ is the default Version used when we execute `CREATE EXTENSION base36`, leading to the import of the `base36--0.0.1.sql` script file. Let’s create another one:

```bash
`cp base36--0.0.1.sql base36--0.0.2.sql `
```

And default to this one:

base36.control

```bash
`# base36 extension comment = 'base36 datatype' default_version = '0.0.2' relocatable = true `
```

And see if it builds:

```bash
`make clean && make && make install && make installcheck `
```

Getting

```bash
`... ERROR:  could not stat file "/usr/local/Cellar/postgresql/9.4.0/share/postgresql/extension/base36--0.0.2.sql": No such file or directory command failed: "/usr/local/Cellar/postgresql/9.4.0/bin/psql" -X -c "CREATE EXTENSION IF NOT EXISTS \"base36\"" "contrib_regression" make: *** [installcheck] Error 2 `
```

Hmmm, it wants to use `extension/base36--0.0.2.sql` but can’t find it.

Let’s fix the Makefile and tell Postgres to use all files following the pattern `*--*.sql`.

Makefile

```makefile
`EXTENSION     = base36                          # the extensions name DATA          = $(wildcard *--*.sql)            # script files to install `
```

In `base36--0.0.2.sql` we can now add the `bigbase36` type

base36–0.0.2.sql

```sql
`-- base36 stuff omitted  CREATE FUNCTION bigbase36_in(cstring) RETURNS bigbase36 AS '$libdir/base36' LANGUAGE C IMMUTABLE STRICT;  CREATE FUNCTION bigbase36_out(bigbase36) RETURNS cstring AS '$libdir/base36' LANGUAGE C IMMUTABLE STRICT;  CREATE TYPE bigbase36 (   INPUT          = bigbase36_in,   OUTPUT         = bigbase36_out,   LIKE           = bigint );  CREATE FUNCTION bigbase36_eq(bigbase36, bigbase36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int8eq';  CREATE FUNCTION bigbase36_ne(bigbase36, bigbase36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int8ne';  CREATE FUNCTION bigbase36_lt(bigbase36, bigbase36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int8lt';  CREATE FUNCTION bigbase36_le(bigbase36, bigbase36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int8le';  CREATE FUNCTION bigbase36_gt(bigbase36, bigbase36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int8gt';  CREATE FUNCTION bigbase36_ge(bigbase36, bigbase36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int8ge';  CREATE FUNCTION bigbase36_cmp(bigbase36, bigbase36) RETURNS integer LANGUAGE internal IMMUTABLE AS 'btint8cmp';  CREATE FUNCTION hash_bigbase36(bigbase36) RETURNS integer LANGUAGE internal IMMUTABLE AS 'hashint8';  CREATE OPERATOR = (   LEFTARG = bigbase36,   RIGHTARG = bigbase36,   PROCEDURE = bigbase36_eq,   COMMUTATOR = '=',   NEGATOR = '<>',   RESTRICT = eqsel,   JOIN = eqjoinsel,   HASHES, MERGES );  CREATE OPERATOR <> (   LEFTARG = bigbase36,   RIGHTARG = bigbase36,   PROCEDURE = bigbase36_ne,   COMMUTATOR = '<>',   NEGATOR = '=',   RESTRICT = neqsel,   JOIN = neqjoinsel );  CREATE OPERATOR < (   LEFTARG = bigbase36,   RIGHTARG = bigbase36,   PROCEDURE = bigbase36_lt,   COMMUTATOR = > ,   NEGATOR = >= ,   RESTRICT = scalarltsel,   JOIN = scalarltjoinsel );  CREATE OPERATOR <= (   LEFTARG = bigbase36,   RIGHTARG = bigbase36,   PROCEDURE = bigbase36_le,   COMMUTATOR = >= ,   NEGATOR = > ,   RESTRICT = scalarltsel,   JOIN = scalarltjoinsel );  CREATE OPERATOR > (   LEFTARG = bigbase36,   RIGHTARG = bigbase36,   PROCEDURE = bigbase36_gt,   COMMUTATOR = < ,   NEGATOR = <= ,   RESTRICT = scalargtsel,   JOIN = scalargtjoinsel );  CREATE OPERATOR >= (   LEFTARG = bigbase36,   RIGHTARG = bigbase36,   PROCEDURE = bigbase36_ge,   COMMUTATOR = <= ,   NEGATOR = < ,   RESTRICT = scalargtsel,   JOIN = scalargtjoinsel );  CREATE OPERATOR CLASS btree_bigbase36_ops DEFAULT FOR TYPE bigbase36 USING btree AS         OPERATOR        1       <  ,         OPERATOR        2       <= ,         OPERATOR        3       =  ,         OPERATOR        4       >= ,         OPERATOR        5       >  ,         FUNCTION        1       bigbase36_cmp(bigbase36, bigbase36);  CREATE OPERATOR CLASS hash_bigbase36_ops DEFAULT FOR TYPE bigbase36 USING hash AS         OPERATOR        1       = ,         FUNCTION        1       hash_bigbase36(bigbase36);  CREATE CAST (bigint as bigbase36) WITHOUT FUNCTION AS ASSIGNMENT; CREATE CAST (bigbase36 as bigint) WITHOUT FUNCTION AS ASSIGNMENT; `
```

As you can see, this is mostly a find and replace for `base36` to `bigbase36` and `int4` to `int8`.

Lets add the C-Part.

## Organizing C-Code

To have the C-Code better organized we’ll put `base36.c` under the `src` dircetory.

```bash
`mkdir src mv base36.c src/ `
```

Now we can add another file for the `bigbase36` input and output function in `src`.

src/bigbase36.c

```c
`PG_FUNCTION_INFO_V1(bigbase36_in); Datum bigbase36_in(PG_FUNCTION_ARGS) {     long result;     char *bad;     char *str = PG_GETARG_CSTRING(0);     result = strtol(str, &bad, 36);     if (bad[0] != '\0' || strlen(str)==0)         ereport(ERROR,             (              errcode(ERRCODE_SYNTAX_ERROR),              errmsg("invalid input syntax for bigbase36: \"%s\"", str)             )         );     if (result < 0)         ereport(ERROR,             (              errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),              errmsg("negative values are not allowed"),              errdetail("value %ld is negative", result),              errhint("make it positive")             )         );     PG_RETURN_INT64((int64)result); }  PG_FUNCTION_INFO_V1(bigbase36_out); Datum bigbase36_out(PG_FUNCTION_ARGS) {     int64 arg = PG_GETARG_INT64(0);     if (arg < 0)         ereport(ERROR,             (              errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),              errmsg("negative values are not allowed"),              errdetail("value %d is negative", arg),              errhint("make it positive")             )         );     char base36[36] = "0123456789abcdefghijklmnopqrstuvwxyz";      /* max 13 char + '\0' */     char buffer[14];     unsigned int offset = sizeof(buffer);     buffer[--offset] = '\0';      do {         buffer[--offset] = base36[arg % 36];     } while (arg /= 36);      PG_RETURN_CSTRING(pstrdup(&buffer[offset])); } `
```

It’s more or less the same code as for base36. In `bigbase36_in`, we don’t need the overflow safe typecast to `int32` anymore and can return the result directly with `PG_RETURN_INT64(result);`. For `bigbase36_out`, we expand the buffer to 14 characters as the result could be that long.

To be able to compile the two files into one shared-library object we need to adapt the Makefile as well.

Makefile

```makefile
`# the extensions name EXTENSION     = base36 DATA          = $(wildcard *--*.sql)            # script files to install TESTS         = $(wildcard test/sql/*.sql)      # use test/sql/*.sql as testfiles  # find the sql and expected directories under test # load plpgsql into test db # load base36 extension into test db # dbname REGRESS_OPTS  = --inputdir=test         \                 --load-extension=base36 \                 --load-language=plpgsql REGRESS       = $(patsubst test/sql/%.sql,%,$(TESTS)) OBJS          = $(patsubst %.c,%.o,$(wildcard src/*.c)) # object files # final shared library to be build from multiple source files (OBJS) MODULE_big    = $(EXTENSION)   # postgres build stuff PG_CONFIG = pg_config PGXS := $(shell $(PG_CONFIG) --pgxs) include $(PGXS) `
```

Here (Line 13) we define that all src/*.c files will become object files that should be build int one shared library from these multiple objects (Line 15).

Thus, we have again generalized the `Makefile` for future use.

If we now build and test the extension then all should be fine.

However, we should also add tests for the `bigbase36` type.

sql/bigbase36_io.sql

```sql
`-- simple input SELECT '120'::bigbase36; SELECT '3c'::bigbase36; -- case insensitivity SELECT '3C'::bigbase36; SELECT 'FoO'::bigbase36; -- invalid characters SELECT 'foo bar'::bigbase36; SELECT 'abc$%2'::bigbase36; -- negative values SELECT '-10'::bigbase36; -- to big values SELECT 'abcdefghijklmn'::bigbase36;  -- storage BEGIN; CREATE TABLE base36_test(val bigbase36); INSERT INTO base36_test VALUES ('123'), ('3c'), ('5A'), ('zZz'); SELECT * FROM base36_test; UPDATE base36_test SET val = '567a' where val = '123'; SELECT * FROM base36_test; UPDATE base36_test SET val = '-aa' where val = '3c'; SELECT * FROM base36_test; ROLLBACK; `
```

If we take a look at `results/bigbase36_io.out` we see again some odd behavior for too-big values.

```bash
`-- to big values SELECT 'abcdefghijklmn'::bigbase36; ERROR:  negative values is not allowed LINE 1: SELECT 'abcdefghijklmn'::bigbase36;                ^ DETAIL:  value -1 is negative HINT:  make it positive``` `
```

You’ll notice `strtol()` returns `LONG_MAX` if the result overflows. If you take a look how converting text to numbers is done in the [postgres source code](https://github.com/postgres/postgres/blob/master/src/backend/utils/adt/numutils.c#L37), you can see that there are lots of platform-specific edge and corner cases. For simplicity, let’s assume that we are on a 64 bit environment having 64 bit long results. On 32 bit machines our test suite and thus `make installcheck` would fail, telling our users that the Extension would not work as expected.

src/bigbase36.c

```c
`#include "postgres.h" #include "fmgr.h" #include "utils/builtins.h" #include <limits.h>  PG_FUNCTION_INFO_V1(bigbase36_in); Datum bigbase36_in(PG_FUNCTION_ARGS) {     long result;     char *bad;     char *str = PG_GETARG_CSTRING(0);     result = strtol(str, &bad, 36);     if (result == LONG_MIN || result == LONG_MAX)         ereport(ERROR,             (              errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),              errmsg("base36 out of range")             )         );      if (bad[0] != '\0' || strlen(str)==0)         ereport(ERROR,             (              errcode(ERRCODE_SYNTAX_ERROR),              errmsg("invalid input syntax for bigbase36: \"%s\"", str)             )         );     if (result < 0)         ereport(ERROR,             (              errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),              errmsg("negative values are not allowed"),              errdetail("value %ld is negative", result),              errhint("make it positive")             )         );     PG_RETURN_INT64((int64)result); }  /* bigbase36_out omitted */ `
```

Here, by including `<limits.h>` we can check if the result overflowed. The same can be applied for `base36_in` checking `result < INT_MIN || result > INT_MAX` and thus getting ride of the `DirectFunctionCall1(int84,result)`. The only caveat here is that we can’t cast `LONG_MAX` and `LONG_MIN` to base36.

Now that we’ve created a bunch of code duplication, let’s improve the readability with a common header file and define the errors in macros.

src/base36.h

```c
`#ifndef BASE36_H #define BASE36_H  #include "postgres.h" #include "utils/builtins.h" #include "utils/int8.h" #include "libpq/pqformat.h" #include <limits.h>  extern const char base36_digits[36];  #define BASE36OUTOFRANGE_ERROR(_str, _typ)                      \   do {                                                          \     ereport(ERROR,                                              \       (errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),             \         errmsg("value \"%s\" is out of range for type %s",      \           _str, _typ)));                                        \   } while(0)                                                    \  #define BASE36SYNTAX_ERROR(_str, _typ)                          \   do {                                                          \     ereport(ERROR,                                              \       (errcode(ERRCODE_SYNTAX_ERROR),                           \       errmsg("invalid input syntax for %s: \"%s\"",             \              _typ, _str)));                                     \   } while(0)                                                    \   #endif // BASE36_H `
```

Also, there is no good reason why we should disallow negative values.

## Migrations

Finally our new Version is ready to be released! Let’s add an update test.

test/sql/update.sql

```sql
`BEGIN; DROP EXTENSION base36; CREATE EXTENSION base36 VERSION '0.0.1'; ALTER EXTENSION base36 UPDATE TO '0.0.2'; SELECT 'abcdefg'::bigbase36; `
```

After we run:

```
`make clean && make && make install && make installcheck `
```

We see:

results/update.out

```sql
`BEGIN; DROP EXTENSION base36; CREATE EXTENSION base36 VERSION '0.0.1'; ALTER EXTENSION base36 UPDATE TO '0.0.2'; ERROR:  extension "base36" has no update path from version "0.0.1" to version "0.0.2" SELECT 'abcdefg'::bigbase36; ERROR:  current transaction is aborted, commands ignored until end of transaction block `
```

Although Version 0.0.2 exists we can’t run the Update command. To make that work we’d need an updated script in the form `extension--oldversion--newversion.sql` that includes all commands needed to upgrade from one version to the other.

So we need to copy all base36 realted sql into `base36--0.0.1--0.0.2.sql`

base36–0.0.1–0.0.2.sql

```sql
`-- complain if script is sourced in psql, rather than via CREATE EXTENSION \echo Use "CREATE EXTENSION base36" to load this file. \quit  CREATE FUNCTION bigbase36_in(cstring) RETURNS bigbase36 AS '$libdir/base36' LANGUAGE C IMMUTABLE STRICT;  CREATE FUNCTION bigbase36_out(bigbase36) RETURNS cstring AS '$libdir/base36' LANGUAGE C IMMUTABLE STRICT;  CREATE TYPE bigbase36 (   INPUT          = bigbase36_in,   OUTPUT         = bigbase36_out,   LIKE           = bigint );  ---... rest omitted `
```

## MODULE_PATHNAME

For each SQL function that uses a C-Function defined `AS '$libdir/base36'`, we are telling Postgres which shared library to use. If we renamed the shared library we’d need to rewrite all the SQL functions. We can do better:

base36.control

```bash
`# base36 extension comment = 'base36 datatype' default_version = '0.0.2' relocatable = true module_pathname = '$libdir/base36' `
```

Here we define the `module_pathname` to point to `'$libdir/base36'` and thus we can define our SQL Functions like this

```sql
`CREATE FUNCTION base36_in(cstring) RETURNS base36 AS 'MODULE_PATHNAME' LANGUAGE C IMMUTABLE STRICT; `
```

## Summary

In the last five articles you saw that you can define your own datatypes and completely specify the behavior you want. However, with great power comes great responsibility. Not only can you confuse users with unexpected results, you can also completely break the server and loose data. Luckily you learned how to debug things and how to write proper tests.

Before you start implementing things, you should first take a look on how Postgres does it and try to reuse as much functionality as you can. So not only do you avoid reinventing the wheel, but you also have trusted code from the well-tested PostgreSQL code base. When you’re done, make sure to always think about the edge cases, write down everything into tests to prevent breaking things, and to try out higher workloads and complex statements to avoid finding bugs in production later.

As testing is so important, we at adjust wrote our own testing tool called `pg_spec`. We’ll cover this in out next post.