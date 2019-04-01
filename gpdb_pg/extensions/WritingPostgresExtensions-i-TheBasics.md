# Writing Postgres Extensions - the Basics

Postgres has a ton of features and offers a wide range of data types, functions, operators, and aggregates. But sometimes it’s just not enough for your use case. Luckily, it’s easy to extend Postgres’ functionality through extension. So why not write your own?

This is the first in a series of articles about extending Postgres through extensions. You can follow the code examples [here on branch part_i](https://github.com/adjust/postgresql_extension_demo/tree/part_i)

## base36

You might already know the trick used by url shorteners. Use some unique random characters such as http://goo.gl/EAZSKW to point to something else. You have to remember what points to where, of course, so you need to store it in a database. But instead of saving 6 characters using `varchar(6)` (and thus wasting 7 bytes) why not use an `integer` with 4 bytes and represent it as [base36](https://en.wikipedia.org/wiki/Base36)?

## The Extension Skeleton

To be able to run the [CREATE EXTENSION](http://www.postgresql.org/docs/9.4/static/sql-createextension.html) command in your database, your extension needs at least two files: a control file in the format `extension_name.control`, which tells Postgres some basics about your extension, and a extension’s SQL script file in the format `extension--version.sql`. So let’s add them into our project directory.

A good starting point for our control file might be:

base36.control

```bash
`# base36 extension comment = 'base36 datatype' default_version = '0.0.1' relocatable = true `
```

As of now, our extension has no functionality. Let’s add some in an SQL script file:

**base36–0.0.1.sql**

```sql
`-- complain if script is sourced in psql, rather than via CREATE EXTENSION \echo Use "CREATE EXTENSION base36" to load this file. \quit CREATE FUNCTION base36_encode(digits int) RETURNS text LANGUAGE plpgsql IMMUTABLE STRICT   AS $$     DECLARE       chars char[];       ret varchar;       val int;     BEGIN       chars := ARRAY[                 '0','1','2','3','4','5','6','7','8','9','a','b','c','d','e','f','g','h',                 'i','j','k','l','m','n','o','p','q','r','s','t', 'u','v','w','x','y','z'               ];        val := digits;       ret := '';      WHILE val != 0 LOOP       ret := chars[(val % 36)+1] || ret;       val := val / 36;     END LOOP;      RETURN(ret);     END;   $$; `
```

The second line ensures that the file won’t be loaded into the database directly, but only via `CREATE EXTENSION`.

The simple plpgsql function allows us to encode any integer into its base36 representation. If we copied these two files into postgres `SHAREDIR/extension` directory, then we could start using the extension with `CREATE EXTENSION`. But we won’t bother users with figuring out where to put these files and how to copy them manually – that’s what Makefiles are made for. So, let’s add one to our project.

## Makefile

Every PostgreSQL installation from 9.1 onwards provides a build infrastructure for extensions called PGXS, allowing extensions to be easily built against an already-installed server. Most of the environment variables needed to build an extension are setup in `pg_config` and can simply be reused.

For our example this Makefile fits our needs.

**Makefile**

```makefile
`EXTENSION = base36        # the extensions name DATA = base36--0.0.1.sql  # script files to install  # postgres build stuff PG_CONFIG = pg_config PGXS := $(shell $(PG_CONFIG) --pgxs) include $(PGXS) `
```

Now we can start using the extension. Run

```bash
`make install `
```

from your project directory and

```sql
`test=# CREATE EXTENSION base36; CREATE EXTENSION Time: 3,329 ms test=# SELECT base36_encode(123456789);  base36_encode ---------------  21i3v9 (1 row)  Time: 0,558 ms `
```

in your database. Awesome!

## Write tests

These days, every serious developer writes tests. And as database developer who deals with data (probably the most valuable thing in your company) you should as well.

You can easily add some regression tests to your project that can be invoked by `make installcheck`after doing `make install`. For this to work you can put test script files in a subdirectory named `sql/`. For each test file there should also be a file containing the expected output in a subdirectory named `expected/` with the same name and the extension `.out`. The `make installcheck` command executes each test script with psql, and compares the resulting output to the matching expected file. Any differences will be written to the file regression.diffs. Let’s do so:

sql/base36_test.sql

```sql
`CREATE EXTENSION base36; SELECT base36_encode(0); SELECT base36_encode(1); SELECT base36_encode(10); SELECT base36_encode(35); SELECT base36_encode(36); SELECT base36_encode(123456789); `
```

We also need to tell our `Makefile` about the tests (Line 3):

Makefile

```makefile
`EXTENSION = base36        # the extensions name DATA = base36--0.0.1.sql  # script files to install REGRESS = base36_test     # our test script file (without extension)  # postgres build stuff PG_CONFIG = pg_config PGXS := $(shell $(PG_CONFIG) --pgxs) include $(PGXS) `
```

If we now run `make install && make installcheck`, then our tests would fail. This is because we didn’t specify the expected output. However, we’d find the new directory `results`, which would contain `base36_test.out` and `base36_test.out.diff`. The former contains the actual output from our test script file. Let’s move it into the desired directory.

```
`mkdir expected mv results/base36_test.out expected `
```

If we now rerun our test, we’d see something like:

```bash
`============== running regression test queries        ============== test base36_test              ... ok  =====================  All 1 tests passed. ===================== `
```

Nice! But hey, we cheated a little bit. If we take a look at our expectations, we’d notice that this isn’t what we should expect.



```
`cat expected/base36_test.out CREATE EXTENSION base36; SELECT base36_encode(0);  base36_encode ---------------  (1 row)  SELECT base36_encode(1);  base36_encode ---------------  1 (1 row)  SELECT base36_encode(10);  base36_encode ---------------  a (1 row)  SELECT base36_encode(35);  base36_encode ---------------  z (1 row)  SELECT base36_encode(36);  base36_encode ---------------  10 (1 row)  SELECT base36_encode(123456789);  base36_encode ---------------  21i3v9 (1 row) `
```

You’ll notice that in line 6, `base36_encode(0)` returns an empty string where we’d expect `0`. If we fix our expectation, our test would fail again.

```
`============== running regression test queries        ============== test base36_test              ... FAILED  ======================  1 of 1 tests failed. ======================  The differences that caused some tests to fail can be viewed in the file "regression.diffs".  A copy of the test summary that you see above is saved in the file "regression.out".  make: *** [installcheck] Error 1 `
```



And we can easily inspect the failing test by looking at the mentioned `regression.diffs`

```sql
`*** 2,8 ****   SELECT base36_encode(0);    base36_encode   --------------- !  0   (1 row)    SELECT base36_encode(1); --- 2,8 ----   SELECT base36_encode(0);    base36_encode   --------------- !   (1 row)    SELECT base36_encode(1); `
```

You can read it as “expected 0 got “.

Now let’s implement the fix in the encoding function to make the tests pass again (Line 12-14):

base36–0.0.1.sql

```sql
`-- complain if script is sourced in psql, rather than via CREATE EXTENSION \echo Use "CREATE EXTENSION base36" to load this file. \quit CREATE FUNCTION base36_encode(digits int) RETURNS character varying LANGUAGE plpgsql IMMUTABLE STRICT   AS $$     DECLARE       chars char[];       ret varchar;       val int;     BEGIN       IF digits = 0         THEN RETURN('0');       END IF;       chars := ARRAY[                 '0','1','2','3','4','5','6','7','8','9','a','b','c','d','e','f','g','h',                 'i','j','k','l','m','n','o','p','q','r','s','t', 'u','v','w','x','y','z'               ];        val := digits;       ret := '';      WHILE val != 0 LOOP       ret := chars[(val % 36)+1] || ret;       val := val / 36;     END LOOP;      RETURN(ret);     END;   $$; `
```

## Optimize for speed, write some C

While shipping related functionality in an extension is a convenient way to share code, the real fun starts when you implement stuff in C. Let’s get the first 1M base36 numbers.

```bash
`test=# SELECT i, base36_encode(i) FROM generate_series(1,1e6::int) i; Time: 11289,610 ms `
```

11s? That’s …well, not so fast.

Let’s see if we can do better in C. Writing [C-Language Functions](http://www.postgresql.org/docs/9.4/static/xfunc-c.html) isn’t that hard.

base36.c

```c
`#include "postgres.h" #include "fmgr.h" #include "utils/builtins.h"  PG_MODULE_MAGIC;  PG_FUNCTION_INFO_V1(base36_encode); Datum base36_encode(PG_FUNCTION_ARGS) {     int32 arg = PG_GETARG_INT32(0);     char base36[36] = "0123456789abcdefghijklmnopqrstuvwxyz";      /* max 6 char + '\0' */     char *buffer = palloc(7 * sizeof(char));     unsigned int offset = sizeof(buffer);     buffer[--offset] = '\0';      do {         buffer[--offset] = base36[arg % 36];     } while (arg /= 36);       PG_RETURN_TEXT_P(cstring_to_text(&buffer[offset])); } `
```

You might have noticed that the actual algorithm is the one [Wikipedia provides](https://en.wikipedia.org/wiki/Base36). Let’s see what we added to make it work with Postgres.

`#include "postgres.h"` includes most of the basic stuff needed for interfacing with Postgres. This line needs to be included in every C-File that declares Postgres functions.

`#include "fmgr.h"` needs to be included to make use of `PG_GETARG_XXX` and `PG_RETURN_XXX` macros.

`#include "utils/builtins.h"` defines some operations on Postgres’ built-in datatypes (`cstring_to_text used later`)

`PG_MODULE_MAGIC` is the “magic block” needed as of PostgreSQL 8.2 in one (and only one) of the module source files after including the header `fmgr.h`.

`PG_FUNCTION_INFO_V1(base36_encode);` introduces the function to Postges as [Version 1 Calling Convention](http://www.postgresql.org/docs/9.4/static/xfunc-c.html#AEN55804), and is only needed if you want the function to interface with Postgres.

`Datum` is the return type of every C-language Postgres function and can be any data type. You can think of it as something similar to a `void *`.

`base36_encode(PG_FUNCTION_ARGS)` our function is named `base36_encode` `PG_FUNCTION_ARGS` and can take any number and any type of arguments.

`int32 arg = PG_GETARG_INT32(0);` get the first argument. The arguments are numbered starting from `0`. You must use the `PG_GETARG_XXX` macros defined in [fmgr.h](https://github.com/postgres/postgres/blob/b7ca57ac0e80b8b511780ef1f19fa2124c901efb/src/include/fmgr.h#L224) to get the actual argument value.

`char *buffer = palloc(7 * sizeof(char));` to prevent memory leaks when allocating memory, always use the PostgreSQL functions palloc and pfree instead of the corresponding C library functions malloc and free. Memory allocated by palloc will be freed automatically at the end of each transaction. You can also use `palloc0` to ensure the bytes are zeroed.

`PG_RETURN_TEXT_P(cstring_to_text(&buffer[offset]));` to return a value to Postgres you always have to use one of the PG_RETURN_XXX macros. `cstring_to_text` converts the cstring to Postgres text type before.

Once we’re finished with the C-part, we need to modify our SQL function.

base36–0.0.1.sql

```sql
`-- complain if script is sourced in psql, rather than via CREATE EXTENSION \echo Use "CREATE EXTENSION base36" to load this file. \quit CREATE FUNCTION base36_encode(integer) RETURNS text AS '$libdir/base36' LANGUAGE C IMMUTABLE STRICT; `
```

To be able to use the function we also need to modify the `Makefile` (Line 4)

Makefile

```makefile
`EXTENSION = base36        # the extensions name DATA = base36--0.0.1.sql  # script files to install REGRESS = base36_test     # our test script file (without extension) MODULES = base36          # our c module file to build  # postgres build stuff PG_CONFIG = pg_config PGXS := $(shell $(PG_CONFIG) --pgxs) include $(PGXS) `
```

Luckily, we already have test and can try it out with `make install && make installcheck`. Opening a database console also proves it to be a lot (30 times) faster:

```sql
`test=# SELECT i, base36_encode(i) FROM generate_series(1,1e6::int) i; Time: 361,054 ms `
```

## Returning errors

You might have noticed that our simple implementation would not work with negative numbers. Just as it did before with `0`, it would return an empty string. We might want to add a `-` sign for negative values or simply error out. Let’s go for the latter. (Line 12-20)

base36.c

```c
`#include "postgres.h" #include "fmgr.h" #include "utils/builtins.h"  PG_MODULE_MAGIC;  PG_FUNCTION_INFO_V1(base36_encode); Datum base36_encode(PG_FUNCTION_ARGS) {     int32 arg = PG_GETARG_INT32(0);     if (arg < 0)         ereport(ERROR,             (              errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),              errmsg("negative values are not allowed"),              errdetail("value %d is negative", arg),              errhint("make it positive")             )         );     char base36[36] = "0123456789abcdefghijklmnopqrstuvwxyz";      /* max 6 char + '\0' */     char *buffer = palloc(7 * sizeof(char));     unsigned int offset = sizeof(buffer);     buffer[--offset] = '\0';      do {         buffer[--offset] = base36[arg % 36];     } while (arg /= 36);       PG_RETURN_TEXT_P(cstring_to_text(&buffer[offset])); } `
```

Which would result in

```sql
`test=# SELECT base36_encode(-10); ERROR:  negative values are not allowed DETAIL:  value -10 is negative HINT:  make it positive `
```

Postgres has some nice [error reporting build in](http://www.postgresql.org/docs/9.4/static/error-message-reporting.html). While for this use case a simple errmsg might have been enough, you can (but don’t need to) add details, hints and more.

For simple debugging, it’s also convenient to use a

```
`elog(INFO, "value here is %d", value); `
```

The `INFO` level error would only result into a log message and not immediately stop the function call. Severity levels range from `DEBUG` to `PANIC`.

## More to come…

Now that we know the basics for writing extensions and C-Language functions, in [the next post](http://big-elephants.com/2015-10/writing-postgres-extensions-part-ii/) we’ll take the next step and implement a complete new datatype.