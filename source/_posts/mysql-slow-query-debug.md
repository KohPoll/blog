title: '记一次 MySQL 的慢查优化'
date: 2015-12-22
tags:
---

最近遇见一个 MySQL 的慢查问题，于是排查了下，这里把相关的过程做个总结。

# 定位原因

我首先查看了 MySQL 的慢查询日志，发现有这样一条 query 耗时非常长（大概在 1 秒多），而且扫描的行数很大（10 多万条数据，差不多是全表了）：

```sql
SELECT * FROM tgdemand_demand t1
WHERE
  (
    t1.id IN
    (
      SELECT t2.demand_id
      FROM tgdemand_job t2
      WHERE (t2.state = 'working' AND t2.wangwang = 'abc')
    )
    AND
    NOT (t1.state = 'needConfirm')
  )
ORDER BY t1.create_date DESC
```

这个查询不是很复杂，首先执行一个子查询，取到任务的状态（state）是 'working' 并且任务的关联人 （wangwang）是'abc'的所有需求 id（这个设计师进行中的任务对应的需求 id），然后再到主表 `tgdemand_demand` 中带入刚才的 id 集合，查询出需求状态（state）不是 'needConfirm' 的所有需求，最后进行一个排序。

按道理子查询筛选出 id 后到主表过滤是直接使用到主键，应该是很快的啊。而且，我检查了子查询的 tgdemand_job 表的索引，where 中用到的查询条件都已经增加了索引。怎么会这样呢？

于是，我对这个 query 执行了一个 explain（输出 sql 语句的执行计划），看看 MySQL 的执行计划是怎样的。输出如下：

![explain](http://img3.tbcdn.cn/L1/461/1/abe7cdbdc386916a4b5ab220b628bcff871d52d0.png)

我们看到，第一行是 t1 表，type 是 ALL（全表扫描），rows（影响行数）是 157089，没有用到任何索引；第二行是 t2 表，用到了索引。和我之前理解的执行顺序完全不一样！

为什么 MySQL 不是先执行子查询，而是对 t1 表进行了全表扫描呢？我们仔细看第二行的 select_type，发现它的值是 DEPENDENT_SUBQUERY，意思是这个子查询的查询方式依赖外层的查询。这是什么意思？

实际上，MySQL 对于这种子查询会进行改写，上面的 SQL 会被改写成下面的形式：

```sql
SELECT * FROM tgdemand_demand t1 WHERE EXISTS (
  SELECT * FROM tgdemand_job t2 WHERE t1.id = t2.demand_id AND (t2.state = 'working' AND t2.wangwang = 'abc')
) AND NOT (t1.state = 'needConfirm')
ORDER BY t1.create_date DESC;
```

这表示，SQL 会去扫描 tgdemand_demand 表的所有数据，每条数据再传入到子查询中与表 tgdemand_job 进行关联，执行子查询，子查询根本不会先执行，而且子查询会执行 157089 次（外层表的记录数量）。还好我们的子查询加了必要的索引，不然结果会更加惨不忍睹。

这个结果真是太坑爹，而且十分违反直觉。对于慢查询，千万不要想当然，还是多多 explain，看看数据库实际上是怎么去执行的。

# 问题修复

既然子查询会被改写，那最简单的解决方案就是不用子查询，将内层获取需求 id 的 SQL 单独拿出来执行，取到结果后再执行一条 SQL 去获取实际的数据。大概像这样（下面的语句是不合法的，只是示意）：

```sql
ids = SELECT t2.demand_id
FROM tgdemand_job t2
WHERE (t2.state = 'working' AND t2.wangwang = 'abc');

SELECT * FROM tgdemand_demand t1
WHERE
  (
    t1.id IN ids
    AND
    NOT (t1.state = 'needConfirm')
  )
ORDER BY t1.create_date DESC;
```

说干咱就干，我找到了下面的代码（是 python 语言写的）：

```python
demand_ids = Job.objects.filter(wangwang=user['wangwang'], state='working').values_list("demand_id", flat=True)

demands = Demand.objects.filter(id__in=demand_ids).exclude(state__in=['needConfirm']).order_by('-create_date')
```

咦！这不是和我想得是一样的嘛？先查出需求 id（代码第一行），然后用 id 集合再去执行实际的查询（代码第二行）。为什么经过 ORM 框架的处理后产出的 SQL 就不一样了呢？

带着这个问题我搜索了一番。原来 Django 自带的 ORM 框架生成的 QuerySet 是懒执行的（lazy evaluated），我们可以将这种 QuerySet 到处传，直到需要时才会实际的执行 SQL。

比如，我们代码里面的 `Job.objects.filter(wangwang=user['wangwang'], state='working').values_list("demand_id", flat=True)` 这个 QuerySet 实际上并没有执行，就被作为参数传递给了 `id__in`，当  `Demand.objects.filter(id__in=demand_ids).exclude(state__in=['needConfirm']).order_by('-create_date')` 这个 QuerySet 执行时，刚才未执行的 QuerySet 才开始作为 SQL 执行，于是生成了最开始的 SQL 语句。

既然如此，我们的目的要让 QuerySet 提前执行，获得结果集。根据文档，对 QuerySet 进行循环、slice、取 len、list 转换的时候被执行。于是我将代码更改为了下面的样子：

```python
demand_ids = list(Job.objects.filter(wangwang=user['wangwang'], state='working').values_list("demand_id", flat=True))

demands = Demand.objects.filter(id__in=demand_ids).exclude(state__in=['needConfirm']).order_by('-create_date')
```

终于，页面打开速度恢复正常了。

实际上，我们也可以对 SQL 进行改写来解决问题：

```sql
select * from tgdemand_demand t1, (select t.demand_id from tgdemand_job t where t.state = 'working' and t.wangwang = 'abc') t2
where t1.id=t2.demand_id and not (t1.state = 'needConfirm')
order by t1.create_date DESC
```

思路是去掉子查询，换用 2 个表进行 join 的方式来取得数据。这里就不展开了。

# 感想

框架可以提高生产率的前提是对背后的原理足够了解，不然应用很可能就会在某个时间暴露出一些隐蔽的要命问题（这些问题在小规模阶段可能根本都发现不了......）。保证应用的健壮真是个大学问，还有很多东西值得我们去探索。

# 参考资料
  - http://www.cnblogs.com/zhengyun_ustc/p/slowquery3.html
  - http://dev.mysql.com/doc/refman/5.5/en/explain-output.html
  - https://docs.djangoproject.com/en/1.9/ref/models/querysets/