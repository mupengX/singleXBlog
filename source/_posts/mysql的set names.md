title: mysql的set names
toc: true
date: 2016-11-13 22:10:35
categories: Database
tags: mysql
---
## 为了Emoji
最近写东西的时候需要支持Emoji表情，用MySQL作存储需要版本5.5.3+并且字符集设置为utf8mb4，由于是跟其他服务公用一个MySQL存储服务所以不能动MySQL的全局配置，在针对数据库和表设置完字符集设置后，应用程序连接数据库时指定default-character-set为utf8mb4会有报错提示（不知道是不是应用程序使用的MySQL驱动不支持的缘故），所以存储Emoji表情的时候还是会提示『Incorrect string value』，最后解决方案是在应用程序连接数据库的时候加上'set names utf8mb4'。
MySQL执行set names utf8mb4后等同于临时设置如下字符编码：
```sql
SET character_set_client = utf8mb4;       

SET character_set_results = utf8mb4;      

SET character_set_connection = utf8mb4;  

```
在对MySQL进行数据插入和查询的时候经过如下的过程：
插入：client→connection→server
查询：server→connection→results

当三者保持一致的时候就不会出现乱码等问题了。