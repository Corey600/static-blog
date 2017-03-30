---
layout: post
title: 用Github和Octopress搭建博客
date: 2013/02/28
toc: true
category: 备忘
tags: [Github, Octopress]
---

最先玩过[网易博客](http://cheesefan.blog.163.com/),但是网易博客广告太多（虽然相比同类型的第三方托管博客广告要少很多，这也是最初选择它的原因），同时定制性太差，代码高亮也在部分主题下面也显示得不是太好。后来分别在[新浪SAE](http://sae.sina.com.cn/)和[百度BAE](http://developer.baidu.com/bae)上搭建过[WordPress](http://cn.wordpress.org/)的博客，[WordPress](http://cn.wordpress.org/)有丰富的主题插件资源，但正是由于它功能太过完善，显得它有点庞大，加载速度让我难以接受。

后来得知[Github](https://github.com/)提供有[GitHub Pages](http://pages.github.com/)功能，可以搭建静态博客，然后尝试了一下确实满足了博主们的最大自由性，因为博客的所有源码都是由你管理可以随意修改的。当然，最重要的是[Github](https://github.com/)免费与稳定性并存，有钱购买域名和空间的自然是选择商业的服务器空间更合适，但相对个人作为交流性的博客，[GitHub Pages](http://pages.github.com/)已经足够了。

<!--more-->

当然，静态博客在简约的同时，我还是希望它能够在主题上好看协调一点，而且在写作上如果直接写html文件的话就显得十分吃力，这时候[OctoPress](http://octopress.org/)和[MarkDown](http://wowubuntu.com/markdown/)，这两位左右手就该上场了。前者是开源的静态博客系统，后者是是一种轻量级的标记语言。这三者互相搭配再加上第三方评论系统就基本能搞定像本博一样的静态博客了。

_以下是以windows7作为环境的全部安装过程_

#### 一、安装软件和Octopress博客系统

[RubyInstaller下载地址](http://rubyforge.org/frs/download.php/75127/rubyinstaller-1.9.2-p290.exe)/[DevKit下载地址](https://github.com/downloads/oneclick/rubyinstaller/DevKit-tdm-32-4.5.2-20111229-1559-sfx.exe)/[Git下载地址](https://msysgit.googlecode.com/files/Git-1.7.10-preview20120409.exe)

RubyInstaller的安装界面

![Alt text](/images/20130228/rbiu.jpg)

DevKit只需解压到一个文件夹

![Alt text](/images/20130228/dkiu.jpg)

然后启动Ruby命令框，用CD进入存放DevKit的目录，执行以下命令

```
ruby dk.rb init
ruby dk.rb install
```

![Alt text](/images/20130228/dkiu2.jpg)

Git安装只需按照提示默认点击下一步，在“Configuring the line ending conversions”界面选择换行格式，选择“Checkout as-is, commit Unix-style line endings”(参考链接：[Win7上Git安装及配置过程](http://www.cnblogs.com/sunny5156/archive/2012/10/23/2735799.html))，如下图

![Alt text](/images/20130228/gitiu.jpg)

安装Octopress前要改变一个软件更新源，运行命令

```
gem sources -a http://ruby.taobao.org/
gem sources -r http://rubygems.org/
gem sources -l
```

![Alt text](/images/20130228/ociu.jpg)

安装Octopress到D:\Blog(参考链接：[Octopress官方安装帮助](http://octopress.org/docs/setup/))，打开Git Bash，运行命令

```
git clone git://github.com/imathis/octopress.git /d/Blog
```

![Alt text](/images/20130228/ociu2.jpg)

打开Octopress安装目录下的``D:\Blog\Gemfile``，将第一行的source改成国内淘宝的``http://ruby.taobao.org/``

![Alt text](/images/20130228/ociu3.jpg)

进入Octopress安装目录，安装bundler，运行命令

```
gem install bundler
bundle install
```

![Alt text](/images/20130228/ociu4.jpg)

安装Octopress默认的主题，运行命令

```
rake install
```

![Alt text](/images/20130228/ociu5.jpg)

最后是生成和预览博客，运行命令

```
rake generate
rake preview
```

![Alt text](/images/20130228/ociu6.jpg)

然后浏览器打开页面``http://localhost:4000/``就可以预览博客了(参考链接：[用Octopress免费静态博客系统在Github免费空间上搭建个人网站](http://www.freehao123.com/octopress/))。

#### 二、更改环境变量以支持生成有中文的文章

打开计算机–属性–高级系统设置–环境变量，新增``LANG``和``LC_ALL``，值都是``zh_CN.UTF-8``，完成后确定保存

![Alt text](/images/20130228/hj.jpg)

#### 三、将博客提交到Github

先申请一个[Github](https://github.com/)账号，再点击右上角用户名旁边的create a new repo创建一个项目(参考链接：[免费开源Github Pages空间可绑域名搭建个人博客存放图片文件](http://www.freehao123.com/github-pages/))，注意repository name必须是``你的用户名.github.com``，这样最后才能直接用``你的用户名.github.com``这个二级域名直接访问你的页面

![Alt text](/images/20130228/gh.jpg)

打开git bash，依次运行下列命令

```
git config --global user.name "用户名"
git config --global user.email "邮箱"
git config --global credential.helper cache
```

![Alt text](/images/20130228/gh2.jpg)

如入命令``cd ~/.ssh``看看本地有没有ssh keys。如果没有的话运行命令``ssh-keygen -t rsa -C "邮箱"``来创建SSH Keys。然后会要你选择保存的位置，直接回车就可。输入密码后创建成功。然后在你刚才保存的文件路径中产生一个id_rsa.pub文件。

![Alt text](/images/20130228/gh3.jpg)

用记事本打开id_rsa.pub，复制里面的东西粘贴到Github项目的SSH Keys中

![Alt text](/images/20130228/gh4.jpg)

输入命令``ssh -T git@github.com``，输入密码后测试是否可以成功连接。如下图就表明连接成功了

![Alt text](/images/20130228/gh5.jpg)

最后进入博客安装目录``D:\Blog``运行如下命令，再打开你的Github的二级域名就可以看到刚刚提交的Octopress博客了

```
rake setup_github_pages
rake generate
rake deploy
```

![Alt text](/images/20130228/gh6.jpg)

#### 四、新建和发布页面

发布一个日志前，先在博客目录``D:\Blog\source\_posts\``中生成一个MD文件，类似``2013-03-10-title.markdown``，是日志的编辑文件。运行命令

```
rake new_post["title"]
```

如果想要新建一个页面，则可以运行命令

```
rake new_page["about"]
```

编辑MD文件需要使用markdown语法(参考链接：[Markdown 语法说明](http://wowubuntu.com/markdown/))。
文章编辑完成后，生成和发布则运行命令

```
rake generate
rake deploy
```

本地预览命令。

```
rake preview
```

退出预览命令是`Ctrl+C`

#### 五、安装主题

我现在用的主题是[justin-kelly](http://blog.justin.kelly.org.au/octopress-theme/)([Github源码](https://github.com/wallace/justin-kelly-theme))，安装方法是先进入博客目录``D:\Blog\``然后运行命令

```
git clone https://github.com/wallace/justin-kelly-theme.git .themes/justin-kelly-theme
rake install['justin-kelly-theme']
rake generate
```

其他不错的主题：[slash](http://zespia.tw/Octopress-Theme-Slash/index_tw.html#overview)/[Fabric](http://panks.me/blog/2013/01/new-octopress-theme-fabric/)/[BlogTheme](https://github.com/rastersize/BlogTheme)/[官方推荐](https://github.com/imathis/octopress/wiki/3rd-Party-Octopress-Themes)

#### 六、添加分享到微博按钮

在文件`source/_includes/post/sharing.html`中加入代码，效果如下

```
<div class="sharing">
  {% if site.weibo_share %}
  <span>
  <iframe 
    width="86" 
    scrolling="no" 
    height="16" 
    frameborder="0" 
    src=
      "http://hits.sinajs.cn/A1/weiboshare.html?url={{ site.url }}{{ page.url }}&amp;type=6&amp;{% if site.weibo_uid %}ralateUid={{ site.weibo_uid }}&amp;{% endif %}language=zh_cn" allowtransparency="true">
  </iframe>
  </span>
  {% endif %}
  {% if site.twitter_tweet_button %}
  <a href="http://twitter.com/share" class="twitter-share-button" data-url="{{ site.url }}{{ page.url }}" data-via="{{ site.twitter_user }}" data-counturl="{{ site.url }}{{ page.url }}" >Tweet</a>
  {% endif %}
  {% if site.google_plus_one %}
  <div class="g-plusone" data-size="{{ site.google_plus_one_size }}"></div>
  {% endif %}
  {% if site.facebook_like %}
    <div class="fb-like" data-send="true" data-width="450" data-show-faces="false"></div>
  {% endif %}
</div>
```

由于Liquid语法的关系，如果代码显示不完全，查看如下截图

![Alt text](/images/20130228/dm.jpg)

然后在博客根目录下的配置文件``_config.yml``文件中加入代码

```
# Weibo
# Please refer to http://weibo.com/tool/weiboshow to get your uid and verifier
weibo_uid: 1234567890       # your uid
weibo_verifier:	abc12ef3    # your verifier
weibo_fansline: 0   # How many lines for the fan list
weibo_show: true    # Whether you want your weibo content to be shown
weibo_pic: true     # Whether you want the pictures in weibo to be shown
weibo_skin: 10      # Please refer to http://weibo.com/tool/weiboshow
weibo_share: true   # Whether show the sharing button
```

其中uid和verifier需要去新浪``http://weibo.com/tool/weiboshow``获取。``rake generate``生成后就可以预览了。(参考链接：[为Octopress追加[分享到微博]按钮](http://programus.github.com/blog/2012/03/04/share-weibo-button/)/[增加微博的侧边栏](http://clark1231.iteye.com/blog/1553939))

然后再推荐[百度分享工具](http://share.baidu.com/)，获取代码添加到``source/_includes/post/sharing.html``中即可，当然也可以分析一下代码，自己改写成如以上分享到微博一样的标准形式。

#### 七、代码高亮

修改``_config.yml``设置``pygments: true``，同时Win7上需要安装和配置好[Python](http://www.python.org/download/)。(参考链接：[用Jekyll和Pygments配置代码高亮](http://zyzhang.github.com/blog/2012/08/31/highlight-with-Jekyll-and-Pygments/))

#### 八、部分界面修改

头部导航菜单的修改可以打开编辑``source/_includes/custom/navigation.html``(参考链接：[Octopress 模板篇](http://linmumu.me/octopress-theme-template.html))。

将最新评论添加到侧边栏，首先新建文件``source/_includes/custom/asides/recent_comment.html``

```
<section>
    <h1>Recent Comments</h1>
    <script type="text/javascript" src="http://corey600.disqus.com/recent_comments_widget.js?num_items=5&hide_avatars=1&avatar_size=32&excerpt_length=200"></script>
</section>
```

然后在配置文件``/_config.yml``中的``default_asides``或者``blog_index_asides``或者``post_asides``或者``page_asides``中添加一项``custom/asides/recent_comment.html``，例如我的配置

```
blog_index_asides: [custom/asides/links.html, asides/recent_posts.html, custom/asides/recent_comment.html]
```

开启disqus评论系统只需在配置文件``/_config.yml``中设置好用户名，当然如果要尝试其他第三方评论系统可以参考链接：[为 Octopress 添加多说评论系统](http://ihavanna.org/Internet/2013-02/add-duoshuo-commemt-system-into-octopress.html)。
