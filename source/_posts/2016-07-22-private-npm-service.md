---
layout: post
title: 私有NPM服务搭建
category : 备忘
tagline: "备忘"
tags : [前端,npm,cnpm]
excerpt_separator: <!--more-->
---

### 简介与背景

由于官方的NPM库是完全开放的，任何人都可以浏览和下载，所以公司私有的、不便对外公开的node模块就没法上传和下载，也无法使用NPM的版本管理功能。这就需要在公司内部建立私有的NPM服务。

目前官方给出的解决方案是 [npm-registry-couchapp](https://github.com/npm/npm-registry-couchapp) + couchDB 的解决方案。但是couchDB的安装配置过程相对来说有点复杂。这里使用较简单快速的方法，也就是使用alibaba团队的 [cnpmjs.org](https://github.com/cnpm/cnpmjs.org) 的方案。

cnpmjs.org 依赖环境如下，数据库只需要选择其中一种就可以：

<!--more-->

>node >=0.11.12, use --harmony    
>Databases: only required one type    
>- sqlite3 >= 3.0.2, we use sqlite3 by default    
>- MySQL >= 0.5.0, include mysqld and mysql cli. I test on mysql@5.6.16.    
>- MariaDB    
>- PostgreSQL    

下文具体介绍安装部署过程，环境为 `CentOS 6.6` 。


### MySQL服务安装与配置

这里选择较为熟悉的 MySQL 来作为数据库服务。使用yum安装名利如下：

```
# sudo yum install -y mysql mysql-server
```

MySQL 的服务名为 `mysqld` ，服务操作命令：

```
// 开启服务
# sudo service mysqld start
// 停止服务
# sudo service mysqld stop
// 重启服务
# sudo service mysqld restart
// 查看服务
# sudo service mysqld status
```

设置开启启动：

```
# sudo chkconfig --list | grep mysqld
# sudo chkconfig mysqld on
```

MySQL 默认有一个 `root` 用户，密码为空，拥有全部操作权限，但是只能在本地登录。登录命令：

```
# mysqsl -u root
```

为了安全性，先给 `root` 用户设置一个密码再登录：

```
// 修改密码
# mysqladmin -u root password 'new-password'
// 登录
# mysqsl -u root -pnew-password
```

注意这里 `-p` 之后直接跟上密码，不要有空格。

如果要指定地址和端口登录：

```
# mysqsl -h 127.0.0.1 -P 3306 -u root -ppassword
```

登录之后首先需要创建名为 `cnpmjs` 的数据库，这里的 `cnpmjs` 的名字可以自定，但是必须和下文在 cnpmjs.org 服务中配置的数据库名字一样。

```
// 创建数据库
> create database cnpmjs;
// 查看已有数据库
> show databases;
// 更改当前使用数据库为 cnpmjs
> use cnpmjs;
```

这里说明 `root` 用户创建的数据库对其他用户是不可见的，需要将该数据库的权限配置给其他用户才可以。这里直接使用 `root` 用户，不需要这一步。


### cnpmjs.org 安装与配置

首先需要到 github 地址 [https://github.com/cnpm/cnpmjs.org](https://github.com/cnpm/cnpmjs.org) 下载源码，可以选择直接打包下载，也可以使用 `git clone` 下来。

解压进入源码目录。运行安装命令：

```
# sudo make install
```

在安装前请保证本机的 `g++` 本版在 `4.8.1` 以上，否在在编译某些依赖的 npm native 模块时会无限出错。

安装完成后可以尝试运行 `make test` 命令测试。

然后 cp 一份 `config/index.js` 到 `config` 目录下，以自定义配置：

```
# cp config/index.js config/config.js
// 使用vim编辑配置
# vim config/config.js
```

需要修改的主要有如下几项。

#### 1. 数据库设置

```
database: {
    db: 'cnpmjs',     // 使用的数据库名称
    username: 'root', // 用户民
    password: 'root', // 密码

    // the sql dialect of the database
    // - currently supported: 'mysql', 'sqlite', 'postgres', 'mariadb'
    dialect: 'mysql',

    // custom host; default: 127.0.0.1
    host: '127.0.0.1',

    // custom port; default: 3306
    port: 3306,
},
```

#### 2. 管理员设置

cnpmjs.org 默认只允许管理员用户 `publish` 模块至服务器。

```
admins: {
    // name: email
    fcx: 'fcx600@163.com'
},
```

#### 3. 允许启动的地址

如果这一项配置为 `127.0.0.1` 则只允许本地访问 npm 服务

```
bindingHost: '',
```

#### 4. 作用域设置

作用域是 npm 2.0 版本以后引入的概念.

>作用域是一种对相关包分组的办法，这样所有属于同一作用域的包都会装在相同的目录node_modules_base_dir/@myScope下，而公共包会装在node_modules_base_dir中。限定作用域的包跟其它包一样有一个名称；此外它还有作用域，用下面这种方式指定：
> `@somescope/somepackagename`

此项设置限制只有如下作用域的模块才被允许上传。

```
// registry scopes, if don't set, means do not support scopes
scopes: [ '@scopes', '@scope' ],
```

#### 5. 同步模式设置

同步模式决定私有 npm 库和 官方 npm 库的同步机制，这里设置为 `none` 。我们没有必要全量同步官方库，下文会描述一般模块使用官方库或者 [https://registry.npm.taobao.org](https://registry.npm.taobao.org)， 而私有模块使用私有库的方法。

```
// sync mode select
// none: do not sync any module, proxy all public modules from sourceNpmRegistry
// exist: only sync exist modules
// all: sync all modules
syncModel: 'none', // 'none', 'all', 'exist'
```

### 启动服务

启动服务前还需要初始化数据库表。进入 cnpmjs.org 源码目录，登录 MySQL 数据库，运行如下命令：

```
// 使用名为 cnpmjs 的数据库
mysql> use cnpmjs;
// 运行初始化脚本
mysql> source docs/db.sql
```

然后退出数据库连接，运行命令：

```
# npm run start
```

启动成功后，尝试访问：

```
// registry地址
# curl http://localhost:7001
// web 浏览地址
# curl http://localhost:7002
```

### cnpm客户端使用

使用npm安装cnpm模块：

```
# npm install -g cnpm --registry=https://registry.npm.taobao.org
```

这里不要直接修改全局 `registry` ，因为我们还要用默认的库来安装一般模块。单独给作用域设置 `registry` 后，只有这些作用域的包才会使用我们设置的私有npm。

单独给设置作用域设置私有npm库的 `registry`:

```
# cnpm config set @myco:registry http://localhost:7001
```

其中 `@myco` 是作用域，`http://localhost:7001` 是私有npm的 `registry` 地址。之后我们就可以使用 `cnpm install` 来安装依赖的，一般模块会使用默认库安装，作用域为 `@myco` 的模块会使用私有npm库安装。

如果要给私有npm库上传模块，首先要在服务配置里增加管理员，在cnpm使用管理员帐户登录：

```
# cnpm login --registry=http://localhost:7001
```

然后在模块的根目录下运行 `cnpm publish` 。

需要查看当前配置情况，可以查看用户目录下的 `.cnpmrc` (对于cnpm)或 `.npmrc` (对于npm)文件。

参考： [npm-scope](https://docs.npmjs.com/misc/scope)
