---
title: github ci
date: 2020/11/23 14:25
categories:
- [git]
tags:
- 知识点整理
---
&emsp;&emsp;github自带的脚本命令完成一键部署。

<!--more-->
## github ci 是什么
&emsp;&emsp;github ci 是 github 提供的一个依据 git 相关事件触发的自动化脚本服务。简单的说，当触发github的事件(push/pull request等)后，github官方会提供一个服务器环境，自动的运行先前配置的脚本，差不多近似于 gitlab 的 ci。

## 为项目创建 github ci
&emsp;&emsp;github 来检测项目是否存在 github ci 就是查看该项目根目录下的`.github/workflows`是否存在yml的配置文件。因此可以通过在根目录下创建 .github/workFlows 文件夹，再创建对应的 .yml 结尾的文件的方式，来为项目创建 github ci。

&emsp;&emsp;github 官网支持在项目中用可视化的方式来创建 github ci。如下图，在项目主目录的 **Actions** 栏目下，点击蓝色的 *set up a workflow yourself* 可以直接为项目创建 github ci。

![github ci](https://pic.downk.cc/item/5fbb9561b18d627113f4ce0d.jpg)

&emsp;&emsp;点击后可以发现，github 创建了一个默认名为 [分支名].yml 的ci模板文件，该文件位于项目的 `/.github/workflows` 文件目录下。
![编辑 github ci](https://pic.downk.cc/item/5fbb9a6ab18d627113f612ce.jpg)

&emsp;&emsp;github ci 配置采用 yml 的语法。yml 类似于 json，是一种标准化数据格式常用于项目的配置。它近似于python的语法，使用 `tab` 来标识缩进，用 `#` 来表示注释。具体语法可以看[这里](https://baike.baidu.com/item/YAML/1067697?fr=aladdin)。

&emsp;&emsp;在 github 上面创建配置文件的时候，github会生成默认的范例。

```yml
# name 标识一下这个 ci 文件名称，纯语义化的项，没有配置意义
name: demo

# on 配置项用于配置该仓库的什么分支触发什么事件后进行该 ci 的调用
on:
  # master 分支 push 后触发
  push:
    branches: [ master ]
  # master 分支 pull_request 后触发
  pull_request:
    branches: [ master ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# jobs是一个功能脚本的维度集合。比如配置检测库语法，发布库，可以作为两个工作(job)。一个yml文件可以运行多个job。
jobs:
  # 一个jobs拥有 setup / build / test 的秩序执行单元。这个维度大于 steps 小于 jobs。
  build:
    # 该操作跑在什么环境之下，支持 Ubuntu/Windows/macOS。
    runs-on: ubuntu-latest

    # step 标识一系列按续执行的最小工作单元
    steps:
      # uses 标识使用创建好的功能脚本，actions/ 标识了这是github公开的脚本库
      # 这里使用了一个名为 checkout@v2 的脚本文件。
      - uses: actions/checkout@v2

      # 运行单个的 shell
      - name: Run a one-line script
        run: echo Hello, world!

      # 运行多个 shell
      - name: Run a multi-line script
        run: |
          echo Add other actions to build,
          echo test, and deploy your project.
```

&emsp;&emsp;再具体的书写方法和配置信息可以点击[这里](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions)查看 github 官方文档。

&emsp;&emsp;了解默认配置之后，试着书写一个最简单的 ci 配置信息查看一下运行的情况。这里设置一个名为`echo demo`的文件在 master 分支 push 的时候触发。输出一个`pwd`和`ls -al`来查看一下实际的运行环境。顺带一提，由于某些今年的原因，github的默认分支不在使用 master，而更改为 main 分支。实际操作的时候注意操作的分支和配置的分支一致。

```yml
name: echo demo
on:
  push:
    branches: [ master ]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: run demo
        run: pwd
        run: ls -al
```

&emsp;&emsp;配置完毕后使用`push`提交。这时候切换到`Actions`分栏下，可以看到对应的ci运行信息。
![Actions](https://pic.downk.cc/item/5fbba905b18d627113fa1939.jpg)


&emsp;&emsp;点击进入 ci 的 workflow 可以看到这次命令的执行结果。github ci 实际上是分配给了项目一小块服务器空间，在这个服务器空间中可以对项目进行任何对应的命令操作。`pwd` 输出值显示目前位于`/home/runner/work/actions-demo/actions-demo`文件下，`ls -al`显示文件内没有任何文件，目前的操作位于 docker 环境。
![workflow](https://pic.downk.cc/item/5fbba976b18d627113fa37cd.jpg)

&emsp;&emsp;接下来深入一下，看看github到底给予了多少权限。修改配置文件尝试进入到 root 目录去新建一个文件。

```yml
name: echo demo
on:
  push:
    branches: [ main ]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: move to root
        run: |
          cd /
          touch demo.txt
```

![权限尝试](https://pic.downk.cc/item/5fbbac26b18d627113faf851.jpg)

&emsp;&emsp;结果可以看到`cd /`成功，但是操作文件失败了。github ci 提供了一个完整的服务器环境和项目维度下的子账号权限。实际上ci也可以使用 `sudo` 权限，仅仅需要在配置文件进行配置，具体内容可以点击[这里](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/sudo-mode)查看。

## github ci 应用实例
&emsp;&emsp;这一章实例来实现一个github ci的自动化博客部署。实现的功能大致为：当在本地更新推送一篇博文之后，github ci能连结到服务器对应网站拉取最新的代码，然后运行npm构建，把最新更新的博文发布出去。

&emsp;&emsp;与 gitlab 不同， github 的脚本运行环境实在github提供的官方服务器上面，不能像gitlab那样拥有直接操作配置服务器的能力，更多是偏向公共包的一些应用。根据 github 文档，github ci 的本意是对于一些公共包的处理，比如上传一些公共的 npm 包后自动进行代码的格式检测，检测无误后进行包的发布，或者用户对公开的库提 pull request 或者 issue 之后，进行基本的样式检验和处理。

&emsp;&emsp;其次github是一个开源网站，配置的 .github/workflows 会以明文的方式暴露到开源库中。使用 ssh 方式进行服务器连结的时候用户名和密码会被暴露到库当中。

&emsp;&emsp;这里使用 github 提供的 `Secrets` 功能来解决问题。在项目的setting 栏目下 选中 `Secrets` 即可进入 `Secrets` 配置页面。Secrets 可以在项目维度下以`key - value`的形式保存一些值，这些值不会在库中公开，也不会被 pull request 拉取。在使用github ci的时候，可以用
<code>$&#123;&#123;secrets&#125;&#125;.&#91;key&#93;</code>的方式来声明 Secrets 中的 key，这些声明的部分会被替换为 value。

![github secrets](https://pic.downk.cc/item/5fbbc122b18d62711301d557.jpg)


&emsp;&emsp;现在来编写博客库的 github ci 文件完成这个实例。首先新建 .github.workflows 目录创建 buildblog.js.yml 文件。完成基本的配置，监听的事件为 push 事件，分支为 master 分支。

&emsp;&emsp;其次来编写对应的脚本文件。连接到发布服务器需要使用到 `sshpass` 命令，它能够直接把服务器密码传递给ssh指令，还不是在输入ssh指令之后敲回车再次进行密码的输入。接着在 ssh 指令后面加上`-o StrictHostKeyChecking=no`参数，github 每次分配 ci 运行的服务器 ip 是不同的，防止首次 ssh 连接的时候安全策略让你输入确认连接输入 yes。最后在 ssh 命令后面加上需要继续执行的脚本 `'cd 目录 && git pull && npm run build'`。然后在用户名和密码的部分使用刚才所说的 `Secrets` 中配置的`key - value` 中的 key 即可完成自动化构建博客的功能了。具体配置文件信息如下，也可以点击[这里](https://github.com/tangdingga1/personal-blog)查看博客项目的具体配置。

```yml
name: build-blog CI

on:
  push:
    branches: [ master ]

jobs:
  build:

    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [14x]

    steps:
      - run: sshpass -p ${{secrets.SECRET}} ssh -o StrictHostKeyChecking=no ${{secrets.NAME_USER}} 'cd ${{secrets.PATH}} && git pull && npm run build'
```
&emsp;&emsp;这篇 github ci 的博文就是使用自动化部署上去的哦~