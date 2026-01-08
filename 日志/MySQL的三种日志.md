# MySQL的三种日志有什么用？

undo log：是 Innodb 存储引擎层生成的日志，实现了事务中的原子性，主要用于事务回滚和 MVCC。  

redo log: 是 Innodb 存储引擎层生成的日志，实现了事务中的持久性，主要用于掉电等故障恢复；  

binlog: 是 Server 层生成的日志，主要用于数据备份和主从复制；

## undo log
undo log 是一种用于撤销回退的日志。在事务没提交之前，MySQL 会先记录更新前的数据到 undo log  日志文件里面，当事务回滚时，可以利用 undo log 来进行回滚。如下图：

<img width="352" height="571" alt="image" src="https://github.com/user-attachments/assets/28e554c5-4546-4852-8671-cd05ab6e9715" />  
每当 InnoDB 引擎对一条记录进行操作（修改、删除、新增）时，要把回滚时需要的信息都记录到 undo log 里，比如：  

在插入一条记录时，要把这条记录的主键值记下来，这样之后回滚时只需要把这个主键值对应的记录删掉就好了；  
在删除一条记录时，要把这条记录中的内容都记下来，这样之后回滚时再把由这些内容组成的记录插入到表中就好了；  
在更新一条记录时，要把被更新的列的旧值记下来，这样之后回滚时再把这些列更新为旧值就好了。  

针对 delete 操作和 update 操作会有一些特殊的处理：
delete操作实际上不会立即直接删除，而是将delete对象打上delete flag，标记为删除，最终的删除操作是purge线程完成的。  
update分为两种情况：update的列是否是主键列。  
如果不是主键列，在undo log中直接反向记录是如何update的。即update是直接进行的。  
如果是主键列，update分两部执行：先删除该行，再插入一行目标行。  

一条记录的每一次更新操作产生的 undo log 格式都有一个 roll_pointer 指针和一个 trx_id 事务id：  
通过 trx_id 可以知道该记录是被哪个事务修改的；  
通过 roll_pointer 指针可以将这些 undo log 串成一个链表，这个链表就被称为版本链；  
<img width="1063" height="457" alt="image" src="https://github.com/user-attachments/assets/8f5848e0-edc0-4243-8467-4e6cfaee5d0c" />  

因此，undo log 两大作用：
实现事务回滚，保障事务的原子性。  事务处理过程中，如果出现了错误或者用户执行了ROLLBACK 语句，MySQL 可以利用 undo log 中的历史数据将数据恢复到事务开始之前的状态。  

实现 MVCC（多版本并发控制）关键因素之一。MVCC 是通过 ReadView + undo log 实现的。undo log 为每条记录保存多份历史数据，MySQL 在执行快照读（普通 select 语句）的时候，会根据事务的 Read View 里的信息，顺着 undo log 的版本链找到满足其可见性的记录。  

## redo log
为了防止断电导致数据丢失的问题，当有一条记录需要更新的时候，InnoDB 引擎就会先更新内存（同时标记为脏页），然后将本次对这个页的修改以 redo log 的形式记录下来，这个时候更新就算完成了。  
后续，InnoDB 引擎会在适当的时候，由后台线程将缓存在 Buffer Pool 的脏页刷新到磁盘里，这就是 WAL （Write-Ahead Logging）技术。  

WAL 技术指的是， MySQL 的写操作并不是立刻写到磁盘上，而是先写日志，然后在合适的时间再写到磁盘上。  
<img width="1292" height="977" alt="image" src="https://github.com/user-attachments/assets/4af19731-51ba-4dc1-a541-d2a815622d21" />  
redo log 是物理日志，记录了某个数据页做了什么修改，比如对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新，每当执行一个事务就会产生这样的一条或者多条物理日志。  
在事务提交时，只要先将 redo log 持久化到磁盘即可，可以不需要等到将缓存在 Buffer Pool 里的脏页数据持久化到磁盘。  
当系统崩溃时，虽然脏页数据没有持久化，但是 redo log 已经持久化，接着 MySQL 重启后，可以根据 redo log 的内容，将所有数据恢复到最新的状态。  

### 被修改 Undo 页面，需要记录对应 redo log 吗？  
需要的。

开启事务后，InnoDB 层更新记录前，首先要记录相应的 undo log，如果是更新操作，需要把被更新的列的旧值记下来，也就是要生成一条 undo log，undo log 会写入 Buffer Pool 中的 Undo 页面。  
不过，在内存修改该 Undo 页面后，也是需要记录对应的 redo log，因为undo log也要实现持久性的保护。  

### redo log 和 undo log 区别在哪？

这两种日志是属于 InnoDB 存储引擎的日志，它们的区别在于：  
redo log 记录了此次事务「修改后」的数据状态，记录的是更新之后的值，主要用于事务崩溃恢复，保证事务的持久性。  
undo log 记录了此次事务「修改前」的数据状态，记录的是更新之前的值，主要用于事务回滚，保证事务的原子性。  

事务提交之前发生了崩溃（这里的崩溃不是宕机崩溃，而是事务执行错误，mysql 还是正常运行的。如果是宕机崩溃的话，其实就不需要通过 undo log 回滚了，因为事务没有提交，事务的数据并不会持久化，还是在内存中，  
宕机崩溃了数据就丢失了，反正事务都没有提交成功，所以数据本身就无意义的，丢失了就丢失了），重启后会通过 undo log 回滚事务。  

事务提交之后发生了崩溃（这里的崩溃是宕机崩溃），重启后会通过 redo log 恢复事务，如下图：  
<img width="551" height="601" alt="image" src="https://github.com/user-attachments/assets/3c82c350-4ed7-4c7b-bc17-63631586543d" />  

### redo log 要写到磁盘，数据也要写磁盘，为什么要多此一举？  
写入 redo log 的方式使用了追加操作， 所以磁盘操作是顺序写，而写入数据需要先找到写入位置，然后才写到磁盘，所以磁盘操作是随机写。  
磁盘的「顺序写 」比「随机写」 高效的多，因此 redo log 写入磁盘的开销更小。  

至此， 针对为什么需要 redo log 这个问题我们有两个答案：  
实现事务的持久性，让 MySQL 有 crash-safe 的能力，能够保证 MySQL 在任何时间段突然崩溃，重启后之前已提交的记录都不会丢失；  
将写操作从「随机写」变成了「顺序写」，提升 MySQL 写入磁盘的性能。  

### redo log 什么时候刷盘？











