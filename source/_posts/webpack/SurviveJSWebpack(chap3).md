---
title: 《SurviveJS - Webpack》读书笔记(3)
date: 2020/10/19 16:00:00
categories:
- [读书笔记, SurviveJSWebpack]
tags:
- webpack
- 读书笔记
---
原文为线上开源的英文书籍。你可以点击[这里](https://survivejs.com/webpack/foreword/)阅读原书籍。
本篇对应书本第三篇，讲了如何处理css。
<!--more-->
&emsp;&emsp;webpack 无法对 css 做到开箱即用，需要配置对应的 loaders 和 plugins 来处理 css 文件。

## 加载css
&emsp;&emsp;加载css文件需要使用到 css-loader 和 style-loader。 css-loader 能让 webpack 可以正常去寻找引入使用 `@import` 和 `url()`方式引入的文件，style-loader则是把 css 文件以行内式（style标签+样式）的方式加载到 html 中。

&emsp;&emsp;引入两个loader并添加对应的配置：
```javascript
// bash
npm add css-loader style-loader --develop

// webpack.parts.js
exports.loadCSS = ({ include, exclude } = {}) => ({
  module: {
    rules: [
      {
        test: /\.css$/,
        include,
        exclude,
        use: ["style-loader", "css-loader"],
      },
    ],
  },
});

// 加入到webpack.config.js
const commonConfig = merge([
  // ...
  parts.loadCSS(),
]);
```

&emsp;&emsp;webpack为各个流行的css库提供了对应的loaders。比如 less 对应 `less-loader`，sass有 `sass-loader`，Stylus有 `stylus-loader`。

&emsp;&emsp;值得一提的是 css-loader 默认只支持相对路径，不支持绝对路径，比如下面两种方式：
```javascript
url("https://mydomain.com/static/demo.png")
url("/static/img/demo.png")
```

## 分割css
&emsp;&emsp;经过上面的配置我们发现 css 文件会以行内式的方式插入的 html 中，这个插入动作实际上是 loaders 放到 js 当中去做了。这样子在开发中能够处理，但在产品打包上线之后不是一个很好的处理方式，我们需要把 css 单独分割出来作为一个文件。这里就要用到 webpack 的 `mini-css-extract-plugin`的 plugins。

```javascript
// bash
npm add mini-css-extract-plugin --develop

const MiniCssExtractPlugin = require("mini-css-extract-plugin");

// webpack.parts.js
exports.extractCSS = ({ options = {}, loaders = [] } = {}) => {
  return {
    module: {
      rules: 
        {
          test: /\.css$/,
          use: [
            { loader: MiniCssExtractPlugin.loader, options },
            "css-loader",
          ].concat(loaders),
          // If you distribute your code as a package and want to
          // use _Tree Shaking_, then you should mark CSS extraction
          // to emit side effects. For most use cases, you don't
          // have to worry about setting flag.
          sideEffects: true,
        },
      ],
    },
    plugins: [
      new MiniCssExtractPlugin({
        filename: "[name].css",
      }),
    ],
  };
};
```

&emsp;&emsp;然后再webpack.config.js中对应 production 和 development 进行单独配置。在开发模式下需要用到热加载的功能。

```javascript
// webpack.config.js
const productionConfig = merge([
  parts.extractCSS(),
]);


const developmentConfig = merge([
  // ...
  parts.extractCSS({ options: { hmr: true } }),
]);
```

## 去除冗余的css
&emsp;&emsp;引入诸如 Bootstrap 或者 Tailwind 这种框架css时会有大量冗余的项目中未使用到的css被一起跟随文件打包进来。此时可以用webpack的 `postcss-loader` 来进行处理，去除那些没用到的css。同时安装 `tailwindcss` 和 `postcss-loader`，来感受下配置前后的文件大小不同。

```javascript
npm add tailwindcss postcss-loader --develop

exports.tailwind = () => ({
  loader: "postcss-loader",
  options: {
    postcssOptions: {
      plugins: [require("tailwindcss")()],
    },
  },
});


// webpack.config.js
const cssLoaders = [parts.tailwind()];

const productionConfig = merge([
  parts.extractCSS({ loaders: cssLoaders }),
]);

const developmentConfig = merge([
  parts.devServer(),
  parts.extractCSS({ options: { hmr: true }, loaders: cssLoaders }),
]);
```

&emsp;&emsp;创建对应的css文件和js文件。引入 tailwindcss。

```javascript
// src/main.css
@tailwind base;
@tailwind components;

/* Write your utility classes here */

@tailwind utilities;

body {
  background: cornsilk;
}

// src/component.js
export default (text = "Hello world") => {
  const element = document.createElement("div");

  element.className = "rounded bg-red-100 border max-w-md m-4 p-4";
  element.innerHTML = text;

  return element;
};
```

&emsp;&emsp;运行打包后可以看到webpack的输出信息，main.css文件达到了1mb以上的大小。

```bash
Hash: 6ed243a3e5aade0133d5
Version: webpack 4.43.0
Time: 1282ms
Built at: 07/09/2020 4:24:36 PM
     Asset       Size  Chunks                    Chunk Names
index.html  237 bytes          [emitted]
  main.css   1.32 MiB       0  [emitted]  [big]  main
   main.js   1.12 KiB       0  [emitted]         main
Entrypoint main [big] = main.css main.js
[0] ./src/main.css 39 bytes {0} [built]
[1] ./src/index.js + 1 modules 315 bytes {0} [built]
    | ./src/index.js 99 bytes [built]
    | ./src/component.js 211 bytes [built]
    + 1 hidden module
...
```
&emsp;&emsp;我们现在引入 purgecss 插件，去除未使用的css。

```javascript
// bash
npm add glob purgecss-webpack-plugin --develop

// webpack.parts.js
const path = require("path");
const glob = require("glob");
const PurgeCSSPlugin = require("purgecss-webpack-plugin");

const ALL_FILES = glob.sync(path.join(__dirname, "src/*.js"));

exports.eliminateUnusedCSS = () => ({
  plugins: [
    new PurgeCSSPlugin({
      paths: ALL_FILES, // Consider extracting as a parameter
      extractors: [
        {
          extractor: (content) =>
            content.match(/[^<>"'`\s]*[^<>"'`\s:]/g) || [],
          extensions: ["html"],
        },
      ],
    }),
  ],
});

// webpack.config.js
const productionConfig = merge([
  parts.extractCSS({ loaders: cssLoaders }),
  parts.eliminateUnusedCSS(),
]);
```

&emsp;&emsp;此时运行 `npm run build` 查看打包文件大小，会发现由原来的 1mb 变为了 7.28 kb。

```bash
Hash: 6ed243a3e5aade0133d5
Version: webpack 4.43.0
Time: 1494ms
Built at: 07/09/2020 4:35:27 PM
     Asset       Size  Chunks             Chunk Names
index.html  237 bytes          [emitted]
  main.css   7.28 KiB       0  [emitted]  main
   main.js   1.12 KiB       0  [emitted]  main
...
```

## 自动补齐前缀
&emsp;&emsp;css中经常有一些兼容浏览器的前缀，比如`-webkit-`或者`-moz-`，借助 `autoprefixer` 可以自动补齐生成。

```javascript
// bash
npm add postcss-loader autoprefixer --develop

// webpack.parts.js
exports.autoprefix = () => ({
  loader: "postcss-loader",
  options: {
    postcssOptions: {
      plugins: [require("autoprefixer")()],
    },
  },
});

// webpack.config.js
const cssLoaders = [parts.autoprefix(), parts.tailwind()];
```

&emsp;&emsp;`autoprefixer`的配置需要借助外部定义的文件来进行设置。我们新建一个名为`.browserslistrc`的文件。

```bash
touch .browserslistrc

# .browserslistrc
> 1% # Browser usage over 1%
Last 2 versions # Or last two versions
IE 8 # Or IE 8
```

&emsp;&emsp;配置完成后运行`npm run build`，css会自动加上浏览器兼容的前缀。