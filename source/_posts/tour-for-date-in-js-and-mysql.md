title: 关于“时间”的一次探索
date: 2016-01-12
tags:
---

最近使用 `sequelize` 过程中发现一个“奇怪”的问题，将某个时间插入到表中后，通过 `sequelize` 查询出来的时间和通过 `mysql` 命令行工具查询出来的时间不一样。非常困惑，于是研究了下，下面是学习成果。

## 基本概念

我们先来介绍一些可能当年在地理课上学习过的基本概念。

说起来，时间真是一个神奇的东西。以前人们通过观察太阳的位置来决定时间（比如：使用日晷），这就使得不同经纬度的地区时间是不一样的。后来人们进一步规定以子午线为中心，向东西两侧延伸，每 15 度划分一个时区，刚好是 24 个时区。然后因为一天有 24 小时，地球自转一圈是 360 度，360 度 / 24 小时 = 15 度/小时，所以每差一个时区，时间就差一个小时。

最开始的标准时间（子午线中心处的时间）是英国伦敦的皇家格林威治天文台的标准时间（因为它刚好在本初子午线经过的地方），这就是我们常说的 `GMT`（Greenwich Mean Time）。然后其他各个时区根据标准时间确定自己的时间，往东的时区时间晚（表示为 GMT+hh:mm）、往西的时区时间早（表示为 GMT-hh:mm）。比如，中国标准时间是东八区，我们的时间就总是比 `GMT` 时间晚 8 小时，他们在凌晨 1 点，我们已经是早晨 9 点了。

但是 `GMT` 其实是根据地球自转、公转计算的（太阳每天经过英国伦敦皇家格林威治天文台的时间为中午 12 点），不是非常准确，于是后面提出了根据原子钟计算的标准时间 `UTC`（Coordinated Universal Time）。

一般情况下，`GMT` 和 `UTC` 可以互换，但是实际上，`GMT` 是一个时区，而 `UTC` 是一个时间标准。

可以在这里看到所有的时区：<http://www.timeanddate.com/time/map/>

所以，当我们“展示”某个时间时，明确时区就变得非常重要了。不然你只说现在是 `2016-01-11 19:30:00`，然后不告诉我时区，我其实是没法准确知道时间的（当然，我可以认为这个时间是我所在时区的当地时间）。如果你说现在是 `2016-01-11 19:30:00 GMT+0800`，那我就知道这个时间是东八区的时间了。如果我在东八区，那时间就是 19:30，如果我在 `GMT` 时区，那时间就是 11:30（减掉 8 小时）。

## JavaScript 中的“时间”

我们现在来介绍下 JavaScript 中的“时间”，包括：`Date`、`Date.parse`、`Date.UTC`、`Date.now`。

注：下面的代码示例可以在 node shell 里面运行，如果你运行的时候结果和下面的不一致，那可能咱们不在一个时区：）

### Date 构造器

构造时间的方法有下面几种：

```javascript
new Date();           // 当前时间
new Date(value);      // 自 1970-01-01 00:00:00 UTC 经过的毫秒数
new Date(dateString); // 时间字符串
new Date(year, month[, day[, hour[, minutes[, seconds[, milliseconds]]]]]);
```

需要注意的是：构造出的日期用来显示时，会被转换为本地时间（调用 `toString` 方法）：

```javascript
> new Date()
Mon Jan 11 2016 20:15:18 GMT+0800 (CST)
```

打印出我写这篇文章时的本地时间。后面的 `GMT+0800` 表示是“东八区”，`CST` 表示是“中国标准时间（China Standard Time）”。

有一个很“诡异”的地方是如果我们直接使用 `Date`，而不是 `new Date`，得到的将会是字符串，而不是 `Date` 类型的对象：

```javascript
> typeof Date()
'string'
> typeof new Date()
'object'
```

#### 时间字符串
我们先说最复杂的时间字符串形式。它实际上支持两种格式：一种是 RFC-2822 的标准；另一种是 ISO 8601 的标准。我们主要介绍后一种。

##### ISO 8601
ISO 8601的标准格式是：`YYYY-MM-DDTHH:mm:ss.sssZ`，分别表示：
  - `YYYY`：年份，0000 ~ 9999
  - `MM`：月份，01 ~ 12
  - `DD`：日，01 ~ 31
  - `T`：分隔日期和时间
  - `HH`：小时，00 ~ 24
  - `mm`：分钟，00 ~ 59
  - `ss`：秒，00 ~ 59
  - `.sss`：毫秒
  - `Z`：时区，可以是：`Z`（UFC）、`+HH:mm`、`-HH:mm`

这里我们主要来说下 `T`、以及 `Z`。

###### T
`T` 也可以用空格表示，但是这两种表示有点不一样，`T` 其实表示 `UTC`，而空格会被认为是本地时区（前提是不通过 `Z` 指定时区）。比如下面的例子：

```javascript
> new Date('1970-01-01 00:00:00')
Thu Jan 01 1970 00:00:00 GMT+0800 (CST)

> new Date('1970-01-01T00:00:00')
Thu Jan 01 1970 08:00:00 GMT+0800 (CST)
```

示例 1 的空格表示法被当做了本地时区，所以显示的时间和传入的时间一致。

示例 2 的 `T` 被当做 `UTC` 时间，所以显示的时间会加上本地时区（东八区）的 8 小时偏移。实际上，`1970-01-01T00:00:00` 等价于 `1970-01-01 00:00:00Z`。

###### Z
`Z` 用来表示传入时间的时区（zone），不指定并且没有使用 `T` 分隔而是使用空格分隔时，就按本地时区处理，比如下面的例子：

```javascript
> new Date('1970-01-01T00:00:00+08:00')
Thu Jan 01 1970 00:00:00 GMT+0800 (CST)

> new Date('1970-01-01 00:00:00')
Thu Jan 01 1970 00:00:00 GMT+0800 (CST)

> new Date('1970-01-01T00:00:00+00:00')
Thu Jan 01 1970 08:00:00 GMT+0800 (CST)
```

示例 1 是东八区时间，显示的时间和传入的时间一致（因为我本地时区是东八区）。

示例 2 和示例 1 结果一样，不指定时区就是本地时区。

示例 3 指定时区为 `GMT` 时区（偏移为 0），显示的时间会加上本地时区的偏移（8 小时）。

##### RFC-2822
RFC-2822 的标准格式大概是这样：`Wed Mar 25 2015 09:56:24 GMT+0100`。其实就是上面显示时间时使用的形式：

```javascript
> new Date('Thu Jan 01 1970 00:00:00 GMT+0800 (CST)')
Thu Jan 01 1970 00:00:00 GMT+0800 (CST)
```

除了能表示基本信息，还可以表示星期，但是一点也不容易读，不建议使用。完整的规范可以在这里查看：<http://tools.ietf.org/html/rfc2822#page-14>

#### 时间戳
`Date` 构造器还可以接受整数，表示想要构造的时间自 `UTC` 时间 `1970-01-01 00:00:00` 经过的毫秒数。比如下面的代码：

```javascript
> new Date(1000 * 1)
Thu Jan 01 1970 08:00:01 GMT+0800 (CST)
```

传人 1 秒，等价于：`1970-01-01 00:00:01Z`，显示的时间加上了本地时区的偏移（8 小时）。

#### 多参数
最后，`Date` 构造器还支持传递多个参数，这种方法就没办法指定时区了，都当做本地时间处理。比如下面的代码：

```javascript
> new Date(1970, 0, 1, 0, 0, 0)
Thu Jan 01 1970 00:00:00 GMT+0800 (CST)
```

显示时间和传入时间一致，均是本地时间。注意：月份是从 0 开始的。

### Date.parse

`Date.parse` 接受一个时间字符串，如果字符串能正确解析就返回自 `UTC` 时间 `1970-01-01 00:00:00` 经过的毫秒数，否则返回 `NaN`：

```javascript
> Date.parse('1970-01-01 00:00:00')
-28800000

> new Date(Date.parse('1970-01-01 00:00:00'))
Thu Jan 01 1970 00:00:00 GMT+0800 (CST)

> Date.parse('1970-01-01T00:00:00')
0

> new Date(Date.parse('1970-01-01T00:00:00'))
Thu Jan 01 1970 08:00:00 GMT+0800 (CST)
```

示例 1，-28800000 换算后刚好是 8 小时表示的毫秒数，`28800000 / (1000 * 60 * 60)`，我们传入的是本地时区时间，等于 `UTC` 时间的 `1969-12-31 16:00:00`，和 `UTC` 时间 `1970-01-01 00:00:00` 相差刚好 -8 小时。

示例 2，将 parse 后的毫秒数传递给构造器，最后显示的时间加上了本地时区的偏移（8 小时），所以结果刚好是 `1970-01-01 00:00:00`。

示例 3，传入的是 `UTC` 时区时间，所以结果为 0。

示例 4，将 parse 后的毫秒数传递给构造器，最后显示的时间加上了本地时区的偏移（8 小时），所以结果刚好是 `1970-01-01 08:00:00`。

### Date.UTC

`Date.UTC` 接受的参数和 `Date` 构造器多参数形式一样，然后返回时间自 `UTC ` 时间 `1970-01-01 00:00:00` 经过的毫秒数：

```javascript
> Date.UTC(1970,0,1,0,0,0)
0

> Date.parse('1970-01-01T00:00:00')
0

> Date.parse('1970-01-01 00:00:00Z')
0
```

可以看出，`Date.UTC` 进行的是一种“绝对运算”，传入的时间就是 `UTC` 时间，不会转换为当地时间。

### Date.now

`Date.now` 返回当前时间距 `UTC` 时间 `1970-01-01 00:00:00` 经过的毫秒数：

```javascript
> Date.now()
1452520484343

> new Date(Date.now())
Mon Jan 11 2016 21:54:55 GMT+0800 (CST)

> new Date()
Mon Jan 11 2016 21:55:00 GMT+0800 (CST)
```

## MySQL 中的“时间”

MySQL 中和时间相关的数据类型主要包括：`YEAR`、`TIME`、`DATE`、`DATETIME`、`TIMESTAMP`。

`DATE`、`YEAR`、`TIME` 比较简单，大概总结如下：

| 名称 | 占用字节 | 取值 |
| :-- | :-- | :-- |
| DATE | 3 字节 | 1000-01-01 ~ 9999-12-31 |
| YEAR | 1 字节 | 1901 ~ 2155 |
| TIME | 3 字节 | -838:59:59 ~ 838:59:59 |

注：`TIME` 的小时范围可以这么大（超过 24 小时），是因为它还可以用来表示两个时间点之差。

### DATEIME vs TIMESTAMP

我们主要来说明下 `DATETIME` 和 `TIMESTAMP`，可以做下面的总结：

| 名称 | 占用字节 | 取值 | 受 time_zone 设置影响
| :-- | :-- | :-- | :-- |
| DATETIME | 8 字节 | 1000-01-01 00:00:00 ~ 9999-12-31 23:59:59 | 否 |
| TIMESTAMP | 4 字节 | 1970-01-01 00:00:00 ~ 2038-01-19 03:14:07 | 是 |

第一个区别是占用字节不同，导致能表示的时间范围也不一样。

第二个区别是 `DATETIME` 是“常量”，保存时就是保存时的值，检索时是一样的值，不会改变；而 `TIMESTAMP` 则是“变量”，保存时数据库服务器将其从`time_zone` 时区转换为 `UTC` 时间后保存，检索时将其转换从 `UTC` 时间转换为 `time_zone` 时区时间后返回。

比如，我们有下面这样一张表：

```sql
CREATE TABLE `tests` (
    `id` INTEGER NOT NULL auto_increment ,
    `datetime` DATETIME,
    `timestamp` TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```

连接到数据库服务器后，可以执行 `SHOW VARIABLES LIKE '%time_zone%'` 查看当前时区设置。类似下面这样的结果：

| Variable_name | Value |
| :------------ | :---- |
| system_time_zone | CST |
| time_zone | SYSTEM |

说明我目前时区是 `CST`（China  Standard Time），也就是东八区。

我们尝试插入下面的数据：

```sql
INSERT INTO `tests` (`id`, `datetime`, `timestamp`) VALUES (DEFAULT, '1970-01-01 00:00:00', '1970-01-01 00:00:00');
```

会发现有一个报错：`Error Code: 1292. Incorrect datetime value: '1970-01-01 00:00:00' for column 'timestamp'`。给 timestamp 这一列提供的值不对，因为我们尝试插入 `1970-01-01 00:00:00` 时，数据库服务器会根据 `time_zone` 的设置将其转换为 `UTC` 时间，也就是 `1969-12-31 16:00:00`，而这个值明显超过了 `TIMESTAMP` 类型的范围。

我们换个大一点的值：

```sql
INSERT INTO `tests` (`id`, `datetime`, `timestamp`) VALUES (DEFAULT, '2000-01-01 00:00:00', '2000-01-01 00:00:00');
```

这次就成功插入了。

再次检索时结果也是正确的（数据库服务器将值从 `UTC` 时间转换为 `time_zone` 设置的时区时间）：

```sql
SELECT * FROM sample.tests;
```

返回：

| id | datetime | timestamp |
| :-- | :-------- | :--------- |
| 1 | 2000-01-01 00:00:00 | 2000-01-01 00:00:00 |

如果我们先将 `time_zone` 设置为一个不同的值后再进行检索就会发现不同的结果：

```sql
SET time_zone = '+00:00';
SELECT * FROM sample.tests;
```

返回：

| id | datetime | timestamp |
| :-- | :-------- | :--------- |
| 1 | 2000-01-01 00:00:00 | 1999-12-31 16:00:00 |

可以看到 datetime 列值没有受 `time_zone` 设置的影响，而 timestamp 列值却改变了。数据库服务器将其从 `UTC` 时区转换为 `time_zone` 时区的时间（首先 `2000-01-01 00:00:00` 在上面进行插入时根据 `time_zone` 被转换为了 `1999-12-31 16:00:00`，此次检索时 `time_zone` 被设置为 `+00:00`，转换回来刚好就是 `1999-12-31 16:00:00`）。

那这两种类型怎么选择呢？建议优先使用 `DATETIME`，表示范围大、不容易受服务器的设置影响。

## 在 JavaScript 和 MySQL 间转换

分别说明了 JavaScript 和 MySQL 中的“时间”后，我们来聊聊 ORM 框架一般都是怎么样在两者间进行正确、合适的转换来避免混乱的。下面的说明将基于 sequelize 框架来解释，主要是一种思路，其他的框架可以阅读框架提供的文档或是源码。

sequelize 实际上有一个 `timezone` 的配置，默认是 `+00:00`（<http://sequelize.readthedocs.org/en/latest/api/sequelize/>）。这个 `timezone` 有下面的用途：
  - 建立数据库连接时，执行 `SET time_zone = opts.timezone`
  - MySQL 的时间类型和 JavaScript 的时间类型的互相转换

第一个用途很简单，体现在源码里就是执行一个 `SQL` 语句：

```javascript
connection.query("SET time_zone = '" + self.sequelize.options.timezone + "'"); /* jshint ignore: line */
```

第二个用途主要体现在两个地方：1）在 JavaScript 中调用 ORM 方法进行插入、更新时，需要将 `Date` 类型转为正确的 SQL 语句；2）从 MySQL 服务器查询数据时，需要将数据库查询到的值转换为 JavaScript 中的 Date 类型。下面我们分别来看一看。

### JavaScript -> MySQL

这个转换的核心代码如下：

```javascript
SqlString.dateToString = function(date, timeZone, dialect) {
  if (moment.tz.zone(timeZone)) {
    date = moment(date).tz(timeZone);
  } else {
    date = moment(date).utcOffset(timeZone);
  }

  if (dialect === 'mysql' || dialect === 'mariadb') {
    return date.format('YYYY-MM-DD HH:mm:ss');
  } else {
    // ZZ here means current timezone, _not_ UTC
    return date.format('YYYY-MM-DD HH:mm:ss.SSS Z');
  }
};
```

代码逻辑如下：
  1. 检查 `timeZone` 是否存在，如果存在（存在指的是类似 `America/New_York` 这样的表示法），调用 `tz` 设置 `date` 的时区。
  2. 如果不存在（类似 `+00:00`、`-07:00` 这样的表示法），调用 `utcOffset` 设置 `date` 的相对 `UTC` 的时区偏移。
  3. 最后使用上面设置的时区偏移将其 `format` 成 MySQL 需要的 `YYYY-MM-DD HH:mm:ss` 格式。

举两个例子。

如果 `timeZone` 等于 `+00:00`，date 等于 `new Date('2016-01-12 09:46:00')`，到 `UTC` 的偏移等于 (timeZone - 本地时区) + timeZone：`(00:00 - 08:00) + 00:00 = -08:00`，即 `2016-01-12 09:46:00-08:00`，于是 `format` 后的结果是 `2016-01-12 01:46:00`。

如果 `timeZone` 等于 `+08:00`，date 等于 `new Date('2016-01-12 09:46:00')`，到 `UTC` 的偏移等于 (timeZone - 本地时区) + timeZone：`(08:00 - 08:00) + 08:00 = 08:00`，即 `2016-01-12 09:46:00+08:00`。于是 `format` 后的结果是 `2016-01-12 09:46:00`。

如果 `timeZone` 等于 `Asia/Shanghai`，结果也会是 `2016-01-12 09:46:00`，和 `+08:00` 等价。

sequelize 的 `timezone` 默认是 `+00:00`，所以，我们在 JavaScript 中的时间最后应用到数据库中都会被转换成 `UTC` 的时间（比实际的时间早 8 小时）。

### MySQL -> JavaScript

这个转换过程实际上是更底层的 node-mysql 库来实现的。核心代码如下：

```javascript
  switch (field.type) {
    case Types.TIMESTAMP:
    case Types.DATE:
    case Types.DATETIME:
    case Types.NEWDATE:
      var dateString = parser.parseLengthCodedString();
      if (dateStrings) {
        return dateString;
      }
      var dt;

      if (dateString === null) {
        return null;
      }

      var originalString = dateString;
      if (field.type === Types.DATE) {
        dateString += ' 00:00:00';
      }

      if (timeZone !== 'local') {
        dateString += ' ' + timeZone;
      }

      dt = new Date(dateString);
      if (isNaN(dt.getTime())) {
        return originalString;
      }

      return dt;
   // 更多代码...
}
```

处理过程大概是这样：
  1. 用 `parser` 将服务器返回的二进制数据解析为时间字符串
  2. 如果配置了强制返回字符串 `dateStrings` 而不是转换回 `Date` 类型，直接返回 `dateString`
  3. 如果字段类型是 `DATE`，时间字符串的时间部分统一为 `00:00:00`
  4. 如果配置的 `timeZone` 不是 `local`（本地时区），时间字符串加上时区信息
  5. 将时间字符串传给 `Date` 构造器，如果构造出的时间不合法，返回原始时间字符串，否则返回时间对象

默认情况下，sequelize 在进行连接时传递给 node-mysql 的 `timeZone` 是 `+00:00`，所以，第 4 步的时间字符串会是类似这样的值 `2016-01-12 01:46:00+00:00`，而这个值传递给 `Date` 构造器，在显示时转换回本地时区时间，就变成了 `2016-01-12 09:46:00`（比数据库中的时间晚 8 小时）。

### 一个例子

在使用 sequelize 定义模型时，其实是没有 `TIMESTAMP` 类型的，sequelize 只提供了一个 `Sequelize.DATE` 类型，生成建表语句时被转换为 `DATETIME`。

如果是在旧表上定义模型，而这张旧表刚好有 `TIMESTAMP` 类型的列，对 `TIMESTAMP` 类型的列定义模型时还是可以使用 `Sequelize.DATE`，对操作没有任何影响。但是 `TIMESTAMP` 是受 `time_zone` 设置影响的，这会引起一些困惑。下面我们来看一个例子。

sequelize 默认将 `time_zone` 设置为 `+00:00`，当我们执行下面代码时：

```javascript
Test.create({
    'datetime': new Date('2016-01-10 20:07:00'),
    'timestamp': new Date('2016-01-10 20:07:00')
  });
```

会进行上面提到的 JavaScript 时间到 MySQL 时间字符串的转换，生成的 SQL 其实是（时间被转换为了 `UTC` 时间，比本地时间早了 8 小时）：

```sql
INSERT INTO `tests` (`id`,`datetime`,`timestamp`) VALUES (DEFAULT,'2016-01-10 12:07:00','2016-01-10 12:07:00');
```

当我们执行 `Test.findAll()` 来查询数据时，会进行上面提到的 MySQL 时间到 JavaScript 时间的转换，其实就是返回这样的结果（显示时时间从 `UTC` 时间转换回了本地时间）：

```javascript
> new Date('2016-01-10 12:07:00+00:00')
Sun Jan 10 2016 20:07:00 GMT+0800 (CST)
```

和我们插入时的时间是一致的。

如果我们通过 MySQL 命令行来查询数据时，发现其实是这样的结果：

| id | datetime | timestamp |
| :-- | :-------- | :--------- |
| 1 | 2016-01-10 12:07:00 | 2016-01-10 20:07:00 |

这很好理解，因为我们数据库服务器的 `time_zone` 默认是东八区，`TIMESTAMP` 是受时区影响的，查询时被数据库服务器从 `UTC` 时间转换回了 `time_zone` 时区时间；`DATETIME` 不受影响，还是 `UTC` 时间。

如果我们先执行 `SET time_zone = '+00:00'`，再进行查询，那结果就都会是 `UTC` 时间了。所以，不要以为数据出错了哦。

总结下就是，sequelize 会将本地时间转换为 `UTC` 时间后入库，查询时再将 `UTC` 时间转换为本地时间。这能达到最好的兼容性，存储总是使用 `UTC` 时间，展示时应用端自己转换为本地时区时间后显示。当然这个的前提是数据类型选用 `DATETIME`。

### 兼容老数据

这里要说的最后一个问题是基于旧表定义 sequelize 模型，并且表中时间值插入时没有转换为 `UTC` 时间（全部是东八区时间），而且 `DATETIME` 和 `TIMESTAMP` 混用，该怎么办？

在默认配置下，情况如下：

查询 `DATETIME` 类型数据时，时间总是会晚 8 小时。比如，数据库中某条老数据的时间是 `2012-01-01 01:00:00`（已经是本地时间了，因为没转换），查询时被 sequelize 转换为 `new Date('2012-01-01 01:00:00+00:00')`，显示时转换为本地时间 `2012-01-01 09:00:00`，结果显然不对。

查询 `TIMESTAMP` 类型数据时，时间是正确的。这是因为 `TIMESTAMP` 受 `time_zone` 影响，sequelize 默认将其设置为 `+00:00`，查询时数据库服务器先将时间转换到 `time_zone` 设置的时区时间，由于没有时区偏移，刚好查出来的就是数据库中的值。比如：`2012-01-01 00:00:00`（注意这个值是 UTC 时间），sequelize 将其转换为 `new Date('2012-01-01 00:00:00+00:00')`，显示时转换为本地时间 `2012-01-01 08:00:00`，刚好“侥幸”正确。

新插入的数据 sequelize 会进行上一部分说的双向转换来保证结果的正确。

维持默认配置显然导致查询 `DATETIME` 不准确，解决方法就是将 sequelize 的 `timezone` 配置为 `+08:00`。这样一来，情况变成下面这样：

查询 `DATETIME` 类型数据时，时间 `2012-01-01 01:00:00` 被转换为 `new Date('2012-01-01 01:00:00+08:00')`，显示时转换为本地时间 `2012-01-01 01:00:00`，结果正确。

查询 `TIMESTAMP` 类型数据时，由于 `time_zone` 被设置为了 `+08:00`，数据库服务器先将库中 `UTC` 时间 `2011-01-01 00:00:00` 转换到 `time_zone` 时区时间（加上 8 小时偏移）为 `2011-01-01 08:00:00`，sequelize 将其转换为 `new Date('2011-01-01 08:00:00+08:00')`，显示时转换为本地时间 `2011-01-01 08:00:00`，结果正确。

插入、更新数据时，所有 JavaScript 时间会转换为东八区时间入库。

这样带来的问题是，所有入库时间都是东八区时间，如果有其他应用的时区不是东八区，那就需要自己基于东八区时间计算偏移并转换时间后显示了。

## 参考资料

一不小心写的有点长了，下面列出参考资料供大家进一步学习：
  - <http://www.timeanddate.com/time/gmt-utc-time.html>
  - <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date>
  - <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/parse>
  - <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/UTC>
  - <http://tools.ietf.org/html/rfc2822#page-14>
  - <http://www.ecma-international.org/ecma-262/5.1/#sec-15.9.1.15>
  - <http://www.w3schools.com/js/js_date_formats.asp>
  - <http://momentjs.com/timezone/docs/>
  - <http://sequelize.readthedocs.org/en/latest/api/sequelize/>
  - <https://github.com/felixge/node-mysql>
  - 《MySQL 技术内幕》

本文采用 [知识共享署名 3.0 中国大陆许可协议](http://creativecommons.org/licenses/by/3.0/cn)，可自由转载、引用，但需署名作者且注明文章出处 。
