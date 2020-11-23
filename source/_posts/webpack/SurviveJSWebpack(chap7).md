---
title: 《SurviveJS - Webpack》读书笔记(7)
date: 2020/11/20 19:00:00
categories:
- [读书笔记, SurviveJSWebpack]
tags:
- webpack
- 读书笔记
---
原文为线上开源的英文书籍。你可以点击[这里](https://survivejs.com/webpack/foreword/)阅读原书籍。
本篇对应书本第七篇，webpack文件输出构建环境。
<!--more-->
## 构建目标
&emsp;&emsp;webpack除了常用于构建web应用之外，还可以构建到NODE（服务器环境）和桌面应用（Electron）。

- web端
&emsp;&emsp;默认打包的构建方式。在webpack5之后browserslist的配置项被增加进了设置中，方便支持在比较老版本的浏览器中运行代码。

- Node（服务器端）
&emsp;&emsp;webpack支持 node 和 async-node 两种模式。详细的内容在ssr小节部分讲解。

- 桌面
&emsp;&emsp;webpack支持node-webkit（NW.js）和atom/electron/electron-main(Electron main process)。

## 多页面
&emsp;&emsp;打包多页面，使用原有的用于打包html的插件（mini-html-webpack-plugin和html-webpack-plugin）就可以很方便的完成，只需要加一点多页面的配置项。我们新增一个名为 webpack.multi.js 的 webpack 配置文件专门用于维护 webpack 的多页面配置打包。

```javascript
// webpack.parts.js
const {
  MiniHtmlWebpackPlugin,
} = require("mini-html-webpack-plugin");

exports.page = ({ url = "", title, chunks } = {}) => ({
  plugins: [
    new MiniHtmlWebpackPlugin({
      publicPath: "/",
      chunks,
      filename: `${url && url + "/"}index.html`,
      context: { title },
    }),
  ],
});

// webpack.multi.js
const { merge } = require("webpack-merge");
const parts = require("./webpack.parts");

module.exports = merge(
  { mode: "production", entry: { app: "./src/multi.js" } },
  parts.page({ title: "Demo" }),
  parts.page({ title: "Another", url: "another" })
);

// 增加指定的js
// src/multi.js
const element = document.createElement("div");
element.innerHTML = "hello multi";
document.body.appendChild(element);

// 增加执行脚本 package.json
{
  "scripts": {

    "build:multi": "wp --config webpack.multi.js",

    // ...
  },
}
```
&emsp;&emsp;处理好配置后运行npm run build:multi，就能够得到两个html文件，在/目录以及/another目录下。

## 服务端渲染
&emsp;&emsp;服务端渲染，Server-Side Rendering (SSR)，是一种在服务器端渲染html文件内容，遇到请求时直接返回的技术。与常规的使用三大框架（react/vue/angular）的浏览器端渲染的技术不同，非常有利于seo和首屏的页面表现。

&emsp;&emsp;这边我们使用react的服务端渲染的技术来作为配置的距离。react框架支持多平台的渲染，除了浏览器端渲染之后，还支持服务器端的渲染。我们重新开始，创建一个配置服务端渲染的webpack项目。首先安装需要的依赖插件：

```bash 
npm add babel-loader @babel/core @babel/preset-react --develop
npm add react react-dom

# .babelrc
{
  "presets": [
    ["@babel/preset-env", { "modules": false }],
    "@babel/preset-react"
  ]
}
```
&emsp;&emsp;配置文件：
```javascript
// 文件 src/ssr.js
const React = require("react");
const ReactDOM = require("react-dom");
const SSR = <div onClick={() => alert("hello")}>Hello world</div>;

// Render only in the browser, export otherwise
if (typeof document === "undefined") {
  module.exports = SSR;
} else {
  ReactDOM.hydrate(SSR, document.getElementById("app"));
}

// 文件 webpack.ssr.js
const path = require("path");
const APP_SOURCE = path.join(__dirname, "src");

module.exports = {
  mode: "production",
  entry: { index: path.join(APP_SOURCE, "ssr.js") },
  output: {
    path: path.join(__dirname, "static"),
    filename: "[name].js",
    libraryTarget: "umd",
    globalObject: "this",
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        include: APP_SOURCE,
        use: "babel-loader",
      },
    ],
  },
};

// 文件 package.json
{
  "scripts": {

    "build:ssr": "wp --config webpack.ssr.js",

    // ...
  }
}
```

&emsp;&emsp;此时运行`npm run build:ssr`可以看见打包的文件出现在`./static/index.js`。接下来配置对应的服务来渲染它。
```javascript
// npm add express --develop

// 创建服务文件server.js
const express = require("express");
const { renderToString } = require("react-dom/server");
const SSR = require("./static");

const app = express();
app.use(express.static("static"));
app.get("/", (req, res) =>
  res.status(200).send(renderMarkup(renderToString(SSR)))
);
app.listen(process.env.PORT || 8080);

function renderMarkup(html) {
  return `<!DOCTYPE html>
<html>
  <head><title>SSR Demo</title><meta charset="utf-8" /></head>
  <body>
    <div id="app">${html}</div>
    <script src="./index.js"></script>
  </body>
</html>`;
}
```

&emsp;&emsp;运行node ./server.js起到服务，到 http://localhost:8080 之后就可以看到Hello World的页面。打开开发者面板查看source中的html，你会发现这部分操作是在服务器就完成，然后发送给你渲染完毕后的页面的。