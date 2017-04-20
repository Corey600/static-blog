---
layout: post
title: React多语言方案
date: 2017/04/20
toc: true
category:
  - 备忘
tags:
  - 前端
  - React
  - i18n
---

由于框架使用的是 [React](https://facebook.github.io/react/) ， 多语言方案也选择了生态系统下的 [react-intl](https://github.com/yahoo/react-intl) 模块，它是雅虎 [FormatJS](https://formatjs.io/) 计划的一部分。

### 翻译字符串替换

使用 [react-intl](https://github.com/yahoo/react-intl) 替换字符串用以翻译很方便。只需将整个业务父组件包裹在 IntlProvider 中：

```
ReactDOM.render(
    <IntlProvider
        locale={usersLocale}
        messages={translationsForUsersLocale}
    >
        <App/>
    </IntlProvider>,
    document.getElementById('container')
);
```

<!--more-->

其中`usersLocale`是当前选择的语言，`translationsForUsersLocale`是用于翻译的键值对数据。

在业务父组件中要显示字符串时插入`FormattedRelative`组件，其中key是上文提到的翻译键值对中的对应key值：

```
<FormattedRelative value={key}/>
```

其他用法参考[官方API文档](https://github.com/yahoo/react-intl/wiki/API)。

需要注意的是，还需要加载针对不同语言的资源包，需要使用哪些语言就要加载对应文件：

```
import {addLocaleData} from 'react-intl';
import en from 'react-intl/locale-data/en';
import fr from 'react-intl/locale-data/fr';
import es from 'react-intl/locale-data/es';

addLocaleData([...en, ...fr, ...es]);
```

如果是在低版本的浏览器环境下，还需要考虑加载 ployfill 库，具体可以参考 [How to Polyfill Browsers](https://formatjs.io/guides/runtime-environments/#polyfill-browsers)。

```
import 'intl/locale-data/jsonp/en';
import 'intl/locale-data/jsonp/fr';
import 'intl/locale-data/jsonp/es';
```

### 获取当前语言

由于当前业务需要，在获取当前语言时，还需要考虑到该用户属性，即保存在后端的语言属性。

基本流程：

- => 有则获取使用后端保存的语言
- => 有则获取使用浏览器端cookies保存的语言
- => 有则获取使用系统的语言

在获取系统使用语言时，使用了 [locale2](https://github.com/moimikey/locale2) 组件。

### 切换语言、按需加载

考虑提供切换语言功能，基本实现原理比较简单，只要修改保存的语言值（包括修改后台保存的值和保存在浏览器cookie中的值），然后reload当前页面。

但是这样做的缺点是，必须事先加载所有可能的语言的资源文件，包括浏览器polyfill和翻译文件。在当前用户只使用一种特定语言的情况下，会造成加载资源的浪费，拖慢加载速度。

但是考虑到，ES6 的 import 是禁止动态引入模块文件的（基于可能的优化角度做的限制），好在我们并不是在原生的环境下使用 ES6 的 import 特性。事实上，在使用 webpack2.x 的情况下，这里的 import 是 webpack 模块方案提供的工具，然而 webpack2.x 是支持动态的 import 的，具体参考官方文档 [webpack - Dynamic import](https://webpack.js.org/guides/code-splitting-async/#dynamic-import-import-)。

动态加载的 demo 就可以这么写：

```
function importLocale(locale) {
  Promise.all([
  // 加载 polyfill
  import(`intl/locale-data/jsonp/${locale}`),
  // 加载 react-intl 资源文件
  import(`react-intl/locale-data/${locale}`),
  // 加载翻译文件
  import(`../../locales/${getFilename(locale)}`),
  ]).
  then((values) => {
    addLocaleData([...values[1]]);
    let messages = values[2].default;
  });
}
```

会注意到，这里使用了 Promise，因为其实动态 import 可以看成一个函数，返回了一个 Promise 对象。动态加载需要并行执行的话，需要使用 Promise.all 。当然在不支持 Promise 的低版本浏览器，需要引入 相应 ployfill ，比如 [es6-promise](https://github.com/stefanpenner/es6-promise)。

最后，我们在 webpack 的打包配置文件中，给 babel 增加一个插件，这个 [webpack - Dynamic import](https://webpack.js.org/guides/code-splitting-async/#dynamic-import-import-) 也有说明。

```
module.exports = {
  entry: './index-es2015.js',
  output: {
    filename: 'dist.js',
  },
  module: {
    rules: [{
      test: /\.js$/,
      exclude: /(node_modules)/,
      use: [{
        loader: 'babel-loader',
        options: {
          presets: [['es2015', {modules: false}]],
          plugins: ['syntax-dynamic-import']
        }
      }]
    }]
  }
};
```

### 已知问题

在IE浏览器环境下，直接使用上文的动态加载文件方案，会报找不到 react-intl 资源文件错误，初步判断是加载顺序的问题。解决方案是把英文当成默认语言，先加载对应资源文件。再根据实际具体语言加载文件。


_后话：_

事实上，很多动态加载多语言资源文件的方案，是直接在 html 中添加 `script` 标签，然后动态修改标签中路径的值。但这里考虑到统一构建方案，翻译文件需要修改而必须添加md5值等等一些比较麻烦的处理，而选择了上文这样的动态加载方式。
