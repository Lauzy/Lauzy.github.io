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

本文简单介绍下 Clean 架构的分层。

1、 Domain Layer

简单来说，领域层就是处理业务逻辑的一层，主要包含了 UseCase(Interactor) 、Repository 等接口。本层为 Java Library ，不包含 Android 的依赖，具体来说，就是控制数据层，通过接口对数据进行增删改查等业务交互。 例如 [Android Clean Architecture](https://github.com/android10/Android-CleanArchitecture) 中的 interactor 与 repository 的交互。

2、 Data Layer

顾名思义，Data 层提供了整个 App 所需要的数据，主要是通过实现 Domain Layer 的接口，根据不同的场景来获取需求的数据。 [Android Clean Architecture](https://github.com/android10/Android-CleanArchitecture) 项目中主要是通过一个工厂方法，来根据不同的条件从不同的数据源中获取数据。实体类的映射转换，cache缓存，数据相关的自定义异常等，均在数据层实现。

3、 Presentation Layer

表现层则主要负责 UI 的逻辑，与数据层完全隔离，根据 Domain Layer 的反馈进行界面展示等。本层即为传统的 UI 层，Activity、Fragment 等都在此层处理 UI 相关的逻辑。
本层可以采用 MVP 架构（MVC,MVVM等均可）进行开发，M层可处理一些数据转换，如 Mapper 转换等。P层则主要根据 Domain 的 use case 反馈来进行视图层的业务处理，[Android Clean Architecture](https://github.com/android10/Android-CleanArchitecture) 项目通过 Dagger 注入进行 use case 接口等的实例化，分层更加明确；V层则为 Activity 或 Fragment，主要用户数据展示，UI控制等。

#### 优缺点

Clean架构有很明确的优点，最主要则为层级分明，各个层级分工明确且只需关注结果。其次，易于测试和代码迭代更新。但是每个架构都不是完美的，clean架构分工明确但视图层可能也采用MVP等模式，这就造成了业务复杂的时候，P 和 V 的接口量是巨大的。

#### 应用

本人写的项目中，相对简化了 Clean 架构。在 Domain 层中，可以只保留 interactor、repository 和 model 即可。在 Data 层，由于是音乐类项目，加上具备最近播放、喜欢等功能，所以 Data 层除了网络外，也用到了 contentprovider 和 数据库等，在 repository 实现类中，由于业务相对简单，所以移除了获取数据源的工厂方法，直接在实现类中获取相应的数据。对于业务较为复杂的场景，可更加完善。 在 Presentation 层，我采用了 MVP 架构，并且采用了 Rxjava + Dagger2 简化代码。

由于很多想法是在写项目的过程中逐渐 get 到的，所以本项目也有一些细节并未严格遵循 clean 架构规则，如音乐播放服务、SharedPreferences 等。其实每种架构均服务于项目的业务，在代码过程中，任何架构都不可能完美服务于业务需求。即便是传统的 MVC，只要做到业务分层，模块清晰，也是不错的架构。所以在平时我们可以多尝试各种架构，多读优秀的开源项目，这样才能不断的提升自己的架构能力和编码水平。

## Gradle 配置

大部分项目都不可避免的用到一些优秀的开源库，至于开源库如何选择，这里就不多说了。这里简单说下使用三方库时的Gradle配置。
由于大部分项目都有很多 module ，所以建议都采用全局的 gradle 配置来进行项目管理。可以参考[Android Clean Architecture](https://github.com/android10/Android-CleanArchitecture) ，在项目根目录下新建一个 dependency.gradle 然后进行相应的配置。如下：

```java

ext {
    //Android
    androidBuildToolsVersion = "26.0.2"
    androidMinSdkVersion = 21
    androidTargetSdkVersion = 25
    androidCompileSdkVersion = 25

    //Libraries
    daggerVersion = '2.11'
    butterKnifeVersion = '8.6.0'
    rxJavaVersion = '2.1.1'
    rxAndroidVersion = '2.0.1'
    gsonVersion = '2.8.1'
    okHttpVersion = '3.8.1'

```

在子 module 中可以进行引用：

版本引用：

```java

def globalConfiguration = rootProject.extensions.getByName("ext")

    compileSdkVersion globalConfiguration["androidCompileSdkVersion"]
    buildToolsVersion globalConfiguration["androidBuildToolsVersion"]

    defaultConfig {
        minSdkVersion globalConfiguration["androidMinSdkVersion"]
        targetSdkVersion globalConfiguration["androidTargetSdkVersion"]
        applicationId globalConfiguration["androidApplicationId"]
        versionCode globalConfiguration["androidVersionCode"]
        versionName globalConfiguration["androidVersionName"]
        testInstrumentationRunner globalConfiguration["testInstrumentationRunner"]
        testApplicationId globalConfiguration["testApplicationId"]
    }

```

依赖库引用：

```java

dependencies {
    def supportDependencies = rootProject.ext.supportDependencies
    def presentationDependencies = rootProject.ext.presentationDependencies
    def testDependencies = rootProject.ext.testDependencies
    compile supportDependencies.appcompatV7
    compile supportDependencies.constraint
    compile presentationDependencies.rxAndroid
    compile presentationDependencies.dagger
    annotationProcessor presentationDependencies.daggerCompiler
    compile presentationDependencies.butterKnife
    compile presentationDependencies.rxPermission
    annotationProcessor presentationDependencies.butterKnifeCompiler
    testCompile testDependencies.junit
}

```

这样，在更改版本或者引用三方库的时候，我们只需要统一管理即可。

### Res 分包

当项目比较庞大时，资源文件会越来越多，一般可以通过模块命名的方式进行区分，但是有时候总感觉很不方便。这时我们可以利用 gradle 进行资源模块的拆分。操作如下：

```java

android {
    ...
    }
  

    sourceSets {
        main {
            manifest.srcFile 'src/main/AndroidManifest.xml'
            java.srcDirs = ['src/main/java', '.apt_generated']
            aidl.srcDirs = ['src/main/aidl', '.apt_generated']
            assets.srcDirs = ['src/main/assets']
            res.srcDirs =
                    [
                            'src/main/res/layouts/main',
                            'src/main/res/layouts/music',
                            'src/main/res/'
                    ]
        }
    }

    buildTypes {
        debug {
          ...
        }
        release {
           ...
        }
    }
}

```

配置完成后，可在 res 根目录下创建 layouts 文件夹，然后在 layouts 文件夹下创建配置的模块文件夹，此文件夹下可创建各种资源文件夹，然后在build 项目，这样就可以使用单独模块的资源了。如下图所示：

<img src = "http://oop6dcmck.bkt.clouddn.com/20171112res_split.png">

## 总结

本篇文章简单介绍了项目的搭建和配置。在项目创建之初，架构的选择还是很重要的，但是在开发过程中，架构不可能满足所有需求，所以我们也不能拘泥于架构。实际项目中，我们可能会因不同的业务进行组件化、模块化等，选择合适的方案，才能更加有效率的解决问题。 

最后，附上项目地址  [TicktockMusic](https://github.com/Lauzy/TicktockMusic) 。


