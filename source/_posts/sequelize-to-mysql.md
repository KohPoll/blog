title: 'Sequelize 和 MySQL 对照'
date: 2015-11-12
tags:
---

*如果你觉得[Sequelize](http://sequelize.readthedocs.org/en/latest/)的文档有点多、杂，不方便看，可以看看这篇。*

在使用`NodeJS`来关系型操作数据库时，为了方便，通常都会选择一个合适的`ORM`（Object Relationship Model）框架。毕竟直接操作`SQL`比较繁琐，通过`ORM`框架，我们可以使用面向对象的方式来操作表。`NodeJS`社区有很多的`ORM`框架，我比较喜欢`Sequelize`，它功能丰富，可以非常方便的进行连表查询。

这篇文章我们就来看看，`Sequelize`是如何在`SQL`之上进行抽象、封装，从而提高开发效率的。

<!-- more -->

# 安装

这篇文章主要使用`MySQL`、`Sequelize`、`co`来进行介绍。安装非常简单：

```bash
$ npm install --save co
$ npm install --save sequelize
$ npm install --save mysql
```

代码模板如下：

```javascript
var Sequelize = require('sequelize');
var co = require('co');

co(function* () {
    // code here
}).catch(function(e) {
    console.log(e);
});
```

基本上，`Sequelize`的操作都会返回一个`Promise`，在`co`的框架里面可以直接进行`yield`，非常方便。

# 建立数据库连接

```javascript
var sequelize = new Sequelize(
    'sample', // 数据库名
    'root',   // 用户名
    'zuki',   // 用户密码
    {
        'dialect': 'mysql',  // 数据库使用mysql
        'host': 'localhost', // 数据库服务器ip
        'port': 3306,        // 数据库服务器端口
        'define': {
            // 字段以下划线（_）来分割（默认是驼峰命名风格）
            'underscored': true
        }
    }
);
```

# 定义单张表

`Sequelize`：

```javascript
var User = sequelize.define(
    // 默认表名（一般这里写单数），生成时会自动转换成复数形式
    // 这个值还会作为访问模型相关的模型时的属性名，所以建议用小写形式
    'user',
    // 字段定义（主键、created_at、updated_at默认包含，不用特殊定义）
    {
        'emp_id': {
            'type': Sequelize.CHAR(10), // 字段类型
            'allowNull': false,         // 是否允许为NULL
            'unique': true              // 字段是否UNIQUE
        },
        'nick': {
            'type': Sequelize.CHAR(10),
            'allowNull': false
        },
        'department': {
            'type': Sequelize.STRING(64),
            'allowNull': true
        }
    }
);

```

`SQL`：

```sql
CREATE TABLE IF NOT EXISTS `users` (
    `id` INTEGER NOT NULL auto_increment ,
    `emp_id` CHAR(10) NOT NULL UNIQUE,
    `nick` CHAR(10) NOT NULL,
    `department` VARCHAR(64),
    `created_at` DATETIME NOT NULL,
    `updated_at` DATETIME NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```

几点说明：

1. 建表`SQL`会自动执行的意思是你主动调用`sync`的时候。类似这样：`User.sync({force: true});`（加`force:true`，会先删掉表后再建表）。我们也可以先定义好表结构，再来定义`Sequelize`模型，这时可以不用`sync`。两者在定义阶段没有什么关系，直到我们真正开始操作模型时，才会触及到表的操作，但是我们当然还是要尽量保证模型和表的同步（可以借助一些`migration`工具）。自动建表功能有风险，使用需谨慎。

2. 所有数据类型，请参考文档[数据类型](http://sequelize.readthedocs.org/en/latest/api/datatypes/)。

3. 模型还可以定义虚拟属性、类方法、实例方法，请参考文档：[模型定义](http://sequelize.readthedocs.org/en/latest/docs/models-definition/)

4. 其他一些特殊定义如下所示：

```javascript
var User = sequelize.define(
    'user',
    {
        'emp_id': {
            'type': Sequelize.CHAR(10), // 字段类型
            'allowNull': false,         // 是否允许为NULL
            'unique': true              // 字段是否UNIQUE
        },
        'nick': {
            'type': Sequelize.CHAR(10),
            'allowNull': false
        },
        'department': {
            'type': Sequelize.STRING(64),
            'allowNull': true
        }
    },
    {
        // 自定义表名
        'freezeTableName': true,
        'tableName': 'xyz_users',

        // 是否需要增加createdAt、updatedAt、deletedAt字段
        'timestamps': true,

        // 不需要createdAt字段
        'createdAt': false,

        // 将updatedAt字段改个名
        'updatedAt': 'utime'

        // 将deletedAt字段改名
        // 同时需要设置paranoid为true（此种模式下，删除数据时不会进行物理删除，而是设置deletedAt为当前时间
        'deletedAt': 'dtime',
        'paranoid': true
    }
);
```

# 单表增删改查

通过`Sequelize`获取的模型对象都是一个`DAO`（Data Access Object）对象，这些对象会拥有许多操作数据库表的实例对象方法（比如：`save`、`update`、`destroy`等），需要获取“干净”的`JSON`对象可以调用`get({'plain': true})`。

通过模型的类方法可以获取模型对象（比如：`findById`、`findAll`等）。

## 增

`Sequelize`：

```javascript
// 方法1：build后对象只存在于内存中，调用save后才操作db
var user = User.build({
    'emp_id': '1',
    'nick': '小红',
    'department': '技术部'
});
user = yield user.save();
console.log(user.get({'plain': true}));

// 方法2：直接操作db
var user = yield User.create({
    'emp_id': '2',
    'nick': '小明',
    'department': '技术部'
});
console.log(user.get({'plain': true}));
```

`SQL`：

```sql
INSERT INTO `users`
(`id`, `emp_id`, `nick`, `department`, `updated_at`, `created_at`)
VALUES
(DEFAULT, '1', '小红', '技术部', '2015-11-02 14:49:54', '2015-11-02 14:49:54');
```

`Sequelize`会为主键`id`设置`DEFAULT`值来让数据库产生自增值，还将当前时间设置成了`created_at`和`updated_at`字段，非常方便。

## 改

`Sequelize`：

```javascript
// 方法1：操作对象属性（不会操作db），调用save后操作db
user.nick = '小白';
user = yield user.save();
console.log(user.get({'plain': true}));

// 方法2：直接update操作db
user = yield user.update({
    'nick': '小白白'
});
console.log(user.get({'plain': true}));
```

`SQL`：

```sql
UPDATE `users`
SET `nick` = '小白白', `updated_at` = '2015-11-02 15:00:04'
WHERE `id` = 1;
```

更新操作时，`Sequelize`将将当前时间设置成了`updated_at`，非常方便。

如果想限制更新属性的白名单，可以这样写：

```javascript
// 方法1
user.emp_id = '33';
user.nick = '小白';
user = yield user.save({'fields': ['nick']});

// 方法2
user = yield user.update(
    {'emp_id': '33', 'nick': '小白'},
    {'fields': ['nick']}
});
```

这样就只会更新`nick`字段，而`emp_id`会被忽略。这种方法在对表单提交过来的一大推数据中只更新某些属性的时候比较有用。

## 删

`Sequelize`：

```javascript
yield user.destroy();
```

`SQL`：

```sql
DELETE FROM `users` WHERE `id` = 1;
```

这里有个特殊的地方是，如果我们开启了`paranoid`（偏执）模式，`destroy`的时候不会执行`DELETE`语句，而是执行一个`UPDATE`语句将`deleted_at`字段设置为当前时间（一开始此字段值为`NULL`）。我们可以使用`user.destroy({force: true})`来强制删除，从而执行`DELETE`语句进行物理删除。

## 查

### 查全部

`Sequelize`：

```javascript
var users = yield User.findAll();
console.log(users);
```

`SQL`：

```sql
SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at` FROM `users`;
```

### 限制字段

`Sequelize`：

```javascript
var users = yield User.findAll({
    'attributes': ['emp_id', 'nick']
});
console.log(users);
```

`SQL`：

```sql
SELECT `emp_id`, `nick` FROM `users`;
```

### 字段重命名

`Sequelize`：

```javascript
var users = yield User.findAll({
    'attributes': [
        'emp_id', ['nick', 'user_nick']
    ]
});
console.log(users);
```

`SQL`：

```sql
SELECT `emp_id`, `nick` AS `user_nick` FROM `users`;
```

### where子句

`Sequelize`的`where`配置项基本上完全支持了`SQL`的`where`子句的功能，非常强大。我们一步步来进行介绍。

#### 基本条件

`Sequelize`：

```javascript
var users = yield User.findAll({
    'where': {
        'id': [1, 2, 3],
        'nick': 'a',
        'department': null
    }
});
console.log(users);
```

`SQL`：

```sql
SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at`
FROM `users` AS `user`
WHERE
    `user`.`id` IN (1, 2, 3) AND
    `user`.`nick`='a' AND
    `user`.`department` IS NULL;
```

可以看到，`k: v`被转换成了`k = v`，同时一个对象的多个`k: v`对被转换成了`AND`条件，即：`k1: v1, k2: v2`转换为`k1 = v1 AND k2 = v2`。

这里有2个要点：
  - 如果v是`null`，会转换为`IS NULL`（因为`SQL`没有`= NULL`
  这种语法）
  - 如果v是数组，会转换为`IN`条件（因为`SQL`没有`=[1,2,3]`这种语法，况且也没数组这种类型）

#### 操作符

操作符是对某个字段的进一步约束，可以有多个（对同一个字段的多个操作符会被转化为`AND`）。

`Sequelize`：

```javascript
var users = yield User.findAll({
    'where': {
        'id': {
            '$eq': 1,                // id = 1
            '$ne': 2,                // id != 2

            '$gt': 6,                // id > 6
            '$gte': 6,               // id >= 6

            '$lt': 10,               // id < 10
            '$lte': 10,              // id <= 10

            '$between': [6, 10],     // id BETWEEN 6 AND 10
            '$notBetween': [11, 15], // id NOT BETWEEN 11 AND 15

            '$in': [1, 2],           // id IN (1, 2)
            '$notIn': [3, 4]         // id NOT IN (3, 4)
        },
        'nick': {
            '$like': '%a%',          // nick LIKE '%a%'
            '$notLike': '%a'         // nick NOT LIKE '%a'
        },
        'updated_at': {
            '$eq': null,             // updated_at IS NULL
            '$ne': null              // created_at IS NOT NULL
        }
    }
});

```

`SQL`：

```sql
SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at`
FROM `users` AS `user`
WHERE
(
    `user`.`id` = 1 AND
    `user`.`id` != 2 AND
    `user`.`id` > 6 AND
    `user`.`id` >= 6 AND
    `user`.`id` < 10 AND
    `user`.`id` <= 10 AND
    `user`.`id` BETWEEN 6 AND 10 AND
    `user`.`id` NOT BETWEEN 11 AND 15 AND
    `user`.`id` IN (1, 2) AND
    `user`.`id` NOT IN (3, 4)
)
AND
(
    `user`.`nick` LIKE '%a%' AND
    `user`.`nick` NOT LIKE '%a'
)
AND
(
    `user`.`updated_at` IS NULL AND
    `user`.`updated_at` IS NOT NULL
);
```

这里我们发现，其实相等条件`k: v`这种写法是操作符写法`k: {$eq: v}`的简写。而要实现不等条件就必须使用操作符写法`k: {$ne: v}`。

#### 条件

上面我们说的条件查询，都是`AND`查询，`Sequelize`同时也支持`OR`、`NOT`、甚至多种条件的联合查询。

##### AND条件

`Sequelize`：

```javascript
var users = yield User.findAll({
    'where': {
        '$and': [
            {'id': [1, 2]},
            {'nick': null}
        ]
    }
});
```

`SQL`：

```sql
SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at`
FROM `users` AS `user`
WHERE
(
    `user`.`id` IN (1, 2) AND
    `user`.`nick` IS NULL
);
```

##### OR条件

`Sequelize`：

```javascript
var users = yield User.findAll({
    'where': {
        '$or': [
            {'id': [1, 2]},
            {'nick': null}
        ]
    }
});
```

`SQL`：

```sql
SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at`
FROM `users` AS `user`
WHERE
(
    `user`.`id` IN (1, 2) OR
    `user`.`nick` IS NULL
);
```

##### NOT条件

`Sequelize`：

```javascript
var users = yield User.findAll({
    'where': {
        '$not': [
            {'id': [1, 2]},
            {'nick': null}
        ]
    }
});
```

`SQL`：

```sql
SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at`
FROM `users` AS `user`
WHERE
NOT (
    `user`.`id` IN (1, 2) AND
    `user`.`nick` IS NULL
);
```

#### 转换规则

我们这里做个总结。`Sequelize`对`where`配置的转换规则的伪代码大概如下：

```javascript
function translate(where) {

    for (k, v of where) {

        if (k == 表字段) {
            // 先统一转为操作符形式
            if (v == 基本值) { // k: 'xxx'
                v = {'$eq': v};
            }
            if (v == 数组) { // k: [1, 2, 3]
                v = {'$in': v};
            }

            // 操作符转换
            for (opk, opv of v) {
                // op将opk转换对应的SQL表示
                => k + op(opk, opv) + AND;
            }
        }

        // 逻辑操作符处理

        if (k == '$and') {
            for (item in v) {
                => translate(item) + AND;
            }
        }

        if (k == '$or') {
            for (item in v) {
                => translate(item) + OR;
            }
        }

        if (k == '$not') {
            NOT +
            for (item in v) {
                => translate(item) + AND;
            }
        }

    }

    function op(opk, opv) {
        switch (opk) {
            case $eq => ('=' + opv) || 'IS NULL';
            case $ne => ('!=' + opv) || 'IS NOT NULL';
            case $gt => '>' + opv;
            case $lt => '<' + opv;
            case $gte => '>=' + opv;
            case $lte => '<=' + opv;
            case $between => 'BETWEEN ' + opv[0] + ' AND ' + opv[1];
            case $notBetween => 'NOT BETWEEN ' + opv[0] + ' AND ' + opv[1];
            case $in => 'IN (' + opv.join(',') + ')';
            case $notIn => 'NOT IN (' + opv.join(',') + ')';
            case $like => 'LIKE ' + opv;
            case $notLike => 'NOT LIKE ' + opv;
        }
    }

}

```

我们看一个复杂例子，基本上就是按上述流程来进行转换。

`Sequelize`：

```javascript
var users = yield User.findAll({
    'where': {
        'id': [3, 4],
        '$not': [
            {
                'id': {
                    '$in': [1, 2]
                }
            },
            {
                '$or': [
                    {'id': [1, 2]},
                    {'nick': null}
                ]
            }
        ],
        '$and': [
            {'id': [1, 2]},
            {'nick': null}
        ],
        '$or': [
            {'id': [1, 2]},
            {'nick': null}
        ]
    }
});
```

`SQL`：

```sql
SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at`
FROM `users` AS `user`
WHERE
    `user`.`id` IN (3, 4)
AND
NOT
(
    `user`.`id` IN (1, 2)
    AND
    (`user`.`id` IN (1, 2) OR `user`.`nick` IS NULL)
)
AND
(
    `user`.`id` IN (1, 2) AND `user`.`nick` IS NULL
)
AND
(
    `user`.`id` IN (1, 2) OR `user`.`nick` IS NULL
);
```

#### 排序

`Sequelize`：

```javascript
var users = yield User.findAll({
    'order': [
        ['id', 'DESC'],
        ['nick']
    ]
});
```

`SQL`：

```sql
SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at`
FROM `users` AS `user`
ORDER BY `user`.`id` DESC, `user`.`nick`;
```

#### 分页

`Sequelize`：

```javascript
var countPerPage = 20, currentPage = 5;
var users = yield User.findAll({
    'limit': countPerPage,                      // 每页多少条
    'offset': countPerPage * (currentPage - 1)  // 跳过多少条
});
```

`SQL`：

```sql
SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at`
FROM `users` AS `user`
LIMIT 80, 20;
```

#### 其他查询方法

##### 查询一条数据

`Sequelize`：

```javascript
user = yield User.findById(1);

user = yield User.findOne({
    'where': {'nick': 'a'}
});
```

`SQL`：

```sql
SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at`
FROM `users` AS `user`
WHERE `user`.`id` = 1 LIMIT 1;

SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at`
FROM `users` AS `user`
WHERE `user`.`nick` = 'a' LIMIT 1;
```

##### 查询并获取数量

`Sequelize`：

```javascript
var result = yield User.findAndCountAll({
    'limit': 20,
    'offset': 0
});
console.log(result);
```

`SQL`：

```sql
SELECT count(*) AS `count` FROM `users` AS `user`;

SELECT `id`, `emp_id`, `nick`, `department`, `created_at`, `updated_at`
FROM `users` AS `user`
LIMIT 20;
```

这个方法会执行2个`SQL`，返回的`result`对象将包含2个字段：`result.count`是数据总数，`result.rows`是符合查询条件的所有数据。

## 批量操作

### 插入

`Sequelize`：

```javascript
var users = yield User.bulkCreate(
    [
        {'emp_id': 'a', 'nick': 'a'},
        {'emp_id': 'b', 'nick': 'b'},
        {'emp_id': 'c', 'nick': 'c'}
    ]
);
```

`SQL`：

```sql
INSERT INTO `users`
    (`id`,`emp_id`,`nick`,`created_at`,`updated_at`)
VALUES
    (NULL,'a','a','2015-11-03 02:43:30','2015-11-03 02:43:30'),
    (NULL,'b','b','2015-11-03 02:43:30','2015-11-03 02:43:30'),
    (NULL,'c','c','2015-11-03 02:43:30','2015-11-03 02:43:30');
```

这里需要注意，返回的`users`数组里面每个对象的`id`值会是`null`。如果需要`id`值，可以重新取下数据。

### 更新

`Sequelize`：

```javascript
var affectedRows = yield User.update(
    {'nick': 'hhhh'},
    {
        'where': {
            'id': [2, 3, 4]
        }
    }
);
```

`SQL`：

```sql
UPDATE `users`
SET `nick`='hhhh',`updated_at`='2015-11-03 02:51:05'
WHERE `id` IN (2, 3, 4);
```

这里返回的`affectedRows`其实是一个数组，里面只有一个元素，表示更新的数据条数（看起来像是`Sequelize`的一个`bug`）。

### 删除

`Sequelize`：

```javascript
var affectedRows = yield User.destroy({
    'where': {'id': [2, 3, 4]}
});
```

`SQL`：

```sql
DELETE FROM `users` WHERE `id` IN (2, 3, 4);
```

这里返回的`affectedRows`是一个数字，表示删除的数据条数。

# 关系

关系一般有三种：一对一、一对多、多对多。`Sequelize`提供了清晰易用的接口来定义关系、进行表间的操作。

当说到关系查询时，一般会需要获取多张表的数据。有建议用连表查询`join`的，有不建议的。我的看法是，`join`查询这种黑科技在数据量小的情况下可以使用，基本没有什么影响，数据量大的时候，`join`的性能可能会是硬伤，应该尽量避免，可以分别根据索引取单表数据然后在应用层对数据进行`join`、`merge`。当然，查询时一定要分页，不要`findAll`。

## 一对一

### 模型定义

`Sequelize`：

```javascript
var User = sequelize.define('user',
    {
        'emp_id': {
            'type': Sequelize.CHAR(10),
            'allowNull': false,
            'unique': true
        }
    }
);
var Account = sequelize.define('account',
    {
        'email': {
            'type': Sequelize.CHAR(20),
            'allowNull': false
        }
    }
);

/*
 * User的实例对象将拥有getAccount、setAccount、addAccount方法
 */
User.hasOne(Account);
/*
 * Account的实例对象将拥有getUser、setUser、addUser方法
 */
Account.belongsTo(User);
```

`SQL`：

```sql
CREATE TABLE IF NOT EXISTS `users` (
    `id` INTEGER NOT NULL auto_increment ,
    `emp_id` CHAR(10) NOT NULL UNIQUE,
    `created_at` DATETIME NOT NULL,
    `updated_at` DATETIME NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS `accounts` (
    `id` INTEGER NOT NULL auto_increment ,
    `email` CHAR(20) NOT NULL,
    `created_at` DATETIME NOT NULL,
    `updated_at` DATETIME NOT NULL,
    `user_id` INTEGER,
    PRIMARY KEY (`id`),
    FOREIGN KEY (`user_id`) REFERENCES `users` (`id`) ON DELETE SET NULL ON UPDATE CASCADE
) ENGINE=InnoDB;
```

可以看到，这种关系中外键`user_id`加在了`Account`上。另外，`Sequelize`还给我们生成了外键约束。

一般来说，外键约束在有些自己定制的数据库系统里面是禁止的，因为会带来一些性能问题。所以，建表的`SQL`一般就去掉约束，同时给外键加一个索引（加速查询），数据的一致性就靠应用层来保证了。

### 关系操作

#### 增

`Sequelize`：

```javascript
var user = yield User.create({'emp_id': '1'});
var account = user.createAccount({'email': 'a'});
console.log(account.get({'plain': true}));
```

`SQL`：

```sql
INSERT INTO `users`
(`id`,`emp_id`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'1','2015-11-03 06:24:53','2015-11-03 06:24:53');

INSERT INTO `accounts`
(`id`,`email`,`user_id`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'a',1,'2015-11-03 06:24:53','2015-11-03 06:24:53');
```

`SQL`执行逻辑是：
  - 使用对应的的`user_id`作为外键在`accounts`表里插入一条数据。

#### 改

`Sequelize`：

```javascript
var anotherAccount = yield Account.create({'email': 'b'});
console.log(anotherAccount);
anotherAccount = yield user.setAccount(anotherAccount);
console.log(anotherAccount);
```

`SQL`：

```sql
INSERT INTO `accounts`
(`id`,`email`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'b','2015-11-03 06:37:14','2015-11-03 06:37:14');

SELECT `id`, `email`, `created_at`, `updated_at`, `user_id`
FROM `accounts` AS `account` WHERE (`account`.`user_id` = 1);

UPDATE `accounts` SET `user_id`=NULL,`updated_at`='2015-11-03 06:37:14' WHERE `id` = 1;
UPDATE `accounts` SET `user_id`=1,`updated_at`='2015-11-03 06:37:14' WHERE `id` = 2;
```

`SQL`执行逻辑是：
  - 插入一条`account`数据，此时外键`user_id`是空的，还没有关联user
  - 找出当前`user`所关联的`account`并将其`user_id`置为`NUL（为了保证一对一关系）
  - 设置新的`acount`的外键`user_id`为`user`的属性`id`，生成关系

#### 删

`Sequelize`：

```javascript
yield user.setAccount(null);
```

`SQL`：

```sql
SELECT `id`, `email`, `created_at`, `updated_at`, `user_id`
FROM `accounts` AS `account`
WHERE (`account`.`user_id` = 1);

UPDATE `accounts`
SET `user_id`=NULL,`updated_at`='2015-11-04 00:11:35'
WHERE `id` = 1;
```

这里的删除实际上只是“切断”关系，并不会真正的物理删除记录。

`SQL`执行逻辑是：
  - 找出`user`所关联的`account`数据
  - 将其外键`user_id`设置为`NULL`，完成关系的“切断”

#### 查

`Sequelize`：

```javascript
var account = yield user.getAccount();
console.log(account);
```

`SQL`：

```sql
SELECT `id`, `email`, `created_at`, `updated_at`, `user_id`
FROM `accounts` AS `account`
WHERE (`account`.`user_id` = 1);
```

这里就是调用`user`的`getAccount`方法，根据外键来获取对应的`account`。

但是其实我们用面向对象的思维来思考应该是获取`user`的时候就能通过`user.account`的方式来访问`account`对象。这可以通过`Sequelize`的`eager loading`（急加载，和懒加载相反）来实现。

`eager loading`的含义是说，取一个模型的时候，同时也把相关的模型数据也给我取过来（我很着急，不能按默认那种取一个模型就取一个模型的方式，我还要更多）。方法如下：

`Sequelize`：

```javascript
var user = yield User.findById(1, {
    'include': [Account]
});
console.log(user.get({'plain': true}));
/*
 * 输出类似：
 { id: 1,
  emp_id: '1',
  created_at: Tue Nov 03 2015 15:25:27 GMT+0800 (CST),
  updated_at: Tue Nov 03 2015 15:25:27 GMT+0800 (CST),
  account:
   { id: 2,
     email: 'b',
     created_at: Tue Nov 03 2015 15:25:27 GMT+0800 (CST),
     updated_at: Tue Nov 03 2015 15:25:27 GMT+0800 (CST),
     user_id: 1 } }
 */
```

`SQL`：

```sql
SELECT `user`.`id`, `user`.`emp_id`, `user`.`created_at`, `user`.`updated_at`, `account`.`id` AS `account.id`, `account`.`email` AS `account.email`, `account`.`created_at` AS `account.created_at`, `account`.`updated_at` AS `account.updated_at`, `account`.`user_id` AS `account.user_id`
FROM `users` AS `user` LEFT OUTER JOIN `accounts` AS `account`
ON `user`.`id` = `account`.`user_id`
WHERE `user`.`id` = 1 LIMIT 1;
```

可以看到，我们对2个表进行了一个外联接，从而在取`user`的同时也获取到了`account`。

#### 其他补充说明

如果我们重复调用`user.createAccount`方法，实际上会在数据库里面生成多条`user_id`一样的数据，并不是真正的一对一。

所以，在应用层保证一致性时，就需要我们遵循良好的编码约定。新增就用`user.createAccount`，更改就用`user.setAccount`。

也可以给`user_id`加一个`UNIQUE`约束，在数据库层面保证一致性，这时就需要做好`try/catch`，发生插入异常的时候能够知道是因为插入了多个`account`。

另外，我们上面都是使用`user`来对`account`进行操作。实际上反向操作也是可以的，这是因为我们定义了`Account.belongsTo(User)`。在`Sequelize`里面定义关系时，关系的调用方会获得相关的“关系”方法，一般为了两边都能操作，会同时定义双向关系（这里双向关系指的是模型层面，并不会在数据库表中出现两个表都加上外键的情况，请放心）。

## 一对多

### 模型定义

`Sequelize`：

```javascript
var User = sequelize.define('user',
    {
        'emp_id': {
            'type': Sequelize.CHAR(10),
            'allowNull': false,
            'unique': true
        }
    }
);
var Note = sequelize.define('note',
    {
        'title': {
            'type': Sequelize.CHAR(64),
            'allowNull': false
        }
    }
);

/*
 * User的实例对象将拥有getNotes、setNotes、addNote、createNote、removeNote、hasNote方法
 */
User.hasMany(Note);
/*
 * Note的实例对象将拥有getUser、setUser、createUser方法
 */
Note.belongsTo(User);
```

`SQL`：

```sql
CREATE TABLE IF NOT EXISTS `users` (
    `id` INTEGER NOT NULL auto_increment ,
    `emp_id` CHAR(10) NOT NULL UNIQUE,
    `created_at` DATETIME NOT NULL,
    `updated_at` DATETIME NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS `notes` (
    `id` INTEGER NOT NULL auto_increment ,
    `title` CHAR(64) NOT NULL,
    `created_at` DATETIME NOT NULL,
    `updated_at` DATETIME NOT NULL,
    `user_id` INTEGER,
    PRIMARY KEY (`id`),
    FOREIGN KEY (`user_id`) REFERENCES `users` (`id`) ON DELETE SET NULL ON UPDATE CASCADE
) ENGINE=InnoDB;
```

可以看到这种关系中，外键`user_id`加在了多的一端（`notes`表）。同时相关的模型也自动获得了一些方法。

### 关系操作

#### 增

##### 方法1

`Sequelize`：

```javascript
var user = yield User.create({'emp_id': '1'});
var note = yield user.createNote({'title': 'a'});
console.log(note);
```

`SQL`：

```sql
NSERT INTO `users`
(`id`,`emp_id`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'1','2015-11-03 23:52:05','2015-11-03 23:52:05');

INSERT INTO `notes`
(`id`,`title`,`user_id`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'a',1,'2015-11-03 23:52:05','2015-11-03 23:52:05');
```

`SQL`执行逻辑：
  - 使用`user`的主键`id`值作为外键直接在`notes`表里插入一条数据。

##### 方法2

`Sequelize`：

```javascript
var user = yield User.create({'emp_id': '1'});
var note = yield Note.create({'title': 'b'});
yield user.addNote(note);
```

`SQL`：

```sql
INSERT INTO `users`
(`id`,`emp_id`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'1','2015-11-04 00:02:56','2015-11-04 00:02:56');

INSERT INTO `notes`
(`id`,`title`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'b','2015-11-04 00:02:56','2015-11-04 00:02:56');

UPDATE `notes`
SET `user_id`=1,`updated_at`='2015-11-04 00:02:56'
WHERE `id` IN (1);
```

`SQL`执行逻辑：
  - 插入一条`note`数据，此时该条数据的外键`user_id`为空
  - 使用`user`的属性`id`值再更新该条`note`数据，设置好外键，完成关系建立

#### 改

`Sequelize`：

```javascript
// 为user增加note1、note2
var user = yield User.create({'emp_id': '1'});
var note1 = yield user.createNote({'title': 'a'});
var note2 = yield user.createNote({'title': 'b'});
// 先创建note3、note4
var note3 = yield Note.create({'title': 'c'});
var note4 = yield Note.create({'title': 'd'});
// user拥有的note更改为note3、note4
yield user.setNotes([note3, note4]);
```

`SQL`：

```sql
/* 省去了创建语句 */
SELECT `id`, `title`, `created_at`, `updated_at`, `user_id`
FROM `notes` AS `note` WHERE `note`.`user_id` = 1;

UPDATE `notes`
SET `user_id`=NULL,`updated_at`='2015-11-04 12:45:12'
WHERE `id` IN (1, 2);

UPDATE `notes`
SET `user_id`=1,`updated_at`='2015-11-04 12:45:12'
WHERE `id` IN (3, 4);
```

`SQL`执行逻辑：
  - 根据`user`的属性`id`查询所有相关的`note`数据
  - 将`note1`、`note2`的外键`user_id`置为`NULL`，切断关系
  - 将`note3`、`note4`的外键`user_id`置为`user`的属性`id`，完成关系建立

这里为啥还要查出所有的`note`数据呢？因为我们需要根据传人`setNotes`的数组来计算出哪些`note`要切断关系、哪些要新增关系，所以就需要查出来进行一个计算集合的“交集”运算。

#### 删

`Sequelize`：

```javascript
var user = yield User.create({'emp_id': '1'});
var note1 = yield user.createNote({'title': 'a'});
var note2 = yield user.createNote({'title': 'b'});
yield user.setNotes([]);
```

`SQL`：

```sql
SELECT `id`, `title`, `created_at`, `updated_at`, `user_id`
FROM `notes` AS `note` WHERE `note`.`user_id` = 1;

UPDATE `notes`
SET `user_id`=NULL,`updated_at`='2015-11-04 12:50:08'
WHERE `id` IN (1, 2);
```

实际上，上面说到的“改”已经有“删”的操作了（去掉`note1`、`note2`的关系）。这里的操作是删掉用户的所有`note`数据，直接执行`user.setNotes([])`即可。

`SQL`执行逻辑：
  - 根据`user`的属性`id`查出所有相关的`note`数据
  - 将其外键`user_id`置为`NULL`，切断关系

还有一个真正的删除方法，就是`removeNote`。如下所示：

`Sequelize`：

```javascript
yield user.removeNote(note);
```

`SQL`：

```sql
UPDATE `notes`
SET `user_id`=NULL,`updated_at`='2015-11-06 01:40:12'
WHERE `user_id` = 1 AND `id` IN (1);
```

#### 查

##### 情况1

查询`user`的所有满足条件的`note`数据。

`Sequelize`：

```javascript
var notes = yield user.getNotes({
    'where': {
        'title': {
            '$like': '%css%'
        }
    }
});
notes.forEach(function(note) {
    console.log(note);
});
```

`SQL`：

```sql
SELECT `id`, `title`, `created_at`, `updated_at`, `user_id`
FROM `notes` AS `note`
WHERE (`note`.`user_id` = 1 AND `note`.`title` LIKE '%a%');
```

这种方法的`SQL`很简单，直接根据`user`的`id`值来查询满足条件的`note`即可。

##### 情况2

查询所有满足条件的`note`，同时获取`note`属于哪个`user`。

`Sequelize`：

```javascript
var notes = yield Note.findAll({
    'include': [User],
    'where': {
        'title': {
            '$like': '%css%'
        }
    }
});
notes.forEach(function(note) {
    // note属于哪个user可以通过note.user访问
    console.log(note);
});
```

`SQL`：

```sql
SELECT `note`.`id`, `note`.`title`, `note`.`created_at`, `note`.`updated_at`, `note`.`user_id`,
`user`.`id` AS `user.id`, `user`.`emp_id` AS `user.emp_id`, `user`.`created_at` AS `user.created_at`, `user`.`updated_at` AS `user.updated_at`
FROM `notes` AS `note` LEFT OUTER JOIN `users` AS `user`
ON `note`.`user_id` = `user`.`id`
WHERE `note`.`title` LIKE '%css%';
```

这种方法，因为获取的主体是`note`，所以将`notes`去`left join`了`users`。

##### 情况3

查询所有满足条件的`user`，同时获取该`user`所有满足条件的`note`。

`Sequelize`：

```javascript
var users = yield User.findAll({
    'include': [Note],
    'where': {
        'created_at': {
            '$lt': new Date()
        }
    }
});
users.forEach(function(user) {
    // user的notes可以通过user.notes访问
    console.log(user);
});
```

`SQL`：

```sql
SELECT `user`.`id`, `user`.`emp_id`, `user`.`created_at`, `user`.`updated_at`,
`notes`.`id` AS `notes.id`, `notes`.`title` AS `notes.title`, `notes`.`created_at` AS `notes.created_at`, `notes`.`updated_at` AS `notes.updated_at`, `notes`.`user_id` AS `notes.user_id`
FROM `users` AS `user` LEFT OUTER JOIN `notes` AS `notes`
ON `user`.`id` = `notes`.`user_id`
WHERE `user`.`created_at` < '2015-11-05 01:51:35';
```

这种方法获取的主体是`user`，所以将`users`去`left join`了`notes`。

##### 一点补充

关于各种join的区别，可以参考：<http://blog.codinghorror.com/a-visual-explanation-of-sql-joins/>。

关于`eager loading`我想再啰嗦几句。`include`里面传递的是去取相关模型，默认是取全部，我们也可以再对这个模型进行一层过滤。像下面这样：

`Sequelize`：

```javascript
// 查询创建时间在今天之前的所有user，同时获取他们note的标题中含有关键字css的所有note
var users = yield User.findAll({
    'include': [
        {
            'model': Note,
            'where': {
                'title': {
                    '$like': '%css%'
                }
            }
        }
    ],
    'where': {
        'created_at': {
            '$lt': new Date()
        }
    }
});
```

`SQL`：

```sql
SELECT `user`.`id`, `user`.`emp_id`, `user`.`created_at`, `user`.`updated_at`,
`notes`.`id` AS `notes.id`, `notes`.`title` AS `notes.title`, `notes`.`created_at` AS `notes.created_at`, `notes`.`updated_at` AS `notes.updated_at`, `notes`.`user_id` AS `notes.user_id`
FROM `users` AS `user` INNER JOIN `notes` AS `notes`
ON `user`.`id` = `notes`.`user_id` AND `notes`.`title` LIKE '%css%'
WHERE `user`.`created_at` < '2015-11-05 01:58:31';
```

注意：当我们对`include`的模型加了`where`过滤时，会使用`inner join`来进行查询，这样保证只有那些拥有标题含有`css`关键词`note`的用户才会返回。

## 多对多关系

在多对多关系中，必须要额外一张关系表来将2个表进行关联，这张表可以是单纯的一个关系表，也可以是一个实际的模型（含有自己的额外属性来描述关系）。我比较喜欢用一个模型的方式，这样方便以后做扩展。

### 模型定义

`Sequelize`：

```javascript
var Note = sequelize.define('note',
    {
        'title': {
            'type': Sequelize.CHAR(64),
            'allowNull': false
        }
    }
);
var Tag = sequelize.define('tag',
    {
        'name': {
            'type': Sequelize.CHAR(64),
            'allowNull': false,
            'unique': true
        }
    }
);
var Tagging = sequelize.define('tagging',
    {
        'type': {
            'type': Sequelize.INTEGER(),
            'allowNull': false
        }
    }
);

// Note的实例拥有getTags、setTags、addTag、addTags、createTag、removeTag、hasTag方法
Note.belongsToMany(Tag, {'through': Tagging});
// Tag的实例拥有getNotes、setNotes、addNote、addNotes、createNote、removeNote、hasNote方法
Tag.belongsToMany(Note, {'through': Tagging});
```

`SQL`：

```sql
CREATE TABLE IF NOT EXISTS `notes` (
    `id` INTEGER NOT NULL auto_increment ,
    `title` CHAR(64) NOT NULL,
    `created_at` DATETIME NOT NULL,
    `updated_at` DATETIME NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS `tags` (
    `id` INTEGER NOT NULL auto_increment ,
    `name` CHAR(64) NOT NULL UNIQUE,
    `created_at` DATETIME NOT NULL,
    `updated_at` DATETIME NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS `taggings` (
    `type` INTEGER NOT NULL,
    `created_at` DATETIME NOT NULL,
    `updated_at` DATETIME NOT NULL,
    `tag_id` INTEGER ,
    `note_id` INTEGER ,
    PRIMARY KEY (`tag_id`, `note_id`),
    FOREIGN KEY (`tag_id`) REFERENCES `tags` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
    FOREIGN KEY (`note_id`) REFERENCES `notes` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB;
```

可以看到，多对多关系中单独生成了一张关系表，并设置了2个外键`tag_id`和`note_id`来和`tags`和`notes`进行关联。关于关系表的命名，我比较喜欢使用动词，因为这张表是用来表示两张表的一种联系，而且这种联系多数时候伴随着一种动作。比如：用户收藏商品（`collecting`）、用户购买商品（`buying`）、用户加入项目（`joining`）等等。

### 增

#### 方法1

`Sequelize`：

```javascript
var note = yield Note.create({'title': 'note'});
yield note.createTag({'name': 'tag'}, {'type': 0});
```

`SQL`：

```sql
INSERT INTO `notes`
(`id`,`title`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'note','2015-11-06 02:14:38','2015-11-06 02:14:38');

INSERT INTO `tags`
(`id`,`name`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'tag','2015-11-06 02:14:38','2015-11-06 02:14:38');

INSERT INTO `taggings`
(`tag_id`,`note_id`,`type`,`created_at`,`updated_at`)
VALUES
(1,1,0,'2015-11-06 02:14:38','2015-11-06 02:14:38');
```

`SQL`执行逻辑：
  - 在`notes`表插入记录
  - 在`tags`表中插入记录
  - 使用对应的值设置外键`tag_id`和`note_id`以及关系模型本身需要的属性（`type: 0`）在关系表`tagging`中插入记录

关系表本身需要的属性，通过传递一个额外的对象给设置方法来实现。

#### 方法2

`Sequelize`：

```javascript
var note = yield Note.create({'title': 'note'});
var tag = yield Tag.create({'name': 'tag'});
yield note.addTag(tag, {'type': 1});
```

`SQL`：

```sql
INSERT INTO `notes`
(`id`,`title`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'note','2015-11-06 02:20:52','2015-11-06 02:20:52');

INSERT INTO `tags`
(`id`,`name`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'tag','2015-11-06 02:20:52','2015-11-06 02:20:52');

INSERT INTO `taggings`
(`tag_id`,`note_id`,`type`,`created_at`,`updated_at`)
VALUES
(1,1,1,'2015-11-06 02:20:52','2015-11-06 02:20:52');
```

这种方法和上面的方法实际上是一样的。只是我们先手动`create`了一个`Tag`模型。

#### 方法3

`Sequelize`：

```javascript
var note = yield Note.create({'title': 'note'});
var tag1 = yield Tag.create({'name': 'tag1'});
var tag2 = yield Tag.create({'name': 'tag2'});
yield note.addTags([tag1, tag2], {'type': 2});
```

`SQL`：

```sql
INSERT INTO `notes`
(`id`,`title`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'note','2015-11-06 02:25:18','2015-11-06 02:25:18');

INSERT INTO `tags`
(`id`,`name`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'tag1','2015-11-06 02:25:18','2015-11-06 02:25:18');

INSERT INTO `tags`
(`id`,`name`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'tag2','2015-11-06 02:25:18','2015-11-06 02:25:18');

INSERT INTO `taggings` (`tag_id`,`note_id`,`type`,`created_at`,`updated_at`)
VALUES
(1,1,2,'2015-11-06 02:25:18','2015-11-06 02:25:18'),
(2,1,2,'2015-11-06 02:25:18','2015-11-06 02:25:18');
```

这种方法可以进行批量添加。当执行`addTags`时，实际上就是设置好对应的外键及关系模型本身的属性，然后在关系表中批量的插入数据。

### 改

`Sequelize`：

```javascript
// 先添加几个tag
var note = yield Note.create({'title': 'note'});
var tag1 = yield Tag.create({'name': 'tag1'});
var tag2 = yield Tag.create({'name': 'tag2'});
yield note.addTags([tag1, tag2], {'type': 2});
// 将tag改掉
var tag3 = yield Tag.create({'name': 'tag3'});
var tag4 = yield Tag.create({'name': 'tag4'});
yield note.setTags([tag3, tag4], {'type': 3});
```

`SQL`：

```sql
/* 前面添加部分的sql，和上面一样*/
INSERT INTO `notes`
(`id`,`title`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'note','2015-11-06 02:25:18','2015-11-06 02:25:18');

INSERT INTO `tags`
(`id`,`name`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'tag1','2015-11-06 02:25:18','2015-11-06 02:25:18');

INSERT INTO `tags`
(`id`,`name`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'tag2','2015-11-06 02:25:18','2015-11-06 02:25:18');

INSERT INTO `taggings`
(`tag_id`,`note_id`,`type`,`created_at`,`updated_at`)
VALUES
(1,1,2,'2015-11-06 02:25:18','2015-11-06 02:25:18'),
(2,1,2,'2015-11-06 02:25:18','2015-11-06 02:25:18');

/* 更改部分的sql */
INSERT INTO `tags`
(`id`,`name`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'tag3','2015-11-06 02:29:55','2015-11-06 02:29:55');

INSERT INTO `tags`
(`id`,`name`,`updated_at`,`created_at`)
VALUES
(DEFAULT,'tag4','2015-11-06 02:29:55','2015-11-06 02:29:55');

/* 先删除关系 */
DELETE FROM `taggings`
WHERE `note_id` = 1 AND `tag_id` IN (1, 2);

/* 插入新关系 */
INSERT INTO `taggings`
(`tag_id`,`note_id`,`type`,`created_at`,`updated_at`)
VALUES
(3,1,3,'2015-11-06 02:29:55','2015-11-06 02:29:55'),
(4,1,3,'2015-11-06 02:29:55','2015-11-06 02:29:55');
```

执行逻辑是，先将`tag1`、`tag2`在关系表中的关系删除，然后再将`tag3`、`tag4`对应的关系插入关系表。

### 删

`Sequelize`：

```javascript
// 先添加几个tag
var note = yield Note.create({'title': 'note'});
var tag1 = yield Tag.create({'name': 'tag1'});
var tag2 = yield Tag.create({'name': 'tag2'});
var tag3 = yield Tag.create({'name': 'tag2'});
yield note.addTags([tag1, tag2, tag3], {'type': 2});

// 删除一个
yield note.removeTag(tag1);

// 全部删除
yield note.setTags([]);
```

`SQL`：

```sql
/* 删除一个 */
DELETE FROM `taggings` WHERE `note_id` = 1 AND `tag_id` IN (1);

/* 删除全部 */
SELECT `type`, `created_at`, `updated_at`, `tag_id`, `note_id`
FROM `taggings` AS `tagging`
WHERE `tagging`.`note_id` = 1;

DELETE FROM `taggings` WHERE `note_id` = 1 AND `tag_id` IN (2, 3);
```

删除一个很简单，直接将关系表中的数据删除。

全部删除时，首先需要查出关系表中`note_id`对应的所有数据，然后一次删掉。

### 查

#### 情况1

查询`note`所有满足条件的`tag`。

`Sequelize`：

```javascript
var tags = yield note.getTags({
    //这里可以对tags进行where
});
tags.forEach(function(tag) {
    // 关系模型可以通过tag.tagging来访问
    console.log(tag);
});
```

`SQL`：

```sql
SELECT `tag`.`id`, `tag`.`name`, `tag`.`created_at`, `tag`.`updated_at`,
`tagging`.`type` AS `tagging.type`, `tagging`.`created_at` AS `tagging.created_at`, `tagging`.`updated_at` AS `tagging.updated_at`, `tagging`.`tag_id` AS `tagging.tag_id`, `tagging`.`note_id` AS `tagging.note_id`
FROM `tags` AS `tag`
INNER JOIN `taggings` AS `tagging`
ON
`tag`.`id` = `tagging`.`tag_id` AND `tagging`.`note_id` = 1;
```

可以看到这种查询，就是执行一个`inner join`。

#### 情况2

查询所有满足条件的`tag`，同时获取每个`tag`所在的`note`。

`Sequelize`：

```javascript
var tags = yield Tag.findAll({
    'include': [
        {
            'model': Note
            // 这里可以对notes进行where
        }
    ]
    // 这里可以对tags进行where
});
tags.forEach(function(tag) {
    // tag的notes可以通过tag.notes访问，关系模型可以通过tag.notes[0].tagging访问
    console.log(tag);
});
```

`SQL`：

```sql
SELECT `tag`.`id`, `tag`.`name`, `tag`.`created_at`, `tag`.`updated_at`,
`notes`.`id` AS `notes.id`, `notes`.`title` AS `notes.title`, `notes`.`created_at` AS `notes.created_at`, `notes`.`updated_at` AS `notes.updated_at`,
`notes.tagging`.`type` AS `notes.tagging.type`, `notes.tagging`.`created_at` AS `notes.tagging.created_at`, `notes.tagging`.`updated_at` AS `notes.tagging.updated_at`, `notes.tagging`.`tag_id` AS `notes.tagging.tag_id`, `notes.tagging`.`note_id` AS `notes.tagging.note_id`
FROM `tags` AS `tag`
LEFT OUTER JOIN
(
    `taggings` AS `notes.tagging` INNER JOIN `notes` AS `notes`
    ON
    `notes`.`id` = `notes.tagging`.`note_id`
)
ON `tag`.`id` = `notes.tagging`.`tag_id`;
```

这个查询就稍微有点复杂。首先是`notes`和`taggings`进行了一个`inner join`，选出`notes`；然后`tags`和刚`join`出的集合再做一次`left join`，得到结果。

#### 情况3

查询所有满足条件的`note`，同时获取每个`note`所有满足条件的`tag`。

`Sequelize`：

```javascript
var notes = yield Note.findAll({
    'include': [
        {
            'model': Tag
            // 这里可以对tags进行where
        }
    ]
    // 这里可以对notes进行where
});
notes.forEach(function(note) {
    // note的tags可以通过note.tags访问，关系模型通过note.tags[0].tagging访问
    console.log(note);
});
```

`SQL`：

```sql
SELECT
`note`.`id`, `note`.`title`, `note`.`created_at`, `note`.`updated_at`,
`tags`.`id` AS `tags.id`, `tags`.`name` AS `tags.name`, `tags`.`created_at` AS `tags.created_at`, `tags`.`updated_at` AS `tags.updated_at`,
`tags.tagging`.`type` AS `tags.tagging.type`, `tags.tagging`.`created_at` AS `tags.tagging.created_at`, `tags.tagging`.`updated_at` AS `tags.tagging.updated_at`, `tags.tagging`.`tag_id` AS `tags.tagging.tag_id`, `tags.tagging`.`note_id` AS `tags.tagging.note_id`
FROM `notes` AS `note`
LEFT OUTER JOIN
(
    `taggings` AS `tags.tagging` INNER JOIN `tags` AS `tags`
    ON
    `tags`.`id` = `tags.tagging`.`tag_id`
)
ON
`note`.`id` = `tags.tagging`.`note_id`;
```

这个查询和上面的查询类似。首先是`tags`和`taggins`进行了一个`inner join`，选出`tags`；然后`notes`和刚`join`出的集合再做一次`left join`，得到结果。

# 其他没有涉及东西

这篇文章已经够长了，但是其实我们还有很多没有涉及的东西，比如：聚合函数及查询（`having`、`group by`）、模型的验证（`validate`）、定义钩子（`hooks`）、索引等等。

这些主题下次再来写写。

本文采用 [知识共享署名 3.0 中国大陆许可协议](http://creativecommons.org/licenses/by/3.0/cn)，可自由转载、引用，但需署名作者且注明文章出处 。
