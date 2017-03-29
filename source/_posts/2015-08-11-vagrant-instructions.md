---
layout: post
title: 使用vagrant虚拟机开发调试
category : 备忘
tagline: "备忘"
tags : [vagrant, nfs, fis]
excerpt_separator: <!--more-->
---

### vagrant介绍和简单使用

#### vagrant介绍

百度百科：

>Vagrant是一个基于Ruby的工具，用于创建和部署虚拟化开发环境。

通俗讲vagrant就是用于管理虚拟机的，但是需要依赖虚拟化软件的支持，比如Oracle的VirtualBox（常用）或者VMwave。它还可以配合puppet或者chef自动化配置环境。

#### 安装和下载

- 虚拟机VirtualBox

下载地址：https://www.virtualbox.org/wiki/Downloads

建议使用4.3.12版本，不要安装最新版。安装流程直接按照提示一直下一步。

- 安装Vagrant

<!--more-->

下载地址：https://www.vagrantup.com/downloads.html

也是直接按照提示下一步完成安装。

- 下载所需系统环境对应的box

下载地址：http://www.vagrantbox.es/

本例使用的是名为opscode_centos-6.5-i386_chef-provisionerless.box的box，即环境为centos 6.5 i386。

vagrant可以将一个虚拟机环境打包，供团队内开发者分发使用，保证相同的开发环境，避免重复配置。 所谓的box文件，就是vagrant将虚拟环境打包后生成的包。这里使用他人打包的标准系统环境，vagrant官网列出了可下载的box文件。当然，也可以自己手动从系统镜像安装开始配置系统环境，然后再打包成box文件。

这里需要注意的是，32位的宿主机环境是无法启动64位虚拟机的，但是64位宿主机可以启动32位虚拟机。

#### 初始化vagrant

选择宿主机下的一个目录作为开发目录，同时该目录将默认作为和虚拟机的共享目录。

开启命令行进入该目录，运行vagrant初始化命令“vagrant init”,之后该目录下会生成名为“Vagrantfile”的默认配置文件。

![基本界面](/images/20150811/1.png)

#### 添加一个box

在开发目录下运行命令“vagrant box add base opscode_centos-6.5-i386_chef-provisionerless.box”，其中“base”是该box的名称，“xxx.box”是对应导入的box文件，可以是当前目录下的也可以带路径的，当然也可以是网络下载路径，但是下载可能较慢，建议下载之后导入。

运行“vagrant box list”可以查看当前已添加的box列表。

![基本界面](/images/20150811/2.png)

#### 启动虚拟机

虚拟机启动前首先需要修改配置文件Vagrantfile。这里仅需要关注两个配置项，其他配置项需要使用时参见文件内说明或参见vagrant官网说明。

- 当前虚拟机使用的box

```
# Every Vagrant virtual environment requires a box to build off of.
config.vm.box = "base"
```

其中，base表示对应已添加box列表中的一个box名称。

- 网络配置

```
# Create a forwarded port mapping which allows access to a specific port
# within the machine from a port on the host machine. In the example below, 
# accessing "localhost:8080" will access port 80 on the guest machine. 
# config.vm.network "forwarded_port", guest: 80, host: 8080
 
# Create a private network, which allows host-only access to the machine 
# using a specific IP. 
# config.vm.network "private_network", ip: "192.168.33.10"
 
# Create a public network, which generally matched to bridged network. 
# Bridged networks make the machine appear as another physical device on 
# your network. 
# config.vm.network "public_network"
```

以上分别是端口映射，私有网络，公有网络三种网络配置的注释说明，需要使用哪种网络模式只需去掉代表注释的#符号，然后配置相应地址。

*端口映射*：就是将宿主机的一个端口A映射到虚拟机特定端口B，访问宿主机端口A就是访问虚拟机端口B。
*私有网络*：给虚拟机配置一个IP地址，但是只有宿主机可以使用该地址访问该虚拟机。
*公有网络*：给虚拟机配置一个宿主机所在网段的IP地址，可以虚拟机当作宿主机所在局域网的一台终端，宿主机和该局域网内的其他终端都可以访问该虚拟机。

如果只需在宿主机上访问虚拟机，网络配置使用默认即可，不必添加。vagrant会默认将宿主机的localhost:2222地址映射到虚拟机的22端口，方便ssh远程登录。
 
配置完成后输入命令“vagrant up”启动虚拟机。第一次启动较慢。

![基本界面](/images/20150811/3.png)

可以看到，上图中将默认映射的宿主机2222端口改成了2200，因为演示前已经打开过一个虚拟机，2222端口已经被占用了。

#### 远程登录

安装xshell客户端，配置ssh地址为127.0.0.1:2200并链接即可。登录名和密码都为vagrant。

#### FAQ

如果vagrant启动的时候报如下错误：

```
The following SSH command responded with a non-zero exit status.
Vagrant assumes that this means the command failed!
 
ARPCHECK=no /sbin/ifup eth1 2> /dev/null
 
Stdout from the command:
 
Device eth1 does not seem to be present, delaying initialization.
 
 
Stderr from the command:
```

此时虚拟机是已经启动了的，只要使用ssh登录，在命令行执行如下命令再重启即可。

```
$ sudo rm -f /etc/udev/rules.d/70-persistent-net.rules
```

问题原因：

>问题就处在在持久网络设备udev规则（persistent network device udev rules）是被原VM设置好的，再用box生成新VM时，这些rules需要被更新。而这和Vagrantfile里对新VM设置private network的指令发生冲突。删除再次启动就没问题了。

### 使用nfs服务挂载路由器

#### 安装nfs服务

首先需要在虚拟机中安装nfs服务器。由于windows环境不支持安装nfs服务器，所以不得不需要像上文一样安装虚拟机来使用。

在命令行执行如下命令以安装nfs服务：
```
$ sudo yum install -y nfs-utils rpcbind
```

其中 nfs-utils 是nfs服务的对应安装包，而 rpcbind 是 nfs 的依赖服务，关于 rpcbind 的解释：

>他是一个RPC服务，主要是在nfs共享时候负责通知客户端，服务器的nfs端口号的。简单理解rpc就是一个中介服务。

#### 启动、停止和查询服务状态

- 启动服务

```
$ sudo service rpcbind start
$ sudo service nfs start
```

- 启动服务

```
$ sudo service rpcbind stop
$ sudo service nfs stop
```

- 查询服务状态

```
$ sudo service rpcbind status
$ sudo service nfs status
```

#### 挂载配置

执行如下命令打开文件：

```
$ sudo vi /etc/exports
```

编辑该文件，在文件末尾添加一行：

```
/html/ 192.168.7.1(rw,no_root_squash,no_all_squash,sync)
```

其中 `/html/` 是linux下被挂载的目录。IP `192.168.7.1` 地址是允许挂载的客户机地址，这里就是路由的局域网IP地址。括号中的内容是 读写和权限配置，使用示例里的规则就可以了。

nfs服务每要被一个客户机挂载就需要在 `/etc/exports` 文件中配置一条规则，否则挂载连接会拒绝。

权限规则配置完成后，执行下面的命令使之生效：

```
$ exportfs -r
```

#### 执行挂载

上文配置的规则中，被挂载的目录 `/html/` 如果不存在的话需要手动创建：

```
$ sudo mkdir /html/
```

由于将挂载目录创建在了根目录下，目录需要配置权限才能正常工作，使用chmod 777命令获取全部权限（这么做其实并不合适，慎用）：

```
$ sudo chmod 777 /html
```

现在可以在客户机命令行执行挂载命令了。

创建对应挂载目录：

```
$ mkdir /tmp/nfs
```

挂载：

```
$ mount -t nfs 192.168.1.188:/html /tmp/nfs  -o nolock
```

其中 IP地址 `192.168.1.188` 是服务器弟子，`/html`是服务器上被挂载的目录，即上文创建的 `/html` 目录。`/tmp/nfs` 是客户机上被挂载的目录。

执行成功后，客户机的 `/tmp/nfs` 目录和 `/html` 目录就可以实现共享。路由的本地页面代码可以直接保存到服务器的 `/html` 目录，然后通过路由来访问。 

### 配置fis的远程部署

进行到这里，如果要在本地开发路由器本地配置的web页面还是不方便。每次修改需要手动把文件复制到虚拟机的 `/html` 目录下面。但是项目使用的fis构建工具可以很好的解决这个问题，它既有的远程部署的功能能自动将构建好的文件上传到我们的虚拟机对应目录下。

#### 安装php服务环境

- 安装http服务apache

命令行执行安装命令：

```
$ sudo yum install -y httpd
```

启动服务：

```
$ sudo /etc/init.d/httpd start
```

停止服务：

```
$ sudo /etc/init.d/httpd stop
```

查看服务状态：

```
$ sudo /etc/init.d/httpd status
```

在浏览器输入服务器地址，看到带有类似 `Apache 2 Test Page` 的页面，就表示服务运行正常。

![基本界面](/images/20150811/httpd.png)

- 安装php

命令行执行安装命令：

```
$ sudo yum install -y php php-devel
```

重启apache使php生效

```
$ sudo /etc/init.d/httpd restart
```

在目录 `/var/www/html/` 下建立一个名为 `info.php` 的文件，内容如下：

```
<?php phpinfo(); ?>
```

在浏览器输入 `服务器地址/info.php`，看到带有php版本和配置信息的页面，就表示php环境运行正常。

![基本界面](/images/20150811/php.png)

#### 部署fis的远程文件接收程序

在上文提到的 `/var/www/html/` 目录下新建一个名为 `receiver.php` 的文件，编辑内容如下：

```
<?php
@error_reporting(E_ALL & ~E_NOTICE & ~E_WARNING);
function mkdirs($path, $mod = 0777) {
    if (is_dir($path)) {
        return chmod($path, $mod);
    } else {
        $old = umask(0);
        if(mkdir($path, $mod, true) && is_dir($path)){
            umask($old);
            return true;
        } else {
            umask($old);
        }
    }
    return false;
}
if($_POST['to']){
    $to = urldecode($_POST['to']);
    if(is_dir($to) || $_FILES["file"]["error"] > 0){
        header("Status: 500 Internal Server Error");
    } else {
        if(file_exists($to)){
            unlink($to);
        } else {
            $dir = dirname($to);
            if(!file_exists($dir)){
                mkdirs($dir);
            }
        }
        echo move_uploaded_file($_FILES["file"]["tmp_name"], $to) ? 0 : 1;
    }
} else {
    echo 'I\'m ready for that, you know.';
}
```

在浏览器输入 `服务器地址/receiver.php`，看到 `I'm ready for that, you know.`这样的文字，就表示接收程序已经准备就绪。

![基本界面](/images/20150811/receiver.png)

#### 修改fis配置文件并构建部署

打开fis的项目配置文件 `fis-conf.js`，加入如下的远程部署配置：

```
// fis的配置文件
fis.config.merge({
    // 部署配置
    deploy: {
        remote: {
            receiver: 'http://192.168.7.188/receiver.php',
            to: '/html/'
        }
    }
});
```

构建时使用如下命令：

```
$ fis release -d remote
```

远程部署在上传文件时偶尔会出错失败，原因未知，但是一般再次尝试后成功。

### 使用iptables的nat表将路由暴露到内网

进行这一步，已经能够很方便的进行开发了。但是还有一个问题是路由器设备资源有限，不能做到人手一台。而且整个开发环境是在局域网内搭建，外部网络终端无法直接访问路由器的配置页面，这样也就无法使用 fiddler 这样的工具将代码重定向到本地进行调试开发。

但是 vagrant 是自带端口映射配置的，只要宿主机有一个暴露到外网的网卡和地址，外部终端就可以通过端口映射访问虚拟机。然后再虚拟机中再使用 iptables 加一层端口映射到路由，就能使外部终端可以访问局域网内的路由的本地配置页面。

#### 宿主机映射到虚拟机

首先将宿主机的 11001 端口映射到虚拟机的 11001 端口，修改vagrant工作目录下的配置文件 `Vagrantfile`，添加如下配置：

```
config.vm.network "forwarded_port", guest: 11001, host: 11001
```

控制台执行如下命令重启虚拟机：

```
$ vagrant reload
```

#### 虚拟机映射到路由

虚拟机映射到路由需要使用 linux 下强大的工具 iptables，更加详细的了解可以移步[iptables原理及应用详解](http://www.adintr.com/article/266)，这里只讲用 nat 表进行端口映射的功能。

##### 安装

一般centos系统已经默认安装了iptables，如果没有安装则命令行执行如下安装命令：

```
$ sudo yum install -y iptables
```

设置 iptables 服务开机启动(如果已经能开机启动，略过该步骤)：

```
$ chkconfig iptables on
```

##### 常用命令

打开服务(如果配置文件为空，打开会有警告，配置过规则以后即可正常启动)：

```
$ sudo service iptables start
```

关闭服务：

```
$ sudo service iptables stop
```

查看目前生效的规则：

```
$ sudo service iptables status
```

重启服务：
 
```
$ sudo service iptables restart
```

##### 添加规则

首先清除目前已有的规则：

```
$ sudo iptables -F
$ sudo iptables -X
$ sudo iptables -Z
```

添加规则，修改目的端口为 11001（通过宿主机和虚拟机端口映射能被外网访问的端口） 的 INPUT 包，目的地址修改为 192.168.7.1（路由局域网地址），目的端口修改为 80（路由的http服务器端口）。

```
$ iptables -t nat -A PREROUTING -p tcp --dport 11001 -j DNAT --to-destination 192.168.7.1:80
```

添加规则，修改目的地址为192.168.7.1（路由局域网地址） 目的端口为 80（路由的http服务器端口） 的 OUTPUT 包，源的地址修改为 192.168.7.200（虚拟机的局域网地址）。

```
$ iptables -t nat -A POSTROUTING -d 192.168.7.1 -p tcp --dport 80 -j SNAT --to 192.168.7.200
```

执行如下命令保存规则：

```
$ sudo service iptables save
```

如果配置成功生效，外网访问宿主机的 11001 端口就相当于访问了局域网内路由器的 80 端口，即路由器的http服务。
