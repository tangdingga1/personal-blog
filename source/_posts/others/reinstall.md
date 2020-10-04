---
title: 记一次服务器更换环境配置
date: 2019/11/5 19:00:00
categories:
- [Ubuntu]
tags:
- 环境配置
---
&emsp;&emsp;双十一大促，老服务器到期。记一次服务器更换环境配置，用于节省以后的操作搜索时间。机器配置`1核 2GB内存 1Mbps带宽`，操作系统`Ubuntu18.0`。
<!--more-->
## 安装nodejs。
&emsp;&emsp;nodejs是前端服务器生态配置的基础支持，首先安装nodejs & npm，用于服务器的访问运行。
&emsp;&emsp;使用Ubuntu自带的生态安装工具apt install即可安装nodejs，不需要到官网curl压缩包解压再进行路径配置。
```bash
sudo apt update -y
sudo apt install nodejs
sudo apt install npm
sudo npm install npm -g #更新npm
```
&emsp;&emsp;接着安装`n`。n是node的一个版本管理工具，可以方便的进行node的版本更新和控制。
```bash
sudo npm install n -g
n 相关指令
n                              显示已安装的Node版本
n latest                       安装最新版本Node
n stable                       安装最新稳定版Node
n lts                          安装最新长期维护版(lts)Node
n <version>                    根据提供的版本号安装Node</pre>
```
&emsp;&emsp;使用`sudo n`切换版本无效之后尝试修改node的path路径。

```bash
export NODE_HOME=/usr/local
export PATH=$NODE_HOME/bin:$PATH
export NODE_PATH=$NODE_HOME/lib/node_modules:$PATH
```

## 配置git。
&emsp;&emsp;配置git以便快速的部署新开发迭代的代码。毕竟平时是不大可能在服务器上面进行前端的开发的。
&emsp;&emsp;配置git要注意，使用`sudo`加权生成的`ssh-key`所在的秘钥位置和不加生成的位置是不同的。在执行`git`相关命令访问的时候，关联的也不是一个`ssh-key`。
```bash
sudo apt-get install git  # 安装git
ssh-keygen -t rsa -C "your github username" # 非`sudo`生成配对的ssh秘钥 默认目录在 ~/.ssh/id_rsa.pub 下
sudo git config ––global user.name "your github username"   # 配置全局的用户名
sudo git config ––global user.email "your github useremail"   # 配置email
```
&emsp;&emsp;生成的秘钥进入github个人仓库setting到SSH and GPG keys，关联后进行正常的git操作。

## 配置Nginx
&emsp;&emsp;配置Nginx，完成反向代理。把默认非http网址代理为https访问，把80端口的访问代理为https的443端口访问。
&emsp;&emsp;内部把443端口的访问代理到自己内部实现的服务器监听的端口上，在服务器端口放开上只需要放开常用的几个端口即可。
&emsp;&emsp;安装Nginx。默认使用`sudo`安装的时候目录在`/etc/nginx`下。
```bash
sudo apt-get install nginx
sudo nginx -t 测试
sudo nginx -s reopen：重启Nginx
sudo nginx -s reload：重新加载Nginx配置文件，然后以优雅的方式重启Nginx
sudo nginx -s stop：强制停止Nginx服务
sudo nginx -s quit：优雅地停止Nginx服务（即处理完所有请求后再停止服务）
```
相关配置
```bash
server {
    listen 80; #监听端口，非https默认为80
    server_name www.tangdingblog.cn; # 访问名称
    rewrite ^(.*) https://$host$1 permanent; #重定向到https
    location / {
       proxy_pass http://122.51.226.110:3000;
       add_header Access-Control-Allow-Origin localhost;
    }
}
```

## 使用scp进行ssl证书传输配置
```bash
scp yourFilePath root@ip:copyedFilePath #传输
scp root@ip:copyedFilePath yourFilePath #拷贝
```


## 配置pm2
&emsp;&emsp;使用`npm`安装`pm2`使得mode的服务常驻，不在ssh访问断开连接后关闭。
```bash
sudo npm install pm2 -g #全局安装pm2
pm2 start app.js #启动服务
pm2 list # 展示当前启动的服务
pm2 kill # 杀掉进程
```

## 安装MongoDB
&emsp;&emsp;数据库使用mongodb，安装完毕后会自动启动。[官方安装文档](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-os-x/)。
```bash
sudo apt-get install mongodb
```