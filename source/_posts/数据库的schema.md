title: 数据库的schema
date: 2015-08-16 15:35:19
categories: Database
tags: PostgreSQL
---
schema是对数据库逻辑的分割，schema隶属于数据库。一个数据库可以包含多个schema。

schema可以包含多种命名对象，例如：数据类型、函数等。不同的schema中可以包含相同的对象名而不会冲突。

使用schema的原因主要有：
1. 允许多个用户使用同一个数据库而互不干扰
2. 将数据库对象进行逻辑分组，便于管理
3. 第三方应用放在单独的schema，不与其他对象发生冲突

下面以PostgresSQL为例说一下schema的使用
###创建schema
```sql
CREATE SCHEMA schema_name [ AUTHORIZATION user_name ] [ schema_element [ ... ] ]

```

*  schema_name 
	将要创建的schema的名字，在当前数据库中schema的名字不能冲突。如果忽略此参数，则以当前数据库用户的名字命名新创建的schema。schema的名字不能以pg_开头，这是系统保留的名字。
*  user_name
	拥有新创建schema的角色名字，如果不指定则为执行当前命令的数据库用户。
*  schema_element
	创建schema名字空间下其他对象的SQL语句。与创建schema完毕后执行单独的SQL来创建对象是一样的，除了如果指定AUTHORIZATION,那么新创建的对象都有指定的角色拥有

每个新建数据库包含一个默认的public模式，数据库中没有指定的schema对象都归属于public schema

###查询schema
```sql
postgres-# \dn
  List of schemas
  Name  |  Owner   
--------+----------
 public | postgres
(1 row)

```

###删除schema
如果schema已经为空对象，可以用一下语句删除schema
```sql
DROP SCHEMA myschema;
```
###schema搜索路径
全限定的名字写起来十分冗长，系统通过一个搜索路径来决定到底使用的是哪一张表，搜索路径是schema的一个列表。搜索路径中第一个匹配的表即是要访问的表。如果搜索路径中没有匹配，会报告一个错误，即使在数据库的其他schema中有相匹配的表。

查看当前额搜索路径：
```sql
postgres=# SHOW search_path;
  search_path   
----------------
 "$user",public
(1 row)

```
默认情况下，搜索路径的第一个schema是与当前用户同名的schema,第二个则是public schema。

可以这样设置搜索路径：
```sql
SET search_path TO myschema,public;
```
**官方建议是这样的：**在管理员创建一个具体数据库后，应该为所有可以连接到该数据库的用户分别创建一个与用户名相同的模式，然后，将search_path设置为"$user"，这样，任何当某个用户连接上来后，会默认将查找或者定义的对象都定位到与之同名的模式中。这是一个好的设计架构。

默认情况下，用户不能访问不属于他的schema中的任何对象。要允许访问，schema的拥有者必须授予用户在这个schema上的USAGE权限。不同访问权限需要不同的授权。

###System Catalog Schema
*  如果不创建任何schema，则所有用户隐式的使用public schema。这等同于不使用schema。这在数据库中只有一个或者极少的用户时推荐使用。
*  可以为每一个用户创建一个与其用户名相同的schema。如果每个用户有一个与其名字相同的单独的schema，则默认他们只能访问自己所属的schema。使用这种schema范式，可以撤销掉对public schema的访问许可，甚至把public schema直接移除，这样每个用户就真正的限定在了他们自己的schema里