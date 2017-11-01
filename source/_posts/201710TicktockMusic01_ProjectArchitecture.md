---
title: Android项目篇（TicktockMusic）：项目搭建及架构
date: 2017-10-15 17:42:27
tags: 
	- Android Project
---

近期空闲时间在写一个音乐播放器 [TicktockMusic](https://github.com/Lauzy/TicktockMusic) ，基本功能已经实现，仍有诸多细节待完善。由于公司新项目开启，加上此项目的坑仍需花不少时间来填，平时敲得略累，所以闲来打算写写博客来分享下项目的经验。计划是打算多写几篇文章，将此项目的搭建，封装，逻辑实现等都写出来，分享的同时也是自己的总结。本文先从项目的搭建及架构开始写起。

<!--more-->

## 架构
在项目创建之初，考虑了诸多架构，众所周知，Android 项目架构有 MVC，MVP，MVVM，Flux或者相互结合等，各个架构的优缺点网上有很多优秀的文章，此处不再介绍。在参考了诸多文章后，我选择了 [Android Clean Architecture](https://github.com/android10/Android-CleanArchitecture) ，此架构分为三层，分别为 领域层 (Domain Layer)、数据层 (Data Layer)、表现层 (Presentation Layer)。

Clean Architecture 有许多文章分析：
- [Architecting Android...The clean way?](https://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/)
- [一种更清晰的Android架构（Architecting Android...The clean way?译文）](https://zhuanlan.zhihu.com/tech-frontier/20001838)
- [Architecting Android...The evolution](https://fernandocejas.com/2015/07/18/architecting-android-the-evolution/)
- [Clean Architecture: Dynamic Parameters in Use Cases](https://fernandocejas.com/2016/12/24/clean-architecture-dynamic-parameters-in-use-cases/)

本文简单介绍下 Clean 架构的分层。

1、 Domain Layer

简单来说，领域层就是处理业务逻辑的一层，主要包含了 UseCase(Interactor) 、Repository 等接口。本层为 Java Library ，不包含 Android 的依赖，具体来说，就是控制数据层，通过接口对数据进行增删改查等业务交互。 例如 [Android Clean Architecture](https://github.com/android10/Android-CleanArchitecture) 中的 interactor 与 repository 的交互。

2、 Data Layer

顾名思义，Data 层提供了整个 App 所需要的数据，主要是通过实现 Domain Layer 的接口，根据不同的场景来获取需求的数据。 [Android Clean Architecture](https://github.com/android10/Android-CleanArchitecture) 项目中主要是通过一个工厂方法，来根据不同的条件从不同的数据源中获取数据。实体类的映射转换，cache缓存，数据相关的自定义异常等，均在数据层实现。

3、 Presentation Layer

表现层则主要负责 UI 的逻辑，与数据层完全隔离，根据 Domain Layer 的反馈进行界面展示等。本层即为传统的 UI 层，Activity、Fragment 等都在此层处理 UI 相关的逻辑。
本层可以采用 MVP 架构（MVC,MVVM等均可）进行开发，M层可处理一些数据转换，如 Mapper 转换等。P层则主要根据 Domain 的 use case 反馈来进行视图层的业务处理，[Android Clean Architecture](https://github.com/android10/Android-CleanArchitecture) 项目通过 Dagger 注入进行 use case 接口等的实例化，分层更加明确；V层则为 Activity 或 Fragment，主要用户数据展示，UI控制等。