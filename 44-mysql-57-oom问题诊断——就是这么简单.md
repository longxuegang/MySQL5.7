# MySQL 5.7 OOM问题诊断——就是这么简单
[原文地址](http://www.innomysql.com/article/25635.html)

Inside君最近把金庸先生的笑傲江湖重看了三遍，感慨良多。很多工作、管理、生活、学习上的问题都能在其中一窥究竟，而那是年轻时所不能体会的一种感悟。比如下面风清扬的这段话：

> 风清扬又道：“单以武学而论，这些魔教长老们也不能说真正已窥上乘武学之门。他们不懂得，招数是死的，发招之人却是活的。死招数破得再妙，遇上了活招数，免不了缚手缚脚，只有任人屠戮。这个‘活’字，你要牢牢记住了。学招时要活学，使招时要活使。倘若拘泥不化，便练熟了几千万手绝招，遇上了真正高手，终究还是给人家破得干干净净。”

今天，来谈谈MySQL的OOM（out of memory）问题诊断。之前，这类问题的定位对于普通用户来说并不怎么简单。但是在MySQL 5.7中，OOM问题的定位变得极其容易。还没掌握的小伙伴赶快来看下吧。通常来说，发生OOM时可在系统日志找到类似的日志提示：

[![](http://www.innomysql.com/wp-content/uploads/2016/09/123-1.png)](http://www.innomysql.com/wp-content/uploads/2016/09/123-1.png)MySQL 5.7的库performance\_schema新增了以下这几张表，用于从各维度查看内存的消耗：

* memory\_summary\_by\_account\_by\_event\_name
* memory\_summary\_by\_host\_by\_event\_name
* memory\_summary\_by\_thread\_by\_event\_name
* memory\_summary\_by\_user\_by\_event\_name
* memory\_summary\_global\_by\_event\_name

简单来说，就是可以根据用户、主机、线程、账号、全局的维度对内存进行监控。同时库sys也就这些表做了进一步的格式化，可以使得用户非常容易的观察到每个对象的内存开销：

```
mysql> select event_name,current_alloc 
-> from memory_global_by_current_bytes limit 10;
+------------------------------------------------------------------------------+---------------+
| event_name                                                                   | current_alloc |
+------------------------------------------------------------------------------+---------------+
| memory/performance_schema/events_statements_history_long                     | 13.66 MiB     |
| memory/performance_schema/events_statements_history_long.sqltext             | 9.77 MiB      |
| memory/performance_schema/events_statements_history_long.tokens              | 9.77 MiB      |
| memory/performance_schema/events_statements_summary_by_digest.tokens         | 9.77 MiB      |
| memory/performance_schema/table_handles                                      | 9.00 MiB      |
| memory/performance_schema/events_statements_summary_by_thread_by_event_name  | 8.80 MiB      |
| memory/performance_schema/memory_summary_by_thread_by_event_name             | 5.62 MiB      |
| memory/performance_schema/events_statements_summary_by_digest                | 4.88 MiB      |
| memory/performance_schema/events_statements_summary_by_user_by_event_name    | 4.40 MiB      |
| memory/performance_schema/events_statements_summary_by_account_by_event_name | 4.40 MiB      |
+------------------------------------------------------------------------------+---------------+
10 rows in set (0.00 sec)
```

细心的同学可能会发现，默认情况下performance\_schema只对performance\_schema进行了内存开销的统计。但是在对OOM进行诊断时，需要对所有可能的对象进行内存监控。因此，还需要做下面的设置：

```
mysql> update performance_schema.setup_instruments 
-> set enabled = 'yes' where name like 'memory%';
Query OK, 310 rows affected (0.00 sec)
Rows matched: 380  Changed: 310  Warnings: 0

mysql> select * from performance_schema.setup_instruments where name like 'memory%innodb%' limit 5;
+-------------------------------------------+---------+-------+
| NAME                                      | ENABLED | TIMED |
+-------------------------------------------+---------+-------+
| memory/innodb/adaptive hash index         | YES     | NO    |
| memory/innodb/buf_buf_pool                | YES     | NO    |
| memory/innodb/dict_stats_bg_recalc_pool_t | YES     | NO    |
| memory/innodb/dict_stats_index_map_t      | YES     | NO    |
| memory/innodb/dict_stats_n_diff_on_level  | YES     | NO    |
+-------------------------------------------+---------+-------+
5 rows in set (0.00 sec)
```

但是这种在线打开内存统计的方法仅对之后新增的内存对象有效：

```
mysql> select event_name,current_alloc from memory_global_by_current_bytes 
    -> where event_name like '%innodb%';
+------------------------+---------------+
| event_name             | current_alloc |
+------------------------+---------------+
| memory/innodb/mem0mem  | 36.52 KiB     |
| memory/innodb/trx0undo | 704 bytes     |
| memory/innodb/btr0pcur | 271 bytes     |
+------------------------+---------------+
3 rows in set (0.01 sec)
```

如想要对全局生命周期中的对象进行内存统计，必须在配置文件中进行设置，然后重启：

```
[mysqld]
performance-schema-instrument='memory/%=COUNTED'

mysql> select event_name,current_alloc from memory_global_by_current_bytes limit 5;
+----------------------------+---------------+
| event_name                 | current_alloc |
+----------------------------+---------------+
| memory/innodb/os0file      | 1.42 GiB      |
| memory/innodb/buf_buf_pool | 1.05 GiB      |
| memory/innodb/os0event     | 51.15 MiB     |
| memory/innodb/hash0hash    | 41.44 MiB     |
| memory/innodb/log0log      | 32.01 MiB     |
+----------------------------+---------------+
5 rows in set (0.00 sec)
```

通过上面的结果，有小伙伴是不是已经发现**可疑的内存使用**了呢？memory/innodb/os0file这个对象使用了1.42G内存，而整个数据库实例的Buffer Pool只有1.05G。那么这时就可以去bugs.mysql.com上去搜索下。果不其然，是一个官方bug，并已在5.7.14修复。而通过类似方法Inside君已经定位了5起OOM问题。当然，这里Inside君只是抛出了一个思路，活学活用，才能达到无招胜有招的至臻境界。

