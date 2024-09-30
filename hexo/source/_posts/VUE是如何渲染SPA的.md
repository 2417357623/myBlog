---
title: VUE是如何渲染SPA的
date: 2024-09-28 13:35:52
tags:   
  - Vue
  - SPA
  - 渲染原理
  - 前端开发
---



## 前置知识

### 进程和线程

进程：就是程序的运行实例，当我们启动一个程序，操作系统会创建一个内存，里面可以有多个线程。
线程：负责当前任务中程序的执行

特点：
- 同一个进程的线程数据共享
- 一个线程崩溃，整个进程崩溃
- 进程关闭后，内存会回收
- 不同进程相互隔离，通过IPC通讯


## chrome多进程架构

chorme架构主要包括了浏览器主进程，网络进程，GPU进程，渲染进程，插件进程

| Name     |
| -------- |
| fasfas   |
| fasdf as |
|          |
| fas asf  |
| asf      |


## 工作原理

浏览器主进程

## 参考链接

- [浏览器历史和渲染原理](https://www.bilibili.com/video/BV1tc41157Va/?spm_id_from=333.337.search-card.all.click&vd_source=115cedcdb1996c6483fb453252e441e6)
- [MDN：浏览器工作原理](https://developer.mozilla.org/zh-CN/docs/Web/Performance/How_browsers_work)

