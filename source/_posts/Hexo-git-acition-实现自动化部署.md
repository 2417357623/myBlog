---
title: Hexo + github action 实现自动化部署
date: 2024-11-05 11:42:48
tags: 
- hexo 
- github action
---


## 准备

首先需要一个可以 build 的 hexo 项目，确保在本地实现 `npm run build` 指令。

## 为什么选择 git action + git pages 模式

Git action 不仅仅是一个 CI/CD 平台，更是一个自动化工作流的平台。他很好的兼容了 github 的功能，除了常规的 ci/cd 流程的兼容，还可以对用户的 issue，贡献者参与等事件都做出反应，之后开始配置过的工作流。
除此之外，github 还提供了静态资源托管服务 git pages，相当于给我们了一个服务器来部署一个人人都可以访问的站点，这样我们就**无需购买配置服务器**。
同时GitHub Pages 在你手动推送新代码到配置的分支（如 main、master 或 gh-pages）时，会**自动重新构建并发布**你的静态网站。这也就意味着我们只需要将 build 的文件发送到项目任意分支就可以。

我们可以在如下标识处找到这两个功能入口

![](attachment/Pasted%20image%2020241105103438.png)


## 让我们开始吧 

来到 setting，配置 github api 的 token，以授权访问我们的 github repo。

![](attachment/Pasted%20image%2020241105112250.png)

再来到仓库的 setting 下把 token 配置进去
![](attachment/Pasted%20image%2020241105112403.png)

这样我们的 actions 就不用去输入明文密码来执行自动化任务了，相应的会使用我们配置的名称来进行身份验证。

接着来到 action 底下新建第一条工作流

![](attachment/Pasted%20image%2020241105112620.png)

Yml 文件里面是我们的工作流配置，包括触发事件，工作流程，复制粘贴我的配置，并且修改里面的仓库地址，node 版本。做你想要的修改。然后提交文档

```yml
name: Deploy Hexo to GitHub Pages1

on:
  push:
    branches:
      - master  # 当推送到 main 分支时触发

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
