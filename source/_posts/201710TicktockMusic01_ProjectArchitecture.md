---
title: Android项目篇（TicktockMusic）：项目搭建及架构
date: 2017-10-15 17:42:27
tags: 
	- Android Project
---

近期空闲时间在写一个音乐播放器 [TicktockMusic](https://github.com/Lauzy/TicktockMusic) ，基本功能已经实现，仍有诸多细节待完善。由于公司新项目开启，加上此项目的坑仍需花不少时间来填，平时敲得略累，所以闲来打算写写博客来分享下项目的经验。计划是打算多写几篇文章，将此项目的搭建，封装，逻辑实现等都写出来，分享的同时也是自己的总结。本文先从项目的搭建及架构开始写起。

## 架构
在项目创建之初，考虑了诸多架构，众所周知，Android 项目架构有 MVC，MVP，MVVM，Flux或者相互结合等，各个架构的优缺点网上有很多优秀的文章，此处不再介绍。在参考了诸多文章后，我选择了 [Android Clean Architecture](https://github.com/android10/Android-CleanArchitecture) ，此架构分为三层，分别为 领域层 (Domain Layer)、数据层 (Data Layer)、表现层 (Presentation Layer)。

Clean Architecture 有许多文章分析：
- [Architecting Android...The clean way?](https://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/)
- [一种更清晰的Android架构（Architecting Android...The clean way?译文）](https://zhuanlan.zhihu.com/tech-frontier/20001838)
- [Architecting Android...The evolution](https://fernandocejas.com/2015/07/18/architecting-android-the-evolution/)
- [Clean Architecture: Dynamic Parameters in Use Cases](https://fernandocejas.com/2016/12/24/clean-architecture-dynamic-parameters-in-use-cases/)