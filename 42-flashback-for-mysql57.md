###Flashback for MySQL

####实现原理

flashback的概念最早出现于Oracle数据库，用于快速恢复用户的误操作。

flashback for MySQL用于恢复由DML语句引起的误操作，目前不支持DDL语句。例如下面的语句：

DELETE FROM XXX;
UPDATE XXX SET YYY=ZZZ;
若没有flashback功能，那么当发生误操作时，用户只能通过全备+二进制日志前滚的方式进行恢复。通常来说，这样所需的恢复时间会非常长。为了缩短误操作恢复的时间，通常可以在slave上搭建LVM，通过定期快照的方式来缩短误操作的恢复时间。但是LVM快照的缺点是会对slave的性能产生一定的影响。

官方mysqlbinlog命令为解析MySQL的二进制日志。当二进制日志的格式为ROW格式时，可以输出每个操作的每条记录的前项与后项。那么通过逆操作即可进行回滚操作，例如：

原始操作：INSERT INTO ...
flashback操作：DELETE ...

原始操作：DELETE FROM ...
flashback操作：INSERT INTO ...

原始操作：UPDATE XXX SET OLD_VALUES ...
flashback操作：UPDATE XXX SET NEW_VALUES ...
目前flashback功能集成于官方mysqlbinlog命令，通过参数的方式进行flashback功能的开启。

####相关参数

–skip_database

解析BinLog时过滤掉该数据库。

–skip_table

解析BinLog时过滤掉该表，一般与skip_datebase配套使用。

–split-size-interval

将BinLog文件按照指定的大小拆分为多个段，解析结果为打印每个段的起始offset位置。

注意，当进行flashback时，flashback的内容先保存在内存中。若你的binlog大小为10G，那么需要额外的10G内存先暂时保存这部分信息。在某些情况下，如云环境、或服务器内存较小，会导致无法输出flashback的日志。这时可以通过此参数来设置内存保存文件的大小，例如将此值设置为100M，那么每100M就会刷新到一个文件。

–datetime_to_pos

基于输入的时间信息，解析出该时间对应的第一个BinLog event偏移位置，格式参照start-datetime，

flashback时要先找到起始的偏移量，DBA可以先通过此参数定位到具体位置，然后再进行flashback操作。

–table

仅解析该表，一般与database配套使用。

–fb_event

仅解析该类型的Log event，一般与database、table选项配套使用。可选的值有：

* DELETE
* INSERT
* UPDATE
关于DDL的flashback功能

flashback功能仅支持DML语句的快速恢复，但是如果误操作为DDL的话，那么就无能为力了，比如：

TRUNCATE TABLE  xxx;
DROP TABLE xxxx;
DROP DATABASE xxx;
若要支持上述的快速flashback功能，需要修改MySQL源代码，将删除的库或者表保存到一个垃圾回收的库中，例如$RECYCLE库。

