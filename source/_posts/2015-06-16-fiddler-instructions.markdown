---
layout: post
title: Fiddler使用说明
category : 备忘
tagline: "备忘"
tags : [前端,Fiddler]
excerpt_separator: <!--more-->
---

### 介绍

>Fiddler是一个http协议调试代理工具，它能够记录并检查所有你的电脑和互联网之间的http通讯，设置断点，查看所有的“进出”Fiddler的数据（指cookie,html,js,css等文件，这些都可以让你胡乱修改的意思）。

### 下载和安装

[下载地址](http://www.telerik.com/download/fiddler) [安装过程](http://docs.telerik.com/fiddler/Configure-Fiddler/Tasks/InstallFiddler)

### 基本界面

![基本界面](/images/20150616/f1.jpg)

- 监听开关：开启或者关闭。`capturing`表示开启状态。
- 监听类型：有 监听所有进程请求(All processes)、监听浏览器请求(Web browsers)、监听非浏览器请求(Non-Browser)和隐藏所有(Hide All)
- 请求列表：请求列表的信息分别有 结果(Result)，协议(Protocol)，主机名(Host)，网页地址(URL)，内容大小(Body)，缓存(Caching)，响应的HTTP内容类型(Content-Type)，请求所运行的程序(Process)，注释(Comments)，自定义(Custom)
- 功能区：包括各种数据流查看器、日志、重定向、请求构造、过滤器、时间轴、统计图表、脚本。

<!--more-->

###### 以下只讲具体功能操作，其他细节请[参阅官方文档](http://docs.telerik.com/fiddler/configure-fiddler/tasks/configurefiddler)和[使用搜索](http://www.bing.com/explore/)

### 抓取并解密HTTPS请求

Fiddler是可以抓取HTTPS的请求，并解密的。但是浏览器会检查数字证书，经过Fiddler解密的请求会被浏览器认为是遭到窃听的。为此，Fiddler通过一个自己的数字证书重新加密HTTPS请求。设置Fiddler为抓取并解密HTTPS请求时，会自动生成一个名为`DO_NOT_TRUST_FiddlerRoot`的CA证书，并能通过该CA证书生成各个域名的TLS证书。只要将`DO_NOT_TRUST_FiddlerRoot`证书设置为浏览器或其他软件的信任CA证书，浏览器就会认为被Fiddler重新加密的HTTPS请求是可信的，不会再提示“证书错误警告”。

设置Fiddler抓取和解密HTTPS请求的方法：

#### 1.开启HTTPS抓取

打开Fiddler选项窗口：`Tools > Fiddler Options > HTTPS`
选择HTTPS标签页，选中`Capture HTTPS CONNECTs` 和 `Decrypt HTTPS traffic` 两个复选框。

![基本界面](/images/20150616/f2.png)

#### 2. 设置CA证书

在开启HTTPS抓取时Fiddler会弹框提示将证书加入浏览器的信任列表，如果只是用于抓取和解密浏览器的HTTPS请求，只要更具提示点击就可以了。但是如果需要抓取和解密其他应用的HTTPS请求，则需要将证书导入系统的信任列表。

点击上图中的`Export Root Certificate to Desktop`按钮，会在桌面生成一个名为`FiddlerRoot.cer`的证书文件：

![基本界面](/images/20150616/f3.bmp)

然后win键+R打开运行窗口输入mmc打开如下窗口：

![基本界面](/images/20150616/f4.png)

选择添加/删除管理单元，选择证书添加到右侧并确定。

![基本界面](/images/20150616/f5.png)

右击`受信任的根证书颁发机构 > 证书` 导入，在弹出框内选择上文生成的桌面上名为 `FiddlerRoot.cer` 的证书文件，然后确定。

![基本界面](/images/20150616/f6.png)

下一步至完成，在如下图列表中找到颁发给`DO_NOT_TRUST_FiddlerRoot`的项时表示导入成功。

![基本界面](/images/20150616/f7.png)

手动导入证书到其他浏览器的过程类似：
- IE:  Internet选项 > 内容 > 证书 > 导入
- chrome:  设置 > 显示高级设置 > HTTPS/SSL > 管理证书 > 导入
- firefox:  选项 > 高级 > 证书 > 查看证书 > 导入

#### 3. 开始监听HTTPS

设置完成之后重启Fiddler开始监听请求，HTTPS请求就能出现在请求列表中了。但是，直接点击HTTPS请求在查看器中body的内容可能是乱码，点击黄色的提示文字就能正常显示了。

![基本界面](/images/20150616/f8.png)

#### 4. 添加例外

如下图有按类型删选和按主机名添加例外两种方式。

![基本界面](/images/20150616/f9.png)

#### 5. 参考链接

[如何配置能让fiddler抓去https的请求？](http://wangsheng14591.blog.163.com/blog/static/3277971020130465730354/)
[在服务器上用Fiddler抓取HTTPS流量](http://yoursunny.com/t/2011/FiddlerHTTPS/)
[fiddler https](http://www.cnblogs.com/yelaiju/archive/2011/09/12/2173893.html)

### 重定向线上JS脚本

#### 1. 找到调试需要修改的文件

先要在请求列表中找到需要修改和调试的JS源文件的对应请求。建议始终关闭缓存，以便修改能及时更新，然后使用过滤器过滤需要的请求。

- 始终关闭缓存

暂时关闭缓存可以在菜单栏中设置： `Rules > Performance > Disable Caching`
需要保存设置在下次开启时有效可以设置Fiddler脚本，打开功能区的FiddlerScript标签，搜索关键字`m_DisableCaching`，上下文代码如下：

```
// Cause Fiddler to delay HTTP traffic to simulate typical 56k modem conditions
public static RulesOption("Simulate &Modem Speeds", "Per&formance")
var m_SimulateModem: boolean = false;
 
// Removes HTTP-caching related headers and specifies "no-cache" on requests and responses
public static RulesOption("&Disable Caching", "Per&formance")
var m_DisableCaching: boolean = false;
 
public static RulesOption("Cache Always &Fresh", "Per&formance")
var m_AlwaysFresh: boolean = false;
 
// Force a manual reload of the script file.  Resets all
// RulesOption variables to their defaults.
public static ToolsAction("Reset Script")
function DoManualReload() {
    FiddlerObject.ReloadScript();
}
```

将`var m_DisableCaching: boolean = false;`中的值`false`修改为`true`，然后点击左上角的`Save Script`按钮保存即可。

- 使用过滤器筛选文件

过滤器是左侧功能区的Filter标签页，选中`Use Filters`复选框。这里使用URL删选出包含`.js`后缀的文件，按照如下图设置：

![基本界面](/images/20150616/f10.png)

开启筛选后仅对之后的请求有效，要想筛选已经记录的请求，单击右上角`Actions`按钮选择`Run Filterset now`。

#### 2. 下载调试文件

找到需要调试的文件对应的请求后，右击该请求项，选择`Svae > Response > Response Body...`，将文件保存到桌面。

![基本界面](/images/20150616/f11.png)

#### 3. 重定向请求到本地文件

将上文保存的文件的对应请求直接拖到`AutoResponder`标签页，在下面的`RuleEditor`的第二个编辑框内下拉选择`Find a file...`，定位到上文保存到桌面的文件，点击`Save`按钮。
然后选中左上角的`Enable automatic responses`和`Unmatched requests passthrough`两个复选框。前者是开启重定向，后者是允许不匹配的请求直接放行。

![基本界面](/images/20150616/f12.png)

#### 4. 修改和运行修改的文件

打开请求重定向的那个本地文件，做一下修改，比如我将百度首页的一个js脚本全部清空，并加入自己的代码 ：

```
var name = '假装是谷歌';
window.document.title = name;
```

刷新百度主页，发现原先请求的脚本变成了本地已经修改过的js脚本，并且运行成功。

![基本界面](/images/20150616/f13.png)

#### 5. 使用重定向伪造数据

重定向功能即能定向脚本文件，其实能截获和修改任何http请求，包括Ajax请求的后台接口，可以将想要接口返回的json数据保存成一个文件，然后使用`Find  a file...`定位到该文件。

截获的http请求特征还可以使用正则表达式来匹配。以`regex:`是使用正则表达式匹配，`method`删选特定的请求方法等等。具体使用可以参考下拉显示的例子。右侧蓝色`Test..`点开可以测试正则表达式。

![基本界面](/images/20150616/f14.png)

#### 6. 参考链接

[如何使用Fiddler调试线上JS代码](http://www.cnblogs.com/RockLi/p/3511132.html)
[Fiddler - 前端开发值得拥有](http://www.cnblogs.com/Darren_code/archive/2011/09/28/Fiddler.html)
[使用Fiddler提高前端工作效率 (实例篇)](http://www.2cto.com/kf/201308/234828.html)

### 其他功能

除了上文这种最常用的功能，Fiddler还有其他很多功能，比如：

查看器、过滤器、请求构造、搜索数据流、比较数据流、作为远程终端的代理服务器并抓包、文本编码和解码、SAZ数据流录制、使用FiddlerScript脚本……

### 参考资料

[使用Fiddler提高前端工作效率.pdf](http://www.open-open.com/doc/view/07de1e1bb7874585a35a6fa7339dd890)
[用Fiddler模拟低速网络环境](http://caibaojian.com/fiddler.html)
[使用Fiddler提高前端工作效率 (实例篇)](http://www.aliued.cn/2010/04/25/use-fiddler-to-improve-efficiency-of-front-development-example.html)
[Fiddler 教程](http://kb.cnblogs.com/page/130367/)
[Fiddler实战深入研究(二)](http://web.jobbole.com/82710/)
[Fiddler (五) Mac下使用Fiddler](http://blog.csdn.net/codenewbie/article/details/17162189)

### QA

#### 1. 在用Fiddler调试本机的网站时，返回如下错误:

>[Fiddler] Connection to localhost. failed.Exception Text: No connection could be made because the target machine actively refused it

解决方法：
打开Fiddler，菜单>Fiddler Options>General>Enable IPv6(if avaible)去掉该选项。

参考链接：
[Fiddler Connection to localhost. failed. ](http://blog.chinaunix.net/uid-20675015-id-1899931.html)
[Using Fiddler with ASP.NET's default local server](http://weblogs.asp.net/mikebosch/archive/2007/10/09/using-fiddler-with-asp-net-s-default-local-server.aspx)

#### 2. 怎样代理本地web服务器的访问请求

Fiddler是捕捉不了本地http请求的。一般访问安装在本地的服务器程序时，会使用的localhost或127.0.0.1地址，但是这样默认会绕过代理，直接访问目标服务器。

解决方法：
使用Fiddler特有的请求方式，可以使本地请求及响应都被fiddler拦截。即在localhost后增加.fiddler或者仅增加一个点。比如把请求`http://localhost:8080`改为`http://localhost.:8080`或者`http://localhost.fiddler:8080`即可。
 
参考链接：
[Fiddler初次使用时应注意的几点](http://www.2cto.com/kf/201308/234833.html)

#### 3. 将请求响应的body保存为文件时偶尔有乱码怎么办

解决方法：
win7：windows按钮+R，输入regedit。在注册表的`HKEY_CURRENT_USER\Software\Microsoft\Fiddler2`项上，右击新建，选字符串值，加上`HeaderEncoding`，然后值输入`GBK`

参考链接：
[Fiddler2中文乱码问题](http://thinktothings.iteye.com/blog/1139336)
