---
title: Android项目篇（二）：项目架构实现
date: 2018-04-11 21:42:27
tags: 
	- Android Project
---

去年年底的时候写了 Android 项目的第一篇博客，当时计划是年前把项目和博客都写完，但是由于公司项目开启，平时比较累，加上项目期间的收获部分也应用到 [TicktockMusic](https://github.com/Lauzy/TicktockMusic) 上边，功能写的较少，原有的东西改动较多，所以进度比较缓慢，直到年后才把剩余的部分功能写完（此项目主要用于学习交流，部分功能可能有所缺失），现在将项目的心得及体会写几篇博客来分享。


上篇文章简单介绍了 clean architecture 的特点，并没有具体说明如何构建使用，本篇文章会根据 [TicktockMusic](https://github.com/Lauzy/TicktockMusic) 介绍下如何使用。

### 1、domain 层

通俗来讲就是业务的接口层。如何定义接口需要根据具体的业务来确定，拿我的音乐类App来讲，用到的数据有本地音乐（包括专辑、歌手等）、网络音乐、增删改查喜欢的音乐和最近播放等。 实际项目中用到最多的就是网络的数据获取，此处就拿网络数据来举例：

首先确定数据模型，也就是实体类 NetSongBean；

其次定义一个数据仓库的接口NetSongRepository：

```java

public interface SongRepository {
    /**
     * 获取网络数据
     * @param method method
     * @param type 类型
     * @param offset 页数
     * @param size 页面加载量
     * @return ob
     */
    Observable<List<NetSongBean>> getSongList(String method, int type, int offset, int size);

    /**
     * 获取缓存数据
     * @param type 类型
     * @return ob
     */
    Observable<List<NetSongBean>> getCacheSongList(int type);
}

```
然后就可以写好用例，此处的业务为获取网络数据和缓存的数据，得到 Observable<NetSongBean> 即可：

```java

public class GetSongListUseCase extends UseCase<List<NetSongBean>, GetSongListUseCase.Params> {

    private final SongRepository mSongRepository;

    @Inject
    GetSongListUseCase(SongRepository songRepository, ThreadExecutor threadExecutor,
                       PostExecutionThread postExecutionThread) {
        super(threadExecutor, postExecutionThread);
        mSongRepository = songRepository;
    }


    @Override
    Observable<List<NetSongBean>> buildUseCaseObservable(GetSongListUseCase.Params params) {
        return mSongRepository.getSongList(params.method, params.type, params.offset, params.size);
    }

    /**
     * 获取缓存数据 Observable
     * @param param 类型
     * @return Observable
     */
    public Observable<List<NetSongBean>> buildCacheObservable(int param) {
        return mSongRepository.getCacheSongList(param);
    }
}

```

这样domain层的一个业务接口及用例便写好了，接下来我们需要实现 data 层。

### 2、data 层

data 层的任务就是获取数据，具体实现业务所需数据及数据处理。 这一层做的工作相对比较多，上一步我们写好了业务接口，此处便要具体的实现从网络中获取数据。
首先我们需要根据网络数据来定义数据模型（如果数据模型和 domain 层的 NetSongBean 一样则无需多此一举）；
其次利用封装好的网络框架来实现网络操作， [https://github.com/android10/Android-CleanArchitecture](https://github.com/android10/Android-CleanArchitecture) 此项目还区分了数据源，我这里相对简化了。

```java

@Singleton
public class SongRepositoryImpl implements SongRepository {

    @Override
    public Observable<List<NetSongBean>> getSongList(final String method, final int type,
                                                     final int offset, final int size) {

        return RetrofitHelper.getInstance().createApi(SongService.class)
                .getMusicData(method, type, offset, size)
				.map(new Function<MusicEntity, List<NetSongBean>>() {
                    @Override
                    public List<NetSongBean> apply(@NonNull MusicEntity musicEntity) throws Exception {
                        SongListMapper mapper = new SongListMapper();
                        return mapper.transform(musicEntity.song_list);
                    }
                }).doOnNext(...);
    }

    @Override
    public Observable<List<NetSongBean>> getCacheSongList(final int type) {
        return Observable.create(...);
    }
}

```

这里说明一下 SongListMapper ，由于我从百度音乐获取到的数据直接转换为 bean 类后属性极多而且大部分无用，所以此处采用数据模型映射来转换为具体需要的业务模型，即 NetSongBean。此方面可以参考 [https://www.jianshu.com/p/c9384eef179e](https://www.jianshu.com/p/c9384eef179e),根据需要来取舍。

这一层实现了具体的网络数据获取，所以接下来我们就可以处理 UI 显示了。

### 3、presentation 层

这一层主要用于 UI 展示及 Android 的业务处理，在这一层我才用了结构相对清晰的 MVP ，这一层和 domain 及 data 层的连接主要是通过 dagger2 来构建对象，然后 presenter 中进行处理 UI 的逻辑。

dagger2 构建对象:

```java

    @Provides
    @Singleton
    SongRepository provideSongRepository() {
        return new SongRepositoryImpl(mApplication);
    }

```

在 presenter 中注入对象并使用：

```java

    @Inject
    NetMusicPresenter(GetSongListUseCase songListUseCase) {
        mSongListUseCase = songListUseCase;
    }

    ...

     @Override
    public void loadNetMusicList() {
        GetSongListUseCase.Params params = GetSongListUseCase.Params.forSongList(METHOD, mType, 0, SIZE);
        mSongListUseCase.buildCacheObservable(mType)
                .flatMap(songListBeen -> mSongListUseCase.buildObservable(params))
                .compose(RxHelper.ioMain())
                .subscribeWith(...);
    }

```

这样，presenter 处理完逻辑，activity 或者 fragment 控制视图显示，便完成了一个流程。


### 总结

采用 clean 架构的好处就是数据独立，层次清晰。 本篇简单介绍了具体的使用，下篇会介绍下三方库的封装和使用。