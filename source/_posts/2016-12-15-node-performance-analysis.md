---
layout: post
title: 由一个简单Node服务引申的Node性能分析方法
date: 2016/12/15
toc: true
category:
  - 备忘
tags:
  - 前端
  - Node
  - 性能
---

### 一、问题现象

一个简单Node服务在500并发的压力下长期运行时会出现内存占用不断增长的问题。

### 二、分析过程及方法

#### 1.性能压测工具http_load

[http_load](http://www.acme.com/software/http_load/) 是用来测试web服务器吞吐量和负载的测试工具。使用方法示例如下：

```bash
// 命令行执行
$ ./http_load -rate 5 -seconds 10 urls.txt

// 结果
49 fetches, 2 max parallel, 289884 bytes, in 10.0148 seconds
5916 mean bytes/connection
4.89274 fetches/sec, 28945.5 bytes/sec
msecs/connect: 28.8932 mean, 44.243 max, 24.488 min
msecs/first-response: 63.5362 mean, 81.624 max, 57.803 min
HTTP response codes:
  code 200 -- 49
```

<!--more-->

其中，urls.txt是一个文本文件，保存了需要请求的url连接。可包含的参数含义如下：

|参数|全称|含义|
|:-:|:-:|:-:|
|-p|-parallel|并发的用户进程数。
|-f|-fetches|总计的访问次数
|-r|-rate|含义是每秒的访问频率
|-s|-seconds|连续的访问时间

本文需要500并发的长时间持续压力，可以运行如下的命令：

```
./http_load -p 500 -f 800000 ./urls.txt
```

参考资料：[通过http_load来测试服务器的性能](https://my.oschina.net/chinacaptain/blog/309212)

#### 2.使用node-inpector进行node程序debug

[node-inpector](https://www.npmjs.com/package/node-inspector) 是用于调试node应用程序的交互界面，类似chrome的开发者面板（Blink Developer Tools）。

使用方法：

(1) 使用npm安装node-inpector

```
npm install -g node-inpector
```

(2) 以debug模式运行node应用程序，比如我要运行一个node的服务：

```
$ node --debug ./bin/www
```

(3) 运行node-inpector

```
$ node-inpector

// 结果
Node Inspector v0.12.8
Visit http://127.0.0.1:8080/?port=5858 to start debugging.
```

(4) 使用浏览器打开地址`http://127.0.0.1:8080/?port=5858`，界面如下：

![imge](/images/20161215/node-inspector.png)


#### 3.抓取内存使用堆栈信息

在node-inpector的调试界面，选择`Profiles`选项卡，选择第三项`Record Heap Allocations`类型。其中，第一项是记录CPU运行信息，展示各个js函数运行的时间；第二项是记录内存堆快照；而第三项会记录一段时间内的内存堆信息。

![imge](/images/20161215/profiles.png)

点击start按钮，并使用http_load开启压力测试。运行一段时间之后，点一左上角的红色圆形按钮（运行结束后变为灰色）。捕获到的快照信息展示如下：

![imge](/images/20161215/snapshot.png)

如上图，时间轴中每个时间点的柱状图表示当时申请的内存数量，其中蓝色的部分表示当前使用的内存，灰色的部分表示曾经使用，但后来被释放的内存。时间轴下面是各个对象对内存的占用情况。可以在最下方看到该对象的信息。

可以看到，在时间轴的前部，表示过去时间的部分，还在存在少量蓝色的柱状，这部分就表示长时间没有被回收的内存。如果存在大量内存长期得不到回收，就很有可能发生了内存泄露。

分析方法可参考：[Chrome开发者工具之JavaScript内存分析](http://www.open-open.com/lib/view/open1421734578984.html)

### 三、问题原因及解决方式

问题原因：从内存堆栈分析，长期得不到释放的内存是和winston日志输出相关的。尝试关闭所有日志输出，内存果然不再增长。再多次尝试后发现其实是每次请求都要输出一条日志到本地磁盘，但是由于处理请求的并发数过大，磁盘IO来不及处理而堵塞，日志输出队列越堆越多，长期驻留内存。由于如果请求处理速率不变，磁盘IO一直处于来不及处理的状态，队列只能越来越长，内存就会不断增长。

解决方法：禁止每个请求都输出日志。
