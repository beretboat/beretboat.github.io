---
layout: post
title: 如何在react-native中动态的导入JavaScript？
description: ""
modified: 2018-01-03
tags: [react-native javascript]
image:
  feature: abstract-3.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
---

## 起因
随着移动设备和浏览器性能越来越好，很多原本只有桌面版本的软件也有了iOS, Android和Web版本。多平台无疑极大的增加了公司的开发成本，而更为严重的是，很多作为平台存在的软件极大的增加了第三方开发者开发插件的难度。在手持设备和Web兴起之前，很多平台化软件一般以native(如C++)的形式开放SDK。显然，native语言很难支持所有平台。如果第三方开发者写一个插件要用C++写一个Windows版，用Objective-C/Swift写一个iOS版，再用JavaScript写一个Web版，可能开发者一开始就直接放弃了。

那什么语言能支持所有的主流平台呢？目前看只有JavaScript一种。react-native的出现则进一步打消了开发者对JavaScript/Web的性能顾虑。react-native适合用来写跨平台插件，但是有个致命的缺陷是不支持动态导入代码:

```javascript
let plugin = GetPluginAtRunTime();
import plugin; // fails because import/require in react-native only accepts static string.
}
```

## 现状
关于动态导入的支持GitHub上很早就有过[讨论](https://github.com/facebook/react-native/issues/6391)，对于react-native不支持动态导入的原因Facebook的开发者有过[解释](https://github.com/facebook/metro/issues/52)。简单的说原因有两个:

1. 动态导入会导致(Flow)[https://github.com/facebook/flow]无法对动态`module`进行类型检查。
2. [react-native packager](https://github.com/facebook/metro)需要将module在bundle阶段打包成一个`.jsbundle`文件，动态的文件名无法打包。

基于以上原因，Facebook认为动态导入对于他们来说不是一个高优先级的需求，所以近期不会做。

对于JavaScript原生支持动态导入的[提案](https://github.com/tc39/proposal-dynamic-import)在ES9也已经进入了stage 3。 不好的消息是目前的提案倾向于由host(浏览器, Node.js, react-native)各自实现。这样的话，react-native官方未必会改变他们不原本的短期不支持动态导入的想法。(另外就算没有ES9，对于Web和Node.js来说原本就有办法动态导入。Web可以动态插入`<script>...</script>`), 而Node.js的`require`本身就支持动态导入。

## 解决方法

首先尝试了改造[react-native packager](https://github.com/facebook/metro)，但是发生坑有点深，而且还要解决react-native不支持多jsbundle的问题。然后想到了`eval()`, 这个被认为“危险”的函数。

```javascript
fetch('https://url_to_plugin_code')
.then((response) => response.text())
.then((responseString) => {
  eval(responseString);
})
.catch((error) => {
});
```

`eval()`不被推荐使用的原因有几个，其中最重要的是:

1. `eval()`会导致性能下降。
2. `eval()`会执行任意传给它的代码，存在安全问题。

关于性能问题，Kyle Simpson在他的["You Don't Know JS"](https://github.com/getify/You-Dont-Know-JS/blob/master/scope%20%26%20closures/ch2.md#cheating-lexical)系列书里面有详细解释。但是对于只是在bootstrap阶段一次执行的情况不会有太大影响。对于安全问题，如果`eval()`只执行通过认证的从特定服务器下载的插件，安全性也是可控的。

## 总结

在react-native的`import`支持动态导入之前，`eval()`是一个不完美但是可用的方案。