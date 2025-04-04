### 多版本并发控制的理解
从 TinyKV 中也可以学习到，多版本并发控制实际上是由事务的创建时间来驱动的，一个事务只能看到比自己事务结束的早的数据。
举个栗子——事务 A 是 56 开始的，事务 B 是 57 开始的，那么事务 A 是不能看到 B 的修改的。
#### Mysql 的 MVCC
一条数据在被修改 or 插入的时候，他的后面其实是记录了对这个数据进行操作的事务 idx
一个事务被创建的时候，会记录一个 ReadView——类似 Snapshot
	｜事务 idx｜当前数据库中还没提交的事务｜创建 RV 时数据库中的最小 idx｜下一事务的 idx
1. 当事务 A 去看一条数据的时候，如果这条数据的（事务）idx小于min_idx，代表可以查看，因为创建RV 的时候最小的 idx 是最小的未提交的事务，小于它代表已经提交
2. 如果这条数据的 idx > max_idx，代表在事务 A 创建后，又有事务创建了，并且这个事务对这个数据进行了修改，所以 A 不能也不应该看到这个事务
3. 当数据 idx > min_idx && idx < max_idx 时
	1. 如果 idx 存在于 A 的还没提交的事务列表中，则 A 不应该查看这个数据，因为在 A 的视角来看，这个数据是还没被提交的
	2. 如果 idx 不存在于 A 的还没提交的事务列表中，则 A 可以查看这个数据，这代表 A 创建的时候，这个对应的事务已经结束了
	3. tip：这里其实是我纠结了，”当前数据库中还没提交的事务“离散的，min_idx 和 max_idx 也是离散的，举个栗子，A 是 56 创建的，此时数据库里最小的没有完成的事务是 32，下一个被创建的是 57，“当前数据库中还没提交的事务”这个字段只记录32～56 这个区间内没完成的事务，所以可能会出现上面的第三种大情况。
总之就一句话，A 只能看到他眼中已经提交的事务。这个是 MVCC 的核心，在 TinyKv 中也是一致的，A 要去查看一个事务有没有被提交（通过 Lock 和 Write 两个列族），而在Mysql 中，这一步是完全被时间戳所驱动。
每个数据后面有一条链子，记录了每个版本的数据，形成一个链表，叫做 undo 日志。这个 undolog 会被定期清理哦。
#### Mysql Explain
可以检查一个查询有没有走索引，查看该 sql 的执行计划
#### Mysql 查询
从 B+ 树根节点开始查询，一直查到叶节点，然后找到对应的页，通过页目录二分查到对应的数据组，定位组后，就能通过链表的结构查询到对应的数据。
#### Mysql 日志层面
##### Undo log
回滚 MVCC
##### Redo log
持久化
刷盘设置 **innodb_flush_log_at_trx_commit**
##### bin log