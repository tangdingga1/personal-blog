---
title: zx库与应用
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