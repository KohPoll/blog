title: 记一次 MySQL 数据库问题排查
date: 2016-01-06
tags:
---

最近遇到应用频繁的响应缓慢，无法正常访问。帮忙一起定位原因，最后定位到的问题说起来真的是很小的细节问题，但是就是这些小细节导致了服务不稳定，真是细节决定成败。这里尝试着来分享下，希望对大家有所帮助。

# 问题 1：占着茅坑不拉屎

遇到问题首先要看的还是服务器错误日志。

错误日志中看到频繁有这样的一个异常报错：`Error: ER_CON_COUNT_ERROR: Too many connections`。这个报错是因为数据库的所有连接被客户端都占有了，没有空闲的连接可以使用。MySQL 默认的最大并发连接数是 100，然而我们的应用这边最多可能的并发也就 30~40 个任务，怎么也不太可能报这样的错误，推测很有可能是代码里面建立连接后没有及时的进行关闭。于是我们重点看了下执行 SQL 部分的代码，大概是下面这样（使用了node-mysql库）：

```javascript
var mysql = require('mysql');
// 建立连接池
var pool = mysql.createPool({
    host: 'host',
    user: 'user',
    password: 'password',
    database: 'db'
});

exports.query = function(sql, cb) {
    // 从池子里面取一个可用连接
    pool.getConnection(function(err, connection) {
        if (err) throw err;
        // 执行sql
        connection.query(sql, function(err, rows, fields) {
            if (err) {
                return cosole.error(err);
            }
            cb(rows);
        });
        // 释放此连接
        connection.release();
    });

};
```

刚开始我还真没看出来有什么问题，后来仔细读了 [node-mysql](https://github.com/felixge/node-mysql#pooling-connections) 的文档及这个 [issue](https://github.com/felixge/node-mysql/issues/712)，终于发现了我们的写法是有问题的。

再次看看上面的代码，`pool.getConnection` 后我们执行 `connection.query`，然后没等 SQL 执行完，直接调用了  `connection.release`，由于 JavaScript 的异步特性（虽然 SQL 可能很快就执行完，但是我们也必须在 `connection.query` 的 callback 里面才明确的知道 SQL 执行完了），这个时候此次连接是不会被释放的！代码里面所有的 SQL 执行都调用到这个函数，这意味着我们占着一堆数据库连接不释放，这时不断的有其他数据库连接过来，直接导致其他连接被阻塞，抛出连接太多的异常。这真是典型的“拉完不及时让坑，占着茅坑不拉屎”的行为。所以，我们一定要在 SQL 执行完成后就将连接及时进行释放。因为 SQL 执行一般是非常快的（零点几秒），如果我们执行完后不释放，在同一时间产生很多数据库连接时很有可能导致连接被阻塞，产生连接过多的异常。于是我们对代码进行了如下修改：

```javascript
exports.query = function(sql, cb) {
    // 从池子里面取一个可用连接
    pool.getConnection(function(err, connection) {
        if (err) throw err;
        // 执行sql
        connection.query(sql, function(err, rows, fields) {
            // 释放连接（一定要在错误处理前，不然出错的时候也会导致该连接得不到释放）
           connection.release();

            if (err) {
                return cosole.error(err);
            }
            cb(rows);
        });
    });

};
```

也可以用更简单的写法 `pool.query`，这个方法内部会在合适的时机来释放连接，不用我们手动操作。

完成此次修改后，这个异常没有再复现，但是响应缓慢的情况依然没有得到缓解。

# 问题 2：一条 UPDATE 引发的血案

我们再次查看了错误日志，发现了另一个异常报错：`Error: ER_LOCK_WAIT_TIMEOUT: Lock wait timeout exceeded; try restarting transaction`。这个报错就非常令人费解了，原因是锁等待超时，当前事务在等待其它事务释放锁资源造成的。

我们先大概说下什么是事务（transaction）。事务应该具有 4 个属性：
  - 原子性（事务作为整体执行，操作要么全部执行、要么全部不执行）
  - 一致性（事务应该确保数据库状态从一个一致状态转变为另一个一致状态）
  - 隔离性（多个事务并发执行时，一个事务执行不影响其他事务执行）
  - 持久性（事务提交后，对数据库修改应该永久保存在数据库中）

对于隔离性，还会分出多个隔离级别：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| :-----: | :-----:| :------: | :----: |
| 未提交读 | 可能 | 可能 | 可能 |
| 已提交读 | 不可能 | 可能 | 可能 |
| 可重复读 | 不可能 | 不可能 | 可能 |
| 串行化 | 不可能 | 不可能 | 不可能 |

  - 脏读（Dirty Read）：A 事务读到 B 事务未提交的修改。
  - 不可重复读（NonRepeatable Read）：A 事务还没有结束时，B 事务也访问同一数据。在 A 事务的两次读取之间，由于 B 事务的修改，A 事务两次读到的数据可能是不一样的。
  - 幻读（Phantom Read）：A 事务对一个表中的数据进行了修改，这种修改涉及到表中的全部数据行。同时，B 事务也修改这个表中的数据，这种修改是向表中插入一行新数据。操作 A 事务的用户发现表中出现了 B 事务插入的行，就好象发生了幻觉一样。

MySQL 默认的级别是 REPEATABLE READ（可重复读），这表示在 MySQL 的默认情况下，“脏读”、“不可重复读”是不会发生的。这就需要在更新的时候进行必要的锁定（InnoDB 是采用行级锁的方式），从而保证一致性。需要注意的是 InnoDB 的行锁是通过给索引上的索引项加锁来实现的，这个特点意味着：只有通过索引条件检索数据，InnoDB 才使用行级锁，否则，InnoDB 将使用表锁！

我们数据库表是 InnoDB 引擎的表，而 MySQL 的 InnoDB 引擎是一个支持事务的引擎，其默认操作模式是 autocommit 自动提交模式。什么意思呢？除非我们显式地开始一个事务，否则每个查询都被当做一个单独的事务自动执行。

回到上面的报错，错误日志里抛出异常时执行的 SQL 语句，都是类似这样的一条 UPDATE 语句：`update testScore set status=1,executionId='946012' where token='f7900c40-8f4b-11e5-b2f1-6feca76a1bf5'`。

问题产生的原因可以这样来描述了：我们在执行 UPDATE 语句时，MySQL 会将其当成一个事务，对表的行进行锁定，这时又有其他连接进来要 UPDATE 同样的表或者 SELECT 这张表时就必须等待锁资源，而这个等待时间太久，导致超时了。

什么？一个 UPDATE 语句居然会这么慢？这我简直不能接受啊！那我只能看看为啥这个语句如此慢了。

查看慢查询（slow_queries.log）日志里面对应的查询信息：

```sql
# Query_time: 56.855324  Lock_time: 48.054343 Rows_sent: 0  Rows_examined: 29400
update testScore set uiTaskId=81041 where token='e7d7d8f0-8f4b-11e5-99be-9dfbb419755e';
```

这样一条 UPDATE 语句花了 56 秒，扫了 29400 条表记录。看到这样的执行日志，也大概猜到原因了，没有为查询字段 token 加索引！这样 MySQL 在进行 update 操作时不会走行锁，直接锁定了整张表，而这个 update 语句本身也够慢（扫了全表），那并发多个 update 更新时导致了等待锁超时。

给 testScore 表的 token 字段增加了索引，终于，这个异常不再复现，响应时间开始回归正常。

# 参考资料

  - https://github.com/felixge/node-mysql#pooling-connections
  - https://github.com/felixge/node-mysql/issues/712
  - http://dev.mysql.com/doc/refman/5.5/en/too-many-connections.html
  - http://www.cnblogs.com/zhoujinyi/p/3437475.html
  - http://stackoverflow.com/questions/5836623/getting-lock-wait-timeout-exceeded-try-restarting-transaction-even-though-im
  - https://www.percona.com/blog/2015/04/09/innodb-locks-deadlocks-without-index-different-isolation-level/

本文采用 [知识共享署名 3.0 中国大陆许可协议](http://creativecommons.org/licenses/by/3.0/cn)，可自由转载、引用，但需署名作者且注明文章出处 。
