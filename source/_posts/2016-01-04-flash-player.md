---
layout: post
title: Flash播放器研究
category : 备忘
tagline: "备忘"
tags : [flash, 播放器]
excerpt_separator: <!--more-->
---

### 背景

原本 Web端 使用的 flash 播放器是 ckplayer，基本能满足当时的使用需求。但是，服务器端推流改为RTMP后，ckplayer 在 播放画面真正出现前会出现短暂的黑屏，这个问题没法解决，最后只能改为使用百度的 cyberplayer 播放器1.5版本。

百度播放器 可以设置onMeta事件 知道画面出现的真正时间，所以可以在画面出现前加一层loading的div覆盖播放器，等到画面出现时再隐藏这一层div。实现代码如下：

```javascript
// metadata事件
t5Player.onMeta(function(event){
    if(t5Player.getState() == 'PLAYING'){
        sum++; // 为避秒长时间显示loading 画面，使用sum设置等待时间上限
    }
    if((!!event.metadata.renderstate && !!event.metadata.stagevideo) || sum == 30){
        setTimeout(function(){
        // todo 隐藏 loading 画面          
        },500);
    }
});
```
 
在决定使用百度播放器前，还试用过 jwplayer 播放器。jwplayer是国外团队开发的比较成熟的Web播放器，支持完善的api接口，包括上文的metdata事件。但是免费版不支持hls流的播放，且不能用于商业活动，收费版需要授权使用。所以只能退而求其次，使用百度播放器。但是百度播放器的源码是不公开的，出了问题很难排查解决，在使用过程中确实遇到了一些问题，所以产生了需要自有的web播放器的需求。

<!--more-->

但是从头开始开发一个web播放器需要投入很大的开发成本，然而由于jwplayer免费版的源码是公开的，预研的版本可以先参考jwplayer的源码，并尝试加入HLS流播放支持，实现一个基本满足目前要求的播放器。

### jwplayer - RTMP播放支持

jwplayer播放器免费版本身就是支持RTMP流播放的。

#### 下载和编译

jwplayer免费版源码地址：https://github.com/jwplayer/jwplayer

在编译jwplayer源码前，需要下载安装flash的开发环境 Flex SDK。目前Flex SDK 已经被 Adobe公司交给apache基金会管理，下载地址：http://flex.apache.org/download-binaries.html

安装完成后设置环境变量 FLEX_HOME 的值为 安装根目录。

jwplayer的源码也是使用grunt构建的，解压后进入目录，按照官方提供的方法就能编译成功了。

>Build Instructions

>1. Install Node.js
>2. Install Adobe AIR SDK - (安装了Flex SDK 就不需要 AIR SDK了，建议安装Flex SDK)
>3. Install Java
>4. Download player.swc 11.2
>5. Rename and move the .swc file to {AIRSDK_Compiler}/frameworks/libs/player/11.2/playerglobal.swc

>```javascript
# First time set up
npm install -g grunt
npm install
# Build using
grunt
```

>After build, the assets will be available in the bin-release folder.

#### 源码分析

以7.0.1版本为例。

找到jwplayer源码目录下的src目录，这里存放的就是所有as3(flash的开发语言)/js和css的源码。flash目录存放了所有as后缀的as3源文件。js目录是js源码，包括html5的播放逻辑。css目录和templates目录是样式和模本，涉及播放器的界面样式，但不涉及播放逻辑。jwplayer源码使用webpack构建，所以css和模版不是直接引入页面，而是会被js/view目录下的文件依赖。

![图片](/images/20160104/1.png)

##### js

js源码的入口文件是jwplayer，主要关注api/controller/providers三个目录下的文件。

![图片](/images/20160104/2.png)

api目录主要定义了一些对外的接口，入口是global-api.js。api-action.js和api-mutators.js是对controller的接口进行了直接透传，callback-deprecate.js是对controller的事件进行了直接透传，而需要另外进行一层处理的接口放在了api.js中。api-deprecate.js是对低版本接口的兼容定义。

![图片](/images/20160104/3.png)

controller目录下代码是播放逻辑的处理，主要是controller.js文件。

![图片](/images/20160104/4.png)

providers目录下的源码是对播放环境封装，主要是html5和flash，还包括jwplayer单独为youtube播放的封装，皆继承于default。入口是providers.js，providers-supported.js是播放环境的判断。

由于我们目前的需求只是pc web 端的flash播放，其实可以尝试把html5的支持去掉，只保留 default.js flash.js providers.js三个文件，并在providers.js中直接返回 flash 的provider，也不需要进行播放环境的判断了。providers.js如下：
 
```javascript
define([
    'providers/flash',
    'utils/helpers',
    'utils/underscore'
    ], function(flash, utils, _) {
  
    // 判断是否是支持的播放类型
    function supports(source) {
        var flashExtensions = {
            'flv': 'video',
            'f4v': 'video',
            'mov': 'video',
            'm4a': 'video',
            'm4v': 'video',
            'mp4': 'video',
            'aac': 'video',
            'f4a': 'video',
            'mp3': 'sound',
            'mpeg': 'sound',
            'smil': 'rtmp',
            'm3u8': 'hls'
        };
        var PLAYABLE = _.keys(flashExtensions);
        if (!utils.isFlashSupported()) {
            return false;
        }
  
        var file = source.file;
        var type = source.type;
  
        if (utils.isRtmp(file, type)) {
            return true;
        }
  
        if(file.indexOf('m3u8') === 0 || type === 'hls'){
            return true;
        }
  
        return _.contains(PLAYABLE, type);
    }
  
    function Providers(config) {
        this.config = config || {};
    }
  
    _.extend(Providers.prototype, {
        // Find the name of the first provider which can support the media source-type
        choose : function(source) {
            // prevent throw on missing source
            source = _.isObject(source) ? source : {};
            if (supports(source)) {
                return {
                    priority: 0,
                    name : 'flash',
                    type: source.type,
                    // If provider isn't loaded, this will be undefined
                    provider : flash
                };
            }
            return null;
        }
    });
  
    return Providers;
});
```

##### flash

flash源码的入口时player目录的Player类。

![图片](/images/20160104/5.png)

flash源码主要关注media目录下面的几个类。

![图片](/images/20160104/6.png)

可以看到有RTMP/Sound/Video三种provider类，分别对应RTMP流直播支持，声音播放支持和视频点播支持。皆继承于MediaProvider类，IMediaProvider是对应接口类。所以，如果要加入HLS流直播支持，就需要加入一个HLSMediaProvider类，继承于MediaProvider类，并实现通用接口就可以了。当然，还要修改Provider选择的逻辑，使之在传入HLS 的播放URL时能正确选择对应Provider。

### flashls - HLS播放支持

flashls是开源的HLS播放库，源码地址：https://github.com/mangui/flashls

官方提供了 Flowplayer 和 OSMF 的适配支持示例，虽然并没有作为jwplayer插件的使用支持，但是由于flashls的性质，我们可以直接在jwplayer源码中使用flashls。

#### 使用

flashls使用文档：https://github.com/mangui/flashls/blob/dev/API.md

拷贝flashls源码的org/mangui/hls目录下的所有文件至jwplayer工程下的对应路径，不存在的文件新建。

拷贝flashls源码的lib目录下的blooddy_crypto.swc文件至jwplayer工程下的lib目录下。

修改gruntfile文件，找到targetCompilerOptions配置，增加一项依赖库目录配置，修改后如下：

```javascript
targetCompilerOptions : [
 '-compiler.library-path+='+ 'lib',
 '-define+=FOXPLAYER::version,\'' + packageInfo.version + '\''
],
```

#### 修改jwplayer源码加入hls支持

第一步在media源码目录下新建一个名为HLSMediaProvider.as的文件，创建一个同名类，使之继承于MediaProvider

```javascript
public class HLSMediaProvider extends MediaProvider {
 // 构造函数
 public function HLSMediaProvider() {
 super('hls');
 }
}
```
 
修改js源码的providers.js文件，使之对于hls格式的url能正确选择flash的provider。其中的supports接口修改如下：

```javascript
// 判断是否是支持的播放类型
function supports(source) {
 var flashExtensions = {
 'flv': 'video',
 'f4v': 'video',
 'mov': 'video',
 'm4a': 'video',
 'm4v': 'video',
 'mp4': 'video',
 'aac': 'video',
 'f4a': 'video',
 'mp3': 'sound',
 'mpeg': 'sound',
 'smil': 'rtmp',
 'm3u8': 'hls' // 新增代码
 };
 var PLAYABLE = _.keys(flashExtensions);
 if (!utils.isFlashSupported()) {
 return false;
 }
 
 var file = source.file;
 var type = source.type;
 
 if (utils.isRtmp(file, type)) {
 return true;
 }
 
 // 新增代码
 if(file.indexOf('m3u8') === 0 || type === 'hls'){
 return true;
 }
 
 return _.contains(PLAYABLE, type);
}
```

修改as3源码，model/Model.as文件中的setupMediaProviders方法如下：

```javascript
protected function setupMediaProviders():void {
setMediaProvider('default', new MediaProvider('default'));
setMediaProvider('video', new VideoMediaProvider());
setMediaProvider('rtmp', new RTMPMediaProvider());
setMediaProvider('sound', new SoundMediaProvider());
setMediaProvider('hls', new HLSMediaProvider()); // 新增代码
 // setActiveMediaProvider('default');
}
```

修改as3源码，model/InstreamPlayer.as文件中的getProvider方法如下：

```javascript
private function getProvider(type:String):MediaProvider {
 // Only accept video, http or rtmp providers for now
 switch (type) {
 case 'video':
 return new VideoMediaProvider();
 case 'rtmp':
 return new RTMPMediaProvider();
 case 'sound':
 return new SoundMediaProvider();
 // 新增代码
 case 'hls':
 return new HLSMediaProvider();
 }
 // ERROR
 SwfEventRouter.triggerJsEvent('instream:error', {
message: 'Unsupported Instream Format; only video or rtmp are currently supported'
 });
 return null;
}
```

另外还需要两处修改，model/PlaylistItem.as文件中的extensionMap和typeMap方法分别增加如下分支：

```javascript
case "m3u8":
 return "hls";
```

```javascript
case "hls":
 return "hls";
```
 
parsers/JWParser.as文件中的extensions对象增加

```javascript
'ogv': 'video',
'm3u8': 'hls'
```

两项，getProvider方法的return 'http';修改为return 'hls'。

最后最重要，也是最困难的就是使用flashls库给media/HLSMediaProvider.as增加通用的接口实现。

全部开发完毕，运行grunt编译成功得到的flash文件和js文件使用时就可以支持hls直播播放了。
