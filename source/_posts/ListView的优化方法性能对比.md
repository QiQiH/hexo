---
title: ListView的优化方法性能对比
date: 2016-03-20 14:23
tags: Android
categories: Android
---

相信大部分像笔者一样的初学者在开发android项目时会使用到ListView来加载多个相同类型的条目。
ListView是一个常见的组件，能够以列表的形式展示内容。说到ListView，就不得不提及Adapter（适配器），Adapter的作用是ListView界面与数据之间的桥梁，当列表里的每一项添加到界面中，都会调用Adapter的getView()方法返回一个View。
但问题来了，如果一个ListView有成千上万个条目时，Adapter为每个item返回一个View，就会占用大量内存，导致滑动卡顿。因此，对ListView进行优化是有必要的。
首先了解一下Adapter中的getView(int position, View convertView, ViewGroup parent )方法。实际上，Android中有个叫做Recycler(反复循环器)的构件，下图是它的工作原理：
![这里写图片描述](http://img.blog.csdn.net/20160320134347006)
<!--more-->
当ListView初始化时，会实例化一数量实例化的View对象，同时ListView会将这些view缓存起来。当屏幕往上滑动时，位于最上方的item会被回收，用于构建新出来的item。getView()中的第二个参数convertView就是被缓存起来的view对象（初始化缓存为空，convertView为null）。
由此可见，利用convertView来返回view可以减少大量的内存使用。那如何使用，如何优化？
先看看Google官方建议的方法(ViewHolder)：
```
@Override
public View getView(int position, View convertView, ViewGroup parent) {
	ViewHolder holder = null;
	if (convertView == null) {
		convertView = mInflater.inflater(R.layout.test, null);
		holder = new ViewHolder();
		holder.icon = (ImageView) convertView.findViewById(R.id.icon);
		holder.text= (TextView) convertView.findViewById(R.id.text);
		convertView.setTag(holder); //添加到缓存
	} else {
		holder = (ViewHolder) convertView.getTag();  //存在convertView,获取缓存
	}
	holder.icon.setImageResourse(R.drawable.icon);
	holder.text.setText("test");
	
	return convertView;
}
final class ViewHolder {  //使用viewHolder来缓存已经解析好的布局，下次再使用时则省去解析布局这一步
	ImageView icon;
	TextView text;
}
```
再看看这个方法（Recycling views）：

```
@Override
public View getView(int position, View convertView, ViewGroup parent) {
	//直接使用convertView，每次都需要再次解析布局
	if (convertView == null) {
		convertView = mInflater.inflater(R.layout.test, null);
	}
	ImageView icon = (ImageView) convertView.findViewById(R.id.icon);
	TextView text= (TextView) convertView.findViewById(R.id.text);
	icon.setImageResourse(R.drawable.icon);
	text.setText("test");
	return convertView; 
}
```
最后测试不进行优化(Dumb)：

```
@Override
public View getView(int position, View convertView, ViewGroup parent) {
	//直接新建View，为每一个item都创建一个view
	View view = mInflater.inflater(R.layout.test, null);
	ImageView icon = (ImageView) view.findViewById(R.id.icon);
	TextView text= (TextView) view.findViewById(R.id.text);
	icon.setImageResourse(R.drawable.icon);
	text.setText("test");
	return view;
}
```
经过测试，明显使用viewHolder的方式优化会使滑动更加流畅。下图为三种方式的性能对比图：
![这里写图片描述](http://img.blog.csdn.net/20160320141907168)

## 其他可能遇到的问题 ##
使用viewHolder优化后仍然卡顿？
解答：考虑是ListView解析的布局中是否含有ImageView，如果有，ImageView加载的图片质量不能太大。使用前尽量使用BitmapFactory.Options里的相关参数压缩图片，可消除卡顿问题。