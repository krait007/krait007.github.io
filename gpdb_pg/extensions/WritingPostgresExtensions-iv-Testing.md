# Writing Postgres Extensions - Testing

In [Part III about Writing Postgres Extensions](http://big-elephants.com/2015-11/writing-postgres-extensions-part-iv/2015-10/writing-postgres-extensions-part-iii/) we fixed a serious bug using LLDB debugger and completed the `base36` type by using type casts. Now it’s time to recover what we’ve actually achieved – and to do some more testing.

You can review the current code base on [on github branch part_iii](https://github.com/adjust/postgresql_extension_demo/tree/part_iii).

## Full-Power Testsuite

Simply trying out some stuff in the Postgres-console and assuming that everything will work just fine is a bad idea, especially since we introduced some serious bugs while developing our extension. Because of this, we learned how important it is to have a fully covered test suite that tests not only the “happy path,” but also the edge and error cases.

We already did a good job on testing in the [first post](http://big-elephants.com/2015-11/writing-postgres-extensions-part-iv/2015-10/writing-postgres-extensions-part-i/), where we used the built-in regression testing for extensions. So let’s write down our findings in some test script.

sql/base36_test.sql

```sql
`CREATE EXTENSION base36; SELECT '120'::base36; SELECT '3c'::base36; CREATE TABLE base36_test(val base36); INSERT INTO base36_test VALUES ('123'), ('3c'), ('5A'), ('zZz'); SELECT * FROM base36_test; SELECT '120'::base36 > '3c'::base36; SELECT * FROM base36_test ORDER BY val; EXPLAIN (COSTS OFF) SELECT * FROM base36_test where NOT val < 'c1'; SELECT 'abcdefghi'::base36; `
```

Note that I added `(COSTS OFF)` to the `EXPLAIN` command to make sure the test won’t fail on different machines with different [cost parameters](http://www.postgresql.org/docs/9.3/static/runtime-config-query.html#RUNTIME-CONFIG-QUERY-CONSTANTS).

If we now run:

```bash
`make clean && make && make install && make installcheck `
```

we get our output in `results/base36_test.out` and can copy it over to `sql/expected/`. But wait – let’s read it carefully first to make sure this all is as expected.

```bash
`SELECT 'abcdefghi'::base36;  base36 --------  r0bprq (1 row) `
```

Well, it’s obviously not. The `base36_in` seems to also have a serious bug when we put too long strings into it. Let’s look into the manual from `strtol`:

```bash
`man strtol strtoimax, strtol, strtoll, strtoq -- convert a string value to a long, long long, intmax_t or quad_t integer `
```

So in line 13 we cast a `long` to an `int which overflows`

```
`result = strtol(str, NULL, 36); `
```

### Reuse Internal DirectFunctionCall

Let’s do the cast correctly by again reusing Postgres internals: So how does Postgres cast a `bigint` to an `integer`?

```sql
`test=# \dC bigint                              List of casts  Source type  |     Target type       |      Function      |   Implicit? --------------+-----------------------+--------------------+---------------  bigint       | bit                   | bit                | no  bigint       | double precision      | float8             | yes  bigint       | integer               | int4               | in assignment `
```

The SQL-function `int4` is used here — how is that defined?

```sql
`test=# \df+ int4                            List of functions  Name | Result data type | Argument data types |  Source code ------+------------------+---------------------+---------------------  int4 | integer          | "char"              |  chartoi4  int4 | integer          | bigint              |  int84  int4 | integer          | bit                 |  bittoint4  int4 | integer          | boolean             |  bool_int4  int4 | integer          | double precision    |  dtoi4  int4 | integer          | numeric             |  numeric_int4  int4 | integer          | real                |  ftoi4  int4 | integer          | smallint            |  i2toi4 (8 rows) `
```

So `int84` is what we are looking for. You’ll find the definition in `utils/int8.h`, which we need to include in our source code to be able to use it. You already learned [in the first post](http://big-elephants.com/2015-11/writing-postgres-extensions-part-iv/2015-10/writing-postgres-extensions-part-i/) that in order to use C-functions in SQL you’ll have to define them using the “version 1” calling convention. Thus, these functions have a specific signature for `int84`. Here it is:

```c
`extern Datum int84(PG_FUNCTION_ARGS); `
```

So we cannot directly call this function from our code. Instead, we have to use the `DirectFunctionCall`macros from `fmgr.h`:

```c
`DirectFunctionCall1(func, arg1) DirectFunctionCall2(func, arg1, arg2) DirectFunctionCall3(func, arg1, arg2, arg3) DirectFunctionCall4(func, arg1, arg2, arg3, arg4) DirectFunctionCall5(func, arg1, arg2, arg3, arg4, arg5) DirectFunctionCall6(func, arg1, arg2, arg3, arg4, arg5, arg6) DirectFunctionCall7(func, arg1, arg2, arg3, arg4, arg5, arg6, arg7) DirectFunctionCall8(func, arg1, arg2, arg3, arg4, arg5, arg6, arg7, arg8) DirectFunctionCall9(func, arg1, arg2, arg3, arg4, arg5, arg6, arg7, arg8, arg9) `
```

With these macros we can directly call any function from our C code, depending on the number of arguments. But be careful using that: these macros are not type-safe, as the arguments passed and returned are just `Datums` which is any kind of data. Using this you won’t get an error from the compiler. You’ll simply get strange results on runtime if you pass the wrong data types around - one more reason to have a fully covered test suite.

As the macro already returns a Datum type, we’d end up with:

base36.c

```c
`PG_FUNCTION_INFO_V1(base36_in); Datum base36_in(PG_FUNCTION_ARGS) {     long result;     char *str = PG_GETARG_CSTRING(0);     result = strtol(str, NULL, 36);     PG_RETURN_DATUM(DirectFunctionCall1(int84,(int64)result)); } `
```

To finally get:

```sql
`# SELECT 'abcdefghi'::base36; ERROR:  integer out of range LINE 1: SELECT 'abcdefghi'::base36; `
```

### Pimp the Makefile

To have a better overview about the different tests, let’s split them up into different files and store them under the `test/sql` directory. To make this work, we need to adapt the Makefile as well.

Makefile

```
`EXTENSION     = base36                          # the extensions name DATA          = base36--0.0.1.sql               # script files to install TESTS         = $(wildcard test/sql/*.sql)      # use test/sql/*.sql as test files  # find the sql and expected directories under test # load base36 extension into test db # load plpgsql into test db REGRESS_OPTS  = --inputdir=test         \                 --load-extension=base36 \                 --load-language=plpgsql REGRESS       = $(patsubst test/sql/%.sql,%,$(TESTS)) MODULES       = base36                          # our c module file to build  # postgres build stuff PG_CONFIG = pg_config PGXS := $(shell $(PG_CONFIG) --pgxs) include $(PGXS) `
```

`TESTS` defines our different test files which you can find under `test/sql/*.sql`. Also we added `REGRESS_OPTS` changing the test input directory to `test` (`--inputdir=test`), that is the directory where the regression runner expects the `sql` directory with the test scripts and the `expected` directory with the expected output. We also define that the extension base36 should be created in the test database beforehand (`--load-extension=base36`), avoiding running the `CREATE EXTENSION` command on top of each test script. We also define to load the `plpgsql` language into the test database, which is actually not needed for our test suite. But it doesn’t hurt, and gives us a more general Makefile for our future projects.

### Test Files Organization

Let’s now add the test files:

test/sql/base36_io.sql

```sql
`-- simple input SELECT '120'::base36; SELECT '3c'::base36; -- case insensitivity SELECT '3C'::base36; SELECT 'FoO'::base36; -- invalid characters SELECT 'foo bar'::base36; SELECT 'abc$%2'::base36; -- negative values SELECT '-10'::base36; -- too big values SELECT 'abcdefghi'::base36;  -- storage BEGIN; CREATE TABLE base36_test(val base36); INSERT INTO base36_test VALUES ('123'), ('3c'), ('5A'), ('zZz'); SELECT * FROM base36_test; UPDATE base36_test SET val = '567a' where val = '123'; SELECT * FROM base36_test; ROLLBACK; `
```

Note I wrapped the state changing commands in a transaction that will be rolled back at the end. This is to ensure that that each script starts with a clean state. If we now look at what we got in `results/base36_io.out` we see that we have again some interesting behavior on malicious input.

```bash
`-- invalid characters SELECT 'foo bar'::base36;  base36 --------  foo (1 row)  SELECT 'abc$%2'::base36;  base36 --------  abc (1 row) `
```

The `strtol` function converts into the given base, stopping at the end of the string or at the first character that does not produce a valid digit in the given base. We definitely don’t want this surprise, so let’s read the man page `man strtol` and fix it.

```
`If endptr is not NULL, strtol() stores the address of the first invalid character in *endptr. If there were no digits at all, however, strtol() stores the original value of str in *endptr. (Thus, if *str is not `\0' but **endptr is `\0' on return, the entire string was valid.) `
```

base36.c

```c
`PG_FUNCTION_INFO_V1(base36_in); Datum base36_in(PG_FUNCTION_ARGS) {     long result;     char *bad;     char *str = PG_GETARG_CSTRING(0);     result = strtol(str, &bad, 36);     if (bad[0] != '\0' || strlen(str)==0)         ereport(ERROR,             (              errcode(ERRCODE_SYNTAX_ERROR),              errmsg("invalid input syntax for base36: \"%s\"", str)             )         );     PG_RETURN_DATUM(DirectFunctionCall1(int84,(int64)result)); } `
```

Now after running `make clean && make && make install && make installcheck`, `results/base36_io.out`looks good. Let’s copy it into the expected folder:

```
`mkdir test/expected cp results/base36_io.out test/expected `
```

And rerun our test suite:

```
`make clean && make && make install && make installcheck `
```

test/sql/operators.sql

```sql
`-- comparison SELECT '120'::base36 > '3c'::base36; SELECT '120'::base36 >= '3c'::base36; SELECT '120'::base36 < '3c'::base36; SELECT '120'::base36 <= '3c'::base36; SELECT '120'::base36 <> '3c'::base36; SELECT '120'::base36 = '3c'::base36;  -- comparison equals SELECT '120'::base36 > '120'::base36; SELECT '120'::base36 >= '120'::base36; SELECT '120'::base36 < '120'::base36; SELECT '120'::base36 <= '120'::base36; SELECT '120'::base36 <> '120'::base36; SELECT '120'::base36 = '120'::base36;  -- comparison negation SELECT NOT '120'::base36 > '120'::base36; SELECT NOT '120'::base36 >= '120'::base36; SELECT NOT '120'::base36 < '120'::base36; SELECT NOT '120'::base36 <= '120'::base36; SELECT NOT '120'::base36 <> '120'::base36; SELECT NOT '120'::base36 = '120'::base36;  --commutator and negator BEGIN; CREATE TABLE base36_test AS SELECT i::base36 as val FROM generate_series(1,10000) i; CREATE INDEX ON base36_test(val); ANALYZE; SET enable_seqscan TO off; EXPLAIN (COSTS OFF) SELECT * FROM base36_test where NOT val < 'c1'; EXPLAIN (COSTS OFF) SELECT * FROM base36_test where NOT 'c1' > val; EXPLAIN (COSTS OFF) SELECT * FROM base36_test where 'c1' > val; -- hash aggregate SET enable_seqscan TO on; EXPLAIN (COSTS OFF) SELECT val, COUNT(*) FROM base36_test GROUP BY 1; ROLLBACK; `
```

Here we played with some [runtime query configuration](http://www.postgresql.org/docs/9.4/static/runtime-config-query.html) to force index usage and a hash aggregate.

```sql
`SET enable_seqscan TO off; EXPLAIN (COSTS OFF) SELECT * FROM base36_test where NOT val < 'c1';                         QUERY PLAN ----------------------------------------------------------  Index Only Scan using base36_test_val_idx on base36_test    Index Cond: (val >= 'c1'::base36) (2 rows)  EXPLAIN (COSTS OFF) SELECT * FROM base36_test where NOT 'c1' > val;                         QUERY PLAN ----------------------------------------------------------  Index Only Scan using base36_test_val_idx on base36_test    Index Cond: (val >= 'c1'::base36) (2 rows)  EXPLAIN (COSTS OFF) SELECT * FROM base36_test where 'c1' > val;                         QUERY PLAN ----------------------------------------------------------  Index Only Scan using base36_test_val_idx on base36_test    Index Cond: (val < 'c1'::base36) (2 rows)  -- hash aggregate SET enable_seqscan TO on; EXPLAIN (COSTS OFF) SELECT val, COUNT(*) FROM base36_test GROUP BY 1;           QUERY PLAN -------------------------------  HashAggregate    Group Key: val    ->  Seq Scan on base36_test (3 rows) `
```

Thus, we can make sure `COMMUTATOR` and `NEGATOR` are set up correctly.

As we didn’t write much own code but used Postgres’ internals we see `results/operators.out` looks good. We’ll copy it over as well.

```bash
`cp results/operators.out test/expected make clean && make && make install && make installcheck `
```

getting

```bash
`============== running regression test queries        ============== test base36_io                ... ok test operators                ... ok  =====================  All 2 tests passed. ===================== `
```

### One more test

So far we implemented input and output functions, reused Postgres’ comparison functions and operators and tested everything. Are we done? Nope! There is one more test we could add:

test/sql/operators.sql

```sql
`-- storage BEGIN; CREATE TABLE base36_test(val base36); INSERT INTO base36_test VALUES ('123'), ('3c'), ('5A'), ('zZz'); SELECT * FROM base36_test; UPDATE base36_test SET val = '567a' where val = '123'; SELECT * FROM base36_test; UPDATE base36_test SET val = '-aa' where val = '3c'; SELECT * FROM base36_test; ROLLBACK; `
```

Here we try to update to a negative value which should fail:

```bash
`UPDATE base36_test SET val = '-aa' where val = '3c'; SELECT * FROM base36_test; ERROR:  negative values are not allowed DETAIL:  value -370 is negative HINT:  make it positive `
```

But it doesn’t…Well, it does, but not on the update step – only when retrieving the value. While we disallow negative values for the `OUTPUT` function, it’s still allowed for the `INPUT`. When we execute the following command:

```
`SELECT '-aa'::base36; ERROR:  negative values are not allowed DETAIL:  value -370 is negative HINT:  make it positive `
```

both `INPUT` and `OUTPUT` functions are called, resulting in the error. But for the `UPDATE` command only input is called, resulting in a negative value on disk which then can never be retrieved. Let’s fix that quickly

base36.c

```
`PG_FUNCTION_INFO_V1(base36_in); Datum base36_in(PG_FUNCTION_ARGS) {     int64 result;     char *bad;     char *str = PG_GETARG_CSTRING(0);     result = strtol(str, &bad, 36);     if (bad[0] != '\0' || strlen(str)==0)         ereport(ERROR,             (              errcode(ERRCODE_SYNTAX_ERROR),              errmsg("invalid input syntax for base36: \"%s\"", str)             )         );     if (result < 0)         ereport(ERROR,             (              errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),              errmsg("negative values are not allowed"),              errdetail("value %ld is negative", result),              errhint("make it positive")             )         );     PG_RETURN_DATUM(DirectFunctionCall1(int84,result)); } `
```

## Is it worth the effort?

While it’s fun to extend Postgres, let’s not forget why we actually built all of this. Let’s compare the `base36` approach to the Postgres-native approach of using `varchar` type. We’ll compare two aspects: the storage requirements for each type and the respective query performance.

### Storage Requirements

Our initial motivation was to save space and just store 4 byte integers instead of 6 characters, which according to the [documentation](http://www.postgresql.org/docs/9.4/static/datatype-character.html) would waste 7 bytes.

So let’s compare it.

```sql
`test=# CREATE TABLE base36_check (val base36); CREATE TABLE test=# CREATE TABLE varchar_check (val varchar(6)); CREATE TABLE test=# INSERT INTO base36_check SELECT i::base36 from generate_series(1,1e6::int) i; INSERT 0 1000000 test=# INSERT INTO varchar_check SELECT i::base36::text from generate_series(1,1e6::int) i; INSERT 0 1000000 test=# SELECT pg_table_size('base36_check') as "base36 size", pg_table_size('varchar_check') as "varchar_check size";  base36 size | varchar_check size -------------+-----------------------     36249600 |              36249600 (1 row) `
```

Oops…we didn’t save a single byte! That’s quite unfortunate for all the effort we put into our datatype. So how does this happen? Well, we have to know how Postgres [actually stores the data](http://www.postgresql.org/docs/9.4/static/storage-page-layout.html). Our little example would end up with the following:

base36_check: 23 bytes for the header + 1 byte for the null bitmap + 4 byte for data = 28 bytes varchar_check: 23 bytes for the header + 1 byte for the null bitmap + 7 byte for data = 31 bytes

So we should indeed save 3 bytes per row but still end up with the same table size. We also need to consider that Postgres stores data in a page which typically contains 8kB (8192 bytes) of data, and that a single row can not span two pages. Each row would also end up with a multiple of maximum data alignment setting, which is 8 bytes on a modern 64bit system.

So in the end, we’d need 32 bytes + 4 bytes tuple pointer per row in both situations.

```
`       (8192 per page - 24 page header) -----------------------------------------------------  = 226 rows per page (32 byte data and alignment + 4 byte tuple pointer) `
```

The picture would (of course) totally change in a real world example:

```sql
`test=# DROP TABLE base36_check; DROP TABLE test=# DROP TABLE varchar_check; DROP TABLE test=# CREATE TABLE base36_check (val base36, num integer); CREATE TABLE test=# CREATE TABLE varchar_check (val varchar(6), num integer); CREATE TABLE test=# INSERT INTO base36_check SELECT i::base36, i from generate_series(1,1e6::int) i; INSERT 0 1000000 test=# INSERT INTO varchar_check SELECT i::base36::text,i from generate_series(1,1e6::int) i; INSERT 0 1000000 test=# SELECT pg_size_pretty(pg_table_size('base36_check')) as "base36 size", pg_size_pretty(pg_table_size('varchar_check')) as "varchar_check size";  base36 size | varchar_check size -------------+--------------------  35 MB       | 42 MB `
```

As we added data into the database, due to alignment 4 wasted bytes on our `base36_check` table it didn’t grow, while the base36_check table grew by 4 bytes of data plus 4 bytes alignment per row.

Now we’re saving a good 20% of space.

### Query Performance

Let’s also do some timing.

```sql
`test=# \timing Timing is on. test=# SELECT * FROM varchar_check ORDER BY VAL LIMIT 10;  val  |  num ------+-------  1    |     1  10   |    36  100  |  1296  1000 | 46656  1001 | 46657  1002 | 46658  1003 | 46659  1004 | 46660  1005 | 46661  1006 | 46662 (10 rows)  Time: 601,551 ms  test=# SELECT * FROM base36_check ORDER BY VAL LIMIT 10;  val | num -----+-----  1   |   1  2   |   2  3   |   3  4   |   4  5   |   5  6   |   6  7   |   7  8   |   8  9   |   9  a   |  10 (10 rows)  Time: 73,575 ms `
```

Besides the fact that the sorting of base36 feels more natural, it’s also 8 times faster. If you keep in mind that sorting is a key operation for databases, then this fact gives us the real optimization. For example, when creating an index:

```sql
`test=# CREATE INDEX ON varchar_check(val); CREATE INDEX Time: 13585,451 ms test=# CREATE INDEX ON base36_check(val); CREATE INDEX Time: 294,433 ms `
```

It’s also useful for join operations or grouping by statements.

## More to come …

Now that we’ve fixed all the bugs and added tests to ensure they won’t come back, our extension is almost complete. In [the next post](http://big-elephants.com/2015-11/writing-postgres-extensions-part-v/) on this series we’ll complete the extension with a `bigbase36` type and see how we can structure our code a bit better.