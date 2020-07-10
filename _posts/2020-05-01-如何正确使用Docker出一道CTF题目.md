---
layout:     post                    # 使用的布局（不需要改）
title:      如何正确使用Docker出一道CTF题目               # 标题 
subtitle:    #副标题
date:       2020-05-01              # 时间
author:     Von                      # 作者
header-img: img/post-bg-universe.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Docker
    - 出题

---

# 写在前面
前几天忙着出minil的题目，Frank要求要用Docker并且要支持动态Flag，看了网上也没有一篇文章彻底详细的讲解这个过程，鼓捣了好久才搞明白的，于是写一篇文章记录一下这个过程，也让后来的出题人可以学习一下不用走那么长的弯路。
在之前的内容中，我们已经学习了Docker的基本用法，但当时我们构建的题目，都只能在一个固定的端口上使用，如果我们要实现像每个用户分配一个动态容器，还是动态flag等功能，就要学习Dockerfile等Docker的高级用法。

# Dockerfile
在Docker入门的文章中，我们可以把构建一道题目的过程分为以下具体三步。  
**1.** 指定具体要使用的镜像  
**2.** 启动镜像，构建一个容器  
**3.** 移入相关的源码，构建容器里面的环境配置
在第一篇文章中，我们第三步里面需要进行的操作只有把源码移入/var/www/html文件夹里面而已，但如果环境配置较为复杂，比如需要构建数据库，安装各种插件等，第三步需要的时间就太长了。如果我们改变下上面的步骤。变成：  
**1.** 指定使用的镜像  
**2.** 配置相关的环境，移入相关的代码  
**3.** 根据第二步的内容，把这些操作以类似于代码，程序的模式写入一个模板，让Docker根据这个模板来生成新的镜像  
**4.** 根据这个新的镜像来生成新的容器  
如果是这么操作的话，带来的好处就是可以方便的构造出一个针对性的镜像。配置题目的时候，我们只需要根据这个我们创作的模板生成特制的镜像，直接按照这个镜像就可以直接生成环境了。这个需要的模板就是Dockerfile。  
我们以0ctf的一道题目的[Dockerfile](https://github.com/CTFTraining/0ctf_2016_unserialize/blob/9c624fad8b7dd380b0e653a1f67cadef741db6c9/Dockerfile)为例来看一下基本Dockerfile的构成和语句
``` 
FROM php:5.6-fpm-alpine

COPY files /tmp/

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories \
    && apk add --update --no-cache nginx mysql mysql-client \
    && docker-php-source extract \
    && docker-php-ext-install mysql \
    && docker-php-source delete \
    && mysql_install_db --user=mysql --datadir=/var/lib/mysql \
    && sh -c 'mysqld_safe &' \
	&& sleep 5s \
    && mysqladmin -uroot password 'root' \
    && mysql -e "source /tmp/db.sql;" -uroot -proot \
    && mkdir /run/nginx \
    && mv /tmp/nginx.conf /etc/nginx/nginx.conf \
    && mv /tmp/vhost.nginx.conf /etc/nginx/conf.d/default.conf \
    && mv /tmp/src/* /var/www/html \
    && chmod -R -w /var/www/html \
    && chmod -R 777 /var/www/html/upload \
    && chown -R www-data:www-data /var/www/html \
    && rm -rf /tmp/* \
    && rm -rf /etc/apk

EXPOSE 80
```
我们来依次分析下这些指令。  
**FROM**语句指定了初始的镜像,必需为Dockerfile文件的第一行。  
**COPY**语句表示将当前文件夹下A目录的内容，复制到对应容器里面的B目录中，在此题里面是把当前目录下files目录下文件复制到容器里面/tmp目录中去(只复制files文件夹内的文件内容,files文件夹本身不被复制)。  
在此题中,files目录包含以下内容
![](/blog_img/dockerfile-1.png)
其中src文件夹为题目源码
**RUN**语句可用来执行相关的命令语句，语句内容基本和执行bash命令相同。其中:
```dockerfile
sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories \
    && apk add --update --no-cache nginx mysql mysql-client \
    && docker-php-source extract \
    && docker-php-ext-install mysql \
    && docker-php-source delete \
    && mysql_install_db --user=mysql --datadir=/var/lib/mysql \
```
用来下载基本的php,mysql环境。  
```dockerfile
sh -c 'mysqld_safe &' \
    && sleep 5s \
    && mysqladmin -uroot password 'root' \
    && mysql -e "source /tmp/db.sql;" -uroot -proot \
```
用来配置Mysql环境，为root用户设置了密码root.同时也运行了db.sql文件来配置sql数据库。
```dockerfile
mkdir /run/nginx \
    && mv /tmp/nginx.conf /etc/nginx/nginx.conf \
    && mv /tmp/vhost.nginx.conf /etc/nginx/conf.d/default.conf \
```  
这部分主要是配置nginx环境
```dockerfile
mv /tmp/src/* /var/www/html \
    && chmod -R -w /var/www/html \
    && chmod -R 777 /var/www/html/upload \
    && chown -R www-data:www-data /var/www/html \
```
这部分首先将/tmp/src文件夹下的题目源码移动到/var/www/html中，同时配置相关的权限，权限需要根据题目的具体情况来配置。  
```dockerfile
rm -rf /tmp/* \
    && rm -rf /etc/apk
```
这部分主要是删除之前/tmp文件夹下的相关配置文件，防止被通过非预期解得到Flag。
**EXPOSE**命令是映射到宿主机的端口，这个命令主要体现在当我们启动容器所执行的命令中.
```bash
docker run -d -p 3389:80 tutum/lamp
```
其中3389:80中的80就是EXPOSE所暴露的端口。  

经过上面这些步骤，一个Dockerfile就制作完成啦。我们可以通过
```bash
docker build -t test .
```
来制作一个专属镜像。其中test是镜像的名称,可以任你随便更改。(记得指令最后还有一个.)
# 快速启动容器
从上面的内容里面我们已经可以实现定制特定的镜像来部署题目，但是对于容器的启动，还是存在一定的不方便之处。比如要启动容器，我们要通过shell执行:
```bash
docker run -d -p 1871:80 test
```
来分配特定的端口给容器，我们可以用对待Dockerfile的思想来思考这个问题，即我们可以也制造一个模板，把我们启动容器需要的镜像，要分配的端口号等其他信息写入这个模板中，运行这个模板就可以快速启动容器了。
为了实现这一功能，我们需要了解下Docker Compose.
## Docker Compose
Compose 是用于定义和运行多容器 Docker 应用程序的工具。通过 Compose，您可以使用 YML 文件来配置应用程序需要的所有服务。然后，使用一个命令，就可以从 YML 文件配置中创建并启动所有服务。  
Docker Compose安装过程可以参考[https://www.runoob.com/docker/docker-compose.html](https://www.runoob.com/docker/docker-compose.html)
## docker-compose.yml
docker-compose.yml就是上面我们所提到的模板。我们把要启动的容器镜像、分配的端口号、环境变量等信息写入docker-compose.yml，Docker Compose就可以根据docker-compose.yml的内容来启动一个容器，达到快速启动的效果。我们仍旧来看上面那道题目的docker-compose.yml
```yml
version: "2"

services:
  web:
    build: .
    image: ctftraining/0ctf_2016_unserialize
    environment:
      - FLAG=flag{test_flag}
    ports:
      - "0.0.0.0:1949:80"
```
**version**说明了yml文件指明的版本号，一般我们使用2,3这两个版本。  
**services**就是yml文件的主体，定义了服务了配置。里面的web标签是我们自己定义的。build表明了以dockerfile类型启动一个容器，后面跟的是dockerfile的路径，支持相对路径和绝对路径，在这个yml文件里面，表明dockerfile与yml处在同个目录下。
- 容器的启动也可以根据已有的镜像,如果定义了image这个标签，就会从本地搜寻相关镜像构建容器，如果本地找不到相关的镜像，就会从网上数据库搜寻相关的镜像。但大家可能会产生疑问了，这里我们定义了build还有image两个不同的标签来构建镜像，那么容器到底要用build还是image来构建呢，这种情况下将按照dockerfile的方式来构建镜像，并且把镜像的名称定义为image标签里面的名称。
- environment标签构建了相关的环境变量,在这里我们是定义了一个FLAG环境变量，并且值为flag{test_flag}，这个地方在后面有大用处，我们后面再说
- ports标签定义了映射的端口,0.0.0.0:1949表示映射到本机的1949端口,后面的80端口则要与Dockerfile文件中EXPOSE的端口保持一致。

到这里，我们便完成了docker-compose.yml的构建，此时我们只需要运行
```bash
docker-compose up -d
```
此时访问主机的1949端口,就可以看到题目了。
![](/blog_img/dockerfile-2.png)
# 如何实现动态Flag?
关于动态Flag的实现，一般是通过CTFd的平台插件，为每一个容器生成一个独立的flag,并把这个flag写入环境变量中(这步出题人无需费心)。我们要做的就是要把这个生成的flag替换掉我们题目环境中比如flag.txt,flag.php里面原来的flag。
我们可以用linux的sed命令对文件中的字符串进行替换.
例如我们的flag文件是flag.php,那么我们只要在原来的文件中写为
```php
<?php
flag="FLAG";
>
```
那么我们只需要执行
```bash
sed -i 's/FLAG/flag{test}/' /var/www/html/flag.php
```
就可以把flag.php中的"flag"替换成flag{test}  
那么我们的思路就很明确了，我们写一个docker-php-entrypoint文件，每次启动容器时运行这个文件，用环境变量中随机生成的flag替换掉flag.php中的flag.就可以成功实现目的了。

还是以上面那道题目为例:编写docker-php-entrypoint
```bash
#!/bin/sh
sed -i "s/flag{0ctf_2016_unserialize_is_very_good!}/$FLAG/" /var/www/html/config.php

export FLAG=not_flag
FLAG=not_flag
```
修改Dockerfile
```
FROM php:5.6-fpm-alpine

LABEL Author="Virink <virink@outlook.com>"
LABEL Blog="https://www.virzz.com"

COPY files /tmp/

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories \
    && apk add --update --no-cache nginx mysql mysql-client \
    && docker-php-source extract \
    && docker-php-ext-install mysql \
    && docker-php-source delete \
    && mysql_install_db --user=mysql --datadir=/var/lib/mysql \
    && sh -c 'mysqld_safe &' \
	&& sleep 5s \
    && mysqladmin -uroot password 'root' \
    && mysql -e "source /tmp/db.sql;" -uroot -proot \
    && mkdir /run/nginx \
    && mv /tmp/docker-php-entrypoint /usr/local/bin/docker-php-entrypoint \
    && mv /tmp/nginx.conf /etc/nginx/nginx.conf \
    && mv /tmp/vhost.nginx.conf /etc/nginx/conf.d/default.conf \
    && mv /tmp/src/* /var/www/html \
    && chmod -R -w /var/www/html \
    && chmod -R 777 /var/www/html/upload \
    && chown -R www-data:www-data /var/www/html \
    && rm -rf /tmp/* \
    && rm -rf /etc/apk

EXPOSE 80

CMD ["/bin/sh", "-c", "docker-php-entrypoint"]
```
此时Dockerfile和原版的区别就是增添了对docker-php-entrypoint的处理。此时用了**CMD**指令来进行处理，CMD指令与RUN类似，都是可以执行命令的语句，只不过RUN语句执行在容器构建之时，CMD语句执行在容器构建之后，这里之所以不能用RUN语句，是因为此时容器还没够构建，环境变量里面还没有FLAG这个变量。用RUN语句无法正确执行替换语句。
此时我们再启动容器.
```bash
docker-compose up -d
```
查看下config.php的内容
![](/blog_img/dockerfile-3.png)
![](/blog_img/dockerfile-4.png)
可以看到config.php中的内容，已经被我们替换为docker-compose.yml中环境变量里面的flag. 
至此，我们已经实现了将环境变量中的FLAG替换掉题目环境中的flag,每次启动容器都会由系统平台随机生成环境变量flag,通过自启动docker-php-entrypoint实现了flag的替换，也就达到了动态flag的效果。
# 一个比较好用的镜像
从上面的过程中，我们看到对于一道题目来说，除了源码以外，最大的不方便之处就是还要有相关的nginx文件配置，在这里我推荐virink写的[base_image_nginx_mysql_php_56](https://github.com/CTFTraining/base_image_nginx_mysql_php_56)来辅助我们快速出题。
这个镜像主要好在不需要我们去配置其他的nginx设置，而且还支持自动导入db.sql文件,支持自动执行flag.sh文件。以我在minil出的一道题为例。题目中主要是需要配置数据库.  
我们只需要src文件夹中放入源码、flag.sh、db.sql(flag.sh、db.sql文件名不能变)，现在我们dockerfile只需要这么写:
```
FROM ctftraining/base_image_nginx_mysql_php_56

COPY src /var/www/html

RUN mv /var/www/html/flag.sh / \
    && chmod +x /flag.sh
```
这个镜像就会自动为我们配置相关的nginx文件，同时自动导入要执行的db.sql文件来配置数据库(默认用户为root,密码也为root,执行后db.sql会被删掉)  
同时镜像还会自动执行flash.sh来从环境变量中读取flag写入到flag.php中(需要注意的是，flag.sh必需在根目录下，也就是我们需要执行mv /var/www/html/flag.sh /这一步的原因所在)。
可以看到，这个镜像极大的方便了我们对Dockerfile的编写，把Dockerfile简化到只需要两三句话就能搞定。  
如果是要在php7的环境下出题,可以采用[base_image_nginx_mysql_php_73](https://github.com/CTFTraining/base_image_nginx_mysql_php_73)  
至于python还是java环境的出题,我暂时还没尝试过，就不班门弄斧了。  
如果还想通过更多的环境学习如何出题，可以在这个[项目](https://github.com/CTFTraining/CTFTraining)中查看更多的题目。
