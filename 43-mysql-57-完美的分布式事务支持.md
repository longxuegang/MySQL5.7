### MySQL 5.7 完美的分布式事务支持  
######[原文地址](http://www.innomysql.com/article/25314.html) 

#### Two Phase Commit Protocol

分布式事务通常采用2PC协议，全称Two Phase Commitment Protocol。该协议主要为了解决在分布式数据库场景下，所有节点间数据一致性的问题。在分布式事务环境下，事务的提交会变得相对比较复杂，因为多个节点的存在，可能存在部分节点提交失败的情况，即事务的ACID特性需要在各个数据库实例中保证。总而言之，在分布式提交时，只要发生一个节点提交失败，则所有的节点都不能提交，只有当所有节点都能提交时，整个分布式事务才允许被提交。

分布式事务通过2PC协议将提交分成两个阶段

1. prepare；
2. commit/rollback
   第一阶段的prepare只是用来询问每个节点事务是否能提交，只有当得到所有节点的“许可”的情况下，第二阶段的commit才能进行，否则就rollback。需要注意的是：prepare成功的事务，则必须全部提交。

#### MySQL分布式事务

一直以来，MySQL数据库是支持分布式事务的，但是只能说是有限的支持，具体表现在：

* 已经prepare的事务，在客户端退出或者服务宕机的时候，2PC的事务会被回滚
* 在服务器故障重启提交后，相应的Binlog被丢失

上述问题存在于MySQL数据库长达数十年的时间，直到MySQL-5.7.7版本，官方才修复了该问题。虽然InnoSQL早已在5.5版本修复，但是对比官方的修复方案，我们真的做的没有那么的优雅。下面将会详细介绍下该问题的具体表现和官方修复方法，这里分别采用官方MySQL-5.6.27版本\(未修复\)和MySQL-5.7.9版本\(已修复\)进行验证。

先来看下存在的问题，我们先创建一个表如下：

create table t\(  
id int auto\_increment primary key,  
a int  
\)engine=innodb;

对于上述表，通过如下操作进行数据插入：  
mysql&gt; XA START 'mysql56';  
mysql&gt; INSERT INTO t VALUES\(1,1\);  
mysql&gt; XA END 'mysql56';  
mysql&gt; XA PREPARE 'mysql56'  
通过上面的操作，用户创建了一个分布式事务，并且prepare没有返回错误，说明该分布式事务可以被提交。通过命令XA RECOVER查看显示如下结果：  
mysql&gt; XA RECOVER;  
+----------+--------------+--------------+---------+  
\| formatID \| gtrid\_length \| bqual\_length \| data    \|  
+----------+--------------+--------------+---------+  
\| 1        \| 7            \| 0            \| mysql56 \|  
+—————+--------------+--------------+---------+  
若这时候用户退出客户端后重连，通过命令xa recover会发现刚才创建的2PC事务不见了

###### 即prepare成功的事务丢失了，不符合2PC协议规范！！！

产生上述问题的主要原因在于：MySQL-5.6版本在客户端退出的时候，自动把已经prepare的事务回滚了，那么MySQL为什么要这样做？这主要取决于MySQL的内部实现，MySQL-5.7以前的版本，对于prepare的事务，MySQL是不会记录binlog的\(官方说是减少fsync，起到了优化的作用\)。只有当分布式事务提交的时候才会把前面的操作写入binlog信息，所以对于binlog来说，分布式事务与普通的事务没有区别，而prepare以前的操作信息都保存在连接的IO\_CACHE中，如果这个时候客户端退出了，以前的binlog信息都会被丢失，再次重连后允许提交的话，会造成Binlog丢失，从而造成主从数据的不一致，所以官方在客户端退出的时候直接把已经prepare的事务都回滚了！

官方的做法，貌似干得很漂亮，牺牲了一点标准化的东西，至少保证了主从数据的一致性。但其实不然，若用户已经prepare后在客户端退出之前，MySQL发生了宕机，这个时候又会怎样？

MySQL在某个分布式事务prepare成功后宕机，宕机前操作该事务的连接并没有断开，这个时候已经prepare的事务并不会被回滚，所以在MySQL重新启动后，引擎层通过recover机制能恢复该事务。当然该事务的Binlog已经在宕机过程中被丢失，这个时候，如果去提交，则会造成主从数据的不一致，即提交没有记录Binlog，从上丢失该条数据。所以对于这种情况，官方一般建议直接回滚已经prepare的事务。

以上是MySQL-5.7以前版本MySQL在分布式事务上的各种问题，那么MySQL-5.7版本官方做了哪些改进？这个可以从官方的WL\#6860描述上得到一些信息，我们还是本着没有实践就没有发言权的态度，从具体的操作上来分析下MySQL-5.7的改进方法：

还是以上面同样的表结构进行同样的操作如下：  
mysql&gt; XA START 'mysql57';  
mysql&gt; INSERT INTO t VALUES\(1,1\);  
mysql&gt; XA END 'mysql57';  
mysql&gt; XA PREPARE 'mysql57'  
这个时候，我们通过mysqlbinlog来查看下Master上的Binlog，结果如下：  
![image](http://www.innomysql.net/wp-content/uploads/2016/01/4.png)

同时也对比下Slave上的Relay log，如下：  
![image](http://www.innomysql.net/wp-content/uploads/2016/01/5.png)

通过上面的操作，明显发现在prepare以后，从XA START到XA PREPARE之间的操作都被记录到了Master的Binlog中，然后通过复制关系传到了Slave上。也就是说MySQL-5.7开始，MySQL对于分布式事务，在prepare的时候就完成了写Binlog的操作，通过新增一种叫XA\_prepare\_log\_event的event类型来实现，这是与以前版本的主要区别\(以前版本prepare时不写Binlog\)

当然仅靠这一点是不够的，因为我们知道Slave通过SQL thread来回放Relay log信息，由于prepare的事务能阻塞整个session，而回放的SQL thread只有一个\(不考虑并行回放\)，那么SQL thread会不会因为被分布式事务的prepare阶段所阻塞，从而造成整个SQL thread回放出现问题？这也正是官方要解决的第二个问题：怎么样能使SQL thread在回放到分布式事务的prepare阶段时，不阻塞后面event的回放？其实这个实现也很简单\(在xa.cc::applier\_reset\_xa\_trans\)，只要在SQL thread回放到prepare的时候，进行类似于客户端断开连接的处理即可\(把相关cache与SQL thread的连接句柄脱离\)。最后在Slave服务器上，用户通过命令XA RECOVER可以查到如下信息：

mysql&gt; XA RECOVER;  
+----------+--------------+--------------+---------+  
\| formatID \| gtrid\_length \| bqual\_length \| data    \|  
+----------+--------------+--------------+---------+  
\| 1        \| 7            \| 0            \| mysql57 \|  
+----------+--------------+--------------+---------+  
至于上面的事务什么时候提交，一般等到Master上进行XA COMMIT  ‘mysql57’后，slave上也同时会被提交。

总结

综上所述，MySQL 5.7对于分布式事务的支持变得完美了，一个长达数十年的bug又被修复了，因而又多了一个升级到MySQL-5.7版本的理由。

