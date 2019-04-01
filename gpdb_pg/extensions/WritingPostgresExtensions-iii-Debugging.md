# Writing Postgres Extensions - Debugging

In the [last post about Writing Postgres Extensions](http://big-elephants.com/2015-10/writing-postgres-extensions-part-ii/) we created a new data type `base36` from ground up. However we left with a serious bug causing our server to crash.

Now let’s hunt that bug down with a debugger and complete the testsuite.

We created a dedicated [github repo](https://github.com/adjust/postgresql_extension_demo) following the content from these series on writing PostgreSQL extensions. The code from the last article could be found on [branch part_ii](https://github.com/adjust/postgresql_extension_demo/tree/part_ii) and today’s changes are on [branch part_iii](https://github.com/adjust/postgresql_extension_demo/tree/part_iii).

## The Bug

First let’s reproduce the bug.

```sql
`test=# CREATE EXTENSION base36; test=# CREATE TABLE base36_test(val base36); test=#  EXPLAIN SELECT * FROM base36_test where '3c'::base36 > val; server closed the connection unexpectedly   This probably means the server terminated abnormally   before or while processing the request. The connection to the server was lost. Attempting reset: Failed. Time: 680,225 ms !> `
```

We definitely don’t want this to happen on our production database, so lets find out where the problem is. We only wrote two relatively simple C-functions `base36_out` and `base36_in`. If we assume that we are not smarter than the folks from the PostgreSQL-Core team - which is at least for me personally a reasonable assumption - then the bug must be in one of these.

base36.c

```c
`#include "postgres.h" #include "fmgr.h" #include "utils/builtins.h"  PG_MODULE_MAGIC;  PG_FUNCTION_INFO_V1(base36_in); Datum base36_in(PG_FUNCTION_ARGS) {     long result;     char *str = PG_GETARG_CSTRING(0);     result = strtol(str, NULL, 36);     PG_RETURN_INT32((int32)result); }  PG_FUNCTION_INFO_V1(base36_out); Datum base36_out(PG_FUNCTION_ARGS) {     int32 arg = PG_GETARG_INT32(0);     if (arg < 0)         ereport(ERROR,             (              errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),              errmsg("negative values are not allowed"),              errdetail("value %d is negative", arg),              errhint("make it positive")             )         );     char base36[36] = "0123456789abcdefghijklmnopqrstuvwxyz";      /* max 6 char + '\0' */     char *buffer        = palloc(7 * sizeof(char));     unsigned int offset = 7 * sizeof(char);     buffer[--offset]    = '\0';      do {         buffer[--offset] = base36[arg % 36];     } while (arg /= 36);      PG_RETURN_CSTRING(&buffer[offset]); } `
```

## Set up debugging environment

In order to use a debugger such as [LLDB](http://lldb.llvm.org/) you’ll need to compile PostgreSQL with debug symbols. The following short guidance through debugging works for me on MacOS having PostgreSQL installed with homebrew and using LLDB with Xcode.

Firstly, let’s shut down any running Postgres instances - you don’t want to mess up your existing DB or work :)

```bash
`$ cd /usr/local/opt/postgresql $ launchctl unload homebrew.mxcl.postgresql.plist # Double check it’s not running: $ psql some_db psql: could not connect to server: No such file or directory   Is the server running locally and accepting   connections on Unix domain socket "/tmp/.s.PGSQL.5432"? `
```

Next we’ll download the PostgreSQL [source code](http://www.postgresql.org/ftp/source/) by executing this script.

```bash
`$ cd ~ $ curl https://ftp.postgresql.org/pub/source/v9.4.4/postgresql-9.4.4.tar.bz2 | bzip2 -d | tar x $ cd postgresql-9.4.4 `
```

And build with debugging options enabled.

```bash
`$  ./configure --enable-cassert --enable-debug CFLAGS="-ggdb" $ make $ sudo make install `
```

We’ll skip the `adduser` command that the [Postgres docs](http://www.postgresql.org/docs/9.4/static/install-short.html) recommend. Instead, I’ll just run Postgres using my own user account to make debugging easier.

```bash
`$ sudo chown $(whoami) /usr/local/pgsql `
```

Then init the data directory

```bash
`$ /usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data `
```

And start the server

```bash
`$ /usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l /usr/local/pgsql/data/postmaster.log start `
```

Add `pgsql/bin` path from the new installation to the `PATH` environment variable

```bash
`$ export PATH=/usr/local/pgsql/bin:$PATH `
```

Install the extension (due to the export above this time `pgxn` from the new installation is used).

```bash
`$ make && make install `
```

Now we can create a test db

```bash
`$ /usr/local/pgsql/bin/createdb test `
```

and connect to it

```bash
`$ /usr/local/pgsql/bin/psql test `
```

Check if it works – well or not

```sql
`test=# CREATE EXTENSION base36 ; CREATE EXTENSION test=# CREATE TABLE base36_test(val base36); CREATE TABLE test=# INSERT INTO base36_test VALUES ('123'), ('3c'), ('5A'), ('zZz'); INSERT 0 4 test=# EXPLAIN SELECT * FROM base36_test where val='3c'; server closed the connection unexpectedly   This probably means the server terminated abnormally   before or while processing the request. The connection to the server was lost. Attempting reset: Failed. !> `
```

## Debugging

Now that we have our debugging environment setup, let’s start the actual chasing of the problem. Firstly, let’s look at the log file. That’s the file we specified with the `-l` flag to `pg_ctl`. In our case `/usr/local/pgsql/data/postmaster.log`.

```bash
`TRAP: FailedAssertion("!(pointer == (void *) (((uintptr_t) ((pointer)) + ((8) - 1)) & ~((uintptr_t) ((8) - 1))))", File: "mcxt.c", Line: 699) LOG:  server process (PID 6515) was terminated by signal 6: Abort trap DETAIL:  Failed process was running: EXPLAIN SELECT * FROM base36_test where val='3c'; LOG:  terminating any other active server processes WARNING:  terminating connection because of crash of another server process DETAIL:  The postmaster has commanded this server process to roll back the current transaction and exit, because another server process exited abnormally and possibly corrupted shared memory. HINT:  In a moment you should be able to reconnect to the database and repeat your command. LOG:  all server processes terminated; reinitializing LOG:  database system was interrupted; last known up at 2015-10-09 15:11:18 CEST LOG:  database system was not properly shut down; automatic recovery in progress LOG:  redo starts at 0/22D0868 LOG:  record with zero length at 0/2359140 LOG:  redo done at 0/2359110 LOG:  last completed transaction was at log time 2015-10-09 15:12:01.344859+02 LOG:  MultiXact member wraparound protections are now enabled LOG:  database system is ready to accept connections LOG:  autovacuum launcher started `
```

Reconnect to the database and find out the pid of your current db session

```sql
`test=# SELECT pg_backend_pid();  pg_backend_pid ----------------            6644 (1 row) `
```

Connect LLDB with the pid (in another terminal)

```bash
`lldb -p 6644 `
```

Run the failing command in the psql session

```sql
`EXPLAIN SELECT * FROM base36_test where val='3c'; `
```

Continue LLDB

```bash
`(lldb) c Process 6644 resuming Process 6644 stopped * thread #1: tid = 0x84aa4, 0x00007fff906d2286 libsystem_kernel.dylib`__pthread_kill + 10, queue = 'com.apple.main-thread', stop reason = signal SIGABRT     frame #0: 0x00007fff906d2286 libsystem_kernel.dylib`__pthread_kill + 10 libsystem_kernel.dylib`__pthread_kill: ->  0x7fff906d2286 <+10>: jae    0x7fff906d2290            ; <+20>     0x7fff906d2288 <+12>: movq   %rax, %rdi     0x7fff906d228b <+15>: jmp    0x7fff906cdc53            ; cerror_nocancel     0x7fff906d2290 <+20>: retq `
```

Get a Backtrace from LLDB

```bash
`(lldb) bt * thread #1: tid = 0x84aa4, 0x00007fff906d2286 libsystem_kernel.dylib`__pthread_kill + 10, queue = 'com.apple.main-thread', stop reason = signal SIGABRT   * frame #0: 0x00007fff906d2286 libsystem_kernel.dylib`__pthread_kill + 10     frame #1: 0x00007fff910f39f9 libsystem_pthread.dylib`pthread_kill + 90     frame #2: 0x00007fff848f19b3 libsystem_c.dylib`abort + 129     frame #3: 0x0000000108328549 postgres`ExceptionalCondition(conditionName="!(pointer == (void *) (((uintptr_t) ((pointer)) + ((8) - 1)) & ~((uintptr_t) ((8) - 1))))", errorType="FailedAssertion", fileName="mcxt.c", lineNumber=699) + 137 at assert.c:54     frame #4: 0x000000010836129d postgres`pfree(pointer=0x00007ff9e1813674) + 173 at mcxt.c:699     frame #5: 0x00000001082ab9e3 postgres`get_const_expr(constval=0x00007ff9e1806708, context=0x00007fff57e824c8, showtype=0) + 707 at ruleutils.c:8002     frame #6: 0x00000001082a5f79 postgres`get_rule_expr(node=0x00007ff9e1806708, context=0x00007fff57e824c8, showimplicit='\x01') + 281 at ruleutils.c:6647     frame #7: 0x00000001082acf22 postgres`get_rule_expr_paren(node=0x00007ff9e1806708, context=0x00007fff57e824c8, showimplicit='\x01', parentNode=0x00007ff9e1806788) + 146 at ruleutils.c:6600  ...more `
```

Ok what do we have? The exception is thrown in `pfree` which is defined in `mcxt.c:699`. `pfree` is called from `get_const_expr` in `ruleutils.c:8002` and so forth. If we go four times up the call stack. We’d end up here:

```bash
`(lldb) up frame #4: 0x000000010836129d postgres`pfree(pointer=0x00007ff9e1813674) + 173 at mcxt.c:699    696     * allocated chunk.    697     */    698    Assert(pointer != NULL); -> 699    Assert(pointer == (void *) MAXALIGN(pointer));    700    701    /*    702     * OK, it's probably safe to look at the chunk header. `
```

Let’s look at the source code in

mcxt.c:699

```c
`/*  * pfree  *    Release an allocated chunk.  */ void pfree(void *pointer) {   MemoryContext context;    /*    * Try to detect bogus pointers handed to us, poorly though we can.    * Presumably, a pointer that isn't MAXALIGNED isn't pointing at an    * allocated chunk.    */   Assert(pointer != NULL);   Assert(pointer == (void *) MAXALIGN(pointer));    /*    * OK, it's probably safe to look at the chunk header.    */   context = ((StandardChunkHeader *)          ((char *) pointer - STANDARDCHUNKHEADERSIZE))->context;    AssertArg(MemoryContextIsValid(context));    (*context->methods->free_p) (context, pointer);   VALGRIND_MEMPOOL_FREE(context, pointer); } `
```

Postgres uses `pfree` to release memory from the current memory context. Somehow we messed up our memory.

Let’s take a look at the pointers content

```bash
`(lldb) print (char *)pointer (char *) $0 = 0x00007ff9e1813674 "3c" `
```

It’s indeed our search condition `3c`. So what did we do wrong here? As mentioned in the first article `pfree` and `palloc` are Postgres counterparts of `free` and `malloc` to safely allocate and free memory in the current memory context. Somehow we messed it up. In `base36_out` we used

```
`char *buffer = palloc0(7 * sizeof(char)); `
```

to allocate 7 bytes of memory. Finally we return a pointer

```
`PG_RETURN_CSTRING(&buffer[offset]); `
```

at offset 4 in this case. The assertion in mcxt.c:699

```
`Assert(pointer == (void *) MAXALIGN(pointer)); `
```

Makes sure that the data to be released are correctly aligned. The condition here is:

```
`(pointer == (void *) (((uintptr_t) ((pointer)) + ((8) - 1)) & ~((uintptr_t) ((8) - 1)))) `
```

To be read as does the pointer start at a multiple of 8 bytes? As we don’t return the same address as the one we allocated from, it causes `pfree` to complain that the pointer is not aligned.

Let’s fix that!

base36.c

```c
`PG_FUNCTION_INFO_V1(base36_out); Datum base36_out(PG_FUNCTION_ARGS) {     int32 arg = PG_GETARG_INT32(0);     if (arg < 0)         ereport(ERROR,             (              errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),              errmsg("negative values are not allowed"),              errdetail("value %d is negative", arg),              errhint("make it positive")             )         );     char base36[36] = "0123456789abcdefghijklmnopqrstuvwxyz";      /* max 6 char + '\0' */     char buffer[7];     unsigned int offset = sizeof(buffer);     buffer[--offset] = '\0';      do {         buffer[--offset] = base36[arg % 36];     } while (arg /= 36);      PG_RETURN_CSTRING(pstrdup(&buffer[offset])); } `
```

Now we allocate the buffer from the stack (Line 18) and finally us `pstrdup` to copy the string freshly allocated memory (Line 26). This implementation is closer – almost equivalent to [Wikipedias](https://en.wikipedia.org/wiki/Base36#C_implementation).

You might have guessed that `pstrdup` is Postgres counterpart of `strdup`. It safely takes memory from the current memory context via `palloc` and frees automatically at the end of a transaction.

## TYPE CASTING

Now that we can input and output data for our type. It would be nice to also cast from and to other types.

base36–0.0.1.sql

```sql
`-- type and operators omitted  CREATE CAST (integer as base36) WITHOUT FUNCTION AS IMPLICIT; CREATE CAST (base36 as integer) WITHOUT FUNCTION AS IMPLICIT; `
```

Wow that is relatively easy. As `integer` and `base36` are binary coercible (that is the binary internal representations are the same) the conversion can be done for free (`WITHOUT FUNCTION`). We also marked this cast as `IMPLICIT` thus telling postgres that it can perform the cast automatically whenever suitable. For example consider this query:

```sql
`test=# SELECT 10::integer + '5a'::base36;  ?column? ----------       200 (1 row) `
```

There is no `integer + base36` operator defined but by implicit casting `base36` to `integer` Postgres can use the `integer + integer` operator and give us the result as integer. However implicit casts should be defined with care as the result of certain operations might be suspicious. For the above operation a user wouldn’t know if the result is integer or base36 and thus might misinterpret it. Queries will totally break if we later decide to add an operator `integer + base36` which returns `base36`.

Even more confusing might be this query result:

```sql
`test=# SELECT -50::base36;  ?column? ----------       -50 (1 row) `
```

Although we disallowed negative values we get one here how is that possible? Internally Postgres does this operation:

```sql
`test=# SELECT -(50::base36)::integer;  ?column? ----------       -50 `
```

We can and should avoid such a confusing behavior. One option would be to add a prefix to base36 output (like it is common for hex or octal numbers) or by giving the responsibility to the user and only allow explicit casts.

Another option to clarify things would be to mark the cast `AS ASSIGNMENT`. With that casting would only be automatically performed if you assign an integer to a base36 type and vice versa. This is typically suitable for INSERT or UPDATE statements. Let’s try this:

base36–0.0.1.sql

```sql
`-- type and operators omitted  CREATE CAST (integer as base36) WITHOUT FUNCTION AS ASSIGNMENT; CREATE CAST (base36 as integer) WITHOUT FUNCTION AS ASSIGNMENT; `
```

and fill our table:

```sql
`test=# INSERT INTO base36_test SELECT i FROM generate_series(1,100) i; INSERT 0 100 SELECT * FROM base36_test ORDER BY val LIMIT 12;  val -----  1  2  3  4  5  6  7  8  9  a  b  c (12 rows) `
```

## More to come…

You have seen how important it is to test everything, not only to find bugs that in the worst case might crash the server, but also to specify the expected output from certain operations such as casts. In [the next post](http://big-elephants.com/2015-11/writing-postgres-extensions-part-iv/) we’ll elaborate on that creating a full-coverage test suite.