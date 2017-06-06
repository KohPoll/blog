title: 详解 sequelize 中的关系
date: 2017-05-15
tags:
---

我在之前的[文章](https://kohpoll.github.io/blog/2015/11/12/sequelize-to-mysql/)中介绍过 [sequelize](http://sequelize.readthedocs.io/en/v3/) 这个 ORM 框架。鉴于当时的篇幅限制，对于“关系”这部分描述的不是很细致，有些情况也没有涉及到。这篇文章将单独对“关系”这部分内容进行详细的介绍。希望对你有帮助。

<!-- more -->

# 概述
模型之间的关系一般有三种：一对一、一对多、多对多。在实际执行时，ORM 框架一般都会“转换”为 SQL JOIN 语句。JOIN 在数据量较小时可以放心使用。数据量大时有可能会比较耗资源，请谨慎使用，建议在应用层对数据分别查询后进行 join。

在接下来的内容，我们将使用下面的数据库表和模型定义来进行讲述：

```sql
CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `nick` varchar(32) NOT NULL,
  `created_at` datetime DEFAULT NULL,
  `updated_at` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `nick_UNIQUE` (`nick`)
);

CREATE TABLE `profiles` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `signature` text NOT NULL,
  `fk_user_id` int(11) DEFAULT NULL,
  `created_at` datetime DEFAULT NULL,
  `updated_at` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
);
```

```javascript
const User = sequelize.define('user', { // -> users 表
  'nick': {
    'type': Sequelize.STRING(32),
    'allowNull': false,
    'unique': true
  }
});
const Profile = sequelize.define('profile', { // -> profiles 表
  'signature': {
    'type': Sequelize.TEXT(),
    'allowNull': false
  }
});
```

# 一对一
![]()

```javascript
User.hasOne(Profile, {'foreignKey': 'fk_user_id'});
Profile.belongsTo(User, {'foreignKey': 'fk_user_id'});
```

一对一的关系比较简单，

# 一对多

# 多对多

