[https://www.kancloud.cn/taobaomysql/monthly/213789](https://www.kancloud.cn/taobaomysql/monthly/213789)

### 1. 背景 {#1-e46e-}

MySQL的masterslave的部署结构，使用binlog日志保持数据的同步，全局有序的binlog在备库按照提交顺序进行回放。  
由于新硬件的发展，SSD的引入和多core的CPU，master节点的并发处理能力持续提升，slave节点完全按照binlog写入顺序的单线程回放，已完全跟不上master节点的吞吐能力，导致HA切换和数据保护带来巨大的挑战。

### 2. 并行复制的演进 {#2-e46e-}

从MySQL5.5版本以后，开始引入并行复制的机制，是MySQL的一个非常重要的特性。

MySQL5.6开始支持以schema为维度的并行复制，即如果binlog row event操作的是不同的schema的对象，在确定没有DDL和foreign key依赖的情况下，就可以实现并行复制。

社区也有引入以表为维度或者以记录为维度的并行复制的版本，不管是schema，table或者record，都是建立在备库slave实时解析row格式的event进行判断，保证没有冲突的情况下，进行分发来实现并行。

MySQL5.7的并行复制，multi-threaded slave即MTS，期望最大化还原主库的并行度，实现方式是在binlog event中增加必要的信息，以便slave节点根据这些信息实现并行复制。

下面我们就来看下MySQL 5.7的实现方式：

### 3. MySQL 5.7 并行复制 {#3-e46e-mysql-5-7-}

MySQL 5.7的并行复制建立在group commit的基础上，所有在主库上能够完成prepared的语句表示没有数据冲突，就可以在slave节点并行复制。   
我们先来回顾一下group commit的情况：

```
1\. group commit的过程：
    1. binlog prepare
    2. InnoDB prepare
    3. binlog commit(ordered commit)
        --3.1 Stage #1: flushing transactions to binary log
        --3.2 Stage #2: Syncing binary log file to disk
        --3.3 Stage #3: Commit all transactions in order.
    4. InnoDB commit
```

在ordered commit的过程中:  
1. 由leader线程帮助FLUSH队列中的线程完成flush binlog操作，  
2. 由leader线程帮助SYNC队列中的线程完成sync binlog操作，

为了表示主库并行度，在binlog row event增加了如下的标识：

```
#160807 15:48:10 server id 100  end_log_pos 739 CRC32 0x2237b2ef        GTID    last_committed=0        sequence_number=3
SET @@SESSION.GTID_NEXT= '8108dc48-47de-11e6-8690-a0d3c1f20ae4:3'/*!*/;
```

即在gtid\_event中增加两个字段：

```
class Gtid_event: public Binary_log_event
{
public:
  /*
    The transaction's logical timestamps used for MTS: see
    Transaction_ctx::last_committed and
    Transaction_ctx::sequence_number for details.
    Note: Transaction_ctx is in the MySQL server code.
  */
  long long int last_committed;
  long long int sequence_number;
  /**
    Ctor of Gtid_event

    The layout of the buffer is as follows
    +-------------+-------------+------------+---------+----------------+
    | commit flag | ENCODED SID | ENCODED GNO| TS_TYPE | logical ts(:s) |
    +-------------+-------------+------------+---------+----------------+
    TS_TYPE is from {G_COMMIT_TS2} singleton set of values

```

代码中为每一个transaction准备了如下的字段：

```
class Transaction_ctx
{
    ......
    
    int64
    last_committed;
    
     int64
     sequence_number;
}
```

MYSQL\_BIN\_LOG全局对象中维护了两个结构：

```
class MYSQL_BIN_LOG: public TC_LOG
{
  ......
  /* Committed transactions timestamp */
   Logical_clock max_committed_transaction;
  /* "Prepared" transactions timestamp */
   Logical_clock transaction_counter;
}
```

事务中的sequence\_number是一个全局有序递增的数字，每个事务递增1，来源mysql\_bin\_log.tranaction\_counter.   
和gtid一对一的关系，即在flush阶段，和gtid生成的时机一致，代码参考：

```
binlog_cache_data::flush
{   
     if (flags.finalized) {
       Transaction_ctx *trn_ctx= thd->get_transaction();
       trn_ctx->sequence_number= mysql_bin_log.transaction_counter.step();
     }
     .......
     mysql_bin_log.write_gtid(thd, this, &writer)))
     mysql_bin_log.write_cache(thd, this, &writer);
}
```

事务中last\_committed表示在这个commit下的事务，都是可以并行的，即没有冲突，  
Transaction\_ctx中的last\_committed在每个语句prepared的时候进行初始化，来源mysql\_bin\_log.max\_committed\_transaction

```
static int binlog_prepare(handlerton *hton, THD *thd, bool all)
{
    ......
    Logical_clock& clock= mysql_bin_log.max_committed_transaction;
    thd->get_transaction()->
      store_commit_parent(clock.get_timestamp());
}
```

而mysql\_bin\_log.max\_committed\_transaction的更新是在group commit提交的时候进行变更。

```
MYSQL_BIN_LOG::process_commit_stage_queue(THD *thd, THD *first)
{
    ......
    for (THD *head= first ; head ; head = head->next_to_commit)
    {
        if (thd->get_transaction()->sequence_number != SEQ_UNINIT)
            update_max_committed(head);
    }
}
```

即获取这个group commit队列中的最大的sequence\_number当成当前的max\_committed\_transaction。

所以，这个机制可以理解成，在group commit完成之前，所有可以成功prepared的语句，没有事实上的冲突，  
分配成相同的last\_committed，就可以在slave节点并行复制。

例如下面时序的事务：

```
session 1：insert into t1 value(100, 'xpchild');
session 2：insert into t1 value(101, 'xpchild');
session 2：commit
session 1：commit
```

Binlog日志片段如下：

    # at 1398
    #160807 15:38:14 server id 100  end_log_pos 1463 CRC32 0xd6141f71       GTID    last_committed=5        sequence_number=6
    SET @@SESSION.GTID_NEXT= '8108dc48-47de-11e6-8690-a0d3c1f20ae4:6'/*!*/;
    '/*!*/;
    ### INSERT INTO `tp`.`t1`
    ### SET
    ###   @1=101 /* INT meta=0 nullable=0 is_null=0 */
    ###   @2='xpchild' /* VARSTRING(100) meta=100 nullable=1 is_null=0 */

    COMMIT/*!*/;
    # at 1658
    #160807 15:38:24 server id 100  end_log_pos 1723 CRC32 0xa24923a8       GTID    last_committed=5        sequence_number=7
    SET @@SESSION.GTID_NEXT= '8108dc48-47de-11e6-8690-a0d3c1f20ae4:7'/*!*/;
    ### INSERT INTO `tp`.`t1`
    ### SET
    ###   @1=100 /* INT meta=0 nullable=0 is_null=0 */
    ###   @2='xpchild' /* VARSTRING(100) meta=100 nullable=1 is_null=0 */

两个insert语句在prepared的时候，没有事实上的冲突，都获取当前最大的committed number = 5.  
提交的过程中，保持sequence\_number生成时候的全局有序性，备库恢复的时候，这两个事务就可以并行完成。

但又如下面的case：

```
session 1: insert into t1 value(100, 'xpchild');

session 2: insert into t1 value(101, 'xpchild');
session 2: commit;

session 3: insert into t1 value(102, 'xpchild');
session 3: commit;

session 1: commit;
```

产生如下的顺序：

    #160807 15:47:58 server id 100  end_log_pos 219 CRC32 0x3f295e2b        GTID    last_committed=0        sequence_number=1
    ### INSERT INTO `tp`.`t1`
    ### SET
    ###   @1=101 /* INT meta=0 nullable=0 is_null=0 */
    .....
    #160807 15:48:05 server id 100  end_log_pos 479 CRC32 0xda52bed0        GTID    last_committed=1        sequence_number=2
    ### INSERT INTO `tp`.`t1`
    ### SET
    ###   @1=102 /* INT meta=0 nullable=0 is_null=0 */
    ......
    #160807 15:48:10 server id 100  end_log_pos 739 CRC32 0x2237b2ef        GTID    last_committed=0        sequence_number=3
    ### INSERT INTO `tp`.`t1`
    ### SET
    ###   @1=100 /* INT meta=0 nullable=0 is_null=0 */
    ....

session 1和session 2语句执行不冲突，分配了相同的last\_committed，   
session 2提交，推高了last\_committed，所以session 3的laste\_committed变成了1，   
最后session 1提交。

注意： 这就是MySQL 5.7.3之后的改进：

在MySQL 5.7.3之前，必须在一个group commit之内的事务，才能够在slave节点并行复制，但如上面的这个case。

session 1 和session 2虽然commit的时间有差，并且不在一个group commit，生成的binlog也没有连续，但事实上是可以并行恢复执行的。

所以从MySQL 5.7.3之后，并行恢复，减少了group commit的依赖，但group commit仍然对并行恢复起着非常大的作用。

### 4. MySQL 5.7 并行复制参数 {#4-e46e-mysql-5-7-}

MySQL 5.7增加了如下参数：

```
mysql> show global variables like '%slave_parallel_type%';
+---------------------+---------------+
| Variable_name       | Value         |
+---------------------+---------------+
| slave_parallel_type | LOGICAL_CLOCK |
+---------------------+---------------+
1 row in set (0.00 sec)
```

slave\_parallel\_type取值：  
1. DATABASE： 默认值，兼容5.6以schema维度的并行复制  
2. LOGICAL\_CLOCK： MySQL 5.7基于组提交的并行复制机制

### 5. MySQL 5.7 并行复制影响因素 {#5-e46e-mysql-5-7-}

group commit delay

首先，并行复制必须建立在主库的真实负载是并行的基础上，才能使MTS有机会在slave节点上完成并行复制，  
其次，MySQL 5.7前面讨论的实现机制，可以人工的增加group commit的delay，打包更多的事务在一起，提升slave复制的并行度。但从5.7.3开始，已经减少了group commit的依赖，  
尽量减少delay参数设置对主库的影响。

合理设置如下参数；

```
mysql> show global variables like '%group_commit%';
+-----------------------------------------+--------+
| Variable_name                           | Value  |
+-----------------------------------------+--------+
| binlog_group_commit_sync_delay          | 100000 |
| binlog_group_commit_sync_no_delay_count | 0      |
+-----------------------------------------+--------+
```

### 6. 一点建议 {#6-e46e-}

1. 尽量使用row格式的binlog
2. slave\_parallel\_workers 太多的线程会增加线程间同步的开销，建议4-8个slave线程，根据测试来最终确定。
3. 如果客户端有并行度，不用刻意增加master的group commit，减少对主库的影响。

另外：   
booking在使用的时候遇到的如下case：

数据库的部署结构是：master-&gt;slave1-&gt;slave2

假设，当t1,t2,t3,t4四个事务在master group commit中，那么slave1线程就可以并行执行这四个事务，  
但在slave1执行的过程中，分成了两个group commit，那么在slave2节点上，并行度就降低了一倍。

booking给出的后续的解法，如果slave不多，建议都挂载在master上，如果slave过多，考虑使用binlog server，来避免这样的问题。

但其实在slave1节点上进行并行恢复的时候，保持着主库的last\_committed和sequence\_number不变，虽然无法保证binlog写入的顺序完全和主库一致，但可以缓解这种情况。

