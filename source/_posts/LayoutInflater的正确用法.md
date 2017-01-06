---
title: LayoutInflater的正确用法（译文）
date: 2016-04-03 16:46
tags: Android
categories: Android
---
原文来自 Dave Smith 写的Layout Inflation as Intented，详细内容请戳原文链接：https://possiblemobile.com/2013/05/layout-inflation-as-intended/

Layout inflation是在android系统中使用的术语，当XML布局资源被解析并转换成View对象时会用到。
在Android SDK中，LayoutInflater是经常使用到的，但你也许会感到惊讶当你发现了一个LayoutInflater的使用误区，而且你的APP可能正在使用这种错误方式！当你的程序在使用LayoutInflater时，如果你写过像下面的代码：

```
inflater.inflate(R.layout.my_layout, null);
```
那么请继续读这篇文章，因为你正在错误的使用它。让我来解释一下为什么。
<!--more-->
## 认识LayoutInflater##
我们首先看看LayoutInflater的工作原理是什么。在一个应用程序中，inflate()方法有两个可用的版本：

```
1. inflate(int resource, ViewGroup root)
2. inflate(int resource, ViewGroup root, boolean attachToRoot)
```
第一个传入的参数resourse是你想要加载的布局资源。第二个传入的参数是指当前载入的视图要将要放入在层级结构中的根视图。如果传入了第三个参数attachToRoot，那么它会决定视图在载入完成后是否附加到传入的根视图中去。
这里，后面的两个参数可能会造成一些错误。如果inflate()方法带有这两个参数，LayoutInflater 会尝试自动地将载入的视图绑定到给定的root视图中。但是，如果你传入的root参数为null的话，系统会在适当的位置有一个检查来试图避免程序崩溃。
许多开发者会这样做，他们认为传入参数为null是就可以关闭绑定操作了；在许多时候甚至没有意识到有三个参数的inflate()方法的存在。这样做的话，我们也会禁用了根视图中的一个非常重要的方法.....不过这个超出我的研究范围了。
## 框架中的一些示例 ##
我们来看一下在Android框架中一些载入视图的情景。
Adapter(适配器)使用LayoutInflater 最常见的情况是自定义ListView然后重写getView()方法，它的方法签名如下：
```
getView(int position, View convertView, ViewGroup parent)
```
Fragment（碎片）通过onCreateView()方法来创建View对象时，也是经常使用inflation操作的，注意它的方法签名：

```
onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState)
```
你是否有注意到，每次框架需要你去载入一个布局时，都会传入一个ViewGroup对象的参数？同样我们会发现，在大多数情况下（包括上面的两个情况），如果允许LayoutInflater自动地将载入的布局绑定根视图root时，随后就会抛出一个异常。
所以你可以想象一下，如果我们不进行绑定的话，为什么要给我们ViewGroup这个参数？事实证明，在载入视图的过程中父视图是一个非常重要的部分，因为计算载入XML资源的根视图的LayoutParams 是必要的。如果传入的参数为null，这就类似告诉框架“对不起，我不知道载入的View视图该绑定到哪个父视图中”。
其中伴随的一个问题是，android:layout_xxx这些属性通常会在父视图中再计算。结果是，因为没有已知的父视图，所有你在XML定义的LayoutParams属性都会被忽略掉，然后你就会想：为什么框架会忽略掉我定义的布局属性呢？我最好上Stack OverFlow去找一下问题，然后提交一个bug。
没有了LayoutParams，ViewGroup最终会为了载入一个默认设置的视图。可能你很幸运（大多数情况下是），这些默认的属性刚好跟你在XML设置的相同......但是它掩盖了出错的事实。
## 程序示例 ##
你能断言你没有在你的程序中出现过这些情况？看看下面我们要为ListView载入的一个简单的布局：
**R.layout.item_row**

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="?android:attr/listPreferredItemHeight"
    android:gravity="center_vertical"
    android:orientation="horizontal">
    <TextView
        android:id="@+id/text1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:paddingRight="15dp"
        android:text="Text1" />
    <TextView
        android:id="@+id/text2"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:text="Text2" />
</LinearLayout>
```
我们想要设置布局中的高度（第三行）为固定高度，这种情况下每个item首选的告诉是当前主题给定的高度......看上去很合理。
但是，当我们用错误的方式载入布局时：

```
public View getView(int position, View convertView, ViewGroup parent) {
    if (convertView == null) {
        convertView = inflate(R.layout.item_row, null);
    }
 
    return convertView;
}
```
我们会得到像这样的结果：
![这里写图片描述](http://img.blog.csdn.net/20160403161201837)
我们设置的固定高度出了什么问题？？这通常因为没有把所有子View都设置固定高度、根布局高度设置wrap_content，接下去不需要知道为什么它会这样（你可能在这个过程中抱怨谷歌这样做）

但是，如果我们用这种方式来载入相同的布局：

```
public View getView(int position, View convertView, ViewGroup parent) {
    if (convertView == null) {
        convertView = inflate(R.layout.item_row, parent, false);
    }
 
    return convertView;
}
```
我们最终就会得到我们刚开始所想的结果：
![这里写图片描述](http://img.blog.csdn.net/20160403162241544)
Hooray!

## 每个规则都有例外 ##
有些情况，我们在载入视图是也能传入为null的父视图，但是这种情况很少。其中一种情况是当我们要将载入的自定义布局绑定到AlertDialog时。看看下面的例子，我们要使用相同的布局，但这次要载入到dialog的视图中：

```
AlertDialog.Builder builder = new AlertDialog.Builder(context);
View content = LayoutInflater.from(context).inflate(R.layout.item_row, null);
 
builder.setTitle("My Dialog");
builder.setView(content);
builder.setPositiveButton("OK", null);
builder.show();
```

这里问题是，AlertDialog.Builder能够支持自定义布局，但是没有实现能够提供布局资源的setView()方法，所以你必须手动地载入XML。然而，最终的结果会加载进dialog，并不会接触到它的根视图（事实上，它现在还不存在），我们不能接触到父视图，所以我们不能载入使用。事实证明，这是无关紧要的，因为AlertDialog会擦除在布局上所有的LayoutParams，然后把它们代替为match_parent。
所以下次你在使用inflate()方法传入参数null时，你就要停下来问一下自己：“我是真的不知道视图最终该载入到哪里吗？”
最后，你可以考虑一下两个参数的inflate()作为方便一点的方法，这个方法忽略第三个参数，它默认为true。同时，你不应该为了方便传入null而忽略了你三个参数默认为false。