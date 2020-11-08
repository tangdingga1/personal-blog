---
title: 《SurviveJS - Webpack》读书笔记(5)
date: 2020/11/08 14:00:00
categories:
- [读书笔记, SurviveJSWebpack]
tags:
- webpack
- 读书笔记
---
原文为线上开源的英文书籍。你可以点击[这里](https://survivejs.com/webpack/foreword/)阅读原书籍。
本篇对应书本第五篇，构建资源的几种处理。
<!--more-->
## Source Maps

&emsp;&emsp;webpack 编译的代码过多之后，在浏览器中进行 debugging 将碰到断点找不到原版本代码的情况。Source Map 是用来解决这个问题的，它相当于源代码和编译后代码的一个映射地图，有点像对象的感觉。`{ 编译代码: 原版代码 }`。

&emsp;&emsp;webpack4以上版本在开发模式的情况下，自动为你生成 Source Map。webpack的 Source Map 大体上可以分为文件内(inline)和文件外(separate)两种。在产品模式下启动 Source Map 的方法也很简单，只需要在 devtool 中配置对应的类型即可。下面介绍文件内和文件外的几种不同功能的 Source Map。

## 文件内(Inline)

- eval
eval 模式让每个模块的函数都被eval函数来执行包裹。

```javascript
/***/ "./src/index.js":
/***/ (function(module, __webpack_exports__, __webpack_require__) {

"use strict";
eval("__webpack_require__.r(__webpack_exports__);\n/* harmony import */ var _main_css__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(\"./src/main.css\");\n/* harmony import */ var _main_css__WEBPACK_IMPORTED_MODULE_0___default = /*#__PURE__*/__webpack_require__.n(_main_css__WEBPACK_IMPORTED_MODULE_0__);\n/* harmony import */ var _component__WEBPACK_IMPORTED_MODULE_1__ = __webpack_require__(\"./src/component.js\");\n\n\ndocument.body.appendChild(Object(_component__WEBPACK_IMPORTED_MODULE_1__[\"default\"])());\n\n//# sourceURL=webpack:///./src/index.js?");

/***/ }),
```

- cheap-eval-source-map
在 eval 的基础上更进一步，拥有base64编码的一个像data url感觉的版本信息。

```javascript
{
  "version": 3,
  "file": "./src/index.js.js",
  "sources": ["webpack:///./src/index.js?3700"],
  "sourcesContent": [
    "import './main.css';\nimport component from \"./component\";\ndocument.body.appendChild(component());"
  ],
  "mappings": "AAAA;AAAA;AAAA;AAAA;AAAA;AACA;AACA",
  "sourceRoot": ""
}
```

- cheap-module-eval-source-map
更进一步，拥有更改的质量和更慢的打包速度。

```javascript
{
  "version": 3,
  "file": "./src/index.js.js",
  "sources": ["webpack:///./src/index.js?b635"],
  "sourcesContent": ["import './main.css';\nimport component ..."],
  "mappings": "AAAA;AAAA;AAAA;AAAA;AAAA;AACA;AAEA",
  "sourceRoot": ""
}
```

- eval-source-map
行内当中最高的质量，和最慢的处理速度。

```javascript
{
  "version": 3,
  "sources": ["webpack:///./src/index.js?b635"],
  "names": ["document", "body", "appendChild", "component"],
  "mappings": "AAAA;AAAA;AAAA;AAAA;AAAA;AACA;AAEAA,QAAQ,CAACC,IAAT,CAAcC,WAAd,CAA0BC,0DAAS,EAAnC",
  "file": "./src/index.js.js",
  "sourcesContent": ["import './main.css';\nimport component ..."],
  "sourceRoot": ""
}
```

## 文件外(separate)
&emsp;&emsp;webpack 提供专门生成的 Source Map 文件，这些文件统一由.map后缀，只在被引入时加载进入浏览器。这种调试方式比较适用于产品打包上线的模式。


- hidden-source-map
和 source-map 类型相同，但是没有在源文件中写入 references。如果你不想在浏览器的 devtool 中暴露 source-map，这是最好的选择。

- nosources-source-map
创建不包含 sourcesContent 的 Source Map。这样你可以不暴露的你的源代码，只暴露编译后的代码。

- cheap-source-map
简化的 Source Map。禁用了外部loader的一些 Source Map。（如css-loader）

```javascript
{
  "version": 3,
  "file": "main.js",
  "sources": [
    "webpack:///webpack/bootstrap",
    "webpack:///./src/component.js",
    "webpack:///./src/index.js",
    "webpack:///./src/main.css"
  ],
  "sourcesContent": [
    "...",
    "// extracted by mini-css-extract-plugin"
  ],
  "mappings": ";AAAA;...;;ACFA;;;;A",
  "sourceRoot": ""
}
```

- cheap-module-source-map
和前面的cheap-source-map功能相同，在原有的基础上生成了一个单独的每一行的对应地图。
```javascript
{
  "version": 3,
  "file": "main.js",
  "sources": [
    "webpack:///webpack/bootstrap",
    "webpack:///./src/component.js",
    "webpack:///./src/index.js",
    "webpack:///./src/main.css"
  ],
  "sourcesContent": [
    "...",
    "// extracted by mini-css-extract-plugin"
  ],
  "mappings": ";AAAA;...;;ACFA;;;;A",
  "sourceRoot": ""
}
```

- source-map
最完整，质量最高的 Source Map。同时也是最慢的 Source Map。

## 代码分割
&emsp;&emsp;webpack默认配置会把所有的js文件打包为一个，使用近似于单页应用的方式来运作。这种方式虽然能够一次加载所有的代码，让后续的操作连贯。但是遇到当模块代码过大，部分模块用户操作不到的情况下，这种方式就不大合适了。我们需要把代码按照页面/模块进行分割，当用户访问到这部分资源的时候再进行代码的加载。

![Code splitting](https://survivejs.com/d0157e4db2b71adc9a7a25316309c3d1.png)


&emsp;&emsp;webpack 提供了两种异步引入代码的方式 import 和 require.ensure。其中 require.ensure 方式为遗留的老版本方式，将被移除，这里不作介绍。

&emsp;&emsp;动态 import 引入的方式按照 Promise 的引入规范：

```javascript
import(/* webpackChunkName: "optional-name" */ "./module").then(
  module => {...}
).catch(
  error => {...}
);
```

&emsp;&emsp;接下来尝试在文件打包中使用异步加载：
```javascript
// src/lazy.js
export default "Hello from lazy";

// src/component.js
export default (text = "Hello world") => {
  const element = document.createElement("div");

  element.className = "rounded bg-red-100 border max-w-md m-4 p-4";
  element.innerHTML = text;
  // 点击时动态加载 lazy 模块。
  element.onclick = () =>
    import("./lazy")
      .then((lazy) => {
        element.textContent = lazy.default;
      })
      .catch((err) => {
        console.error(err);
      });

  return element;
};
```

然后运行npm run build，我们可以在webpack的打包日志中看到一个 34.js 的文革在 man.js 外面的js文件，这就是异步分割出去的lazy.js。
```bash
⬡ webpack: Build Finished
⬡ webpack: assets by status 7.95 KiB [compared for emit]
    asset main.css 7.72 KiB [compared for emit] (name: main) 1 related asset
    asset index.html 237 bytes [compared for emit]
  assets by status 3.06 KiB [emitted]
    asset main.js 2.88 KiB [emitted] [minimized] (name: main) 1 related asset
    # 独立在 main.js外面的分割js
    asset 34.js 187 bytes [emitted] [minimized] 1 related asset
  Entrypoint main 10.6 KiB (14.3 KiB) = main.css 7.72 KiB main.js 2.88 KiB 2 auxiliary assets
  runtime modules 6.76 KiB 11 modules
  orphan modules 544 bytes [orphan] 2 modules
  code generated modules 624 bytes (javascript) 1.99 MiB (css/mini-extract) [code generated]
    ./src/index.js + 1 modules 591 bytes [built] [code generated]
    css ./node_modules/css-loader/dist/cjs.js!./node_modules/postcss-loader/dist/cjs.js??ruleSet[1].rules[1].use[2]!./src/main.css 1.99 MiB [code generated]
    ./src/lazy.js 33 bytes [built] [code generated]
  webpack 5.1.3 compiled successfully in 3846 ms
...
```

&emsp;&emsp;这种异步加载的方式是默认的行为，你可以在配置中强行禁止代码分割的行为出现。
```javascript
const config = {
  plugins: [
    new webpack.optimize.LimitChunkCountPlugin({
      maxChunks: 1,
    }),
  ],
};
```

## 包分割
&emsp;&emsp;我们在之前的章节学习过了，webpack可以在生成的文件中植入hash和base64版本号。这样做的目的是在代码更新的时候，一并更新文件名称，让浏览器及时更新运行代码，而不是从缓存中拿过时的代码。

&emsp;&emsp;但在webpack使用时，我们经常会用到外部的包，比如`react/vue/angular`。实际上在你的代码更新的时候，这些包的代码应该是不需要更新的。所以我们需要把这些三方包的代码单独打包出来，在后续代码更新的时候，这些三方包使用缓存，而我们的代码部分不使用缓存进行更新。

&emsp;&emsp;完成这一个功能我们首先需要在原有的项目文件中引入三方的库。
```bash
npm add react react-dom

# src/index.js
import "react";
import "react-dom";

npm run build
# result
⬡ webpack: Build Finished
⬡ webpack: assets by path *.js 127 KiB
    asset main.js 127 KiB [emitted] [minimized] (name: main) 2 related assets
    asset 34.js 187 bytes [compared for emit] [minimized] 1 related asset
  asset main.css 7.72 KiB [compared for emit] (name: main) 1 related asset
  asset index.html 237 bytes [compared for emit]
  Entrypoint main 135 KiB (323 KiB) = main.css 7.72 KiB main.js 127 KiB 2 auxiliary assets
  ...
  webpack 5.1.3 compiled successfully in 5401 ms
```

&emsp;&emsp;我们看到main文件变为了127k，引入了三方包的代码后，文件变大了。接下来我们把这些三方包的代码单独分割打一个包出去。

```javascript
// webpack.config.js
const productionConfig = merge([
  // ...
  {
    optimization: {
      splitChunks: {
        chunks: "all",
      },
    },
  },
]);

// npm run build
webpack: Build Finished
⬡ webpack: assets by status 128 KiB [emitted]
    asset 935.js 124 KiB [emitted] [minimized] (id hint: vendors) 2 related assets
    asset main.js 3.24 KiB [emitted] [minimized] (name: main) 1 related asset
    asset index.html 267 bytes [emitted]
  assets by status 7.9 KiB [compared for emit]
    asset main.css 7.72 KiB [compared for emit] (name: main) 1 related asset
    asset 34.js 187 bytes [compared for emit] [minimized] 1 related asset
  Entrypoint main 135 KiB (326 KiB) = 935.js 124 KiB main.css 7.72 KiB main.js 3.24 KiB 3 auxiliary assets
  ...
  webpack 5.1.3 compiled successfully in 4847 ms
```

&emsp;&emsp;一个名为 935.js 的包被单独打包了出来，这就是三方包的集合。和代码的关系树类似于下图这样：
![](https://survivejs.com/cc11f7e53c579fff28de1b3ed5b9f53a.png)

&emsp;&emsp;三方打包的规则和目录还可以在webpack中进行配置。

```javascript
const productionConfig = merge([
  // ...
  {
    optimization: {
      splitChunks: {
        // 被打包的最小文件
        minSize: {
          javascript: 20000,
          // This type is injected by mini-css-extract-plugin
          "css/mini-extra": 10000,
        },
        cacheGroups: {
          commons: {
            // 指定打包的引入
            test: /[\\/]node_modules[\\/]/,
            // 打包的文件名称
            name: "vendor",
            // chunks 的类型 Entry/Normal/Initial
            chunks: "initial",
          },
        },
      },
    },
  },
]);
```

## 一些其它的打包优化
- 清理打包的文件夹
清理打包文件夹，让每次 webpack 构建之后之前的错误文件消失。需要使用 clean-webpack-plugin 插件。
```javascript
// npm add clean-webpack-plugin --develop

// webpack.parts.js
const { CleanWebpackPlugin } = require("clean-webpack-plugin");

exports.clean = () => ({
  plugins: [new CleanWebpackPlugin()],
});

// webpack.config.js
const path = require("path");

const commonConfig = merge([

  {
    output: {
      path: path.resolve(process.cwd(), "dist"),
    },
  },
  parts.clean(),

  // ...
]);
```

- 关联版本到构建文件
关联诸如迭代的或者git的版本信息到构建出来的文件
```javascript
// npm add git-revision-webpack-plugin --develop

// webpack.parts.js
const webpack = require("webpack");
const GitRevisionPlugin = require("git-revision-webpack-plugin");

exports.attachRevision = () => ({
  plugins: [
    new webpack.BannerPlugin({
      banner: new GitRevisionPlugin().version(),
    }),
  ],
});

// webpack.config.js
const productionConfig = merge([
  // ...
  parts.attachRevision(),

]);

```
之后使用`npm run build`，就可以发现文件的结尾会带上版本号的注释。`/*! 0b5bb05 */` 或者 `/*! v1.7.0-9-g5f82fe8 */`。





