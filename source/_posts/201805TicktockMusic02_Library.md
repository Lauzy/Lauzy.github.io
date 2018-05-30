---
title: Android项目篇（二）：开源库及工具的封装
date: 2018-05-28 17:42:27
tags: 
	- Android Project
---

在我们的项目中，总会不可避免的用到三方的开源项目。在开源库的选择上，我们一般会选择成熟稳定，不断更新，作者及时解决 issue 的项目。而且大部分开源项目开放的 api 已经非常方便，使用简单，容易入手。但是，在我们做项目的过程中，最好再将开源项目进行封装。本文以图片加载框架和工具的封装为例来讨论下封装的好处。

### 必要性

1、统一入口，逻辑改动方便。
2、若三方库停止维护或者业务不满足需求需要更换三方库时，封装后方便改动。

## 图片加载框架的封装

Android 项目中常用的图片加载框架基本上都是 Picasso、Glide、Fresco、android-universal-image-loader，就近年来说，Picasso 和 Glide 由于其便捷性，更为受欢迎。本文就介绍下 Glide 的封装。

首先，最简单的封装如下：

```java

public class ImageLoader { 
    public static void display(Context context, String imageUrl, ImageView imageView) { 
        Glide.with(context).load(imageUrl).into(imageView);  
    } 
}


```

其实这样已经可以应对一般的场景了，但是业务相对复杂的话，比如改变图片的缓存策略、仅在wifi下加载图片、图片圆角等，这种简单的封装就无法满足了。 接下来我们采用策略模式进行封装。

1、 创建基础策略接口，所有的策略均实现此接口

```java

public interface IBaseImageStrategy {

    void display(Context context, ImageConfig imageConfig);

    void clean(Context context, ImageView imageView);
}

```

2、 图片加载配置类，此类包含用到的图片加载参数，采用 Build 模式进行参数的配置，这样使用更加灵活。

```java

public class ImageConfig {

    private Object url;
    private ImageView imageView;
    ...

    public ImageConfig(Builder builder) {
        this.url = builder.url;
        this.imageView = builder.imageView;
        ...
    }

    ... //getter and setter
   
    public static class Builder {
        private Object url;
        private ImageView imageView;
       
        ...       

        public ImageConfig build() {
            return new ImageConfig(this);
        }
    }
}


```

3、具体的策略，比如我们采用 Glide ，则创建 GlideImageLoaderStrategy，可以根据 config 的属性作为参数，来配置 Glide 具体的加载参数。

```java

public class GlideImageLoaderStrategy implements IBaseImageStrategy {

    @Override
    public void display(Context context, ImageConfig imageConfig) {
        RequestOptions options = getOptions(context, imageConfig);
        Object url = getPath(imageConfig);
        if (!imageConfig.isAsBitmap()) {
            RequestBuilder<Drawable> requestBuilder = Glide.with(context)
                    .load(url)
                    .apply(options);
            if (!imageConfig.isRound() && imageConfig.getDuration() != 0) {
                requestBuilder = requestBuilder.transition(new DrawableTransitionOptions()
                        .crossFade(imageConfig.getDuration()));
            }
            requestBuilder.into(imageConfig.getImageView());
        } else {
            RequestBuilder<Bitmap> requestBuilder = Glide.with(context)
                    .asBitmap()
                    .load(url)
                    .apply(options);
            if (!imageConfig.isRound() && imageConfig.getDuration() != 0) {
                requestBuilder = requestBuilder.transition(new BitmapTransitionOptions()
                        .crossFade(imageConfig.getDuration()));
            }
            requestBuilder.into(imageConfig.getTarget());
        }
    }

    /**
     * Glide 配置
     *
     * @param context     context
     * @param imageConfig 配置
     * @return Glide配置
     */
    private RequestOptions getOptions(Context context, ImageConfig imageConfig) {
        RequestOptions options = new RequestOptions()
                .placeholder(imageConfig.getDefaultRes())
                .error(imageConfig.getErrorRes());
        ...
        return options;
    }


    @Override
    public void clean(Context context, ImageView imageView) {
        Glide.with(context).clear(imageView);
    }

```

4、统一入口，此处采用单例模式。 

```java

public class ImageLoader implements IBaseImageStrategy {

    private static ImageLoader INSTANCE;
    private IBaseImageStrategy mImageStrategy;


    {
        mImageStrategy = new GlideImageLoaderStrategy();
    }

    public static ImageLoader getInstance() {
        if (INSTANCE == null) {
            synchronized (ImageLoader.class) {
                if (INSTANCE == null) {
                    INSTANCE = new ImageLoader();
                }
            }
        }
        return INSTANCE;
    }

    @Override
    public void display(Context context, ImageConfig imageConfig) {
        mImageStrategy.display(context, imageConfig);
    }

    @Override
    public void clean(Context context, ImageView imageView) {
        mImageStrategy.clean(context, imageView);
    }
}

```

5、使用，所有的加载均使用统一的 ImageLoader 进行加载。

```java

 ImageLoader.getInstance().display(context, new ImageConfig.Builder()
                .url(url)
                .placeholder(R.drawable.ic_placeholder)
                .into(imageView)
                .build());

```

6、当我们因为一些原因需要切换图片加载库，比如之前用的 Picasso（策略类为 PicassoImageLoaderStrategy）,由于要加载 gif 图片，所以切换到 Glide，这样我们只用新建一个 GlideImageLoaderStrategy，在 ImageLoader 中改变 IBaseImageStrategy 的实例类即可切换。

#### 图片加载框架封装总结

综上，这种封装模式的好处就是便于根据业务及其他需求进行扩展和维护。当然封装并不是万能的，比如使用 Glide 可能用到的 Target ，在 Picasso 中并没有此类，所以用到 Target 的地方我们仍然需要一个个手动的更改。另外，Fresco 由于使用时涉及 xml 文件等，用法相对特殊，所以使用 Fresco 的话，封装很难顾及到。本文的封装主要是提供思路，以一种相对维护性高的方式进行封装。


## 工具的封装
 
上边以 Glide 为例介绍了开源项目的封装。我们在项目过程中，也会用到各种工具，接下来介绍下 SharedPrefrences 的封装。 SharedPrefrences 经常用来保存一些简单的信息，如各种状态，部分缓存等。由于其特殊性，所以很多人会写一个工具类来简单处理下 SharedPrefrences 的使用，但是 SharedPrefrences 作为一种数据存储的手段，我觉得还是需要重视起来，以便处理后续的需求。接下来就介绍下 SharedPrefrences 的封装。

1、定义数据仓库的接口，所有的数据仓库接口都继承此接口，并且创建此接口的实现类。

数据仓库接口：

```java

public interface DataRepo {

    void put(String key, String value);

    void put(String key, int value);

    void put(String key, boolean value);

    void put(String key, long value);

    String getString(String key);

    String getString(String key, String defaultValue);

    int getInt(String key);

    int getInt(String key, int defaultValue);

    long getLong(String key);

    boolean getBoolean(String key);

    boolean getBoolean(String key, boolean defaultValue);

    void remove(String key);

    Map<String, ?> getAll();

    boolean contains(String key);

    void clear();
}

```

实现类：

```java

public class SharedPreferenceDataRepo implements DataRepo {

    private final SharedPreferences mSharedPreferences;

    public SharedPreferenceDataRepo(Context context, String fileName, int mode) {
        mSharedPreferences = context.getSharedPreferences(fileName, mode);
    }

    @Override
    public void put(String key, String value) {
        mSharedPreferences.edit().putString(key, value).commit();
    }

    ...

    @Override
    public void clear() {
        mSharedPreferences.edit().clear().commit();
    }
}


```


2、定义具体的仓库接口，如 CacheRepo，并创建此接口的实现类。

仓库接口：

```java

public interface CacheRepo extends DataRepo{

    void setCache(String key, String value);

    String getCache(String key);

}


```

实现类：

```java

public class CacheRepoImpl extends SharedPreferenceDataRepo implements CacheRepo {

    private static final String FILE_NAME = "cache_sp";

    public CacheRepoImpl(Context context) {
        super(context, FILE_NAME, Context.MODE_PRIVATE);
    }

    @Override
    public void setCache(String key, String value) {
        put(key, value);
    }

    @Override
    public String getCache(String key) {
        return getString(key);
    }
}

```

3、统一入口，这里类似一个工厂方法，根据需要来获取不同的数据仓库。

```java

public class DataManager {

    private final CacheRepo mCacheRepo;
    private final ConfigRepo mConfigRepo;
    private volatile static DataManager INSTANCE;

    public DataManager(Context context) {
        mCacheRepo = new CacheRepoImpl(context);
        mConfigRepo = new ConfigRepoImpl(context);
    }

    public static DataManager getInstance(Context context) {
        if (INSTANCE == null) {
            synchronized (DataManager.class) {
                if (INSTANCE == null) {
                    INSTANCE = new DataManager(context);
                }
            }
        }
        return INSTANCE;
    }

    public CacheRepo getCacheRepo() {
        return mCacheRepo;
    }

    public ConfigRepo getConfigRepo() {
        return mConfigRepo;
    }
}


```

使用，统一通过 DataManager 进行加载：

```java

DataManager.getInstance(mContext).getCacheRepo().getCache("key");

```

#### SharedPrefrences 封装总结

SharedPrefrences 的封装和 Glide 类似，其实也都是为了提高可维护性。假如 SharedPrefrences 缓存的内容偏多不再合适时，我们可以很方便的转化为数据库，文件等其他的存储方式。

## 总结

对于封装，也要考虑开源项目的本身因素，假如侵入性比较强或者作者更新及时、项目比较复杂之类的，也可以根据人力考虑是否封装及如何封装。
本文以 Glide 和 SharedPrefrences 为例分别介绍了开源项目和工具的封装，封装的好处是为了提高代码的可维护性和扩展性等，以便应付不断变化的需求及其他因素。所有的代码都在 [TicktockMusic](https://github.com/Lauzy/TicktockMusic)  中，欢迎讨论。