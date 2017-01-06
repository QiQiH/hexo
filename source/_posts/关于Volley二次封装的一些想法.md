---
title: 关于Volley二次封装的一些想法
date: 2016-05-24 20:20
tags: Android
categories: Android
---
![Volley适用于发送小而多的请求](http://img.blog.csdn.net/20160524201922927)
最近忙于课程和项目，好久没有更新博客了。最近在项目中使用了Volley框架来请求数据，一开始接触Volley感觉使用起来挺简单，只需几个简单的操作就可以实现发送请求。以StringRequest 为例：
<!--more-->
```
//创建一个请求队列
RequestQueue requestQueue = Volley.newRequestQueue(this);
        String url = "这里是请求的链接...";
        //创建StringRequest对象，以Post数据为例
        StringRequest request = new StringRequest(Request.Method.POST, url, new Response.Listener<String>() {
            @Override
            public void onResponse(String response) {
                Log.i("TAG", response);
            }
        }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
				 Log.i("TAG", error.toString());
            }
        }) {
	        //填写需要post的表单
            @Override
            protected Map<String, String> getParams() throws AuthFailureError {
                HashMap<String, String> map = new HashMap<>();
                map.put("userId", "0"); //测试数据
                map.put("sessionId", "0");
                return map;
            }
        };
        requestQueue.add(request);
```
以上，就是Volley请求简单的实现。可是，在开发中，往往需要减少代码的冗余，并且请求队列也应该全局共享。
本着再封装Volley的想法，上网找了许多方案，都觉得不太适合。于是，想着怎么设计一个方案来封装。既然，Volley的请求队列需要全局共享，那么可以创建一个MyApplicantion类继承自Application，然后在里面共享一个请求队列：

```
public class MyApplication extends Application{
	private static RequestQueue queue;
	private final static String QUE_TAG = "REQUEST";
	/**
	 * 加载时新建一个请求队列
	 */
	@Override
	public void onCreate() {
		super.onCreate();
		//请求队列在onCreate()方法里创建，只会执行一次
		queue = Volley.newRequestQueue(getApplicationContext());
	}
	/**
	 * 
	 * @return 返回请求队列
	 */
	public static RequestQueue getHttpQueue() {
		return queue;
	}
	
	public static void addToQueue(StringRequest request) {
		if (request != null) {
			request.setTag(QUE_TAG);
			queue.add(request);
		}
	}
	
	/**
	 * 取消请求队列，在onStop()方法中调用
	 */
	public static void cancelRequest() {
		if (queue != null) {
			queue.cancelAll(QUE_TAG);
		}
	}	
}
```
有了这个请求队列后，再设法将StringRequest这个请求封装好，以便在需要的地方直接调用，这里创建了一个VolleyHelper类：

```
/**
 * volley二次封装，用于添加和管理当前的请求
 * @author qiHuang
 *
 */
public class VolleyHelper {
	/**
	 * 当前请求
	 */
	private StringRequest request;
	
	public VolleyHelper() {
	}
	/**
	 * 发送请求数据
	 * @param url 请求的地址
	 * @param callBack 响应后的回调接口
	 */
	public void postRequest(String url, final ICallBack callBack) {
		request = new StringRequest(Request.Method.POST, url, new Response.Listener<String>() {
			@Override
			public void onResponse(String response) {
				try {
					callBack.onSuccess(new JSONObject(response));
				} catch (JSONException e) {
					e.printStackTrace();
				}
			}
		},  new Response.ErrorListener() {

			@Override
			public void onErrorResponse(VolleyError arg0) {
				callBack.onFailed(arg0);
			}
		}) {
			@Override
			protected Map<String, String> getParams() throws AuthFailureError {
				return callBack.postParams();
			}
		};
		MyApplication.addToQueue(request);
	}
	
	/**
	 * 取消请求
	 */
	public void cancelRequest() {
		if (request != null) {
			request.cancel();
		}
	}
	
	/**
	 * 清空请求队列
	 */
	public static void cancelAllRequest() {
		MyApplication.cancelRequest();
	}
}
```
上面涉及到一个回调接口ICallBack:

```
public interface ICallBack {
	/**
	 * 成功时回调的函数
	 * @param object json
	 */
	public void onSuccess(JSONObject object);
	
	/**
	 * 失败时回调的函数
	 * @param error 错误原因
	 */
	public void onFailed(VolleyError error);
	
	/**
	 * post参数
	 * @return
	 */
	public Map<String, String> postParams();
}

```
通过VolleyHelper的postRequest方法与回调接口ICallBack的结合，便可实现请求的封装。这样子，就可以在需要请求的地方执行下面的语句来实现请求：

```
VolleyHelper helper = new VolleyHelper();
helper.postRequest(URL, new ICallBack() {
			@Override
			public Map<String, String> postParams() {
				HashMap<String, String> map = new HashMap<>(2);
				map.put("useId", "0");
				map.put("sessionId", "0");
				return map;
			}
			@Override
			public void onSuccess(JSONObject object) {
 				//成功，UI界面更新
			}
			
			@Override
			public void onFailed(VolleyError error) {
				//失败，处理失败结果
			}
		});
```
通过上面的设计，个人感觉在逻辑上比较清晰，也便于请求的添加与管理。可能还存在不妥的地方，慢慢改善吧。