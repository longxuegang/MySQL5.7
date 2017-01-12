##  5.7新增rename index的online功能

5.7新增加online rename index， 仅仅通过修改元信息就可以完成。

## 5.7新增通过performance schema来查看改表的进度

[官方文档连接](https://dev.mysql.com/doc/refman/5.7/en/monitor-alter-table-performance-schema.html)

一、打开功能

```
mysql> UPDATE setup_instruments SET ENABLED = 'YES' WHERE NAME LIKE 'stage/innodb/alter%';
Query OK, 7 rows affected (0.00 sec)
Rows matched: 7  Changed: 7  Warnings: 0

mysql> UPDATE setup_consumers SET ENABLED = 'YES' WHERE NAME LIKE '%stages%';
Query OK, 3 rows affected (0.00 sec)
Rows matched: 3  Changed: 3  Warnings: 0
```



二、执行改表操作

```
mysql> ALTER TABLE employees.employees ADD COLUMN middle_name varchar(14) AFTER first_name;
Query OK, 0 rows affected (9.27 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

三、查看改表进度

```
mysql> SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED FROM events_stages_current;
+------------------------------------------------------+----------------+----------------+
| EVENT_NAME                                           | WORK_COMPLETED | WORK_ESTIMATED |
+------------------------------------------------------+----------------+----------------+
| stage/innodb/alter table (read PK and internal sort) |            280 |           1245 |
+------------------------------------------------------+----------------+----------------+
1 row in set (0.01 sec)
```

### 腾讯GSC引擎

腾讯游戏通过修改compact引擎实现的GCS引擎，实现在线加字段。具体原理如下：

1. 在每一个行记录头中增加Field Count计数；增加一个系统表SYS\_ADDED\_COLS\_DEFAULT存储字段的default值。

2. 增加列的时候，仅修改Innodb元信息，修改完元信息之后，新的数据直接按照新的结构来存储；

3. 出现了一个问题，老结构数据和新结构数据并存，select、insert、update、delete需要支持“混合存储”；

   * select读取时如果发现Field Count计数小于当前表结构字段数，则其他的字段以NULL或者DEFAULT值填充；

   * insert直接按照当前的表结构来构造；

   * update原地更新不变；非原地更新走delete+insert，会更新为新的field count；

   * delete不变；

限制：

1. 表必须是innodb的GCS表，原Compact表不支持在线加字段功能；

2. 不支持临时表；

3. 一次alter table仅允许加一列或多列，但不允许同时进行多个alter table的不同操作（如增删索引、删字段、修改字段等）；

4. 加字段不支持指定Before或After关键字表示定义在某列之前或之后；

5. 所加字段不能包含除not null外的任何约束，包括外键约束、唯一约束；

6. 不支持允许为NULL并指定默认值的加字段操作（同oracle 11g\)；

7. 所加字段不能自增列\(auto\_increment\)；

详情可以查看[链接](http://got.qq.com/webplat/info/news_version3/8616/8622/8625/8628/m7025/201407/271174.shtml)。



## MySQL Online DDL

里面有4个操作是只需要修改元信息即可，包括rename index、drop index、设置default值、设置表级别的[persistent statistics](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_persistent_statistics)；这4个操作真正的做到了online。

add index、add column这些操作目前还得依赖pt-online-schema-change或者gh-ost之类的工具，因为虽然这些操作已经做到了in-place，比之前的copy table的方式要快很多，不阻塞DML；如果直接改表还是会造成Slave延迟，见[bug73196](https://bugs.mysql.com/bug.php?id=73196)。

[一篇对比online ddl和osc工具文章](http://www.fromdual.com/online-ddl_vs_pt-online-schema-change)，有兴趣可以阅读。



| Operation | In-Place? | Copies Table? | Allows Concurrent DML? | Allows Concurrent Query? | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| [`CREATE INDEX`](http://dev.mysql.com/doc/refman/5.7/en/create-index.html),[`ADD INDEX`](http://dev.mysql.com/doc/refman/5.7/en/alter-table.html) | Yes\* | No\* | Yes | Yes | Some restrictions for`FULLTEXT`index; see next row. |
| [`ADD FULLTEXT INDEX`](http://dev.mysql.com/doc/refman/5.7/en/alter-table.html) | Yes | No\* | No | Yes | Creating the first`FULLTEXT`index for a table involves a table copy, unless there is a user-supplied`FTS_DOC_ID`column. Subsequent`FULLTEXT`indexes on the same table can be created in-place. |
| [`ADD SPATIAL INDEX`](http://dev.mysql.com/doc/refman/5.7/en/alter-table.html) | Yes | No | No | Yes | In-place support was added in MySQL 5.7.5. Bulk load is not supported. |
| [`RENAME INDEX`](http://dev.mysql.com/doc/refman/5.7/en/alter-table.html) | Yes | No | Yes | Yes | Only modifies table metadata. |
| [`DROP INDEX`](http://dev.mysql.com/doc/refman/5.7/en/drop-index.html) | Yes | No | Yes | Yes | Only modifies table metadata. |
| [`OPTIMIZE TABLE`](http://dev.mysql.com/doc/refman/5.7/en/optimize-table.html) | Yes | Yes | Yes | Yes | Uses`ALGORITHM=INPLACE`as of MySQL 5.7.4.`ALGORITHM=COPY`is used if`old_alter_table=1`or[**mysqld**](http://dev.mysql.com/doc/refman/5.7/en/mysqld.html)`--skip-new`option is enabled.`OPTIMIZE TABLE`using online DDL \(`ALGORITHM=INPLACE`\) is not supported for tables with FULLTEXT indexes. |
| Set default value for a column | Yes | No | Yes | Yes | Only modifies table metadata. |
| Change[auto-increment](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_auto_increment)value for a column | Yes | No | Yes | Yes | Modifies a value stored in memory, not the data file. |
| Add a[foreign key constraint](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_foreign_key_constraint) | Yes\* | No\* | Yes | Yes | To avoid copying the table, disable[`foreign_key_checks`](http://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_foreign_key_checks)during constraint creation. |
| Drop a[foreign key constraint](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_foreign_key_constraint) | Yes | No | Yes | Yes | The[`foreign_key_checks`](http://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_foreign_key_checks)option can be enabled or disabled. |
| Rename a column | Yes\* | No\* | Yes\* | Yes | To allow concurrent DML, keep the same data type and only change the column name. Prior to MySQL 5.7.8,`ALGORITHM=INPLACE`is supported for renaming a[generated virtual column](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_generated_virtual_column)but not for renaming a[generated stored column](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_generated_virtual_column). As of MySQL 5.7.8,`ALGORITHM=INPLACE`is not supported for renaming a[generated column](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_generated_column). |
| Add a column | Yes\* | Yes\* | Yes\* | Yes | Concurrent DML is not allowed when adding an[auto-increment](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_auto_increment)column. Although`ALGORITHM=INPLACE`is allowed, the data is reorganized substantially, so it is still an expensive operation.`ALGORITHM=INPLACE`is supported for adding a[generated virtual column](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_generated_virtual_column)but not for adding a[generated stored column](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_generated_virtual_column). Adding a generated virtual column does not require a table copy. |
| Drop a column | Yes | Yes\* | Yes | Yes | Although`ALGORITHM=INPLACE`is allowed, the data is reorganized substantially, so it is still an expensive operation.`ALGORITHM=INPLACE`is supported for dropping a generated column. Dropping a[generated virtual column](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_generated_virtual_column)does not require a table copy. |
| Reorder columns | Yes | Yes | Yes | Yes | Although`ALGORITHM=INPLACE`is allowed, the data is reorganized substantially, so it is still an expensive operation. |
| Change`ROW_FORMAT`property | Yes | Yes | Yes | Yes | Although`ALGORITHM=INPLACE`is allowed, the data is reorganized substantially, so it is still an expensive operation. |
| Change`KEY_BLOCK_SIZE`property | Yes | Yes | Yes | Yes | Although`ALGORITHM=INPLACE`is allowed, the data is reorganized substantially, so it is still an expensive operation. |
| Make column`NULL` | Yes | Yes | Yes | Yes | Although`ALGORITHM=INPLACE`is allowed, the data is reorganized substantially, so it is still an expensive operation. |
| Make column`NOT NULL` | Yes\* | Yes | Yes | Yes | `STRICT_ALL_TABLES`or`STRICT_TRANS_TABLES`[`SQL_MODE`](http://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_sql_mode)is required for the operation to succeed. The operation fails if the column contains NULL values. The server prohibits changes to foreign key columns that have the potential to cause loss of referential integrity. For more information, see[Section14.1.8, “ALTER TABLE Syntax”](http://dev.mysql.com/doc/refman/5.7/en/alter-table.html). Although`ALGORITHM=INPLACE`is allowed, the data is reorganized substantially, so it is still an expensive operation. |
| Change data type of column | No\* | Yes\* | No | Yes | Exception:[`VARCHAR`](http://dev.mysql.com/doc/refman/5.7/en/char.html)size may be increased using online[`ALTER TABLE`](http://dev.mysql.com/doc/refman/5.7/en/alter-table.html). See[InnoDB Online DDL Column Properties](http://dev.mysql.com/doc/refman/5.7/en/innodb-create-index-overview.html#innodb-online-ddl-column-properties)for more information. |
| Add[primary key](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_primary_key) | Yes\* | Yes | Yes | Yes | Although`ALGORITHM=INPLACE`is allowed, the data is reorganized substantially, so it is still an expensive operation.`ALGORITHM=INPLACE`is not allowed under certain conditions if columns have to be converted to`NOT NULL`. See[Example15.9, “Creating and Dropping the Primary Key”](http://dev.mysql.com/doc/refman/5.7/en/innodb-create-index-examples.html#online-ddl-ex-primary-key). |
| Drop[primary key](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_primary_key)and add another | Yes | Yes | Yes | Yes | `ALGORITHM=INPLACE`is only allowed when you add a new primary key in the same[`ALTER TABLE`](http://dev.mysql.com/doc/refman/5.7/en/alter-table.html); the data is reorganized substantially, so it is still an expensive operation. |
| Drop[primary key](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_primary_key) | No | Yes | No | Yes | Restrictions apply when you drop a primary key without adding a new one in the same`ALTER TABLE`statement. |
| Convert character set | No | Yes | No | Yes | Rebuilds the table if the new character encoding is different. |
| Specify character set | No | Yes | No | Yes | Rebuilds the table if the new character encoding is different. |
| Rebuild with`FORCE`option | Yes | Yes | Yes | Yes | Uses`ALGORITHM=INPLACE`as of MySQL 5.7.4.`ALGORITHM=COPY`is used if`old_alter_table=1`or[**mysqld**](http://dev.mysql.com/doc/refman/5.7/en/mysqld.html)`--skip-new`option is enabled. Table rebuild using online DDL \(`ALGORITHM=INPLACE`\) is not supported for tables with FULLTEXT indexes. |
| Rebuild with“null”`ALTER TABLE ... ENGINE=INNODB` | Yes | Yes | Yes | Yes | Uses`ALGORITHM=INPLACE`as of MySQL 5.7.4.`ALGORITHM=COPY`is used if`old_alter_table=1`or[**mysqld**](http://dev.mysql.com/doc/refman/5.7/en/mysqld.html)`--skip-new`option is enabled. Table rebuild using online DDL \(`ALGORITHM=INPLACE`\) is not supported for tables with FULLTEXT indexes. |
| Set table-level[persistent statistics](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_persistent_statistics)options \(`STATS_PERSISTENT`,```STATS_AUTO_RECALC``STATS_SAMPLE_PAGES```\) | Yes | No | Yes | Yes | Only modifies table metadata. |



