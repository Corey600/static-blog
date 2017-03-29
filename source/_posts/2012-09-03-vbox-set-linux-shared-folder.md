---
layout: post
title: VBox中设置Linux的共享文件夹
category : 备忘
tagline: "备忘"
tags : [Linux, VBox]
excerpt_separator: <!--more-->
---

1.首先在Vbox的菜单选项中控制=>设置=>共享文件夹中分配一个共享文件夹在主机下的路径，我在WIin8下的路径是``E:\Program Files\Oracle\shared``，设置了共享文件夹名称为shared，这个名称在Linux中挂载时输入命令时用到。

![Alt text](/images/20120903/1.jpg)

<!--more-->

2.在Vbox的菜单选项中设备=>安装增强功能，然后你会发现虚拟机中挂载了一个名叫VBOXADDITIONS_xxx的盘，具体名字根据Vbox版本而定。在虚拟机路径``/media/``下出现一个同名文件夹，这个就是挂载的位置。在该路径下输入命令``#sudo ./VBoxLinuxAdditions.run``具体的文件名根据实际情况有所区别。这样安装增强功能就结束了。

![Alt text](/images/20120903/2.jpg)

3.在虚拟机路径``/home/用户名/``下建一个对应的文件夹，注意这里的文件夹名不能和共享文件夹取相同名字，这里我就取名share。命令行输入``#mount -t vboxsf shared /home/fcx/share``(我的用户文件夹是fcx)，这样共享文件夹就设置完成了。可以弹出卷 VBOXADDITIONS_xxx 了。

4.开机自动挂载。vim打开``/etc/rc.local``文件把挂载命令写在exit 0之前，保存。