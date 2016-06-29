# **pt-online-schema-change**

## NAME

**pt-online-schema-change** - ALTER tables without locking them.

## SYNOPSIS

### Usage

```
pt-online-schema-change [OPTIONS] DSN
```

**pt-online-schema-change** alters a table’s structure without blocking reads or writes. Specify the database and table in the DSN. Do not use this tool before reading its documentation and checking your backups carefully.

Add a column to sakila.actor:

```
pt-online-schema-change --alter "ADD COLUMN c1 INT" D=sakila,t=actor

```

Change sakila.actor to InnoDB, effectively performing OPTIMIZE TABLE in a non-blocking fashion because it is already an InnoDB table:

```
pt-online-schema-change --alter "ENGINE=InnoDB" D=sakila,t=actor

```

## RISKS

The following section is included to inform users about the potential risks, whether known or unknown, of using this tool. The two main categories of risks are those created by the nature of the tool (e.g. read-only tools vs. read-write tools) and those created by bugs.

**pt-online-schema-change** modifies data and structures. You should be careful with it, and test it before using it in production. You should also ensure that you have recoverable backups before using this tool.

At the time of this release, we know of no bugs that could cause harm to users.

The authoritative source for updated information is always the online issue tracking system. Issues that affect this tool will be marked as such. You can see a list of such issues at the following URL: [http://www.percona.com/bugs/pt-online-schema-change](http://www.percona.com/bugs/pt-online-schema-change).

See also “BUGS” for more information on filing bugs and getting help.

## DESCRIPTION

**pt-online-schema-change** emulates the way that MySQL alters tables internally, but it works on a copy of the table you wish to alter. This means that the original table is not locked, and clients may continue to read and change data in it.

**pt-online-schema-change** works by creating an empty copy of the table to alter, modifying it as desired, and then copying rows from the original table into the new table. When the copy is complete, it moves away the original table and replaces it with the new one. By default, it also drops the original table.

The data copy process is performed in small chunks of data, which are varied to attempt to make them execute in a specific amount of time (see [*--chunk-time*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--chunk-time)). This process is very similar to how other tools, such as pt-table-checksum, work. Any modifications to data in the original tables during the copy will be reflected in the new table, because the tool creates triggers on the original table to update the corresponding rows in the new table. The use of triggers means that the tool will not work if any triggers are already defined on the table.

When the tool finishes copying data into the new table, it uses an atomic RENAME TABLE operation to simultaneously rename the original and new tables. After this is complete, the tool drops the original table.

Foreign keys complicate the tool’s operation and introduce additional risk. The technique of atomically renaming the original and new tables does not work when foreign keys refer to the table. The tool must update foreign keys to refer to the new table after the schema change is complete. The tool supports two methods for accomplishing this. You can read more about this in the documentation for [*--alter-foreign-keys-method*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--alter-foreign-keys-method).

Foreign keys also cause some side effects. The final table will have the same foreign keys and indexes as the original table (unless you specify differently in your ALTER statement), but the names of the objects may be changed slightly to avoid object name collisions in MySQL and InnoDB.

For safety, the tool does not modify the table unless you specify the [*--execute*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--execute) option, which is not enabled by default. The tool supports a variety of other measures to prevent unwanted load or other problems, including automatically detecting replicas, connecting to them, and using the following safety checks:

- The tool refuses to operate if it detects replication filters. See *--[no]check-replication-filters* for details.
- The tool pauses the data copy operation if it observes any replicas that are delayed in replication. See [*--max-lag*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--max-lag) for details.
- The tool pauses or aborts its operation if it detects too much load on the server. See [*--max-load*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--max-load) and [*--critical-load*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--critical-load)for details.
- The tool sets its lock wait timeout to 1 second so that it is more likely to be the victim of any lock contention, and less likely to disrupt other transactions. See [*--lock-wait-timeout*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--lock-wait-timeout) for details.
- The tool refuses to alter the table if foreign key constraints reference it, unless you specify [*--alter-foreign-keys-method*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--alter-foreign-keys-method).
- The tool cannot alter MyISAM tables on “Percona XtraDB Cluster” nodes.

## Percona XtraDB Cluster

**pt-online-schema-change** works with Percona XtraDB Cluster (PXC) 5.5.28-23.7 and newer, but there are two limitations: only InnoDB tables can be altered, and wsrep_OSU_method must be set to TOI (total order isolation). The tool exits with an error if the host is a cluster node and the table is MyISAM or is being converted to MyISAM (ENGINE=MyISAM), or if wsrep_OSU_method is not TOI. There is no way to disable these checks.

## OUTPUT

The tool prints information about its activities to STDOUT so that you can see what it is doing. During the data copy phase, it prints [*--progress*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--progress) reports to STDERR. You can get additional information by specifying [*--print*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--print).

If [*--statistics*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--statistics) is specified, a report of various internal event counts is printed at the end, like:

```
# Event  Count
# ====== =====
# INSERT     1

```

## OPTIONS

[*--dry-run*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--dry-run) and [*--execute*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--execute) are mutually exclusive.

This tool accepts additional command-line arguments. Refer to the “SYNOPSIS” and usage information for details.

- --alter

  type: stringThe schema modification, without the ALTER TABLE keywords. You can perform multiple modifications to the table by specifying them with commas. Please refer to the MySQL manual for the syntax of ALTER TABLE.The following limitations apply which, if attempted, will cause the tool to fail in unpredictable ways:The RENAME clause cannot be used to rename the table.Columns cannot be renamed by dropping and re-adding with the new name. The tool will not copy the original column’s data to the new column.If you add a column without a default value and make it NOT NULL, the tool will fail, as it will not try to guess a default value for you; You must specify the default.DROP FOREIGN KEY constraint_name requires specifying _constraint_name rather than the realconstraint_name. Due to a limitation in MySQL, **pt-online-schema-change** adds a leading underscore to foreign key constraint names when creating the new table. For example, to drop this contraint:`CONSTRAINT `fk_foo` FOREIGN KEY (`foo_id`) REFERENCES `bar` (`foo_id`)`You must specify --alter "DROP FOREIGN KEY _fk_foo".The tool does not use LOCK IN SHARE MODE with MySQL 5.0 because it can cause a slave error which breaks replication:`Query caused different errors on master and slave. Error on master:'Deadlock found when trying to get lock; try restarting transaction' (1213),Error on slave: 'no error' (0). Default database: 'pt_osc'.Query: 'INSERT INTO pt_osc.t (id, c) VALUES ('730', 'new row')'`The error happens when converting a MyISAM table to InnoDB because MyISAM is non-transactional but InnoDB is transactional. MySQL 5.1 and newer handle this case correctly, but testing reproduces the error 5% of the time with MySQL 5.0.This is a MySQL bug, similar to [http://bugs.mysql.com/bug.php?id=45694](http://bugs.mysql.com/bug.php?id=45694), but there is no fix or workaround in MySQL 5.0. Without LOCK IN SHARE MODE, tests pass 100% of the time, so the risk of data loss or breaking replication should be negligible.**Be sure to verify the new table if using MySQL 5.0 and converting from MyISAM to InnoDB!**


- --alter-foreign-keys-method

  type: stringHow to modify foreign keys so they reference the new table. Foreign keys that reference the table to be altered must be treated specially to ensure that they continue to reference the correct table. When the tool renames the original table to let the new one take its place, the foreign keys “follow” the renamed table, and must be changed to reference the new table instead.The tool supports two techniques to achieve this. It automatically finds “child tables” that reference the table to be altered.autoAutomatically determine which method is best. The tool uses rebuild_constraints if possible (see the description of that method for details), and if not, then it uses drop_swap.rebuild_constraintsThis method uses ALTER TABLE to drop and re-add foreign key constraints that reference the new table. This is the preferred technique, unless one or more of the “child” tables is so large that the ALTER would take too long. The tool determines that by comparing the number of rows in the child table to the rate at which the tool is able to copy rows from the old table to the new table. If the tool estimates that the child table can be altered in less time than the [*--chunk-time*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--chunk-time), then it will use this technique. For purposes of estimating the time required to alter the child table, the tool multiplies the row-copying rate by [*--chunk-size-limit*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--chunk-size-limit), because MySQL’s ALTER TABLE is typically much faster than the external process of copying rows.Due to a limitation in MySQL, foreign keys will not have the same names after the ALTER that they did prior to it. The tool has to rename the foreign key when it redefines it, which adds a leading underscore to the name. In some cases, MySQL also automatically renames indexes required for the foreign key.drop_swapDisable foreign key checks (FOREIGN_KEY_CHECKS=0), then drop the original table before renaming the new table into its place. This is different from the normal method of swapping the old and new table, which uses an atomicRENAME that is undetectable to client applications.This method is faster and does not block, but it is riskier for two reasons. First, for a short time between dropping the original table and renaming the temporary table, the table to be altered simply does not exist, and queries against it will result in an error. Secondly, if there is an error and the new table cannot be renamed into the place of the old one, then it is too late to abort, because the old table is gone permanently.This method forces --no-swap-tables and --no-drop-old-table.noneThis method is like drop_swap without the “swap”. Any foreign keys that referenced the original table will now reference a nonexistent table. This will typically cause foreign key violations that are visible in SHOW ENGINEINNODB STATUS, similar to the following:`Trying to add to index `idx_fk_staff_id` tuple:DATA TUPLE: 2 fields;0: len 1; hex 05; asc  ;;1: len 4; hex 80000001; asc     ;;But the parent table `sakila`.`staff_old`or its .ibd file does not currently exist!`This is because the original table (in this case, sakila.staff) was renamed to sakila.staff_old and then dropped. This method of handling foreign key constraints is provided so that the database administrator can disable the tool’s built-in functionality if desired.


- --ask-pass

  Prompt for a password when connecting to MySQL.


- --charset

  short form: -A; type: stringDefault character set. If the value is utf8, sets Perl’s binmode on STDOUT to utf8, passes the mysql_enable_utf8 option to DBD::mysql, and runs SET NAMES UTF8 after connecting to MySQL. Any other value sets binmode on STDOUT without the utf8 layer, and runs SET NAMES after connecting to MySQL.


- --check-interval

  type: time; default: 1Sleep time between checks for [*--max-lag*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--max-lag).


- --[no]check-alter

  default: yesParses the [*--alter*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--alter) specified and tries to warn of possible unintended behavior. Currently, it checks for:Column renamesIn previous versions of the tool, renaming a column with CHANGE COLUMN name new_namewould lead to that column’s data being lost. The tool now parses the alter statement and tries to catch these cases, so the renamed columns should have the same data as the originals. However, the code that does this is not a full-blown SQL parser, so you should first run the tool with [*--dry-run*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--dry-run) and [*--print*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--print) and verify that it detects the renamed columns correctly.DROP PRIMARY KEYIf [*--alter*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--alter) contain DROP PRIMARY KEY (case- and space-insensitive), a warning is printed and the tool exits unless [*--dry-run*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--dry-run) is specified. Altering the primary key can be dangerous, but the tool can handle it. The tool’s triggers, particularly the DELETE trigger, are most affected by altering the primary key because the tool prefers to use the primary key for its triggers. You should first run the tool with [*--dry-run*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--dry-run) and [*--print*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--print) and verify that the triggers are correct.


- --[no]check-plan

  default: yesCheck query execution plans for safety. By default, this option causes the tool to run EXPLAIN before running queries that are meant to access a small amount of data, but which could access many rows if MySQL chooses a bad execution plan. These include the queries to determine chunk boundaries and the chunk queries themselves. If it appears that MySQL will use a bad query execution plan, the tool will skip the chunk of the table.The tool uses several heuristics to determine whether an execution plan is bad. The first is whether EXPLAIN reports that MySQL intends to use the desired index to access the rows. If MySQL chooses a different index, the tool considers the query unsafe.The tool also checks how much of the index MySQL reports that it will use for the query. The EXPLAIN output shows this in the key_len column. The tool remembers the largest key_len seen, and skips chunks where MySQL reports that it will use a smaller prefix of the index. This heuristic can be understood as skipping chunks that have a worse execution plan than other chunks.The tool prints a warning the first time a chunk is skipped due to a bad execution plan in each table. Subsequent chunks are skipped silently, although you can see the count of skipped chunks in the SKIPPED column in the tool’s output.This option adds some setup work to each table and chunk. Although the work is not intrusive for MySQL, it results in more round-trips to the server, which consumes time. Making chunks too small will cause the overhead to become relatively larger. It is therefore recommended that you not make chunks too small, because the tool may take a very long time to complete if you do.


- --[no]check-replication-filters

  default: yesAbort if any replication filter is set on any server. The tool looks for server options that filter replication, such as binlog_ignore_db and replicate_do_db. If it finds any such filters, it aborts with an error.If the replicas are configured with any filtering options, you should be careful not to modify any databases or tables that exist on the master and not the replicas, because it could cause replication to fail. For more information on replication rules, see [http://dev.mysql.com/doc/en/replication-rules.html](http://dev.mysql.com/doc/en/replication-rules.html).


- --check-slave-lag

  type: stringPause the data copy until this replica’s lag is less than [*--max-lag*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--max-lag). The value is a DSN that inherits properties from the the connection options ([*--port*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--port), [*--user*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--user), etc.). This option overrides the normal behavior of finding and continually monitoring replication lag on ALL connected replicas. If you don’t want to monitor ALL replicas, but you want more than just one replica to be monitored, then use the DSN option to the [*--recursion-method*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--recursion-method) option instead of this option.


- --chunk-index

  type: stringPrefer this index for chunking tables. By default, the tool chooses the most appropriate index for chunking. This option lets you specify the index that you prefer. If the index doesn’t exist, then the tool will fall back to its default behavior of choosing an index. The tool adds the index to the SQL statements in a FORCE INDEX clause. Be careful when using this option; a poor choice of index could cause bad performance.


- --chunk-index-columns

  type: intUse only this many left-most columns of a [*--chunk-index*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--chunk-index). This works only for compound indexes, and is useful in cases where a bug in the MySQL query optimizer (planner) causes it to scan a large range of rows instead of using the index to locate starting and ending points precisely. This problem sometimes occurs on indexes with many columns, such as 4 or more. If this happens, the tool might print a warning related to the *--[no]check-plan* option. Instructing the tool to use only the first N columns of the index is a workaround for the bug in some cases.


- --chunk-size

  type: size; default: 1000Number of rows to select for each chunk copied. Allowable suffixes are k, M, G.This option can override the default behavior, which is to adjust chunk size dynamically to try to make chunks run in exactly [*--chunk-time*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--chunk-time) seconds. When this option isn’t set explicitly, its default value is used as a starting point, but after that, the tool ignores this option’s value. If you set this option explicitly, however, then it disables the dynamic adjustment behavior and tries to make all chunks exactly the specified number of rows.There is a subtlety: if the chunk index is not unique, then it’s possible that chunks will be larger than desired. For example, if a table is chunked by an index that contains 10,000 of a given value, there is no way to write a WHERE clause that matches only 1,000 of the values, and that chunk will be at least 10,000 rows large. Such a chunk will probably be skipped because of [*--chunk-size-limit*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--chunk-size-limit).


- --chunk-size-limit

  type: float; default: 4.0Do not copy chunks this much larger than the desired chunk size.When a table has no unique indexes, chunk sizes can be inaccurate. This option specifies a maximum tolerable limit to the inaccuracy. The tool uses <EXPLAIN> to estimate how many rows are in the chunk. If that estimate exceeds the desired chunk size times the limit, then the tool skips the chunk.The minimum value for this option is 1, which means that no chunk can be larger than [*--chunk-size*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--chunk-size). You probably don’t want to specify 1, because rows reported by EXPLAIN are estimates, which can be different from the real number of rows in the chunk. You can disable oversized chunk checking by specifying a value of 0.The tool also uses this option to determine how to handle foreign keys that reference the table to be altered. See [*--alter-foreign-keys-method*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--alter-foreign-keys-method) for details.


- --chunk-time

  type: float; default: 0.5Adjust the chunk size dynamically so each data-copy query takes this long to execute. The tool tracks the copy rate (rows per second) and adjusts the chunk size after each data-copy query, so that the next query takes this amount of time (in seconds) to execute. It keeps an exponentially decaying moving average of queries per second, so that if the server’s performance changes due to changes in server load, the tool adapts quickly.If this option is set to zero, the chunk size doesn’t auto-adjust, so query times will vary, but query chunk sizes will not. Another way to do the same thing is to specify a value for [*--chunk-size*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--chunk-size) explicitly, instead of leaving it at the default.


- --config

  type: ArrayRead this comma-separated list of config files; if specified, this must be the first option on the command line.


- --critical-load

  type: Array; default: Threads_running=50Examine SHOW GLOBAL STATUS after every chunk, and abort if the load is too high. The option accepts a comma-separated list of MySQL status variables and thresholds. An optional =MAX_VALUE (or :MAX_VALUE) can follow each variable. If not given, the tool determines a threshold by examining the current value at startup and doubling it.See [*--max-load*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--max-load) for further details. These options work similarly, except that this option will abort the tool’s operation instead of pausing it, and the default value is computed differently if you specify no threshold. The reason for this option is as a safety check in case the triggers on the original table add so much load to the server that it causes downtime. There is probably no single value of Threads_running that is wrong for every server, but a default of 50 seems likely to be unacceptably high for most servers, indicating that the operation should be canceled immediately.


- --default-engine

  Remove ENGINE from the new table.By default the new table is created with the same table options as the original table, so if the original table uses InnoDB, then the new table will use InnoDB. In certain cases involving replication, this may cause unintended changes on replicas which use a different engine for the same table. Specifying this option causes the new table to be created with the system’s default engine.


- --defaults-file

  short form: -F; type: stringOnly read mysql options from the given file. You must give an absolute pathname.


- --[no]drop-new-table

  default: yesDrop the new table if copying the original table fails.Specifying --no-drop-new-table and --no-swap-tables leaves the new, altered copy of the table without modifying the original table. The new table name is like _TBL_new where TBL is the table name.–no-drop-new-table does not work with alter-foreign-keys-method drop_swap.


- --[no]drop-old-table

  default: yesDrop the original table after renaming it. After the original table has been successfully renamed to let the new table take its place, and if there are no errors, the tool drops the original table by default. If there are any errors, the tool leaves the original table in place.If --no-swap-tables is specified, then there is no old table to drop.


- --dry-run

  Create and alter the new table, but do not create triggers, copy data, or replace the original table.


- --execute

  Indicate that you have read the documentation and want to alter the table. You must specify this option to alter the table. If you do not, then the tool will only perform some safety checks and exit. This helps ensure that you have read the documentation and understand how to use this tool. If you have not read the documentation, then do not specify this option.


- --help

  Show help and exit.


- --host

  short form: -h; type: stringConnect to host.


- --lock-wait-timeout

  type: int; default: 1Set the session value of innodb_lock_wait_timeout. This option helps guard against long lock waits if the data-copy queries become slow for some reason. Setting this option dynamically requires the InnoDB plugin, so this works only on newer InnoDB and MySQL versions. If the setting’s current value is greater than the specified value, and the tool cannot set the value as desired, then it prints a warning. If the tool cannot set the value but the current value is less than or equal to the desired value, there is no error.


- --max-lag

  type: time; default: 1sPause the data copy until all replicas’ lag is less than this value. After each data-copy query (each chunk), the tool looks at the replication lag of all replicas to which it connects, using Seconds_Behind_Master. If any replica is lagging more than the value of this option, then the tool will sleep for [*--check-interval*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--check-interval) seconds, then check all replicas again. If you specify [*--check-slave-lag*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--check-slave-lag), then the tool only examines that server for lag, not all servers. If you want to control exactly which servers the tool monitors, use the DSN value to [*--recursion-method*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--recursion-method).The tool waits forever for replicas to stop lagging. If any replica is stopped, the tool waits forever until the replica is started. The data copy continues when all replicas are running and not lagging too much.The tool prints progress reports while waiting. If a replica is stopped, it prints a progress report immediately, then again at every progress report interval.


- --max-load

  type: Array; default: Threads_running=25Examine SHOW GLOBAL STATUS after every chunk, and pause if any status variables are higher than their thresholds. The option accepts a comma-separated list of MySQL status variables. An optional =MAX_VALUE (or :MAX_VALUE) can follow each variable. If not given, the tool determines a threshold by examining the current value and increasing it by 20%.For example, if you want the tool to pause when Threads_connected gets too high, you can specify “Threads_connected”, and the tool will check the current value when it starts working and add 20% to that value. If the current value is 100, then the tool will pause when Threads_connected exceeds 120, and resume working when it is below 120 again. If you want to specify an explicit threshold, such as 110, you can use either “Threads_connected:110” or “Threads_connected=110”.The purpose of this option is to prevent the tool from adding too much load to the server. If the data-copy queries are intrusive, or if they cause lock waits, then other queries on the server will tend to block and queue. This will typically cause Threads_running to increase, and the tool can detect that by running SHOW GLOBAL STATUS immediately after each query finishes. If you specify a threshold for this variable, then you can instruct the tool to wait until queries are running normally again. This will not prevent queueing, however; it will only give the server a chance to recover from the queueing. If you notice queueing, it is best to decrease the chunk time.


- --password

  short form: -p; type: stringPassword to use when connecting.


- --pid

  type: stringCreate the given PID file. The file contains the process ID of the tool’s instance. The PID file is removed when the tool exits. The tool checks for the existence of the PID file when starting; if it exists and the process with the matching PID exists, the tool exits.


- --port

  short form: -P; type: intPort number to use for connection.


- --print

  Print SQL statements to STDOUT. Specifying this option allows you to see most of the statements that the tool executes. You can use this option with [*--dry-run*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--dry-run), for example.


- --progress

  type: array; default: time,30Print progress reports to STDERR while copying rows. The value is a comma-separated list with two parts. The first part can be percentage, time, or iterations; the second part specifies how often an update should be printed, in percentage, seconds, or number of iterations.


- --quiet

  short form: -qDo not print messages to STDOUT (disables [*--progress*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--progress)). Errors and warnings are still printed to STDERR.


- --recurse

  type: intNumber of levels to recurse in the hierarchy when discovering replicas. Default is infinite. See also [*--recursion-method*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--recursion-method).


- --recursion-method

  type: array; default: processlist,hostsPreferred recursion method for discovering replicas. Possible methods are:`METHOD       USES===========  ==================processlist  SHOW PROCESSLISThosts        SHOW SLAVE HOSTSdsn=DSN      DSNs from a tablenone         Do not find slaves`The processlist method is the default, because SHOW SLAVE HOSTS is not reliable. However, the hosts method can work better if the server uses a non-standard port (not 3306). The tool usually does the right thing and finds all replicas, but you may give a preferred method and it will be used first.The hosts method requires replicas to be configured with report_host, report_port, etc.The dsn method is special: it specifies a table from which other DSN strings are read. The specified DSN must specify a D and t, or a database-qualified t. The DSN table should have the following structure:`CREATE TABLE `dsns` (  `id` int(11) NOT NULL AUTO_INCREMENT,  `parent_id` int(11) DEFAULT NULL,  `dsn` varchar(255) NOT NULL,  PRIMARY KEY (`id`));`To make the tool monitor only the hosts 10.10.1.16 and 10.10.1.17 for replication lag, insert the values h=10.10.1.16and h=10.10.1.17 into the table. Currently, the DSNs are ordered by id, but id and parent_id are otherwise ignored.


- --retries

  type: int; default: 3Retry a chunk this many times when there is a nonfatal error. Nonfatal errors are problems such as a lock wait timeout or the query being killed. This option applies to the data copy operation.


- --set-vars

  type: string; default: wait_timeout=10000Set these MySQL variables. Immediately after connecting to MySQL, this string will be appended to SET and executed.


- --socket

  short form: -S; type: stringSocket file to use for connection.


- --statistics

  Print statistics about internal counters. This is useful to see how many warnings were suppressed compared to the number of INSERT.


- --[no]swap-tables

  default: yesSwap the original table and the new, altered table. This step completes the online schema change process by making the table with the new schema take the place of the original table. The original table becomes the “old table,” and the tool drops it unless you disable *--[no]drop-old-table*.


- --user

  short form: -u; type: stringUser for login if not current user.


- --version

  Show version and exit.


- --version-check

  type: string; default: offSend program versions to Percona and print suggested upgrades and problems. Possible values for –version-check:https, http, auto, offauto first tries using https, and resorts to http if that fails. Keep in mind that https might not be available ifIO::Socket::SSL is not installed on your system, although --version-check http should work everywhere.The version check feature causes the tool to send and receive data from Percona over the web. The data contains program versions from the local machine. Percona uses the data to focus development on the most widely used versions of programs, and to suggest to customers possible upgrades and known bad versions of programs.For more information, visit [http://www.percona.com/version-check](http://www.percona.com/version-check).

## DSN OPTIONS

These DSN options are used to create a DSN. Each option is given like option=value. The options are case-sensitive, so P and p are not the same option. There cannot be whitespace before or after the = and if the value contains whitespace it must be quoted. DSN options are comma-separated. See the percona-toolkit manpage for full details.

- A

> dsn: charset; copy: yes
>
> Default character set.

- D

> dsn: database; copy: yes
>
> Database for the old and new table.

- F

> dsn: mysql_read_default_file; copy: yes
>
> Only read default options from the given file

- h

> dsn: host; copy: yes
>
> Connect to host.

- p

> dsn: password; copy: yes
>
> Password to use when connecting.

- P

> dsn: port; copy: yes
>
> Port number to use for connection.

- S

> dsn: mysql_socket; copy: yes
>
> Socket file to use for connection.

- t

> dsn: table; copy: no
>
> Table to alter.

- u

> dsn: user; copy: yes
>
> User for login if not current user.

## ENVIRONMENT

The environment variable PTDEBUG enables verbose debugging output to STDERR. To enable debugging and capture all output to a file, run the tool like:

```
PTDEBUG=1 pt-online-schema-change ... > FILE 2>&1

```

Be careful: debugging output is voluminous and can generate several megabytes of output.

## SYSTEM REQUIREMENTS

You need Perl, DBI, DBD::mysql, and some core packages that ought to be installed in any reasonably new version of Perl.

This tool works only on MySQL 5.0.2 and newer versions, because earlier versions do not support triggers.

## BUGS

For a list of known bugs, see [http://www.percona.com/bugs/pt-online-schema-change](http://www.percona.com/bugs/pt-online-schema-change).

Please report bugs at [https://bugs.launchpad.net/percona-toolkit](https://bugs.launchpad.net/percona-toolkit). Include the following information in your bug report:

- Complete command-line used to run the tool
- Tool [*--version*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--version)
- MySQL version of all servers involved
- Output from the tool including STDERR
- Input files (log/dump/config files, etc.)

If possible, include debugging output by running the tool with PTDEBUG; see “ENVIRONMENT”.

## DOWNLOADING

Visit [http://www.percona.com/software/percona-toolkit/](http://www.percona.com/software/percona-toolkit/) to download the latest release of Percona Toolkit. Or, get the latest release from the command line:

```
wget percona.com/get/percona-toolkit.tar.gz

wget percona.com/get/percona-toolkit.rpm

wget percona.com/get/percona-toolkit.deb

```

You can also get individual tools from the latest release:

```
wget percona.com/get/TOOL

```

Replace TOOL with the name of any tool.

## AUTHORS

Daniel Nichter and Baron Schwartz

## ACKNOWLEDGMENTS

The “online schema change” concept was first implemented by Shlomi Noach in his tool oak-online-alter-table, part of [http://code.google.com/p/openarkkit/](http://code.google.com/p/openarkkit/). Engineers at Facebook then built another version calledOnlineSchemaChange.php as explained by their blog post: [http://tinyurl.com/32zeb86](http://tinyurl.com/32zeb86). This tool is a hybrid of both approaches, with additional features and functionality not present in either.

## ABOUT PERCONA TOOLKIT

This tool is part of Percona Toolkit, a collection of advanced command-line tools developed by Percona for MySQL support and consulting. Percona Toolkit was forked from two projects in June, 2011: Maatkit and Aspersa. Those projects were created by Baron Schwartz and developed primarily by him and Daniel Nichter, both of whom are employed by Percona. Visit [http://www.percona.com/software/](http://www.percona.com/software/) for more software developed by Percona.

## COPYRIGHT, LICENSE, AND WARRANTY

This program is copyright 2011-2013 Percona Ireland Ltd.

THIS PROGRAM IS PROVIDED “AS IS” AND WITHOUT ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, WITHOUT LIMITATION, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE.

This program is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, version 2; OR the Perl Artistic License. On UNIX and similar systems, you can issue `man perlgpl’ or `man perlartistic’ to read these licenses.

You should have received a copy of the GNU General Public License along with this program; if not, write to the Free Software Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA 02111-1307 USA.

## VERSION

**pt-online-schema-change** 2.1.10