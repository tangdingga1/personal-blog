---
title: 《SurviveJS - Webpack》读书笔记(1-2)
date: 2020/10/14 16:00:00
categories:
- [读书笔记, SurviveJSWebpack]
tags:
- webpack
- 读书笔记
---
原文为线上开源的英文书籍。你可以点击[这里](https://survivejs.com/webpack/foreword/)阅读原书籍。
本篇对应书本前两篇，讲了什么是webpack，webpack的基本开发。
<!--more-->
## 什么是webpack
&emsp;&emsp;webpack是一个模块打包器。通过定义入(input)出(output)，webpack能自行遍历模块之间的引入，根据自定义的配置，生成对应的文件。

&emsp;&emsp;webpack支持各种模块和标准，诸如 ES2015, CommonJS, MJS, AMD 甚至 WebAssembly，通过导入器(loader)也能很好支持诸如css的文件。你可以使用插件(plugins)来拓展各种便利的诸如热更新压缩代码等便利的功能。

&emsp;&emsp;下图webpack执行流程简图：
![webpack process](https://survivejs.com/538c4af0d21e375d6d252d38cbb8a993.png)

&emsp;&emsp;webpack依赖 loader 和 plugins ，通过一份配置文件来进行文件处理的定义。webpack 通过定义 loader 能够处理任何类型的文件，所有定义的 loader 将被从下往上，从右往左的顺序进行执行。plugins则更多的对webpack本身的进程运行进行拓展，比如代码热更新，浏览器自动刷新，css/js 的代码批量分割。

## webpack开发

### 开始
&emsp;&emsp;开始使用 webpack 前，需要安装好对应的 [node](https://nodejs.org/en/) 和 npm 环境。

&emsp;&emsp;首先需要新建一个文件夹设置对应的 `package.json` 文件。package.json 是一个管理 npm 项目依赖的配置文件。

```bash
mkdir webpack-demo
cd webpack-demo
npm init -y # -y 根据默认配置生成package.json文件，也可以直接使用 npm init 手动生成配置。
```

&emsp;&emsp;使用 npm 来安装 webpack 和 webpack-nano。随后使用 `npx wp` 运行服务。

```bash
npm add webpack webpack-nano --develop # --develop === -D
npx wp

# result
⬡ webpack: Build Finished
⬡ webpack: Hash: 80f545d7d31df2164016
  Version: webpack 4.44.1
  Time: 29ms
  Built at: 08/21/2020 9:24:56 AM

WARNING in configuration
The 'mode' option has not been set, webpack will fallback to 'production' for this value. Set 'mode' option to 'development' or 'production' to enable defaults for each environment.
You can also set it to 'none' to disable any default behavior. Learn more: https://webpack.js.org/configuration/mode/

ERROR in Entry module not found: Error: Can't resolve './src' in '/tmp/webpack-demo'
```

&emsp;&emsp;错误提示没有对应的文件来供 webpack 进行编译，来新增src的文件夹以及对应的文件来进行处理。

```javascript
// src/component.js
export default (text = "Hello world") => {
  const element = document.createElement("div");
  element.innerHTML = text;
  return element;
};

// src/index.js
import component from "./component";

document.body.appendChild(component());
```
&emsp;&emsp;此时再次运行`npx wp`，运行成功，出现了 dist 文件夹，里面包含了编译的 js 文件。我们继续添加新的插件让编译出来的文件直接到浏览器可用。

&emsp;&emsp;添加名为`mini-html-webpack-plugin`的一个插件，这个插件能帮你根据配置生成对应的 html 文件。然后新建一个名为 `webpack.config.js` 的文件，这个文件用于webpack的启动配置。
```bash
npm add mini-html-webpack-plugin --develop
touch webpack.config.js
```

&emsp;&emsp;mini-html-webpack-plugin 的插件需要配置在webpack配置文件中的 `plugins` 中使用。

```javascript
// webpack.config.js
const { mode } = require("webpack-nano/argv");
const {
  MiniHtmlWebpackPlugin,
} = require("mini-html-webpack-plugin");

module.exports = {
  mode,
  plugins: [
    new MiniHtmlWebpackPlugin({
      context: {
        title: "Webpack demo",
      },
    }),
  ],
};
```

&emsp;&emsp;此时再运行命令，同时加入`mode`的命令配置，mode对应 development/none/production。编译完毕后进入dist文件运行`npx serve`。webpack 会帮你自动安装服务并运行在浏览器上。

```bash
npx wp --mode production
cd dist
npx serve
```

&emsp;&emsp;运行`npx wp --mode production`后你会看到类似于下面的输出:

```bash
⬡ webpack: Build Finished
⬡ webpack: Hash: b3d548da335f2c806f02
  Version: webpack 4.44.1
  Time: 60ms
  Built at: 08/21/2020 9:29:03 AM
       Asset       Size  Chunks             Chunk Names
  index.html  198 bytes          [emitted]
     main.js   1.04 KiB       0  [emitted]  main
  Entrypoint main = main.js
  [0] ./src/index.js + 1 modules 219 bytes {0} [built]
      | ./src/index.js 77 bytes [built]
      | ./src/component.js 142 bytes [built]
```

- Hash: b3d548da335f2c806f02，本次构建的hash值，可以添加到文件名当中，后续会讲到详细的用法。
- Version: webpack 4.44.1，webpack的版本号
- Time: 60ms， 本次构建的用时
- main.js 1.04 KiB 0 [emitted] main 生成的文件名以及对应的文件大小

&emsp;&emsp;还可以在`package.json`文件中添加对应的快捷命令，如下：
```javascript
// package.json
{
  "scripts": {
    "build": "wp --mode production"
  }
}
```
&emsp;&emsp;后续只要运行 npm run + scripts 中对应的key值就能执行后面的命令。比如运行`npm run build`实际上等于运行 `wp --mode production`。

### 开发服务
&emsp;&emsp;本章介绍 webpack 为开发者提供的各种便捷的服务功能。

### 监听
&emsp;&emsp;使用 `--watch` 能够开启一个进程让 webpack 监听依赖文件的内容变化，每次变化自动进行编译，不用每次都执行一遍 build 命令。

```bash
npm run build -- --watch
```

### 热加载
&emsp;&emsp;添加`webpack-plugin-serve`插件来添加浏览器热加载功能，执行`npm add webpack-plugin-serve --develop`。

&emsp;&emsp;随后在 package.json 中添加两个快捷命令，修改 webpack.config.js 文件的 webpack 配置。

```javascript
// package.json
{
  "scripts": {

    "start": "wp --mode development",

    "build": "wp --mode production"
  },
  ...
}

// webpack.config.js
const { mode } = require("webpack-nano/argv");
const {
  MiniHtmlWebpackPlugin,
} = require("mini-html-webpack-plugin");
const { WebpackPluginServe } = require("webpack-plugin-serve");

module.exports = {
  watch: mode === "development",
  entry: ["./src", "webpack-plugin-serve/client"],
  mode,
  plugins: [
    new MiniHtmlWebpackPlugin({
      context: {
        title: "Webpack demo",
      },
    }),
    new WebpackPluginServe({
      port: process.env.PORT || 8080,
      static: "./dist",
      liveReload: true,
      waitForBuild: true,
    }),
  ],
};
```

&emsp;&emsp;此时运行`npm run start`，服务成功启动在8080端口，可以通过输入`http://localhost:8080/`访问。每次你更改你代码时候，webpack都会自动根据更改进行编译，将更新的代码同步到浏览器上面。你也可以通过查询本机的ip来把你的服务同步到网络上供其他人访问。在mac上运行`ifconfig | grep inet`，win上为`ipconfig`，查到本机的ip，提供给其它人。

&emsp;&emsp;使用`nodemon`来让每次你更改 webpack.config.js 的时候自动重启 webpack 服务，不用每次手动结束进程重启。使用`npm add nodemon --develop`进行安装。然后修改 package.json 中的配置。

```javascript
// package.json

"scripts": {
  "start": "nodemon --watch webpack.* --exec \"wp --mode development\"",
  "build": "wp --mode production"
},
```

### 合并配置
&emsp;&emsp;当后续章节越来越多的时候，webpack 的配置文件内容也会越来越多。需要在这一步先对配置文件做一个合理的划分规划。这里会用到一个`webpack-merge`的库。运行来安装`npm add webpack-merge --develop`。它的作用类似于这样：

```javascript
> { merge } = require("webpack-merge")
...
> merge(
... { a: [1], b: 5, c: 20 },
... { a: [2], b: 10, d: 421 }
... )
{ a: [ 1, 2 ], b: 10, c: 20, d: 421 }
```

&emsp;&emsp;我们新建一个`webpack.parts.js`文件，来分离一些配置。

```javascript
const { WebpackPluginServe } = require("webpack-plugin-serve");
const {
  MiniHtmlWebpackPlugin,
} = require("mini-html-webpack-plugin");

exports.devServer = () => ({
  watch: true,
  plugins: [
    new WebpackPluginServe({
      port: process.env.PORT || 8080,
      static: "./dist", // Expose if output.path changes
      liveReload: true,
      waitForBuild: true,
    }),
  ],
});

exports.page = ({ title }) => ({
  plugins: [
    new MiniHtmlWebpackPlugin({
      context: {
        title,
      },
    }),
  ],
});
```

&emsp;&emsp;然后我们更改对应的`webpack.config.js`内容。这样处理后能使webpack配置文件变得可拓展，阅读性也更高。
```javascript
const { mode } = require("webpack-nano/argv");
const { merge } = require("webpack-merge");
const parts = require("./webpack.parts");

const commonConfig = merge([
  {
    entry: ["./src"],
  },
  parts.page({ title: "Webpack demo" }),
]);

const productionConfig = merge([]);

const developmentConfig = merge([
  {
    entry: ["webpack-plugin-serve/client"],
  },
  parts.devServer(),
]);

const getConfig = (mode) => {
  switch (mode) {
    case "production":
      return merge(commonConfig, productionConfig, { mode });
    case "development":
      return merge(commonConfig, developmentConfig, { mode });
    default:
      throw new Error(`Trying to use an unknown mode, ${mode}`);
  }
};

module.exports = getConfig(mode);
```
