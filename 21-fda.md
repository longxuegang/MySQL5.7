## 安全相关的特性
#####[原文地址](http://mysql.taobao.org/monthly/2016/05/02/)

### 1 认证插件

mysql.user表中的plugin更改成not null，5.7开始不再支持mysql\_old\_password的认证插件，推荐全部使用mysql\_native\_password。从低版本升级到5.7的时候，需要处理两个兼容性问题。

**\[兼容性\]**  
需要先迁移mysql\_old\_password的用户，然后进行user表结构的升级：

**1. 迁移mysql\_old\_password用户**  
MySQL 5.7.2之前的版本，是根据password的hash value来判断使用的认证插件类型，5.7.2以后的版本，plugin字段为not null，就直接根据plugin来判断了。新的密码从password字段中，保存到新的字段authentication\_string中，password字段废弃处理。

如果user是隐式的mysql\_native\_password。直接使用sql进行变更：

```
UPDATE mysql. user SET plugin = 'mysql_native_password'WHERE plugin = ''AND (Password = ''OR LENGTH(Password ) = 41 ); FLUSH PRIVILEGES ;
```

如果user是隐式的或者显示的mysql\_old\_password， 首先通过以下sql进行查询:

```markdown
SELECT User , Host, Password FROM mysql. user WHERE (plugin = ''AND LENGTH(Password ) = 16 ) OR plugin = 'mysql_old_password';
```

如果存在记录，就表示还有使用mysql\_old\_password的user，使用以下sql进行用户的迁移：

```
ALTER USER 'user1'@ 'localhost'IDENTIFIED WITH mysql_native_password BY 'DBA-chosen-password';
```

**2. user表结构升级**  
通过mysql\_upgrade直接进行升级，步骤如下\[5.6-&gt;5.7\]：

1. stop MySQL 5.6实例
2. 替换5.7的mysqld二进制版本
3. 使用5.7启动实例
4. run mysql\_upgrade升级系统表
5. 重启MySQL 5.7实例

### 2 密码过期

用户可以通过`ALTER USER 'jeffrey'@'localhost' PASSWORD EXPIRE;`这样的语句来使用户的密码过期。  
并新增加 default\_password\_lifetime来表示用户密码自动过期时间，从5.7.10开始，其默认值从0变更到了360，也就是默认一年过期。  
可以通过以下两种方法禁止过期：

```
1. SET GLOBAL default_password_lifetime = 0 ;
2. ALTER USER 'jeffrey'@ 'localhost'PASSWORD EXPIRE NEVER;
```

**\[兼容性\]**  
只需要通过mysql\_upgrade升级mysql.user系统表就可以使用密码过期新功能。

### 3 账号锁定

用户可以通过以下语法进行账号锁定，阻止这个用户进行登录：

```
ALTER USER 'jeffrey'@ 'localhost'ACCOUNT LOCK ; ALTER USER 'jeffrey'@ 'localhost'ACCOUNT UNLOCK ;
```

**\[兼容性\]**  
只需要通过mysql\_upgrade升级mysql.user系统表就可以使用密码过期新功能。

### 4 SSL连接

如果mysqld编译使用的openssl，在启动的时候，默认创建SSL， RSA certificate 和 key 文件。  
但不管是openssl还是yassl，如果没有设置ssl相关的参数，mysqld都会在data directory里查找ssl认证文件，来尽量打开ssl特性。

**\[兼容性\]**  
不存在兼容性的问题

### 5 安装数据库

5.7开始建议用户使用`mysqld --initialize`来初始化数据库，放弃之前的mysql\_install\_db的方式，新的方式只创建了一个root@localhost的用户，随机密码保存在~/.mysql\_secret文件中，并且账号是expired，第一次使用必须reset password，并且不再创建test db。

**\[兼容性\]**  
不存在兼容性的问题

