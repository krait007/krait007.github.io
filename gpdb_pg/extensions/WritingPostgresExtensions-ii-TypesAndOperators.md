# Writing Postgres Extensions - Types and Operators

In the [last post about Writing Postgres Extensions](http://big-elephants.com/2015-10/writing-postgres-extensions-part-i/), we covered the basics of extending PostgresSQL with extension. Now it’s time for the fun part – developing our own type.

## A small disclaimer

It’s in your best interest to resist the urge to copy and paste the code found within this article. There are some serious bugs along the lines, which were intentionally left in for illustrative purposes. If you’re looking for a production-ready `base36` type definition, then take a look at [here](https://github.com/adjust/pg-base36).

## A refresher on base36

What we’re after is the solid implementation of a `base36` data type to use for storing and retrieving [base36 numbers](https://en.wikipedia.org/wiki/Base36). We already created the basic skeleton for our extension, including `base36.control`and `Makefile`, which you can find in the GitHub repo dedicated to this series of blog posts. You can check out what we ended up with in [Part 1](https://github.com/adjust/postgresql_extension_demo/tree/part_i) and the code from this post can be found on the [part_ii branch](https://github.com/adjust/postgresql_extension_demo/tree/part_ii).

base36.control

```
`# base36 extension comment = 'base36 datatype' default_version = '0.0.1' relocatable = true `
```

Makefile

```makefile
`EXTENSION = base36              # the extension name DATA      = base36--0.0.1.sql   # script files to install REGRESS   = base36_test         # our test script file (without extension) MODULES   = base36              # our c module file to build  # Postgres build stuff PG_CONFIG = pg_config PGXS := $(shell $(PG_CONFIG) --pgxs) include $(PGXS) `
```

## Custom data type in Postgres

Let’s rewrite the SQL script file to show our own data type:

base36–0.0.1.sql

```sql
`-- complain if script is sourced in psql, rather than via CREATE EXTENSION \echo Use "CREATE EXTENSION base36" to load this file. \quit  CREATE FUNCTION base36_in(cstring) RETURNS base36 AS '$libdir/base36' LANGUAGE C IMMUTABLE STRICT;  CREATE FUNCTION base36_out(base36) RETURNS cstring AS '$libdir/base36' LANGUAGE C IMMUTABLE STRICT;  CREATE TYPE base36 (   INPUT          = base36_in,   OUTPUT         = base36_out,   LIKE           = integer ); `
```

This is the minimum required to create a [base type](http://www.postgresql.org/docs/9.4/static/sql-createtype.html) in Postgres: We need the two functions `input` and `output` that tell Postgres how to convert the input text to the internal representation (`base36_in`) and back from the internal representation to text (`base36_out`). We also need to tell Postgres to treat our type like `integer`. This can also be achieved by specifying these additional parameters in the type definition as in the example below.

```bash
`INTERNALLENGTH = 4,     -- use 4 bytes to store data ALIGNMENT      = int4,  -- align to 4 bytes STORAGE        = PLAIN, -- always store data inline uncompressed (not toasted) PASSEDBYVALUE           -- pass data by value rather than by reference `
```

Now let’s do the C-Part:

**base36.c**

```c
`#include "postgres.h" #include "fmgr.h" #include "utils/builtins.h"  PG_MODULE_MAGIC;  PG_FUNCTION_INFO_V1(base36_in); Datum base36_in(PG_FUNCTION_ARGS) {     long result;     char *str = PG_GETARG_CSTRING(0);     result = strtol(str, NULL, 36);     PG_RETURN_INT32((int32)result); }  PG_FUNCTION_INFO_V1(base36_out); Datum base36_out(PG_FUNCTION_ARGS) {     int32 arg = PG_GETARG_INT32(0);     if (arg < 0)         ereport(ERROR,             (              errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),              errmsg("negative values are not allowed"),              errdetail("value %d is negative", arg),              errhint("make it positive")             )         );     char base36[36] = "0123456789abcdefghijklmnopqrstuvwxyz";      /* max 6 char + '\0' */     char *buffer        = palloc(7 * sizeof(char));     unsigned int offset = 7 * sizeof(char);     buffer[--offset]    = '\0';      do {         buffer[--offset] = base36[arg % 36];     } while (arg /= 36);      PG_RETURN_CSTRING(&buffer[offset]); } `
```

We basically just reused our `base36_encode` function to be our `OUTPUT` and added an `INPUT` decoding function - easy.

Now we can store and retrieve `base36` numbers in our database. Let’s build and test it.

```bash
`make clean && make && make install `
```



```sql
`test=# CREATE TABLE base36_test(val base36); CREATE TABLE test=# INSERT INTO base36_test VALUES ('123'), ('3c'), ('5A'), ('zZz'); INSERT 0 4 test=# SELECT * FROM base36_test;  val -----  123  3c  5a  zzz (4 rows) `
```



Works so far. Let’s order the output.

```sql
`test=# SELECT * FROM base36_test ORDER BY val; ERROR:  could not identify an ordering operator for type base36 LINE 1: SELECT * FROM base36_test ORDER BY val;                                            ^ HINT:  Use an explicit ordering operator or modify the query. `
```

Hmmm… looks like we missed something.

## Operators

Keep in mind that we’re dealing with a completely bare data type. In order to do any sorting, we need to define what it means for an instance of the data type to be less than another instance, for it to be greater than another instance or for two instances to be equal.

This shouldn’t be too strange – in fact, it resembles how you would include the `Enumerable` mixin in a Ruby class or implement the `sort.Interface` in a Golang type to introduce the ordering rules for your objects.

Let’s add the comparison functions and operators to our SQL script.

base36–0.0.1.sql

```sql
`-- type definition omitted  CREATE FUNCTION base36_eq(base36, base36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int4eq';  CREATE FUNCTION base36_ne(base36, base36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int4ne';  CREATE FUNCTION base36_lt(base36, base36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int4lt';  CREATE FUNCTION base36_le(base36, base36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int4le';  CREATE FUNCTION base36_gt(base36, base36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int4gt';  CREATE FUNCTION base36_ge(base36, base36) RETURNS boolean LANGUAGE internal IMMUTABLE AS 'int4ge';  CREATE FUNCTION base36_cmp(base36, base36) RETURNS integer LANGUAGE internal IMMUTABLE AS 'btint4cmp';  CREATE FUNCTION hash_base36(base36) RETURNS integer LANGUAGE internal IMMUTABLE AS 'hashint4';  CREATE OPERATOR = (   LEFTARG = base36,   RIGHTARG = base36,   PROCEDURE = base36_eq,   COMMUTATOR = '=',   NEGATOR = '<>',   RESTRICT = eqsel,   JOIN = eqjoinsel,   HASHES, MERGES );  CREATE OPERATOR <> (   LEFTARG = base36,   RIGHTARG = base36,   PROCEDURE = base36_ne,   COMMUTATOR = '<>',   NEGATOR = '=',   RESTRICT = neqsel,   JOIN = neqjoinsel );  CREATE OPERATOR < (   LEFTARG = base36,   RIGHTARG = base36,   PROCEDURE = base36_lt,   COMMUTATOR = > ,   NEGATOR = >= ,   RESTRICT = scalarltsel,   JOIN = scalarltjoinsel );  CREATE OPERATOR <= (   LEFTARG = base36,   RIGHTARG = base36,   PROCEDURE = base36_le,   COMMUTATOR = >= ,   NEGATOR = > ,   RESTRICT = scalarltsel,   JOIN = scalarltjoinsel );  CREATE OPERATOR > (   LEFTARG = base36,   RIGHTARG = base36,   PROCEDURE = base36_gt,   COMMUTATOR = < ,   NEGATOR = <= ,   RESTRICT = scalargtsel,   JOIN = scalargtjoinsel );  CREATE OPERATOR >= (   LEFTARG = base36,   RIGHTARG = base36,   PROCEDURE = base36_ge,   COMMUTATOR = <= ,   NEGATOR = < ,   RESTRICT = scalargtsel,   JOIN = scalargtjoinsel );  CREATE OPERATOR CLASS btree_base36_ops DEFAULT FOR TYPE base36 USING btree AS         OPERATOR        1       <  ,         OPERATOR        2       <= ,         OPERATOR        3       =  ,         OPERATOR        4       >= ,         OPERATOR        5       >  ,         FUNCTION        1       base36_cmp(base36, base36);  CREATE OPERATOR CLASS hash_base36_ops     DEFAULT FOR TYPE base36 USING hash AS         OPERATOR        1       = ,         FUNCTION        1       hash_base36(base36); `
```

Wow…that’s a lot. To break it down: First, we defined a comparison function to power each comparison operator (`<`, `<=`, `=`, `>=` and `>`). We then put them together in an operator class that will enable us to create indexes on our new data type.

For the functions themselves we could simply reuse the corresponding, built-in functions for the integer type: `int4eq`, `int4ne`, `int4lt`, `int4le`, `int4gt`, `int4ge`, `btint4cmp` and `hashint4`.

Now let’s take a look at the operator definitions.

Each operator has a left argument (`LEFTARG`), a right argument (`RIGHTARG`), and a function (`PROCEDURE`).

So, if we write:



```sql
`SELECT 'larg'::base36 < 'rarg'::base36;  ?column? ----------  t (1 row) `
```

Postgres will use the `base36_lt` function and do a `base36_lt('larg','rarg')`.

### COMMUTATOR and NEGATOR

Each operator also has a `COMMUTATOR` and a `NEGATOR` (see Line 52-53). These are used by the query planer to do optimizations. A commutator is the operator that should be used to denote the same result, but with the arguments flipped. Thus, since `(x < y)` equals `(y > x)` for all possible values `x`and `y`, the operator `>` is the commutator of the operator `<`. For the same reason `<` is the commutator of `>`.

The negator is the operator that would negate the boolean result of an operator. That is, `(x < y)`equals `NOT(x >= y)` for all possible values `x` and `y`.

So why is that important? Suppose you’ve indexed the column `val`:

```sql
`EXPLAIN SELECT * FROM base36_test where 'c1'::base36 > val;                                            QUERY PLAN -------------------------------------------------------------------------------------------------  Index Only Scan using base36_test_val_idx on base36_test  (cost=0.42..169.93 rows=5000 width=4)    Index Cond: (val < 'c1'::base36) (2 rows) `
```



As you can see, Postgres has to rewrite the query from `'c1'::base36 > val` to `val < 'c1'::base36` in order to be able to use the index.

The same is true for the negator.

```sql
`base36_test=# explain SELECT * FROM base36_test where NOT val > 'c1';                                            QUERY PLAN -------------------------------------------------------------------------------------------------  Index Only Scan using base36_test_val_idx on base36_test  (cost=0.42..169.93 rows=5000 width=4)    Index Cond: (val <= 'c1'::base36) (2 rows) `
```

Here `NOT val > 'c1'::base36` is rewritten to `val <= 'c1'::base36`.



And finally you can see that it would rewrite `NOT 'c1'::base36 < val` to `val <= 'c1'::base36`.

```sql
`base36_test=# explain SELECT * FROM base36_test where NOT 'c1' < val;                                            QUERY PLAN -------------------------------------------------------------------------------------------------  Index Only Scan using base36_test_val_idx on base36_test  (cost=0.42..169.93 rows=5000 width=4)    Index Cond: (val <= 'c1'::base36) (2 rows) `
```

So while `COMMUTATOR` and `NEGATOR` clauses are not strictly required in a custom Postgres type definition, without them the above rewrites won’t be possible. Therefore, the respective queries won’t use the index and in most situations lose performance.

### RESTRICT and JOIN

Luckily, we don’t need to write our own `RESTRICT` function (see Line 54-55) and can use simply use this:

```
`eqsel for = neqsel for <> scalarltsel for < or <= scalargtsel for > or >= `
```

These are restriction selectivity estimation functions which give Postgres a hint on how many rows will satisfy a WHERE-clause given a constant as the right argument. If the constant is the left argument, we can flip it to the right using the commutator.

You may already know that Postgres collects some statistics of each table when you or the autovacuum daemon run an `ANALYZE`. You can also take a look at these statistics on the [pg_stats view](http://www.postgresql.org/docs/9.4/static/view-pg-stats.html).

```sql
`SELECT * FROM pg_stats WHERE tablename = 'base36_test'; `
```

All the estimation function does is to give a value between 0 and 1, indicating the estimated fraction of rows based on these statistics. This is quite important to know as typically the `=` operator satisfies fewer rows than the `<>` operator. Since you are relatively free in naming and defining your operators, you need to tell how they work.

If you really want to know what the estimation functions look like, take a look at the source code in [src/backend/utils/adt/selfuncs.c](https://github.com/postgres/postgres/blob/39df0f150ca69fac1c89537065ddc97af18921b8/src/backend/utils/adt/selfuncs.c). Disclaimer: your eyes might start bleeding.

So, it’s pretty great that we don’t need to write our own `JOIN` selectivity estimation function. This one is for queries where an operator is used to join tables in the form `table1.column1 OP table2.column2`, but it has essentially the same idea: it estimates how many rows will be returned by the operation to finally decide which of the possible plans (i.e. which join order) to use.

So if you have something like:

```sql
`SELECT * FROM table1 JOIN table2 ON table1.c1 = table2.c1 JOIN table3 ON table2.c1 = table2.c1 `
```

Here table3 has only a few rows, while table1 and table2 are really big. So it makes sense to first join table3, amass a few rows and then join the other tables.

### HASHES and MERGES

For the equality operator, we also define the parameters `HASHES` and `MERGES` (Line 35). When we do this, we’re telling Postgres that it’s suitable to use this function for hash to respectively merge join operations. To make the hash join actually work, we also need to define a hash function and put both together in an operator class. You can read further in the [PostgreSQL Documentation](http://www.postgresql.org/docs/9.4/static/xoper-optimization.html) about the different Operator Optimization clauses.

## More to come…

So far you’ve seen how to implement a basic data type using `INPUT` and `OUTPUT` functions. On top of this we added comparison operators by reusing Postgres internals. This allows us to order tables and use indexes.

However, if you followed the implementation on your computer step-by-step, you might find that the above mentioned `EXPLAIN` command doesn’t really work.

```sql
`# EXPLAIN SELECT * FROM base36_test where 'c1'::base36 > val; server closed the connection unexpectedly   This probably means the server terminated abnormally   before or while processing the request. The connection to the server was lost. Attempting reset: Failed. Time: 275,327 ms !> `
```

That’s because right we just did the worst possible thing: in some situations, our code makes the whole server crash.

In the [next post](http://big-elephants.com/2015-10/writing-postgres-extensions-part-iii/) we’ll see how we can debug our code using LLDB, and how to avoid these errors with the proper testing.