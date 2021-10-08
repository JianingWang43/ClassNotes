# 数据库隔离级别

## tips：

在阅读了一些MySQL文档、技术博客、SQL标准文档之后，个人认为很多数据库所实现的未必是完全符合SQL标准的隔离性级别（老旧的SQL1992标准对MVCC和快照隔离缺乏认识），比如Oracle实际上只有两个隔离级别；MySQL 8.0版本虽然实现了四个级别，但其使用的并非传统的基于锁的实现，而是使用了一部分基于[多版本并发控制](https://zh.wikipedia.org/wiki/多版本并发控制)(MVCC)的快照隔离。

什么是快照隔离？[Wikipedia Link](https://zh.wikipedia.org/wiki/%E5%BF%AB%E7%85%A7%E9%9A%94%E7%A6%BB)

很多重要的[数据库管理系统](https://zh.wikipedia.org/wiki/数据库管理系统)已经采用了快照隔离，，如[MySQL](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)。原因是快照隔离比[可串行性](https://zh.wikipedia.org/wiki/可串行性)隔离级别的性能更好，且能避免绝大多数并发异常。快照隔离一般用[多版本并发控制](https://zh.wikipedia.org/wiki/多版本并发控制)(MVCC)实现。 快照隔离避免了ISO SQL-92所列举的并发异常现象，但不是SQL-92定义的无并发异常的[可串行化](https://zh.wikipedia.org/wiki/可串行化)。

## SQL1992标准定义

[SQL1992 Link](https://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)

隔离级别有四种，分别是未提交读READ UNCOMMITTED，提交读READ COMMITTED，可重复读REPEATABLE READ，可串行化SERIALIZABLE。根据不同的隔离级别会出现三种问题，分别是P1脏读Dirty read，P2不可重复读Non-repeatable read，P3幻读Phantom。

四种隔离级别会出现的问题如下表所示，越高的隔离级别出现的问题越少，SERIALIZABLE可以完全避免三个问题。

```
Level__________________P1______P2_______P3________
READ UNCOMMITTED       y       y        y
READ COMMITTED         n       y        y
REPEATABLE READ        n       n        y
SERIALIZABLE           n       n        n
```

## 使用锁的实现方法

### 未提交读

未提交读（READ UNCOMMITTED）是最低的隔离级别。允许“脏读”（dirty reads），事务可以看到其他事务“尚未提交”的修改。这种隔离级别不需要使用锁就可以实现。

### 提交读：

在提交读（READ COMMITTED）级别中，基于锁机制并发控制的 DBMS 需要对选定对象的写锁一直保持到事务结束，但是读锁在 SELECT 操作完成后马上释放（因此“不可重复读”现象可能会发生，见下面描述）。和前一种隔离级别一样，也不要求“范围锁”。

### 可重复读

在可重复读（REPEATBLE READS）隔离级别中，基于锁机制并发控制的 DBMS 需要对选定对象的读锁（read locks）和写锁（write locks）一直保持到事务结束，但不要求“范围锁”（在mysql中叫做gap lock），因此可能会发生“幻影读”。

### 可串行化

在基于锁机制并发控制的 DBMS 实现可串行化，要求在选定对象上的读锁和写锁保持直到事务结束后才能释放。在 SELECT 的查询中使用一个 “WHERE” 子句来描述一个范围时应该获得一个“范围锁”（range-locks）。这种机制可以避免“幻影读”（phantom reads）现象，详见下文。
当采用不基于锁的并发控制时不用获取锁。但当系统探测到几个并发事务有“写冲突”的时候，只有其中一个是允许提交的。这种机制的详细描述见“[快照隔离](https://zh.wikipedia.org/wiki/%E5%BF%AB%E7%85%A7%E9%9A%94%E7%A6%BB)”。

以上四段引用自[维基百科](https://zh.wikipedia.org/wiki/%E4%BA%8B%E5%8B%99%E9%9A%94%E9%9B%A2#%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB)

## Mysql 8.0实践：

上面的tips已经叙述过，当前的mysql8.0版本结合使用锁和快照两种机制，实现了四种隔离级别。

mysql 8.0版本已经部分改为使用[快照隔离](https://zh.wikipedia.org/wiki/%E5%BF%AB%E7%85%A7%E9%9A%94%E7%A6%BB)，首先说明，快照隔离确实可以避免SQL1992标准定义的几种并发异常现象，但是快照隔离实现的并非SQL1992定义的可串行化（即database system concepts教材中所说的冲突可串行化）。

由于mysql8.0使用了一部分mvcc实现快照隔离，又使用了一部分锁。所以不能用传统的可串行化来思考和评价它，最好是用那三种异常现象来评价。

### Mysql8.0的实现方法：

以下部分引用自[Mysql Doc Link](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#:~:text=InnoDB%20offers%20all%20four%20transaction,%2C%20REPEATABLE%20READ%20%2C%20and%20SERIALIZABLE%20.)

注：以下所说“对索引加锁”，当一个表没有显式建立的索引也没有可以自动建立索引的主键时，使用InnoDB自动为其生成的隐藏聚类索引，什么是隐藏聚类索引？定义：`If a table has no PRIMARY KEY or suitable UNIQUE index, InnoDB generates a hidden clustered index named GEN_CLUST_INDEX on a synthetic column that contains row ID values. The rows are ordered by the row ID that InnoDB assigns. The row ID is a 6-byte field that increases monotonically as new rows are inserted. Thus, the rows ordered by the row ID are physically in order of insertion.`

首先，在提交读和可重复读两个级别下，将读操作分为[Locking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html)和[Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)，并且这两种读操作在不同级别下各自行为不同。

在其他两个级别下，都是和前两个相比有一些区别又有一些相同点。

#### Locking Read

在提交读和可重复读两个级别下， 指的是`SELECT ... FOR UPDATE`语句和`SELECT ... LOCK IN SHARE MODE`（与`SELECT ... FOR SHARE`同义）语句，可以翻译为显式指定在对应行加共享锁。

文档中更细节的描述是：

[`SELECT ... FOR SHARE`](https://dev.mysql.com/doc/refman/8.0/en/select.html)

`Sets a shared mode lock on any rows that are read. Other sessions can read the rows, but cannot modify them until your transaction commits. If any of these rows were changed by another transaction that has not yet committed, your query waits until that transaction ends and then uses the latest values.`

在阅读这部分和另一篇介绍InnoDB中所有锁的类型的mysql文档：[InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)后，其实我这里有些疑问，对于mysql中的共享锁和排他锁有以下两种理解方式（个人倾向于第一种，因为这比较符合大多数时候大多数人的理解）：

第一种理解方式：这种共享锁和排他锁都是传统意义上的共享锁和排他锁，并且mysql中根据不同的隔离级别，实际上在进行locking read和update和delete操作时会自动加上这两种锁中的一个（可能根据不同的隔离级别，这两种锁的作用也可能能不同）。

第一种理解方式的引用原文：

```
Shared and Exclusive Locks
InnoDB implements standard row-level locking where there are two types of locks, shared (S) locks and exclusive (X) locks.

A shared (S) lock permits the transaction that holds the lock to read a row.

An exclusive (X) lock permits the transaction that holds the lock to update or delete a row.

If transaction T1 holds a shared (S) lock on row r, then requests from some distinct transaction T2 for a lock on row r are handled as follows:

A request by T2 for an S lock can be granted immediately. As a result, both T1 and T2 hold an S lock on r.

A request by T2 for an X lock cannot be granted immediately.

If a transaction T1 holds an exclusive (X) lock on row r, a request from some distinct transaction T2 for a lock of either type on r cannot be granted immediately. Instead, transaction T2 has to wait for transaction T1 to release its lock on row r.
```

```
shared lock
A kind of lock that allows other transactions to read the locked object, and to also acquire other shared locks on it, but not to write to it. The opposite of exclusive lock.
```

```
exclusive lock
A kind of lock that prevents any other transaction from locking the same row. Depending on the transaction isolation level, this kind of lock might block other transactions from writing to the same row, or might also block other transactions from reading the same row. The default InnoDB isolation level, REPEATABLE READ, enables higher concurrency by allowing transactions to read rows that have exclusive locks, a technique known as consistent read.
```



第二种理解方式：mysql文档中定义的这种共享锁，不同于传统意义上的与排他锁配合对应的共享锁，这种共享锁只需要自己一个就可以起作用，来阻止其他事务的写操作。锁加上之后，其他事务只能读取被锁住的内容但是无法写入，并且如果想要读取的行已经被其他事务修改过但没有提交，那必须要等到那个事务结束后，当前读操作才能开始。而且，这个锁一旦加上，只有在事务提交或回滚后才会释放。

在mysql中只有当autocommit关闭时，这种锁才可以使用，因为在mysql中如果开启autocommit，将使得每一条语句都被当作一个事务，一条语句提交一次，即事务已经失去意义了。

#### Consistent Nonlocking Read

与consistent read、nonlocking read等词同义。

在提交读和可重复读两个级别下，普通select语句（与上面的locking read相反，这种查询不显式指定加锁）会被视为consistent Nonblocking read，mysql会使用快照来实现这种查询语句。

#### 可重复读

这是InnoDB引擎的默认隔离级别

情况一：对于Consistent Read，一个事务中的多次Consistent Read操作，那么第一次读操作将建立一个快照，后续的读操作都会使用这个快照，也就是说其他事务对这个事务完全没有了影响，保证了一致性。

doc中原话如下：

`Suppose that you are running in the default REPEATABLE READ isolation level. When you issue a consistent read (that is, an ordinary SELECTstatement), InnoDB gives your transaction a timepoint according to which your query sees the database. If another transaction deletes a row and commits after your timepoint was assigned, you do not see the row as having been deleted. Inserts and updates are treated similarly.`

情况二：而对于locking read和update和delete这几种操作来说，加什么锁取决于对应的条件。

如果索引和where条件都是唯一的，那innoDB引擎只会在那一条索引上加普通锁，如果是locking read语句，则加的显然是[上面说的那种共享锁](#Locking Read)，如果是update和delete那可能加的是排他锁。

对于其他情况的搜索条件，innoDB引擎会对扫描的索引的一个范围加gap lock，gap lock是一种很特殊的锁，对一个范围起作用，效果是使得这个范围内不可以再插入记录，直到锁被释放。由于这种锁起的是抑制作用，所以多个事务可以持有同一范围上的gap lock。gap lock也分共享的和排他的，个人认为是根据操作是locking read还是update/delete来决定加共享还是排他。

按照doc的描述，这种级别下consistent read会导致幻读，locking read因为加了gap lock应该是不会产生幻读的。也就是说情况一会幻读，情况二不会。

#### 提交读

*情况一*：对于一个事务中的多次Consistent Read操作，每次都会使用一个全新的快照。

`With [READ COMMITTED] isolation level, each consistent read within a transaction sets and reads its own fresh snapshot.`

个人认为这种所谓的全新快照会获得其他事务已提交的更改，但不会获得未提交的更改。

*情况二*：对于locking read和update和delete语句，innoDB只对索引记录加普通的共享锁或排他锁，而不是上面说过的那种gap lock（区间锁），所以提交读不能防止幻读。

按照标准规定，提交读是会出现不可重复读的，显然consistent read如果按照我的推理，确实会出现不可重复读。但对于情况二如果加的锁直到事务结束再释放那确实是可以防止不可重复读的。也就是说情况一会幻读和不可重复读，情况二只会幻读。

#### 未提交读

[`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statements are performed in a nonlocking fashion, but a possible earlier version of a row might be used. Thus, using this isolation level, such reads are not consistent. This is also called a [dirty read](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_dirty_read). Otherwise, this isolation level works like [`READ COMMITTED`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_read-committed).

对于select语句，都不加锁，其他语句和提交读一样。

因为select语句没有锁，所以会出现全部三种异常现象。

#### 可串行化

This level is like [`REPEATABLE READ`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read), but `InnoDB` implicitly converts all plain [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statements to [`SELECT ... FOR SHARE`](https://dev.mysql.com/doc/refman/8.0/en/select.html) if [`autocommit`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_autocommit) is disabled. If [`autocommit`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_autocommit) is enabled, the [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) is its own transaction. It therefore is known to be read only and can be serialized if performed as a consistent (nonlocking) read and need not block for other transactions. (To force a plain [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) to block if other transactions have modified the selected rows, disable [`autocommit`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_autocommit).)

如果autocommit关闭，那么在可重复读基础上，隐式把所有查询语句后加上FOR SHARE。

如果autocommit开启，那么每一条语句都是一个事务，也就没意义了。

在这种隔离级别下，由于所有的查询语句都加锁，并且直到事务结束才释放，那么直到当前事务结束之前，不可能有其他事务写操作出现在当前事务的读操作之后，也就不可能出现幻读了。



