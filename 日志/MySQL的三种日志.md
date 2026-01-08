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

产生的 redo log不是直接写入磁盘，而是因为这样会产生大量的 I/O 操作，而且磁盘的运行速度远慢于内存。  
所以，redo log 也有自己的缓存—— redo log buffer，每当产生一条 redo log 时，会先写入到 redo log buffer，后续在持久化到磁盘如下图：

<img width="1398" height="1344" alt="image" src="https://github.com/user-attachments/assets/864cba36-534f-4b73-9af3-1ab33690d8a2" />  
redo log buffer 默认大小 16 MB，可以通过 innodb_log_Buffer_size 参数动态的调整大小，增大它的大小可以让 MySQL 处理「大事务」是不必写入磁盘，进而提升写 IO 性能。  

缓存在 redo log buffer 里的 redo log 还是在内存中，它什么时候刷新到磁盘？  

主要有下面几个时机：  
MySQL 正常关闭时；  
当 redo log buffer 中记录的写入量大于 redo log buffer 内存空间的一半时，会触发落盘；  
InnoDB 的后台线程每隔 1 秒，将 redo log buffer 持久化到磁盘。  
每次事务提交时都将缓存在 redo log buffer 里的redo log 直接持久化到磁盘，这个策略可由 innodb_flush_log_at_trx_commit 参数控制。  

innodb_flush_log_at_trx_commit 参数控制的是什么？  
单独执行一个更新语句的时候，InnoDB 引擎会自己启动一个事务，在执行更新语句的过程中，生成的 redo log 先写入到 redo log buffer 中，然后等事务提交的时候，再将缓存在 redo log buffer 中的 redo log 按组的方式「顺序写」到磁盘。  
上面这种 redo log 刷盘时机是在事务提交的时候，这个默认的行为。  
除此之外，InnoDB 还提供了另外两种策略，由参数 innodb_flush_log_at_trx_commit 参数控制，可取的值有：0、1、2，默认值为 1，这三个值分别代表的策略如下：  

当设置该参数为 0 时，表示每次事务提交时，还是将 redo log 留在 redo log buffer 中，该模式下在事务提交时不会主动触发写入磁盘的操作。  
当设置该参数为 1 时，表示每次事务提交时，都将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘，这样可以保证 MySQL 异常重启之后数据不会丢失。  
当设置该参数为 2 时，表示每次事务提交时，都只是缓存在 redo log buffer 里的 redo log 写到 redo log 文件，注意写入到「 redo log 文件」并不意味着写入到了磁盘，因为操作系统的文件系统中有个 Page Cache，Page Cache 是专门用来缓存文件数据的，所以写入「 redo log文件」意味着写入到了操作系统的文件缓存。  
<img width="1061" height="863" alt="image" src="https://github.com/user-attachments/assets/85ec495f-5641-4d1b-8a98-7e7b4b910bbe" />  

数据安全性：参数 1 > 参数 2 > 参数 0  
写入性能：参数 0 > 参数 2> 参数 1  

### redo log文件写满了怎么办
默认情况下， InnoDB 存储引擎有 1 个重做日志文件组( redo log Group），「重做日志文件组」由有 2 个 redo log 文件组成，这两个 redo 日志的文件名叫 ：ib_logfile0 和 ib_logfile1 。  
<img width="350" height="101" alt="image" src="https://github.com/user-attachments/assets/27c11f50-d6fa-4092-b408-6b1c76238f0d" />  

在重做日志组中，每个 redo log File 的大小是固定且一致的，假设每个 redo log File 设置的上限是 1 GB，那么总共就可以记录 2GB 的操作。  
重做日志文件组是以循环写的方式工作的，从头开始写，写到末尾就又回到开头，相当于一个环形。  
<img width="441" height="261" alt="image" src="https://github.com/user-attachments/assets/5732fc4c-fe40-460f-a90e-f7b57e9b87fd" />  

redo log 是循环写的方式，相当于一个环形，InnoDB 用 write pos 表示 redo log 当前记录写到的位置，用 checkpoint 表示当前要擦除的位置，如下图：  
<img width="1362" height="906" alt="image" src="https://github.com/user-attachments/assets/84dd62a3-bce9-407b-8c52-e961c660ef46" />  

write pos 和 checkpoint 的移动都是顺时针方向；  
write pos ～ checkpoint 之间的部分（图中的红色部分），用来记录新的更新操作；  
check point ～ write pos 之间的部分（图中蓝色部分）：待落盘的脏数据页记录；  

如果 write pos 追上了 checkpoint，就意味着 redo log 文件满了，这时 MySQL 不能再执行新的更新操作，也就是说 MySQL 会被阻塞（因此所以针对并发量大的系统，适当设置 redo log 的文件大小非常重要），此时会停下来将 Buffer Pool 中的脏页刷新到磁盘中，然后标记 redo log 哪些记录可以被擦除，接着对旧的 redo log 记录进行擦除，等擦除完旧记录腾出了空间，checkpoint 就会往后移动（图中顺时针），然后 MySQL 恢复正常运行，继续执行新的更新操作。  

## binlog
MySQL 在完成一条更新操作后，Server 层还会生成一条 binlog，等之后事务提交的时候，会将该事物执行过程中产生的所有 binlog 统一写 入 binlog 文件。  
binlog 文件是记录了所有数据库表结构变更和表数据修改的日志，不会记录查询类的操作，比如 SELECT 和 SHOW 操作。  

### binlog 和 redo log 有什么区别？

1.适用对象不同  
binlog 是 MySQL 的 Server 层实现的日志，所有存储引擎都可以使用；  
redo log 是 Innodb 存储引擎实现的日志；  

2.文件格式不同
binlog 有 3 种格式类型，分别是 STATEMENT（默认格式）、ROW、 MIXED，区别如下：  

STATEMENT：每一条修改数据的 SQL 都会被记录到 binlog 中（相当于记录了逻辑操作，所以针对这种格式， binlog 可以称为逻辑日志），主从复制中 slave 端再根据 SQL 语句重现。  
但 STATEMENT 有动态函数的问题，比如你用了 uuid 或者 now 这些函数，你在主库上执行的结果并不是你在从库执行的结果，这种随时在变的函数会导致复制的数据不一致；  

ROW：记录行数据最终被修改成什么样了（这种格式的日志，就不能称为逻辑日志了），不会出现 STATEMENT 下动态函数的问题。但 ROW 的缺点是每行数据的变化结果都会被记录，比如执行批量 update 语句，更新多少行数据就会产生多少条记录，使 binlog 文件过大，而在 STATEMENT 格式下只会记录一个 update 语句而已；  

MIXED：包含了 STATEMENT 和 ROW 模式，它会根据不同的情况自动使用 ROW 模式和 STATEMENT 模式；  

redo log 是物理日志，记录的是在某个数据页做了什么修改，比如对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新；  

3、写入方式不同：  
binlog 是追加写，写满一个文件，就创建一个新的文件继续写，不会覆盖以前的日志，保存的是全量的日志。  
edo log 是循环写，日志空间大小是固定，全部写满就从头开始，保存未被刷入磁盘的脏页日志。  

4、用途不同：  
binlog 用于备份恢复、主从复制；  
redo log 用于掉电等故障恢复。  

如果不小心整个数据库的数据被删除了，能使用 redo log 文件恢复数据吗？不可以使用 redo log 文件恢复，只能使用 binlog 文件恢复。  

主从复制是怎么实现？  

MySQL 的主从复制依赖于 binlog ，也就是记录 MySQL 上的所有变化并以二进制形式保存在磁盘上。复制的过程就是将 binlog 中的数据从主库传输到从库上。  
这个过程一般是异步的，也就是主库上执行事务操作的线程不会等待复制 binlog 的线程同步完成。  
<img width="991" height="401" alt="image" src="https://github.com/user-attachments/assets/6e072dab-36aa-4f42-9373-0053e8b4b9c9" />  
MySQL 集群的主从复制过程梳理成 3 个阶段：  

写入 Binlog：主库写 binlog 日志，提交事务，并更新本地存储数据。  
同步 Binlog：把 binlog 复制到所有从库上，每个从库把 binlog 写到暂存日志中。  
回放 Binlog：回放 binlog，并更新存储引擎中的数据。  

在完成主从复制之后，你就可以在写数据时只写主库，在读数据时只读从库，这样即使写请求会锁表或者锁记录，也不会影响读请求的执行。  

<img width="451" height="471" alt="image" src="https://github.com/user-attachments/assets/3d76b40a-a292-48ce-82af-83010a9bf5d9" />  

从库是不是越多越好？不是的。

因为从库数量增加，从库连接上来的 I/O 线程也比较多，主库也要创建同样多的 log dump 线程来处理复制的请求，对主库资源消耗比较高，同时还受限于主库的网络带宽。  
所以在实际使用中，一个主库一般跟 2～3 个从库（1 套数据库，1 主 2 从 1 备主），这就是一主多从的 MySQL 集群结构。  

MySQL 主从复制还有哪些模型？  
主要有三种：  
同步复制：MySQL 主库提交事务的线程要等待所有从库的复制成功响应，才返回客户端结果。这种方式在实际项目中，基本上没法用，原因有两个：一是性能很差，因为要复制到所有节点才返回响应；二是可用性也很差，主库和所有从库任何一个数据库出问题，都会影响业务。  

异步复制（默认模型）：MySQL 主库提交事务的线程并不会等待 binlog 同步到各从库，就返回客户端结果。这种模式一旦主库宕机，数据就会发生丢失。  

半同步复制：MySQL 5.7 版本之后增加的一种复制方式，介于两者之间，事务线程不用等待所有的从库复制成功响应，只要一部分复制成功响应回来就行，比如一主二从的集群，只要数据成功复制到任意一个从库上，主库的事务线程就可以返回给客户端。这种半同步复制的方式，兼顾了异步复制和同步复制的优点，即使出现主库宕机，至少还有一个从库有最新的数据，不存在数据丢失的风险。  

### binlog 什么时候刷盘
事务执行过程中，先把日志写到 binlog cache（Server 层的 cache），事务提交的时候，再把 binlog cache 写到 binlog 文件中。  

一个事务的 binlog 是不能被拆开的，因此无论这个事务有多大（比如有很多条语句），也要保证一次性写入。这是因为有一个线程只能同时有一个事务在执行的设定，所以每当执行一个 begin/start transaction 的时候，就会默认提交上一个事务，这样如果一个事务的 binlog 被拆开的时候，在备库执行就会被当做多个事务分段自行，这样破坏了原子性，是有问题的。  

MySQL 给每个线程分配了一片内存用于缓冲 binlog ，该内存叫 binlog cache，参数 binlog_cache_size 用于控制单个线程内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。  

什么时候 binlog cache 会写到 binlog 文件？  
在事务提交的时候，执行器把 binlog cache 里的完整事务写入到 binlog 文件中，并清空 binlog cache。  
<img width="721" height="461" alt="image" src="https://github.com/user-attachments/assets/38a474bc-d05f-4c9b-bc18-3bb289ebd218" />  

虽然每个线程有自己 binlog cache，但是最终都写到同一个 binlog 文件：  

图中的 write，指的就是指把日志写入到 binlog 文件，但是并没有把数据持久化到磁盘，因为数据还缓存在文件系统的 page cache 里，write 的写入速度还是比较快的，因为不涉及磁盘 I/O。  
图中的 fsync，才是将数据持久化到磁盘的操作，这里就会涉及磁盘 I/O，所以频繁的 fsync 会导致磁盘的 I/O 升高。  

MySQL提供一个 sync_binlog 参数来控制数据库的 binlog 刷到磁盘上的频率：  
sync_binlog = 0 的时候，表示每次提交事务都只 write，不 fsync，后续交由操作系统决定何时将数据持久化到磁盘；  
sync_binlog = 1 的时候，表示每次提交事务都会 write，然后马上执行 fsync；  
sync_binlog =N(N>1) 的时候，表示每次提交事务都 write，但累积 N 个事务后才 fsync。  

如果能容少量事务的 binlog 日志丢失的风险，为了提高写入的性能，一般会 sync_binlog 设置为 100~1000 中的某个数值。  

## 为什么需要两阶段提交？
事务提交后，redo log 和 binlog 都要持久化到磁盘，但是这两个是独立的逻辑，可能出现半成功的状态，这样就造成两份日志之间的逻辑不一致。  

可以看到，在持久化 redo log 和 binlog 这两份日志的时候，如果出现半成功的状态，就会造成主从环境的数据不一致性。这是因为 redo log 影响主库的数据，binlog 影响从库的数据，所以 redo log 和 binlog 必须保持一致才能保证主从数据一致。  

MySQL 为了避免出现两份日志之间的逻辑不一致的问题，使用了「两阶段提交」来解决，两阶段提交其实是分布式事务一致性协议，它可以保证多个逻辑操作要不全部成功，要不全部失败，不会出现半成功的状态。  

两阶段提交把单个事务的提交拆分成了 2 个阶段，分别是「准备（Prepare）阶段」和「提交（Commit）阶段」。  

## 两阶段提交的过程是什么？

在 MySQL 的 InnoDB 存储引擎中，开启 binlog 的情况下，MySQL 会同时维护 binlog 日志与 InnoDB 的 redo log，为了保证这两个日志的一致性，MySQL 使用了内部 XA 事务，内部 XA 事务由 binlog 作为协调者，存储引擎是参与者。  
当客户端执行 commit 语句或者在自动提交的情况下，MySQL 内部开启一个 XA 事务，分两阶段来完成 XA 事务的提交，如下图：  

<img width="1157" height="842" alt="image" src="https://github.com/user-attachments/assets/5baf44ac-8173-4d9e-8c79-40aacc10ec20" />
从图中可看出，事务的提交过程有两个阶段，就是将 redo log 的写入拆成了两个步骤：prepare 和 commit，中间再穿插写入binlog，具体如下：  

prepare 阶段：将 XID（内部 XA 事务的 ID） 写入到 redo log，同时将 redo log 对应的事务状态设置为 prepare，然后将 redo log 持久化到磁盘。  
commit 阶段：把 XID 写入到 binlog，然后将 binlog 持久化到磁盘（sync_binlog = 1 的作用），接着调用引擎的提交事务接口，将 redo log 状态设置为 commit，此时该状态并不需要持久化到磁盘，只需要 write 到文件系统的 page cache 中就够了，因为只要 binlog 写磁盘成功，就算 redo log 的状态还是 prepare 也没有关系，一样会被认为事务已经执行成功；  

## 两阶段提交的问题是什么？  
两阶段提交虽然保证了两个日志文件的数据一致性，但是性能很差，主要有两个方面的影响：  

磁盘 I/O 次数高：对于“双1”配置，每个事务提交都会进行两次 fsync（刷盘），一次是 redo log 刷盘，另一次是 binlog 刷盘。  

锁竞争激烈：两阶段提交虽然能够保证「单事务」两个日志的内容一致，但在「多事务」的情况下，却不能保证两者的提交顺序一致，因此，在两阶段提交的流程基础上，还需要加一个锁来保证提交的原子性，从而保证多事务的情况下，两个日志的提交顺序一致。  

## 组提交
MySQL 引入了 binlog 组提交（group commit）机制，当有多个事务提交的时候，会将多个 binlog 刷盘操作合并成一个，从而减少磁盘 I/O 的次数，如果说 10 个事务依次排队刷盘的时间成本是 10，那么将这 10 个事务一次性一起刷盘的时间成本则近似于 1。  
引入了组提交机制后，prepare 阶段不变，只针对 commit 阶段，将 commit 阶段拆分为三个过程：  

flush 阶段：多个事务按进入的顺序将 binlog 从 cache 写入文件（不刷盘）；  
sync 阶段：对 binlog 文件做 fsync 操作（多个事务的 binlog 合并一次刷盘）；  
commit 阶段：各个事务按顺序做 InnoDB commit 操作；  

上面的每个阶段都有一个队列，每个阶段有锁进行保护，因此保证了事务写入的顺序，第一个进入队列的事务会成为 leader，leader领导所在队列的所有事务，全权负责整队的操作，完成后通知队内其他事务操作结束。  
<img width="852" height="828" alt="image" src="https://github.com/user-attachments/assets/ee2d3657-74a3-4434-b47e-7834d9858c5f" />
对每个阶段引入了队列后，锁就只针对每个队列进行保护，不再锁住提交事务的整个过程，可以看的出来，锁粒度减小了，这样就使得多个阶段可以并发执行，从而提升效率。  























