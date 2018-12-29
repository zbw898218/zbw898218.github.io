---
title: HEXO+GITPAGES 搭建个人技术博客
tags: tags
---
本文旨在利用hexo和gitpages来搭建一个免费的个人技术博客。搭建过程参考了网络上很多的成功范例，并结合自己实操，形成一个完整详实的搭建指南。

## 前期准备工作

### 第一步：申请github账号，并创建博客仓库

每个GitHub账号只能有一个仓库用来存放个人主页，仓库名格式：
username/username.github.io【username：GitHub账户名称，不可改。可以通过http://username.github.io 来访问你的个人主页】

### 第二步：安装Git/Node.js

Hexo，需要系统支持Nodejs和git

``` Node.js下载地址
https://nodejs.org/en/
```
``` Git下载地址
http://git-scm.com/download/
```

### 第三步：安装Hexo

```1.新建hexo目录
 【例：e:/hexo】
```
```2.进入hexo目录，打开命令窗口【ctrl+shift+右键】,顺序执行如下命令
- $ npm install hexo-cli -g
- $ hexo init blog
- $ cd blog
- $ npm install
- $ hexo generate
- $ hexo server
```
执行完成【hexo server】命令，浏览器打开：http://localhost:4000/

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/deployment.html)
