---
title: package.json里面有啥
date: 2019/4/5 14:00
categories:
- [前端, node, npm]
tags:
- npm
- 知识点整理
---
## 前言
&emsp;&emsp;今年的2月19日是令我印象深刻的一天，不是因为这天是元宵节，而是因为这天晚上9点我生平头一回直面阿里面试官的面试，问倒了我一半问题。
&emsp;&emsp;在前端工程化部分他问了我两个问题：
1. 你知道`1，表示什么版本吗?
2. depends有四种依赖形式，你知道他们分别是什么吗？知道哪一种depends不会被安装吗?

&emsp;&emsp;对这方面仅仅是了解的我，愣是一个没答上来。
&emsp;&emsp;今天趁着清明假期，整理一下package.json当中到底有些什么。
<!--more-->
## Package.json是个啥
&emsp;&emsp;package.json到底是啥，它就是一个json文件呗，放在项目的主目录下面。
那么这个json文件到底用来干嘛，它是用来描述这个项目**相关的一切信息**，方便诸如在npm上发布项目包，使用npm安装项目包依赖。简而言之，就是按照规范写给npm的项目的说明书。让npm根据这份说明书去进行具体的逻辑操作。
&emsp;&emsp;你可以使用`npm init`命令来创建一个初始化的package.json项目包。
## 包发布相关字段
&emsp;&emsp;包发布相关字段，指这些字段用于描述你的项目，以便在npm上面能够查询和使用。
&emsp;&emsp;核心字段**name、version**。按照npm文档上面的描述，`the most important things in your package.json are the name and version fields as they will be required`。这两个字段是必须的，也是最重要的。用于描述你包的名字和版本号。其他的都可以不要。
&emsp;&emsp;其中，**name**建议使用`@pacakname/module`的方式进行命名，以免重复。比如babel的包，就采用`@babel/core`这样的@包名/模块名的方式来进行命名。而react就很任性的使用`react-dom`这种`-`的方式来进行命名。
&emsp;&emsp;**version**则建议使用1.0.0，三段式进行声明，意思为大版本.小版本.小修复。基本上小的一些bug修复都只改第三个数字，兼容这一版本的一些改动，会去修改第二个的小版本号。如果是极大的影响架构兼容性的改动，就放到大版本号。比如著名的angular1到angular2的颠覆式修改。这里的数字就是大版本号。(ps，这也就是angular成为垫底前端框架原因之一吧)。
## 包描述相关字段
&emsp;&emsp;这些字段是用来描述你的包一些相关信息，以便在npm上面可以使用标签等搜索到你的包，也可以介绍包的一些相关情况。
&emsp;&emsp;**description**和**keywords**,就等同于html中meta标签的相关字段。用于npm查询的字段，介绍你的表关键词和一些描述。
&emsp;&emsp;**homepage**、**bugs**、**repository**、**license**和**people fields**。这些都是一些作者信息，仓库主页，提交bug的地方,包采用的开源协议。这里的**people fields**指的是包作者相关的字段，比如**author**，**email**，不是这个字段名叫做peopke。比较简单，所以看一下react的包示例，不多做介绍了。
```json
"homepage": "https://reactjs.org/",
"bugs": {
    "url": "https://github.com/facebook/react/issues"
},
"repository": {
    "type": "git",
    "url": "git+https://github.com/facebook/react.git",
    "directory": "packages/react"
},
"license": "MIT",
```
## 操作相关字段
&emsp;&emsp;操作相关指的是npm在进行包读取或者指令运行时候，设置的一些相关字段。
- **main**字段接的是一个文件路径的字符串。值得是你这个包引入的时候，主入口是哪里。比如你填写`"main": "./app.js"`，你的包名为`exm`。那么当别人require或者import你的包的时候，npm会根据这个main字段去自动的查找这个主入口文件。即`require('exm')`，相当于`require('exm/app.js')`。
- **bin**字段会创建一个bash的全局链接到该字段下设置的路径当中。创建的地方跟随系统的不同而不同。比如使用`{ "bin" : { "myapp" : "./cli.js" } }`，在linux系统下面会创建一个/usr/local/bin/myapp的软链接到cli.js文件当中。
- **scripts**字段应该是常用字段了，它能够自定义一段脚本，当你使用`npm run 你定义的key`的时候，就能够跑对应的指令。这样就可以避免老是去敲打很多长串的指令。比如：
```json
"scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
}
```
  当我运行`npm run test`的时候，实际上运行`echo \"Error: no test specified\" && exit 1`的指令。很多包的指令是npm run start，现在就可以去scripts字段当中去看看，实际上到底运行了什么指令了。

## 依赖相关字段
&emsp;&emsp;依赖相关就是dependencies，也就是上面的我的那两道面试题了。简单来说，你的包依赖了那些npm上的其他轮子，就会定义在dependencies当中。dependencies分为4种，其中比较重要的就是**dependencies**和**devDependencies**。dependencies表现形式就是一个**包名**和**版本号**的**映射对象**。下例：
```json
"dependencies": {
    "loose-envify": "^1.1.0",
},
```
**dependencies**和**devDependencies**区别就是一个运行态(product)的打包相关依赖，另一个是开发状态(dev)的打包相关依赖。举个很简单的例子，比如我在开发状态会用到`babel`包的一些编译，诸如`css-loader`用来处理css的打包工具，这些只有在**开发**的时候才会被需要，而如果用到像`react`这种涉及到整个项目的包，就需要放到dependencies当中去。那么现在你应该知道了，`npm install`的时候，哪个在包中的dependencies不会被安装了吧。

&emsp;&emsp;这里也简单说一下引入包的版本号描述方式。前面说过了，版本号的三个数字分别表示大版本，次版本和小修改号。在项目使用过程当中，很有可能引入的包是需要某个大版本，或者兼容某个版本以下的。所以需要一些特殊的表示符号去表示这些版本号的范围。这里直接拿一下官网上面的例子：
```json
{ "dependencies" :
  { "foo" : "1.0.0 - 2.9999.9999"
  , "bar" : ">=1.0.2 <2.1.2"
  , "baz" : ">1.0.2 <=2.3.4"
  , "boo" : "2.0.1"
  , "qux" : "<1.0.0 || >=2.3.1 <2.4.5 || >=2.5.2 <3.0.0"
  , "asd" : "http://asdf.com/asdf.tar.gz"
  , "til" : "~1.2"
  , "elf" : "~1.2.3"
  , "two" : "2.x"
  , "thr" : "3.3.x"
  , "lat" : "latest"
  , "dyl" : "file:../dyl"
  }
}
```

- `1.1.0` 明确的版本号
- `>1.1.0` 大于1.1.0的所有版本
- `>=1.1.0` 大于等于1.1.0的所有版本
- `<version` 小于1.1.0的所有版本
- `<=1.1.0` 小于等于1.1.0的所有版本
- `1.2.x` 表示1.2.下面的任意版本，x为任意版本
- `~1.1.0` 安装1.1.x 下面最新版本，前两个版本号必须为1.1
- `^1.1.0` 安装1.x.x 下面最新版本，第一个大版本号必须为1
- `*` 表示任意版本
- `""` 空字符串，功能和单个的*一样，表示任意版本
- `1.1.0 - 1.2.0` 大于1.1.0，小于版本1.2.0
- `1.1.0 || 1.2.0` 1.1.0或者1.2.0均可

&emsp;&emsp;了解package.json并没有那么难，但这部分知识点属于前端工程化基础知识点，还是需要掌握的。前几天在和同事聊天的时候，发现不少人到现在连lib和src的区别都没搞清楚。平时就执行个`npm install`完成了也就没有过多思考了。还是要多思考，多学习，多总结，争取不做一个curd boy。
> [本文抽取了一些比较核心的部分，如果想要更多详细内容可以参考npm官方文档](http://caibaojian.com/npm/files/package.json.html)
