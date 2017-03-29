---
layout: post
title: Linux命令行总汇(20120922更新)
category : 备忘
tagline: "备忘"
tags : [Linux]
excerpt_separator: <!--more-->
---

#### 1.修改全局环境变量操作

运行以下命令

	$ getdit /etc/profile

在``/etc/profile``最后加上``export PATH=$PATH:[绝对路径]``，然后运行以下命令使之生效

	$ source /etc/profile

##### 小技巧：
其实如果打开``/etc/profile``，在最后我们可以发现有这样一段代码：

	if [ -d /etc/profile.d ]; then
	  for i in /etc/profile.d/*.sh; do
	    if [ -r $i ]; then
	      . $i
	    fi
	  done
	  unset i
	fi

<!--more-->

也就是说，在``/etc/profile``执行的最后，它会自动的执行``/etc/profile.d``目录下的所有可读的文件。这就是我们将设置__JDK__环境变量的工作放在``/etc/profile.d/development.sh``的原因，可以不用修改操作系统自带的``/etc/profile``内容，方便系统的移植。

#### 2.察看环境变量操作

	$ echo $PATH

#### 3.解决ubuntu 12.04下TXT乱码问题

	$ gsettings set org.gnome.gedit.preferences.encodings auto-detected "['UTF-8','GB18030','GB2312','GBK','BIG5','CURRENT','UTF-16']"

#### 4.Ubuntu更改文件夹权限

	$ sudo chmod 600 ××× （只有所有者有读和写的权限）
	$ sudo chmod 644 ××× （所有者有读和写的权限，组用户只有读的权限）
	$ sudo chmod 700 ××× （只有所有者有读和写以及执行的权限）
	$ sudo chmod 666 ××× （每个人都有读和写的权限）
	$ sudo chmod 777 ××× （每个人都有读和写以及执行的权限）

其中×××指文件名（也可以是文件夹名，不过要在chmod后加-ld）。命令的形式是

	$ sudo chmod -（代表类型）×××（所有者）×××（组用户）×××（其他用户）

三位数的每一位都表示一个用户类型的权限设置。取值是0～7，即二进制的[000]~[111]。这个三位的二进制数的每一位分别表示读、写、执行权限。如000表示三项权限均无，而100表示只读。这样，我们就有了下面的对应：

0 [000] 无任何权限    
4 [100] 只读权限    
6 [110] 读写权限    
7 [111] 读写执行权限    
#### 5.查看内核版本

	$ cat /proc/version

#### 6.修改目录所有者

	$ chown 用户名 文件名

#### 7.删除非空目录

	$ rm -rf 目录名        ;其中参数-f表示force.

#### 8.执行make menuconfig可能看到如下这样的错误

	*** Unable to find the ncurses libraries or the
	*** required header files.
	*** ‘make menuconfig’ requires the ncurses libraries.
	*** Install ncurses (ncurses-devel) and try again.

Ubuntu上解决办法如下：

	$ sudo apt-get insatll ncurses-dev
