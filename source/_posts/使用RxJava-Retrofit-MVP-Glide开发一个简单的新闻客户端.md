---
title: 使用RxJava+Retrofit+MVP+Glide开发一个简单的新闻客户端
date: 2016-11-18 12:37
tags: Android
categories: Android
---
好久没有更新博客了，有点懒。。之前在网上看到很多有趣又高效的开源库，于是就想着写一个简单的项目来运用这些开源库，恰巧看见聚合数据上有个新闻头条的API，所以就尝试写了一下。先上效果图：
<!--more-->
![这里写图片描述](http://img.blog.csdn.net/20161118121930729)![这里写图片描述](http://img.blog.csdn.net/20161118121955026)![这里写图片描述](http://img.blog.csdn.net/20161118122043792)![这里写图片描述](http://img.blog.csdn.net/20161118124024413)

在这个小项目中，使用RxJava和Retrofit网络请求框架来实现异步数据的获取是比较重要的环节。
关于RxJava和Retrofit的使用可以参考：http://gank.io/post/56e80c2c677659311bed9841

首先，需要封装好网络请求逻辑：

```
public class Api {
    private Retrofit retrofit;
    private static ApiService apiService;

    private Api() {
        OkHttpClient client = new OkHttpClient.Builder()
                .connectTimeout(10, TimeUnit.SECONDS)
                .build();
        retrofit = new Retrofit.Builder()
                .baseUrl(AppConfig.baseUrl)
                .addConverterFactory(GsonConverterFactory.create())
                //添加Rx适配
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .client(client)
                .build();
        apiService = retrofit.create(ApiService.class);
    }

    public static ApiService getDefault() {
        if (apiService == null) {
            synchronized (Api.class) {
                if (apiService == null) {
                    new Api();
                }
            }
        }
        return Api.apiService;
    }


}
```

```
public interface ApiService {
	//获取新闻请求，返回Observable对象
    @GET("index")
    Observable<News> getNews(@Query("type") String type, @Query("key") String apiKey);
}
```
网络请求封装好了，由于整体采用了MVP架构，所以在Model层需要异步请求数据：

```
public class NewsModel implements NewsContract.Model {
    @Override
    public Observable<List<Data>> getChannelList(String type) {
        return Api.getDefault()
                .getNews(type, AppConfig.apiKey)
                .map(news -> news.getResult().getData())
                .compose(RxSchedulers.io_main()); //对Observable进行转换，用于切换执行异步任务的线程
    }
}
```
整个项目设计中也同时实现了简单的事件总线RxBus，用于发送用户定义的Event：

```
public class RxBus {
    private static RxBus rxBus;

    private RxBus() {

    }

    public static RxBus getInstance() {
        if (rxBus == null) {
            synchronized (RxBus.class) {
                if (rxBus == null) {
                    rxBus = new RxBus();
                }
            }
        }
        return rxBus;
    }

    public final Subject<Object, Object> _bus = new SerializedSubject<>(PublishSubject.create());

    public final Vector<Subscription> subscriptions = new Vector<>();

	//发送事件
    public void send(Object o) {
        _bus.onNext(o);
    }

	//取出事件
    public Observable<Object> toObserverable() {
        return _bus;
    }

	//添加订阅
    public void addSubscription(Subscription s) {
        subscriptions.add(s);
    }

	//取消所有的订阅
    public void unSubscribeAll() {
        for (Subscription s : subscriptions) {
            if (!s.isUnsubscribed()) {
                s.unsubscribe();
            }
        }
    }
}
```

更多细节也可以参考：http://git.oschina.net/QiHuangQi/News

