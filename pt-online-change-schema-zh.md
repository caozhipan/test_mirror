# **pt-online-schema-change**

## NAME

**pt-online-schema-change** - 无锁表修改表schema

## SYNOPSIS

### Usage

```shell
pt-online-schema-change [OPTIONS] DSN
```

**pt-online-schema-change**  无阻塞修改表结构. 在DSN部分指定库名和表明， 使用本工具修改表之前要认真阅读该文档并做数据备份

在 sakila.actor 表中添加一个字段:

```shell
pt-online-schema-change --alter "ADD COLUMN c1 INT" D=sakila,t=actor
```

修改 sakila.actor 表的引擎 为 InnoDB, 修改表引擎为InnoDB后能够更有效的执行修改表操作

```
pt-online-schema-change --alter "ENGINE=InnoDB" D=sakila,t=actor
```

## 风险

用户需要知道该工具的潜在风险，主要有两类风险包括该工具自身特点所导致的和bugs所导致的

**pt-online-schema-change** 修改表数据和结构，你必须小心地使用它，生产环境使用前需要先测试，还有你必须确保使用该工具前已经对要修改的表做了备份。

截至到目前，我们没有发现能造成严重影响的bug。

官方更新信息会发布到在线问题追踪系统，影响该工具的问题会被列出来，你可以从这里看到问题列表URL: [http://www.percona.com/bugs/pt-online-schema-change](http://www.percona.com/bugs/pt-online-schema-change).

See also “BUGS” for more information on filing bugs and getting help.

## DESCRIPTION

**pt-online-schema-change** 模拟mysql修改表的方式，但它是在需要修改表的副本表上操作的，这就意味着原始表不会被锁定，客户端可以继续读写原始表

**pt-online-schema-change** 会先创建一个空的副本表，在此基础上进行修改表结构操作，然后从原始表拷贝数据到副本表，拷贝完成时，原始表会被副本表替代，默认原始表会被删除。

数据拷贝过程是以小数据块的形式进行的，这样可以让每次操作都在指定时间内完成，该过程类似其他工具如pt-table-checksum，任何基于原始表的修改操作都会更新的副本表，因为该工具在原始表创建了触发器来更新响应的记录到副本表，触发器的使用意味着该工具不能用在意境创建了触发器的表上

该工具完成数据拷贝工作后，会用一个rename table原子操作同时rename原始表和副本表，完成之后该工具会drop原始表

外键会使该工具的操作更加复杂，并且会带来额外的风险，由于表外键，原子rename操作技术将不能运行，该工具必须在副本表结构修改完毕后更新表外键，该工具提供两种方案实现这种需求，详情请 查看[*--alter-foreign-keys-method*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--alter-foreign-keys-method).文档

外键也会造成一些间接影响，最终表会和原始表拥有相同的外键和索引（除了指定修改的部分），但是对象名可能会为了比避免与在mysql和innodb中的对象名冲突而做一些轻微的改动。



安全起见，该工具不会执行修改操作除非指定—excute选项（默认是不可用状态），该工具提供很多方案来预防非必要的load和其他问题，包括自动检测样本，并连接，使用以下方式安全检测：

- 如果表中没有主键或者unique索引,否则该工具不会执行操作 See [*--alter*](https://www.percona.com/doc/percona-toolkit/2.2/pt-online-schema-change.html#cmdoption-pt-online-schema-change--alter) for details.
- 如果检测到有复制过滤器，该工具将不会执行操作， See *--[no]check-replication-filters* for details.

- 如果有复制延时超限，该工具会暂停执行 See *--[no]check-replication-filters* for details.
- 如果发现server负载过高，该工具会暂停甚至终止操作 See *--[no]check-replication-filters* for details.
- 该工具设置锁超市时间为1s，所以容易受到锁资源竞争的影响，但不会破坏事务操作See [*--lock-wait-timeout*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--lock-wait-timeout) for details.
- 除非指定[*--alter-foreign-keys-method*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--alter-foreign-keys-method). 选项，否则如果原始表有外键约束，该工具将不会修改标结构
- 改工具不能在“Percona XtraDB Cluster” 节点上修改MyISAM引擎的表

## Percona XtraDB Cluster

**pt-online-schema-change** 支持Percona XtraDB Cluster（(PXC) 5.5.28-23.7以后的版本）,但是有两个限制:只有InnoDB引擎的表才能被修改，并且wsrep_OSU_method必须被设置为TOI(total order isolation), 当在Percona XtraDB Cluster集群节点上修改MyISAM表或者引擎正在被修改为MyISAM的表，或者wsrep_OSU_method选项没有设置为TOI时会报错，该检查不能跳过

## OUTPUT

该工具会答应执行过程到标准输出以便查看，数据拷贝阶段，会输出—progress报告到标准错误输出，还可以通过指定 --print参数获取更多信息

如果指定[*--statistics*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--statistics) 参数，任务结束时内部各种事件数报表都会打印出来，例如：

```
# Event  Count
# ====== =====
# INSERT     1
```

## OPTIONS

[*--dry-run*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--dry-run) 和  [*--execute*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--execute) 参数互斥

该工具支持附件命令行参数，参考大纲和用法

---



## --alter

#### type:string #schema修改语句(去掉 ALTER TABLE 关键字)

#### 使用逗号分割多个修改语句

####  TABLE 语法参考 MYSQL手册

#### 一下限制会导致该工具意外失败，

* RENAME 语句不能rename该表表名

* 字段不能先drop 然后添加，

* 该工具不复制原始表中的数据到新表

* 如果添加了未指定默认值但是设置为NOT NULL的字段，该工具会执行失败，因为该工具不设置默认值

* DROP FOREIGN KEY constraint_name 必须指定 _<约束名> 而不是<约束名>,这是由于MYSQL自身的限制，pt-online-schema-change 会在新表上创建_<约束名>的外键约束 例如删除下面的约束：

  ```mysql
  CONSTRAINT fk_foo FOREIGN KEY (foo_id) REFERENCES bar (foo_id)
  ```

   你必须指定 

  ``` shell
  --alter "DROP FOREIGN KEY _fk_foo"
  ```

  ​

* 该工具在mysql 5.0版本上不使用共享锁,因为这会导致主从复制报错，如下

  ``` shell
  - Query caused different errors on master and slave. Error on master:

  'Deadlock found when trying to get lock; try restarting transaction' (1213),

  Error on slave: 'no error' (0). Default database: 'pt_osc'.

  Query: 'INSERT INTO pt_osc.t (id, c) VALUES ('730', 'new row')'
  ```

  ​
  当修改MyISAM引擎为InnoDB时会引发该错误，因为MyISAM引擎是非事务的，而InnoDB是事务的，MySQL5.1及更新的版本解决了这个问题，但是MySQL5.0上有5%的复现率

  ---

  ​

## --alter-foreign-keys-method

### type:string

如何修改外键，让其关联到新表上, 关联到原始表上的外键必须被特殊处理以确保其关联到正确的表上，当该工具替换原始表时该外键必须关联到新表上，

该工具提供两种技术来实现这个操作，自动发现关联到原始表的"child tables"

* auto模式

```
自动选择最佳方式，优先使用rebuild_constraints方法，如果失败，则使用drop_swap.
```

* rebuild_constraints

```shell
该方法 使用ALTER TABLE 先drop然后添加外键约束到新表，除非原始表有非常大的字表，ALTER操作时非常耗时，否则此方法为优先选择方法，
该工具通过对比子表的行数与需要拷贝行数的比率来确定是否适合采用该方法，如果该工具估算alter 子表的时间少于--chunk-time，将会选择该方法，
出于估算修改字表时间的目的，该工具会将row-coping比例和--chunk-size-limit相乘，因为MySQL ALTER TABLE操作比拷贝数据过程快得多
由于MySQL自身的限制，外键不能重名。该工具必须在重新定义外键的时候重命名(通过在外键名加下划线(_)前缀) 个别情况下MySQL还会自动重命名外键索引
```

* drop_swap

```
禁用外键检查(FOREIGN_KEY_CHECKS=0) 然后在rename新表时删除掉原始表，这不同于通常替换新旧表的方法(RENAME 原子操作，应用层无感知)
该方法虽然很快并且无阻塞，但是仍然存在风险，原因如下
	* 首先在删除原始表盒重命名临时表之间的短暂时间内，该表是不存在的，此时该表上的query会报错
	* 如过重命名临时表时报错，此时终止就太晚了，因为原始表已被彻底删除
该方法强制添加--no-swap-tables 和 --no-drop-old-table.
```

  ```

  * none
  ```
  该方法类似 drop_swap 但是没有"swap"操作,所有关联到原始表的外键都会被关联到一个不存在的表上，这会导致外键不合法，详情可在SHOW ENGINE INNODB STATUS中看到
  Trying to add to index `idx_fk_staff_id` tuple:
  DATA TUPLE: 2 fields;
  0: len 1; hex 05; asc  ;;
  1: len 4; hex 80000001; asc     ;;
  But the parent table `sakila`.`staff_old`
  or its .ibd file does not currently exist!
  这是因为原始表(sakila.staff)已经被重命名为sakila.staff_old 然后被drop掉了.
  这种处理外键的方法可以使数据库管理员在必要的情况下禁用该工具的内建方法

---

### --[no]analyze-before-swap

默认: yes

  替换原始表之前在新表上执行ANALYZE TABLE，默认情况下，只有当该工具运行在MySQL5.6及更新的版本上，并且innodb_stats_persistent配置可用时才会出现这种情况，明确指定该选项可用或者不可用无论MySQL的版本和innodb_stats_persistent配置。

  这会对InnoDB优化器造成潜在的严重影响， 如果表正在被修改，并且该工具很快就结束了，新表替换后不会有优化器统计数据，这会造成原本命中索引的query扫描全表，直到优化器数据更新(一般有10s延时)，但如果表很大，并且服务繁忙，就会造成服务终止。

---

### -ask-pass

连接MySQL时提示密码.

---

--charset

短类型标示: -A

type: string

默认字符集，如果值为utf8，设置标准输出Perl的字节处理为utf8,通过DBD::mysql对象的DBD::mysql选项，在连接MySQL后运行 SET NAMES UTF8。设置非utf8字符集时，连接mysql后运行 SET NAMES

---



### --check-interval

type: time; default: 1Sleep time between checks for [*--max-lag*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--max-lag).

类型：time

默认值：1

max-lag时间内check的时间间隔

---

### --[no]check-alter

default: yes

Parses the [*--alter*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--alter) specified and tries to warn of possible unintended behavior. Currently, it checks for:Column renamesIn previous versions of the tool, renaming a column with CHANGE COLUMN name new_namewould lead to that column’s data being lost. The tool now parses the alter statement and tries to catch these cases, so the renamed columns should have the same data as the originals. However, the code that does this is not a full-blown SQL parser, so you should first run the tool with [*--dry-run*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--dry-run) and [*--print*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--print) and verify that it detects the renamed columns correctly.DROP PRIMARY KEYIf [*--alter*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--alter) contain DROP PRIMARY KEY (case- and space-insensitive), a warning is printed and the tool exits unless [*--dry-run*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--dry-run) is specified. Altering the primary key can be dangerous, but the tool can handle it. The tool’s triggers, particularly the DELETE trigger, are most affected by altering the primary key because the tool prefers to use the primary key for its triggers. You should first run the tool with [*--dry-run*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--dry-run) and [*--print*](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--print) and verify that the triggers are correct.

---



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



