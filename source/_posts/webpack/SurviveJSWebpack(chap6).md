---
title: 《SurviveJS - Webpack》读书笔记(6)
date: 2020/11/16 14:00:00
categories:
- [读书笔记, SurviveJSWebpack]
tags:
- webpack
- 读书笔记
---
原文为线上开源的英文书籍。你可以点击[这里](https://survivejs.com/webpack/foreword/)阅读原书籍。
本篇对应书本第六篇，webpack优化的几种方式。
<!--more-->
## 压缩

### 压缩js
&emsp;&emsp;webpack4以上的版本，在 mode 为 production 的模式下，会自动为 javascript 进行压缩，压缩采用的是原先的插件UglifyJS。

&emsp;&emsp;自定义配置压缩js能够进一步的进行代码的压缩甚至于执行js提速，但同时也是*不安全*的。这些压缩插件在编译过程中会重命名变量(time -> a)，移除无意义的代码(if(false))，这些方式在某些特定情况下会对原有的程序造成破坏。

&emsp;&emsp;webpack中对js的压缩配置项主要在`optimization.minimize/minimizer`中(minimizer接收一个配置数组)。现在来引入`terser-webpack-plugin`插件来作为示例改变webpack默认的压缩行为。

```javascript
// npm add terser-webpack-plugin --develop

// webpack.parts.js
const TerserPlugin = require("terser-webpack-plugin");

exports.minifyJavaScript = () => ({
  optimization: { minimizer: [new TerserPlugin()] },
});

// webpack.config.js
const productionConfig = merge([
  parts.minifyJavaScript(),
  // ...
]);
```

### 压缩html/css

&emsp;&emsp;html压缩的收益不高，因为标签名称和标签写法都是定死的，最多压缩一些空格部分。常用的压缩html插件有 `posthtml-minifier` 以及`posthtml-minify-classnames`。后者可以用来压缩class的名称。

&emsp;&emsp;css压缩大多使用 `css-minimizer-webpack-plugin` 插件。

```javascript
// npm add css-minimizer-webpack-plugin --develop

// webpack.parts.js
const CssMinimizerPlugin = require("css-minimizer-webpack-plugin");

exports.minifyCSS = ({ options }) => ({
  optimization: {
    minimizer: [
      new CssMinimizerPlugin({ minimizerOptions: options }),
    ],
  },
});

// webpack.config.js
const productionConfig = merge([
  // ...
  parts.minifyJavaScript(),
  parts.minifyCSS({ options: { preset: ["default"] } }),
  //...
]);
```

&emsp;&emsp;此时运行`npm run build`，就会发现css的体积变小了。css的注释会被删除，部分类会被合并。

## Tree Shaking

&emsp;&emsp;Tree Shaking 是 webpack 依据ES2015(ES6)的模块规范开发的一个只引入用到模块代码的打包优化方式，来举个例子：

```javascript
// src/shake.js
const shake = () => console.log("shake");
const bake = () => console.log("bake");

export { shake, bake };

// src/index.js
import { bake } from "./shake";

bake();
```

&emsp;&emsp;这里我们引入bake函数，而没有引入shake。执行`npm run build`之后到编译的源码当中去查找，是查找不到`console.log("shake")`这行代码的。

&emsp;&emsp;插件`babel-plugin-transform-imports`是一个能够依据webpack的 Tree Shaking 逻辑来只引入使用到的外部三方包（诸如antd,React等）的一个插件。如果你是一个三方包的作者，你可以使用`sideEffects": false`和 es2015 的语法来告诉 webpack可以放心的对这个三方包使用 Tree Shaking 逻辑来进行优化。

## 环境变量
&emsp;&emsp;在诸如react这种三方包在打包过程中经常会用到仅在开发模式下的代码。比如React的一些开发模式下的错误排查和错误信息提示，这部分代码需求仅仅在开发模式下被打包进来。一般在不使用配置的情况下是这样进行代码配置：

```javascript
if (process.env.NODE_ENV === "development") {
  console.log("Hello during development");
}
```
&emsp;&emsp;实际上这部分代码在产品模式下是不会被使用到的，应该被移除掉。此时就需要使用 DefinePlugin 插件进行配置。现在假设我们有如下的代码：

```javascript
var foo;

if (foo === "bar") {
  console.log("bar");
}

if (bar === "bar") {
  console.log("bar");
}
```

&emsp;&emsp;此时如果我们使用 DefinePlugin 把 bar 变量全部替换为诸如`foobar`的字符串。这里代码的第二个if条件等同于if(false)就会被直接移除掉。使用这个逻辑，可以来编写仅在开发模式下使用的代码，在产品模式下这部分代码会被移除。

```javascript
var foo;

if (foo === "bar") {
  console.log("bar");
}

if ("foobar" === "bar" /* 等同于 if(false) */) {
  console.log("bar");
}
```

&emsp;&emsp;我们来看下 DefinePlugin 的具体使用：

```javascript
// webpack.parts.js
exports.setFreeVariable = (key, value) => {
  const env = {};
  env[key] = JSON.stringify(value);

  return {
    plugins: [new webpack.DefinePlugin(env)],
  };
};

// webpack.config.js
const commonConfig = merge([
  //...
  parts.setFreeVariable("HELLO", "hello from config"),
]);

// src/component.js
export default (text = HELLO) => {
  const element = document.createElement("div");
  // ...
};
```

&emsp;&emsp;如果你运行这个应用，HELLO变量全部会被替换为字符串'hello from config'。

## 给打包后的文件名增加hash值
&emsp;&emsp;webpack提供了很多文件名配置相关的占位符，用于动态的配置打包后的文件名称。

- [id] chunk id
- [path] 文件路径
- [name] 文件名称
- [ext] 文件拓展名
- [fullhash] 构建的hash，每次构建都有一个唯一的hash
- [chunkhash] 包hash，每次entry的文件更改都会改变
- [contenthash] 根据内容生成的hash

&emsp;&emsp;这些配置在任意webpack设计到文件名相关配置的地方，包括webpack自身的配置，插件等。

```javascript
const config = {
  output: {
    path: PATHS.build,
    filename: "[name].[contenthash].js",
  },
};

const productionConfig = merge([
  {
    output: {
      chunkFilename: "[name].[contenthash].js",
      filename: "[name].[contenthash].js",
      assetModuleFilename: "[name].[contenthash][ext][query]",
    },
  },
  // ...
]);


exports.extractCSS = ({ options = {}, loaders = [] } = {}) => {
  return {
    // ...
    plugins: [
      new MiniCssExtractPlugin({
        filename: "[name].[contenthash].css",
      }),
    ],
  };
};
```

## 分割运行时
&emsp;&emsp;webpack可以配置分割出运行时文件，用于后续的更改只改变代码改变包的hash。

```javascript
// webpack.config.js
const productionConfig = merge([
  // ...
  {
    optimization: {
      splitChunks: {
        // ...
      },
      runtimeChunk: { name: "runtime" },
    },
  },
]);
```

&emsp;&emsp;更改配置以后运行`npm run build`。
```bash
npm run build
⬡ webpack: Build Finished
⬡ webpack: assets by path *.js 130 KiB
    asset vendor.16...22.js 126 KiB [emitted] [immutable] [minimized] (name: vendor) (id hint: commons) 2 related assets
    asset runtime.41...f8.js 3.01 KiB [emitted] [immutable] [minimized] (name: runtime) 2 related assets
    asset main.ed...dd.js 633 bytes [emitted] [immutable] [minimized] (name: main) 2 related assets
    asset 34.a4...c5.js 257 bytes [emitted] [immutable] [minimized] 2 related assets
  asset main.ac...a1.css 1.87 KiB [emitted] [immutable] (name: main)
  asset index.html 324 bytes [emitted]
...
webpack 5.1.3 compiled successfully in 7209 ms
```

&emsp;&emsp;此时可以发现打包出来的文件多了一个`runtime`的文件。该文件不会被index.html引用，仅作为webpack自身记录、配置用。这时候更改`index.js`的内容再运行`npm run build`，可以发现只有 runtime 和 app 的文件名上的哈希值改变，而vendor的没有改变。

## 优化
&emsp;&emsp;webpack开箱即用的功能只能适用于小的项目，当项目越来越大之后，必须要进行对应的构建/打包项目优化。

### 构建
- 在开发模式下使用更快速的 source map 或者不用。
- 使用 @babel/preset-env 插件在开发模式下替代 source map。
- 在开发模式下不使用polyfills
- 跳过不必要的loader，比如在开发模式下使用现代浏览器可用跳过 babel-loader 插件或者 equivalent。
- 尽量使用 include 和 exclude 明确的指定插件范围。在不指定的情况下，webpack会去遍历所有的文件包括node_modules。
- 缓存开销特别大loader，比如可以使用 cache-loader 。

### 热更新
&emsp;&emsp;可以通过指定三方包不解析，直接引入打包后文件的方式来进行构建上的优化。如下面的例子，指定不要解析react包，而直接引入已经打包好的产品模式下的react包。

```javascript
exports.dontParse = ({ name, path }) => ({
  module: { noParse: [new RegExp(path)] },
  resolve: { alias: { [name]: path } },
});

dontParse({
  name: "react",
  path: path.resolve(
    __dirname, "node_modules/react/cjs/react.production.min.js",
  ),
}),
```



