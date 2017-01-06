---
title: RxJava&Retrofit如何优雅地处理错误？
date: 2016-12-22 15:11
tags: Android
categories: Android
---
RxJava和retrofit配合使用进行网络请求在实际开发中还是很强大的。在实际开发中，再对网络请求返回的结果往往要先进行预处理。即先过滤了错误的信息，在执行onNext()方法时只需要考虑正确的结果下如何处理就行。
以一个正常的Json返回为例：

```
{
    "status": 1,
    "errmsg": "OK",
    "data": {...}
}

{
    "status": 0,
    "errmsg": "error",
    "data": {}
}
```
status表示请求结果正确（1）还是错误（0），errmsg为错误信息，data为正请求正确返回的数据。
此时，在Java代码中就可以创建一个JavaBean对象来对应该Json格式：

<!--more-->
```
public class BaseResponse<T> {
	public int status;
	public String errmsg;
	public T data;

	public boolean isStatusOK() {
		return status == 1;
	}
}
```
由于在onNext()中只需要正确结果中的data，所以需要进行错误信息的过滤：

```
   /**
     * 结果预处理，此方法用于规范的JSON格式
     * Observable<BaseResponse<T>> -> Observable<T>
     * @param <T>
     * @return 
     */
    public static <T> Observable.Transformer<BaseResponse<T>, T> handleResult() {
        return new Observable.Transformer<BaseResponse<T>, T>() {
            @Override
            public Observable<T> call(Observable<BaseResponse<T>> rObservable) {
                return rObservable.flatMap(new Func1<BaseResponse<T>, Observable<T>>() {
                    @Override
                    public Observable<T> call(BaseResponse<T> tBaseResponse) {
                        if (tBaseResponse.isStatusOK()) {
                            return createData(tBaseResponse.data);
                        } else {
		                    //如果发生错误就会在onError()方法处进行处理
                            return Observable.error(new ServerException(tBaseResponse.errmsg));
                        }
                    }
                });
            }
        };
    }
    
private static <T> Observable<T> createData(final T data) {
        return Observable.create(new Observable.OnSubscribe<T>() {
            @Override
            public void call(Subscriber<? super T> subscriber) {
                try {
                    subscriber.onNext(data);
                    subscriber.onCompleted();
                } catch (Exception e) {
                    subscriber.onError(e);
                }
            }
        });
    }
```
然后在网络请求时便可传入上述变换器来进行转换,以登录为例：

```
//定义请求借口
public interface ApiService {
    @FormUrlEncoded
    @POST("login")
    Observable<BaseResponse<MyData>> login(@Field("userPhone") String userPhone
            ,@Field("userPass") String psw, @Field("SeriesID") int SeriesID);
}
```

```
	//此方法位于Model层，对于onNext的处理在presenter层
	@Override
    public Observable<MyData> login(String phone, String psw) {
        return Api.getDefault()
                .login(phone, psw, AppConfig.SERIAL_ID)
                .compose(RxTransformer.<BaseResponse<MyData>>schedules_io_main()) //线程调度
                .compose(RxTransformer.<MyData>handleResult());  //转换为正确的结果
    }
    
```

```
	//presenter层
	@Override
    public void login(String phone, String psw) {
        mManager.add(mModel.login(phone, psw).subscribe(new RxSubscriber<MyData>(mContext) {
            @Override
            public void _onNext(MyData myData) {
                //处理myData
            }
        }));
    }
```

以上就可以实现规范的Json数据处理。
但是，有时候服务器端返回的Json并非都是完全规范的，比如，最近遇到的一种格式：

```
{
    "status": 1,
    "errmsg": "OK",
    "data": {...}
}

{
    "status": 0,
    "errmsg": {
        "code": 101,
        "content": "error!"
    },
    "data": {}
}
```
其中errmsg是不确定的类型，可以为String，也可以为JavaBean，**由status决定**，导致无法像data一样用泛型直接处理。
这时候就需要自己手动去根据status的值来解析数据，BaseResponse也有所修改：

```
//不能直接解析errmsg，如果类型不正确会报错
public class BaseResponse<T> {
    public int status;
    public T data;

    public boolean isStatusOK() {
        return status == 1;
    }
}
```
此时，定义请求接口时返回需要修改:
	

```
public interface ApiService {
    @FormUrlEncoded
    @POST("login")
    //返回String，用来手动解析
    Observable<String> login(@Field("userPhone") String userPhone
            ,@Field("userPass") String psw, @Field("SeriesID") int SeriesID);
}
```
这里，由于返回的是String类型，所以要为retrofit添加转换器：

> compile 'com.squareup.retrofit2:converter-scalars:2.0.0-beta4'
> 其他转换器也可在https://github.com/square/retrofit 中查找

```
 retrofit = new Retrofit.Builder()
                .client(okHttpClient)
                .addConverterFactory(ScalarsConverterFactory.create()) //添加字符串转换
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .baseUrl(BASE_URL)
                .build();
```
变换器就应该有所修改：

```
/**
     * 结果预处理，此方法用于处理不规范的JSON格式，代码需根据不规范的形式进行修改
     * Observable
     * @param cls 直接解析的JavaBean类型
     * @param <T>
     * @return
     */
    public static <T> Observable.Transformer<String, T> handleResultFromString(final Class<T> cls) {
        if (cls == null) {
            throw new NullPointerException("cls cannot be null!");
        }
        return new Observable.Transformer<String, T>() {
            @Override
            public Observable<T> call(Observable<String> sObservable) {
                return sObservable.flatMap(new Func1<String, Observable<T>>() {
                    @Override
                    public Observable<T> call(String s) {
                        Gson gson = new Gson();
                        Type type = new TypeToken<BaseResponse<T>>(){}.getType();
                        //解析Json对应到BaseResponse
                        BaseResponse<T> response = gson.fromJson(s, type);
                        if (response.isStatusOK()) { //结果正确
                           if (response.data == null) {
                                try {
                                    return createData(null);
                                } catch (InstantiationException e) {
                                    e.printStackTrace();
                                } catch (IllegalAccessException e) {
                                    e.printStackTrace();
                                }
                                return createData(null);
                            } else { //data不为空，解析到传入的泛型T中
                                T t = gson.fromJson(response.data.toString(), cls);
                                return createData(t);
                            }
                        } else {
	                        //如果发生错误则解析到BaseError中，最终由onError处理
                            BaseError error = gson.fromJson(s, BaseError.class);
                            return Observable.error(new ServerException(error.errmsg.content));
                        }
                    }
                });
            }
        };
    }
``` 

```
public class BaseError {
    public ErrMsg errmsg;
}
public class ErrMsg {
    public int code;
    public String content;
}
```
###
此时，在model层应用转换器：

```
	@Override
    public Observable<MyData> login(String phone, String psw) {
        return Api.getDefault()
                .login(phone, psw, AppConfig.SERIAL_ID)
                .compose(RxTransformer.<String>schedules_io_main())
                .compose(RxTransformer.handleResultFromString(MyData.class));
    }
```
通过上述的实现则完成了错误的预处理，当然，不规范的情况可能不止这一种，其他类型的也需要手动去进行解析。
