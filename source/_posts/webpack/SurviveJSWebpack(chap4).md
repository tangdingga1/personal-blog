---
title: 《SurviveJS - Webpack》读书笔记(4)
date: 2020/10/27 14:00:00
categories:
- [读书笔记, SurviveJSWebpack]
tags:
- webpack
- 读书笔记
---
原文为线上开源的英文书籍。你可以点击[这里](https://survivejs.com/webpack/foreword/)阅读原书籍。
本篇对应书本第四篇，几种常见静态资源的处理。
<!--more-->
&emsp;&emsp;webpack 除了能开箱即用 js 文件外，其它类型的文件都需要配置对应的 loader 才能进行引入使用。


## 配置 loaders
&emsp;&emsp;通过设置 loaders 以及配置关联文件名等方式来启用 webpack 的 loader。下面的例子就是配置了 loader，使用 babel 来加载js。
```javascript
const config = {
  module: {
    rules: [
      {
        // **Conditions** to match files using RegExp, function.
        test: /\.js$/,

        // **Restrict** matching to a directory.
        include: path.join(__dirname, "app"),
        exclude: (path) => path.match(/node_modules/);

        // **Actions** to apply loaders to the matched files.
        use: "babel-loader",
      },
    ],
  },
};
```
&emsp;&emsp;loaders的执行顺序为定义的从右往左，如下例：

```javascript
{
  test: /\.css$/,
  use: ["style-loader", "css-loader"],
},
```

&emsp;&emsp;或者从下往上。上面的例子可以拆散开：
```javascript
{
  test: /\.css$/,
  use: "style-loader",
},
{
  test: /\.css$/,
  use: "css-loader",
},
```

&emsp;&emsp;可以使用 enforce 指令，来指定该配置项最先还是最后执行。该项接收 `pre/post` 两个值，来制定 loaders `最先/最后`执行。 

```javascript
{
  // Conditions
  test: /\.js$/,
  enforce: "pre", // "post" too
  // Actions
  use: "eslint-loader",
},
```

&emsp;&emsp;loaders的配置项可以使用行内式的方式，使用类似于 url 的 query 方式来进行传递：

```javascript
{
  // Conditions
  test: /\.js$/,
  include: PATHS.app,

  // Actions
  use: "babel-loader?presets[]=env",
},
```

&emsp;&emsp;上面的例子也可以拆开来使用 options 来进行单独配置，两种书写方式是一致的：
```javascript
{
  // Conditions
  test: /\.js$/,
  include: PATHS.app,

  // Actions
  use: {
    loader: "babel-loader",
    options: {
      presets: ["env"],
    },
  },
},
```

&emsp;&emsp;使用多个 loaders 的时候直接传递数组给 use 配置项即可：

```javascript
{
  test: /\.js$/,
  include: PATHS.app,
  // loaders
  use: [
    {
      loader: "babel-loader",
      options: {
        presets: ["env"],
      },
    },
  ],
},
```

&emsp;&emsp;除了直接编写 `webpack.config.js` 引入行内定义也可以来指定对应的 loaders。我们定义一张图片 foo.png，使用 url-loader 来进行引入。

```javascript
// Process foo.png through url-loader and other possible ones.
import "url-loader!./foo.png";

// 最高优先级的使用 url-loader 插件
import "!!url-loader!./bar.png";
```

&emsp;&emsp;配置文件的入口也可通过这种方式制定loaders。
```javascript
{
  entry: {
    app: "babel-loader!./app",
  },
},
```

&emsp;&emsp;webpack 支持多种匹配文件的方式。我们常用的test的其实包含了 `include` 和 `exclude` 的两个配置项。支持的配置如下：

-  test。支持正则、函数、字符串对象或者数组。
- include。如test。所有包含的文件。
- exclude。支持配置内容桶test，但表示所有忽略的文件。
- resource: /inline/。匹配所有路径下的文件，包括query。比如匹配/path/，那么/path/foo.inline.js和/path/bar.png?inline都会被匹配。
- issuer: /bar.js/。匹配所有文件下的请求。比如匹配/bar.js/，那么所有 bar.js 文件内的引入都会被匹配。
- resourcePath: /inline/。匹配所有路径下不带query的文件。比如匹配/path/，所有path下面的文件都会被匹配。
- resourceQuery: /inline/。匹配所有路径带query的文件。比如匹配/path/，/path/foo.png?inline会被匹配。

&emsp;&emsp;配置项中还能使用特定的参数来进行文件的匹配，这些字段包括`not/and/or`：

```javascript
{
  test: /\.css$/,
  rules: [
    { // CSS imported from other modules is added to the DOM
      issuer: { not: /\.css$/ },
      use: "style-loader",
    },
    { // Apply css-loader against CSS imports to return CSS
      use: "css-loader",
    },
  ],
}
```

&emsp;&emsp;你还可以使用函数形式来配置 use 字段，可以根据不同的环境路径来进行不同的 loaders 加载，函数执行返回项将会作为 use 的配置值。
```javascript
{
  test: /\.css$/,
  // `resource` refers to the resource path matched.
  // `resourceQuery` contains possible query passed to it
  // `issuer` tells about match context path
  use: ({ resource, resourceQuery, issuer }) => {
    // You have to return something falsy, object, or a
    // string (i.e., "style-loader") from here.
    //
    // Returning an array fails! Nest rules instead.
    if (env === "development") {
      return {
        use: {
          loader: "css-loader", // css-loader first
          rules: [
            "style-loader", // style-loader after
          ],
        },
      };
    }
  },
},
```
## 图片加载
&emsp;&emsp;`url-loader` 能够限制引入图片文件的大小，当文件小于限制时，直接以 base64 的方式嵌入到文件当中，而大于限制时，变为正常的url地址进行引入。

```javascript
{
  test: /\.(jpg|png)$/,
  use: {
    loader: "url-loader",
    // 25kB以下图片文件会转换为 base64 的方式行嵌到文件当中
    options: {
      limit: 25000,
    },
  },
},
```

&emsp;&emsp;如果不需要根据文件体积进行base64转换，可以直接使用 `file-loader`。该loader可以导入图片路径，根据打包配置生成对应的图片文件到文件夹。

```javascript
{
  test: /\.(jpg|png)$/,
  use: {
    loader: "file-loader",
    options: {
      // path - 路径
      // name - 文件名
      // contenthash - hash戳
      // ext 拓展名
      name: "[path][name].[contenthash].[ext]",
    },
  },
},
```

&emsp;&emsp;现在安装两个 loaders，并增加对应的配置项：
```bash
npm add file-loader url-loader --develop

// webpack.parts.js
exports.loadImages = ({ options } = {}) => ({
  module: {
    rules: [
      {
        test: /\.(png|jpg)$/,
        use: {
          loader: "url-loader",
          options,
        },
      },
    ],
  },
});

// webpack.config.js
const commonConfig = merge([
  ...
  parts.loadImages({
    options: {
      limit: 15000,
      name: "[name].[ext]",
    },
  }),
]);
```

&emsp;&emsp;webpack还提供了多种图片优化的loader：
- 生成图片的 srcsets。（responsive-loader / html-loader-srcset）
- 压缩图片。（image-webpack-loader / imagemin-webpack-plugin）
- 雪碧图。（webpack-spritesmith）

## 加载js文件
&emsp;&emsp;webpack支持转译js代码（支持到老版浏览器），压缩js代码。默认将会以 ES2015 的 module 语法进行编译。

&emsp;&emsp;更全面的使用处理js的功能，建议使用 Babel 来进行对应的处理。

```bash
# install
npm add babel-loader @babel/core --develop


# webpack.parts.js
const APP_SOURCE = path.join(__dirname, "src");
exports.loadJavaScript = () => ({
  module: {
    rules: [
      {
        test: /\.js$/,
        include: APP_SOURCE, // Consider extracting as a parameter
        use: "babel-loader",
      },
    ],
  },
});

# webpack.config.js
const commonConfig = merge([
  ...
  parts.loadJavaScript(),
]);
```

&emsp;&emsp;Babel 插件需要进行独立的文件配置，我们还需要安装 @babel/preset-env，并新建一个名为.babelrc的文件进行 babel 的配置。

```bash
# install
npm add @babel/preset-env --develop

touch .babelrc

# .babelrc
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "modules": false
      }
    ]
  ]
}
```

&emsp;&emsp;Polyfilling 是一种编译新的语法，使得其能支持旧浏览器的功能。Babel中如果想要使用 Polyfilling ，需要在配置项中打开 useBuiltIns，然后安装 core-js 的库。如果想要编译诸如 async 函数的语法，还需要安装 regenerator-runtime 的库。

&emsp;&emsp;在实际项目中会遇到需要生成多个js包的场景，版本新一些的浏览器使用新的包，老浏览器使用 Polyfilling 的js包，如下面的html所示：

```html
<!-- Browsers with ES module support load this file. -->
<script type="module" src="main.mjs"></script>

<!-- 老旧浏览器 (and module-supporting -->
<!-- browsers know *not* to load this file). -->
<script nomodule src="main.es5.js"></script>
```

&emsp;&emsp;我们使用之前定义的浏览器版本列表，现在修改配置使得打包出来的js为两个包，一个支持老浏览器（IE8），一个支持到新浏览器。

```bash
# .browserslistrc

# Let's support old IE
[legacy]
IE 8

# Make this more specific if you want
[modern]
> 1% # Browser usage over 1%

```

```javascript
// webpack.config.js
const getConfig = (mode) => {
  switch (mode) {
    case "production:legacy":
      process.env.BROWSERSLIST_ENV = 'legacy';

      return merge(commonConfig, productionConfig, { mode });
    case "production:modern":
      process.env.BROWSERSLIST_ENV = 'modern';

      return merge(commonConfig, productionConfig, { mode });
    ...
    default:
      throw new Error(`Trying to use an unknown mode, ${mode}`);
  }
};
// package.json

"scripts": {
  "build": "wp --mode production:legacy && wp --mode production:modern",
  ...
},
```

&emsp;&emsp;除了基础的js编译压缩功能外，webpack还支持拓展js的编译。拓展js指诸如typescript（微软研发的强类型的js），flow（facebook的强类型的js）编译成js。也支持把WebAssembly编译成js。




