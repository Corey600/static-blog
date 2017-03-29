---
layout: post
title: gulp前端构建工作流
category : 备忘
tagline: "备忘"
tags : [前端,gulp,webpack]
excerpt_separator: <!--more-->
---

在前端工程化中，目前最流行的是使用gulp构建前端代码。但gulp只是提供了一种流式的文件处理方式，具体的功能需要各种插件来实现。gulp插件种类繁多，各自实现了特定的功能，无法全部了解和熟悉，那么在决定构建方案的时候难免选择困难，无法快速准确地选择出自己需要插件。

但事实上一些常用的功能，已经有一些流行而成熟的插件被广泛地应用了，我们不需要自己大海捞针一样去搜寻和评估一大片插件才能选择到合适的插件。本文就是基于‘HTML5页面node化’的项目中使用到的插件，形成一个暂时较为通用的gulp构建工作流。随着项目的积累和深入，以后将会补充和修改。

_注_ ：gulp的基本使用方法将不会赘述。

#### 1. *第一步* - 清理上次构建的目标目录

清理即是删除上次构建产生的文件。一般情况下，构建的目标目录需要和源码目录分开。这样在清理时，直接将构建的目标目录（这里为`dist`目录）下的所有文件删除即可。

<!--more-->

```javascript
var del = require('del');

gulp.task('clean', function () {
    del(['dist/*']);
});
```

#### 2. *第二步* - 增加语法检查

一般的集成开发环境会集成语法检查，而类似SublimeTest的一些文本编辑器则需要插件支持。这里直接在构建时，加入js语法检查，保证代码质量。

一般的js的语法检查工具有`jshint`、`jslint`和`eslint`等，各自的规则有所区别。这里使用`gulp-jshint`插件实现语法检查。

```javascript
var jshint = require('gulp-jshint');

gulp.task('jshint', function () {
    return gulp.src([
        'public/**/*.js',
    ])
        .pipe(jshint())
        .pipe(jshint.reporter('default'));
});
```

#### 3. *第三步* - 处理图片

一般情况下，处理图片可能涉及到的压缩和生成雪碧图等。这里还未有这部分需求，所以只做了md5文件名后缀的添加。其中`rev()`用于给文件名扩展名前增加一个md5后缀，为文件内容的hash值，`rev.manifest()`用于生成增加后缀名前后的文件路径映射表，用于文件名的替换。

为什么要使用md5文件名后缀？这里是基于文件缓存策略的考虑。
具体可以参考：[前端工程精粹（一）：静态资源版本更新与缓存](http://www.infoq.com/cn/articles/front-end-engineering-and-performance-optimization-part1/)

```javascript
var rev = require('gulp-rev');

gulp.task('image', function () {
    return gulp.src([
        'public/**/*.+(ico|png|jpeg|jpg|gif|svg)'
    ])
        .pipe(rev())
        .pipe(gulp.dest('dist'))
        .pipe(rev.manifest())
        .pipe(gulp.dest('dist/manifest/img'));
});
```

#### 4. *第四步* - 处理样式文件

相比于没有前端构建过程的原始方式，构建可以很方便的加入各种css预处理器，`Less`、`Sass`和`Stylus`都有对应的预处理gulp插件。这里选择了`gulp-less`插件作为less的处理器。并使用`gulp-minify-css`对处理后的css进行压缩。其中`gulp-rev-replace`插件用于将css中引用的图片文件路径替换为增md5后缀后的路径。

```javascript
var less = require('gulp-less');
var minifycss = require('gulp-minify-css');
var replace = require('gulp-rev-replace');

gulp.task('less', ['image'], function () {
    // 读取 图片  的路径映射表
    var manifest = gulp.src('dist/manifest/img/rev-manifest.json');
    return gulp.src([
        'public/**/pages/**/*.less',
        'public/**/common/styles/style.less'
    ])
        .pipe(rev())
        // css预处理
        .pipe(less())
        // 替换图片引用路径
        .pipe(replace({manifest: manifest, prefix: prefix}))
        // 压缩css代码
        .pipe(minifycss())
        // 生成到目标路径
        .pipe(gulp.dest('dist'))
        // 生成css文件路径映射
        .pipe(rev.manifest())
        .pipe(gulp.dest('dist/manifest/css'));
});
```

#### 5. *第五步* - 处理脚本文件

gulp可以直接处理js脚本文件的压缩合并等工作，但在考虑js代码的模块化时，还要引入require.js或者seajs等模块加载器的对应插件。我们知道，webpack本身就是一个完备的前端构建工具，而且引入了一种新的模块化解决方案。这里我们将使用webpack的处理变成一个gulp任务，利用它的js模块化方案和打包压缩等功能，其他事情交给gulp来处理。

在gulp中使用webpack，我们需要使用`webpack-stream`插件。它直接将`main`作为唯一的入口entry，但是我们的工程不是单页面的，这样显然不行。例如：

```javascript
gulp.task('webpack', function () {
    return gulp.src([ 'public/pages/**/*.js' ])
        .pipe(webpackStream({
            output: {
                filename: '[name].js'
            },
            resolve: {
                extensions: ['', '.js', '.jsx']
            },
            module: {
                loaders: [{
                    test: /\.(js|jsx)$/,
                    loader: 'babel-loader!jsx-loader?harmony'
                }]
            }
        }))
        .pipe(gulp.dest('dist'));
});
```

只生成了`dist`目录下的一个main.js文件。

为了将`gulp.src`中的文件直接作为webpack的entry，我们又引入了`vinyl-named`插件，它可以使用回调函数将文件路径中的匹配部分作为entry中的name配置。例如：

```javascript
gulp.task('webpack', function () {
    return gulp.src([
        'public/pages/**/*.js'
    ])
        .pipe(named(function (file) {
            var dir = path.dirname(file.path);
            // 替换掉 public 目录路径
            dir = dir.replace(/^.*public(\\|\/)/, '');
            // 将文件路径及 basename 作为 name
            return path.join(dir, path.basename(file.path, path.extname(file.path)));
        }))
        .pipe(webpackStream({
            output: {
                // 指定输出文件名
                filename: '[name].js'
            },
            resolve: {
                extensions: ['', '.js', '.jsx']
            },
            module: {
                // 设置加载器
                loaders: [{
                    test: /\.(js|jsx)$/,
                    loader: 'babel-loader!jsx-loader?harmony'
                }]
            }
        }))
        .pipe(gulp.dest('dist'));
});
```

生成了如下四个文件：

```
pages\safe_grade\index\index.js
pages\safe_grade\user_info\user_info.js
pages\safe_grade\video_store\video_store.js
```

webpack要实现代码的压缩等功能也需要使用到插件，这里使用到两个功能：

(1) 代码压缩

使用时在plugins配置的列表中增加一项：

```javascript
new webpack.optimize.UglifyJsPlugin({
    compress: {
        warnings: false
    },
    mangle: {
        except: ['$', 'm', 'window', 'webpackJsonpCallback']
    }
}),
```

其中mangle.except配置了在压缩过程中不被简化的变量，以防出现错误。
参考资料：[http://webpack.github.io/docs/list-of-plugins.html#uglifyjsplugin](http://webpack.github.io/docs/list-of-plugins.html#uglifyjsplugin)

(2) 公共代码提取

使用时在plugins配置的列表中增加一项：

```javascript
new webpack.optimize.CommonsChunkPlugin({
    name: 'common/js/common',
    filename: 'common/js/common.js',
    minChunks: 2
})
```

参考资料：[http://webpack.github.io/docs/list-of-plugins.html#commonschunkplugin](http://webpack.github.io/docs/list-of-plugins.html#commonschunkplugin)

代码构建及压缩后不容易阅读和调试，webpack也支持增加source map。只需设置devtool选项为'source-map'。当然，它还支持其他很多模式，具体可参考：[webpack sourcemap 选项多种模式的一些解释](http://www.07net01.com/2016/01/1120167.html)

最终js构建代码如下：

```javascript
var named = require('vinyl-named');
var webpack = require('webpack');
var webpackStream = require('webpack-stream');

gulp.task('webpack', ['image'], function () {
    var revall = new Revall({hashLength: 10});
    var manifest = gulp.src('dist/manifest/img/rev-manifest.json');
    return gulp.src([
        'public/pages/**/*.js'
    ])
        .pipe(named(function (file) {
            var dir = path.dirname(file.path);
            dir = dir.replace(/^.*public(\\|\/)/, '');
            return path.join(dir, path.basename(file.path, path.extname(file.path)));
        }))
        .pipe(webpackStream({
            output: {
                filename: '[name].js'
            },
            resolve: {
                extensions: ['', '.js', '.jsx']
            },
            module: {
                loaders: [{
                    test: /\.(js|jsx)$/,
                    loader: 'babel-loader!jsx-loader?harmony'
                }]
            },
            plugins: [
                // 使用生产环境版本'react'
                new webpack.DefinePlugin({
                    "process.env": {
                        NODE_ENV: JSON.stringify("production")
                    }
                }),
                //js文件的压缩
                new webpack.optimize.UglifyJsPlugin({
                    compress: {
                        warnings: false
                    },
                    mangle: {
                        except: ['$', 'm', 'window', 'webpackJsonpCallback']
                    }
                }),
                //将公共代码抽离出来合并为一个文件
                new webpack.optimize.CommonsChunkPlugin({
                    name: 'common/js/common',
                    filename: 'common/js/common.js',
                    minChunks: 2
                })
            ],
            devtool: 'source-map'
        }))
        .pipe(replace({manifest: manifest, prefix: prefix}))
        .pipe(revall.revision())
        .pipe(gulp.dest(dest))
        .pipe(revall.manifestFile())
        .pipe(gulp.dest('dist/manifest/webpack'));
});
```

#### 6. *第六步* - 增加文件监听

gulp已经为我们提供了监听的工具，就是`watch`接口。使用方法也很简单，参看下面的例子。

```javascript
gulp.task('watch', ['default'], function () {
    gulp.watch(['public/**/*.+(ico|png|jpeg|jpg|gif|svg)'], ['less', 'webpack']);
    gulp.watch(['public/**/*.less'], ['less']);
    gulp.watch(['public/**/*.js'], ['js', 'webpack']);
});
```

#### *补充：*

暂无。