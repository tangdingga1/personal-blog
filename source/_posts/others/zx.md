---
title: 深入浅出zx库
date: 2022/02/2 19:00:00
categories:
- [前端, node]
tags:
- node
---
&emsp;&emsp;在 `js` 中便捷自由的写 `bash`，想想都畅快。
<!--more-->
## zx是什么
&emsp;&emsp;[zx](https://github.com/google/zx) 是谷歌实现的一个能在 `node` 中写 `bash` 的库。就像这样：
```javascript
await $`echo "hello world"`;
```
&emsp;&emsp;使用<code>$\`\`</code>框起想要执行的命令，就可以直接执行 `bash`。

&emsp;&emsp;这个库的最大的便捷在于，`node` 和标准的 `bash` 都是我们非常熟悉的东西。只需要知道最基本的库的使用，就可以很快上手，写出功能很强大的工具。

## 基本使用
&emsp;&emsp;只需要安装 `zx` 库，就可以便捷的在项目中使用了。
```bash
mkdir zx-demo
cd zx-demo
npm init -y
npm install zx --save
touch index.mjs
```
&emsp;&emsp;需要一提的是，`zx` 命令执行的文件，需要以 `.mjs` 命名。

&emsp;&emsp;在 `.mjs` 结尾的文件中，`zx` 可以允许在最外层使用 `await`，并提供了一系列常用的 `node` 库和函数，可以直接使用。

&emsp;&emsp;现在打开刚刚创建的 `index.mjs` 文件，输入下面的指令，然后运行 `npx zx index.mjs`。

```javascript
await $`pwd`;
/*
输出
$ pwd
/Users/tang/Desktop/zx-demo
*/
```
&emsp;&emsp;运行成功。

## <code>$\`command\`</code>
&emsp;&emsp;执行指令的两个飘折号<code>\`\`</code>等同于ES6语法。你可以在内部直接使用`${变量名}`来嵌入命令。
```javascript
var log = 'log';
await $`git ${log}`;
```
&emsp;&emsp;这种使用方法需要注意的是，zx会自动给以`${变量名}`方式嵌入的内容加上括号，作为一个字符串。比如我要创建一个名为`测试 测试`的文件夹。在 `bash` 中直接执行是会失败的，如下：
```bash
mkdir 测试 测试
# 失败，创建了一个名为 测试 的文件夹，然后报错 mkdir: 测试: File exists
mkdir '测试 测试'
# 成功
```
&emsp;&emsp;而使用`zx`则直接可以执行成功：
```javascript
let name = '测试 测试';
await $`mkdir ${name}`;
// 输出：$ mkdir $'测试 测试' 成功
```

&emsp;&emsp;除了这点，如果插入的变量是数组，`zx` 还会自动的解析数组，如下：
```javascript
let flags = [
  '--oneline',
  '--decorate',
  '--color',
];
await $`git log ${flags}`;
```

&emsp;&emsp;但是这个特性也会带来麻烦。当你直接键入命令的时候，`zx` 会强行解析。

```javascript
let demo = 'git log';
await $`${demo}`; // 相当于在 bash 中执行 'git log'
/*
报错
$ $'git log'
/bin/bash: git log: command not found
Error: /bin/bash: git log: command not found
    at file:///Users/tang/Desktop/zx-demo/index.mjs:2:8
    exit code: 127 (Command not found)
*/
```
&emsp;&emsp;此时可以利用 `zx` 会解析数组的特性来这样操作。
```javascript
let demo = `git log`;
await $`${demo.split(' ')}`;
// 执行成功
```

&emsp;&emsp;<code>$\`command\`</code>的返回值是一个继承Promise的类。可以追踪该命令的`输入`，`输出`，`错误`，`退出码`。还提供了两个方法，`pipe`和`kill`。

```typescript
class ProcessPromise<T> extends Promise<T> {
  readonly stdin: Writable
  readonly stdout: Readable
  readonly stderr: Readable
  readonly exitCode: Promise<number>
  pipe(dest): ProcessPromise<T>
  kill(signal = 'SIGTERM'): Promise<void>
}
```
&emsp;&emsp;其中 `pipe` 可以将进程的输出重定向，类似于 `bash` 中的管道语法。而 `kill` 则可以直接杀死进程。

```javascript
// pipe
await $`cat file.txt`.pipe(process.stdout);
// kill
await $`npx service`.kill();
```

## 全局能力
&emsp;&emsp;前文也提到过，`zx` 在执行以 `.mjs` 结尾的文件中，提供了便捷的能力，以及一些常用的函数和库。可以直接使用的库有：
- chalk
- fs
- globby
- os
- path
- minimist

关于这些库的作用和使用方法可以自行去 `npm` 上面查找，本文不做赘述。

&emsp;&emsp;接下来重点聊一聊 `zx` 提供的两个非常有用的函数。

### cd
&emsp;&emsp;当你执行 `npx zx index.mjs` 时，默认的执行路径会是 `index.mjs` 文件的路径。而在文件中使用的每一个 `bash`，都会是以子进程的形式来执行的。所以，当使用 `bash` 来进行路径变更时，变更的是对应的子进程的路径，主进程的路径是不变的。

```javascript
await $`pwd`; // 文件路径
await $`cd ../../`;
await $`pwd`; // 仍然是文件路径
```
&emsp;&emsp;函数cd则可以改变主进程的路径。
```javascript
await $`pwd`; // 文件路径
cd('../');
await $`pwd`; // ../文件路径
```

### question
&emsp;&emsp;`question` 函数是 `readline` 的包装，可以停住程序，等待用户输入敲击回车才完成。
```javascript
let mustBeYse = await question('Do you like me ？');
// Do you like me ？
```
&emsp;&emsp;`question` 还接受第二个参数，可以支持用户在输入的时候，使用 `tab` 键来补全。

```javascript
// @type { choices: string[] }
let mustBeYse = await question('Do you like me ？', { choices: ['yes'] });
```

## 服务进程与子进程
&emsp;&emsp;`zx`最棒的地方就在于他可以方便的开多个子进程。下面我愉快的起三个服务，三个服务的输出都会转到主进程。
```javascript
$`npx serve`;
$`npx serve`;
$`npx serve`;
/*
输出
npx: 90 安装成功，用时 8.892 秒
npx: 90 安装成功，用时 8.957 秒
npx: 90 安装成功，用时 9.047 秒
INFO: Accepting connections at http://localhost:3000
INFO: Accepting connections at http://localhost:52857
INFO: Accepting connections at http://localhost:52858
*/
```
&emsp;&emsp;此时如果我退出主进程，子进程的服务也会全部停止。

&emsp;&emsp;在上文中，使用<code>$\`npx serve\`</code>的时候，没有加上 `await`。是因为类似于服务这种需要手动关闭的进程，如果使用 `await` 将永远执行不到下一步。

```javascript
await $`npx serve`;
console.log(1); // 此时这一行永远不会执行到。
```

&emsp;&emsp;如果你需要监听一个服务进程启动成功了，再去执行下一步的操作，可以使用`for await` 去监听日志信息。

```javascript
let serve = $`npx serve -p 5000`;
for await (let chunk of serve.stdout) {
  // 启动服务成功
  if (chunk.includes('Accepting connections')) break;
}
// 拉取服务信息
await $`curl http://localhost:5000`;
serve.kill('SIGINT');
```

## 尝试实践
&emsp;&emsp;了解了初步的使用方法之后，来实现一个小小的脚本。在实际的按迭代的开发工作中，每次开发之后都会清理掉所有的本地迭代分支。就写一个清空除了`master` 分支以外分支的脚本。

&emsp;&emsp;首先需要获取到本地所有需要删除的分支名。首先需要执行 `git branch` 获取到所有的本地分支，获取的信息是一个字符串。
字符串中的分支名按照换行符隔开的，因此可以使用 `split('\n')` 把字符串按照分支分割成数组。

&emsp;&emsp;接着需要摘除分支中的 `master` 分支和当前所在的分支。因为当前所在的分支是无法使用 `git branch -d` 来进行删除的。

```javascript
const branchBash = await $`git branch`;
const localBranchNames =
  branchBash
    .stdout
    .split('\n')
    .filter(branchName => !branchName.includes('master') && !branchName.includes('*'))
    .map(filteredBranchName => filteredBranchName.trim());
```
&emsp;&emsp;最后执行删除分支的命令，然后抓到无法删除分支的错误信息，提示给用户。
```javascript
  try {
    const deleteBranch = await $`git branch -d ${localBranchNames}`;
  } catch(p) {
    let remainBranch = p.stderr.match(/\'(\S+?)\'/gi);
    console.log(`remain local branches: ${remainBranch}`);
  }
```

&emsp;&emsp;最后使用 `zx` 执行文件，就可以敲击下回车清空所有的本地分支了。

## 深入一下
&emsp;&emsp;`zx` 的源码非常的少，两个代码相关的文件加起来总才共五百多行，这还包括了 `markdown` 和 `xml` 的解析功能。如果去掉诸如**执行命令解析**，**错误匹配**以及一些**格式化输入**的代码，整个 `bash` 核心功能相关的代码才100行左右。

&emsp;&emsp;`zx` 的 `bash` 相关的功能本质上是在 node 的 [child_process](http://nodejs.cn/api/child_process.html#child-process) 模块上一个封装。正如库本身声明的那样， 它是一个为了更好编写脚本的工具（`A tool for writing better scripts`）。因此深入源码前，可以带上这三个有趣的问题：
1. 为什么 `zx` 要使用 <code>\`\`</code> 的方式去执行`$`函数。
2. <code>$\`command\`</code>是如何解析 command，处理引号的问题的。
3. 为什么一个核心功能仅为包装了一下 node 功能的库，能够成为2021年最受欢迎的前端库。

&emsp;&emsp;`zx` 库 代码文件有 `zx.mjs` 和 `index.mjs`。其中 `zx.mjs` 是用来解析用 `zx` 执行命令行文件的。这个文件的代码可以跳过，只看 `index.mjs` 功能的部分。以下为了文章内容的连贯性和阅读的方便性，会省略部分不重要的解析处理代码，具体代码以源码为准。

我们首先来看核心的`$`函数是怎么出入输入的 `command` 的：

```javascript
function $(pieces, ...args) {
  let cmd = pieces[0], i = 0
  while (i < args.length) {
    let s
    // 转换数组类型的入参
    if (Array.isArray(args[i])) {
      s = args[i].map(x => $.quote(substitute(x))).join(' ')
    } else {
      s = $.quote(substitute(args[i]))
    }
    cmd += s + pieces[++i]
  }
}
```

&emsp;&emsp;先来回顾一下`$`函数的使用方法，<code>$\`command\`</code>。这里使用的是[带标签的模板字符串](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Template_literals#%E5%B8%A6%E6%A0%87%E7%AD%BE%E7%9A%84%E6%A8%A1%E6%9D%BF%E5%AD%97%E7%AC%A6%E4%B8%B2)语法。这种方式可以快速的提取在<code>\`\`</code>中插入的变量，也这样就能方便的实现前文提到过的**自动转换数组**和**自动加上引号**的功能。

接下来看下解析转换的 `substitute` 和 `$.quote` 函数。
```javascript
function substitute(arg) {
  if (arg instanceof ProcessOutput) {
    return arg.stdout.replace(/\n$/, '')
  }
  return `${arg}`
}
```

&emsp;&emsp;函数 substitute 很简单，就是如果输入变量为一个 `ProcessOutput` 类，会解析成类的 `stdout` 输出返回。

```javascript
function quote(arg) {
  if (/^[a-z0-9/_.-]+$/i.test(arg) || arg === '') {
    return arg
  }
  // 如果其中包含空格会走这里
  return `$'`
    + arg
      .replace(/\\/g, '\\\\')
      .replace(/'/g, '\\\'')
      .replace(/\f/g, '\\f')
      .replace(/\n/g, '\\n')
      .replace(/\r/g, '\\r')
      .replace(/\t/g, '\\t')
      .replace(/\v/g, '\\v')
      .replace(/\0/g, '\\0')
    + `'`
}
```
&emsp;&emsp;`quote` 部分也很简单，如果输入变量仅包含 `/^[a-z0-9/_.-]+$/i` 或者为 `空字符串`，就直接返回。如果有其它内容，则会加上 <code>$''</code>，把这个输入作为一个字符串返回。这也就是前文提到的过的现象的原因：

```javascript
let name = '测试 测试';
await $`mkdir ${name}`;
// 输出：$ mkdir $'测试 测试' 成功

let demo = 'git log';
await $`${demo}`;
// 报错
```

&emsp;&emsp;这么设计的原因在我和库提的一个[issue](https://github.com/google/zx/issues/284)中有讨论过原因，主音是为了安全（`Main idea for $ to be safe`）。至此，三个问题中的两个已经解决了，我们接下来看完`$`函数中的剩余内容。

```javascript
let resolve, reject
let promise = new ProcessPromise((...args) => [resolve, reject] = args)
promise._run = () => { /* 暂时省略中间内容 */ }
setTimeout(promise._run, 0) // 确保所有的协程已经开始
return promise
```

&emsp;&emsp;剩下的就是生成了一个 `ProcessPromise` 类的实例，取出其中的 `resolve, reject` ，声明了一个`_run` 函数，然后把实例返回了。总的`$`函数的逻辑就是这么简单。我们来过一下 `ProcessPromise` 类。

```javascript
class ProcessPromise extends Promise {
  get stdin() {
    this._run()
    return this.child.stdin
  }

  get stdout() {
    this._run()
    return this.child.stdout
  }

  get stderr() {
    this._run()
    return this.child.stderr
  }

  get exitCode() {
    return this
      .then(p => p.exitCode)
      .catch(p => p.exitCode)
  }

  then(onfulfilled, onrejected) {
    if (this._run) this._run()
    return super.then(onfulfilled, onrejected)
  }

  pipe() {}

  kill() {}
}
```

&emsp;&emsp;类 `ProcessPromise` 继承了 `Promise`，重写了它的 then 方法，去执行 _run。然后声明了文档提供的属性和方法。本质上就是一个取值需要的工具类。现在我们回到 `_run` 函数。

```javascript
import {spawn} from 'child_process'

let child = spawn(prefix + cmd, {
  cwd: process.cwd(),
  shell: typeof shell === 'string' ? shell : true,
  stdio: [promise._inheritStdin ? 'inherit' : 'pipe', 'pipe', 'pipe'],
  windowsHide: true,
})

child.on('exit', code => {
  child.on('close', () => {
    /* 省略错误处理 */
  })
})

let onStderr = data => {
  if (verbose) process.stderr.write(data)
  stderr += data
  combined += data
}
child.stderr.on('data', onStderr)
```

&emsp;&emsp;可以看到 `_run` 函数中核心的功能就在这里。每次使用`$`函数去执行一个 `bash` 本质上是使用 `spawn` 分出去一个以当前主线程路径的子线程。去监听它的**日志**、**错误**和**退出**。

## 写在最后
&emsp;&emsp;文章的最后，来聊聊前文提出的三个问题中的最后一个问题。我写代码到现在为止一直在思考一件事情，想要做一个成功的程序员，到哪一点才是最重要的。我一直有一个观点，程序中蕴含的哲学思想，是远远要比现在大多数程序员在努力追求的诸如**算法**，**底层**，**炫酷的代码**是要重要的多的。

&emsp;&emsp;比如**操作系统**是一个伟大的创作，但是如果学习和明白了**操作系统**中核心的设计理念，用任何语言，一般水平的程序员也能写出简易的操作系统中的核心功能。但是**操作系统**中，**线程池**，**文件**，**任务调度**这些高度抽象的哲学设计和它们成功解决的问题，这应该才是**操作系统**最伟大的地方。

&emsp;&emsp;就像 `zx` 库一样，它的核心功能代码几乎不到100行，而且基本上是对 node 功能的一个封装，但是却能成为2021最受欢迎的前端库，不正是因为它巧妙的设计和解决的问题吗。