---
title: 《SurviveJS - Webpack》读书笔记(7)
date: 2020/12/13 19:00:00
categories:
- [读书笔记, SurviveJSWebpack]
tags:
- webpack
- 读书笔记
---
原文为线上开源的英文书籍。你可以点击[这里](https://survivejs.com/webpack/foreword/)阅读原书籍。
本篇对应书本第八篇，构建外的一些小技巧。
<!--more-->
## 动态加载
&emsp;&emsp;require.context 提供了动态加载静态资源网页的方法。require.context 可以不通过webpack的配置文件来进行文件的引入，并指定解析引用的插件。require.context 函数返回一个对象，拥有文件的路径和id信息。

```javascript
const req = require.context(
  "json-loader!yaml-frontmatter-loader!./pages",
  true, // Load files recursively. Pass false to skip recursion.
  /^\.\/.*\.md$/ // 匹配所有结尾为 md 的文件
);

req.keys(); // ["./demo.md", "./another-demo.md"]
req.id; // 42
```

&emsp;&emsp;你也可以使用 require.context 函数来为 webpack 添加需要追踪的文件目标。

&emsp;&emsp;webpack还支持动态路径的加载，你可以指定不同的路径变量。然后使用 import 方法进行引入。

```javascript
import(`translations/${target}.json`).then(...).catch(...);
```


## 国际化
&emsp;&emsp;多语言国际化（i18n）是一个非常庞大的项目工程。现在来看结合 webpack 如何来使用。首先配置不同语言的翻译文件。

```javascript
// translations/en.json
{ "hello": "Hello world" }

// translations/fi.json
{ "hello": "Terve maailma" }
```

&emsp;&emsp;更改webpack，babel配置，并添加运行命令到package.json。
```javascript
// webpack.i18n.js
const path = require("path");
const {
  MiniHtmlWebpackPlugin,
} = require("mini-html-webpack-plugin");
const APP_SOURCE = path.join(__dirname, "src");

module.exports = {
  mode: "production",
  entry: { index: path.join(APP_SOURCE, "i18n.js") },
  module: {
    rules: [
      {
        test: /\.js$/,
        include: APP_SOURCE,
        use: "babel-loader",
      },
    ],
  },
  plugins: [new MiniHtmlWebpackPlugin()],
};

// Babel
{
  "presets": [
    ["@babel/preset-env", { "modules": false }],
    "@babel/preset-react"
  ]
}

// package.json
{
  "scripts": {
    "build:i18n": "wp --config webpack.i18n.js",
  }
}
```

&emsp;&emsp;最后建立国家化的文件。

```javascript
// src/i18n.js
import "regenerator-runtime/runtime";
import React, { useEffect, useState } from "react";
import ReactDOM from "react-dom";

const App = () => {
  const [language, setLanguage] = useState("en");
  const [hello, setHello] = useState("");

  const changeLanguage = () =>
    setLanguage(language === "en" ? "fi" : "en");

  useEffect(() => {
    translate(language, "hello")
      .then(setHello)
      .catch(console.error);
  }, [language]);

  return (
    <div>
      <button onClick={changeLanguage}>Change language</button>
      <div>{hello}</div>
    </div>
  );
};

function translate(locale, text) {
  return getLocaleData(locale).then((messages) => messages[text]);
}

async function getLocaleData(locale) {
  return import(`../messages/${locale}.json`);
}

const root = document.createElement("div");

root.setAttribute("id", "app");
document.body.appendChild(root);

ReactDOM.render(<App />, root);
```

&emsp;&emsp;最后运行`npm run build:i18n && npx serve dist`，就可以发现动态的国际化语言加载会根据点击的按钮来进行。

## 处理包文件


### resolve.alias
&emsp;&emsp;resolve.alias可以用一个名称来表示一段路径。比如下图的配置表示使用 `demo` 来表示当前文件夹到`node_modules/demo/dist/demo.js`的路径。

```javascript
const config = {
  resolve: {
    alias: {
      demo: path.resolve(
        __dirname,
        "node_modules/demo/dist/demo.js"
      ),
    },
  },
};
```

### resolve.modules
&emsp;&emsp;webpack 默认回去 node_modules 当中寻找依赖的包引用文件。这个配置命令可以让webpack不进入到node_modules，而去你指定的文件里面寻找包依赖引用。这个配置在开发时自行调试开发三方包代码时十分方便。

```javascript
const config = { resolve: { modules: ["demo", "node_modules"] } };
```

### resolve.extensions
&emsp;&emsp;webpack 默认只会加载处理 `js/mjs/json` 后缀文件。使用该配置能够增加默认处理文件后缀。比如添加 jsx 后缀到 webpack 默认处理中。

```javascript
const config = { resolve: { extensions: [".js", ".jsx"] } };
```

### 定义webpack外部引用

&emsp;&emsp;当我想要使用外部的链接资源，而不本地的包文件的时候，可以使用 externals 配置文件。比如我引入的jquery是挂载与外链下载的，而不是本地的包文件时，我这么进行配置。

```javascript
const config = { externals: { jquery: "jquery" } };

<script src="//ajax.googleapis.com/ajax/libs/jquery/3.1.1/jquery.min.js"></script>
<script>
  window.jQuery ||
    document.write(
      '<script src="js/jquery-3.1.1.min.js"><\/script>'
    );
</script>
```

### 处理全局变量
&emsp;&emsp;imports-loader 等于你往全局的所有文件前注入一句引用。比如我注入`import $ from 'jquery';`需要这么配置。

```javascript
const config = {
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: "imports-loader",
        options: {
          imports: ["default jquery $"],
        },
      },
    ],
  },
};
```

&emsp;&emsp;有时你需要暴露一个三方包。
```javascript
const config = {
  test: require.resolve("react"),
  loader: "expose-loader",
  options: {
    exposes: ["React"],
  },
};
```

## 链接
&emsp;&emsp;在开发模式下，可以通过 npm link 创建一些包的默认链接。让开发模式的包引用从 node_modules 改为你链接的文件目录。webpack默认也是支持npm link的链接的，你可以通过更改 resolve.symlinks 的配置为 false 来取消这一行为。


## 移除没有使用的包
&emsp;&emsp;即使包引入能够开箱即用，但是他们经常会带来很多没有使用到的依赖，比如Moment.js，打包时会把 locale data (多国语言信息)给打包进来。我们可以使用`IgnorePlugin`插件来解决这个问题。

```javascript
const config = {
  plugins: [
    new webpack.IgnorePlugin({
      resourceRegExp: /^\.\/locale$/,
      contextRegExp: /moment$/,
    }),
  ],
};
```
