---
title: 关于字符串String，你所需要注意的细节
date: 2016-03-26 23:32
tags: Java
categories: Java
---

以前在编写Java程序时为了图个方便，拼接字符时总是用"+"来实现，而不用StringBuilder。直到近来看了《Thinking in java》这本书关于字符串的介绍才恍然大悟！原来这样子做反而会降低效率，有时甚至会使程序出错！
关于String首先要知道String对象是一个不可变的。什么是不可变？即每次对String的对象进行修改都会创建一个全新的String对象，然后保存修改后的字符串。而之前的String对象却没有做丝毫改变。我们看看下面的代码吧：

```
public class Main {
	public static void main(String[] args) {
		String s = "java";
		System.out.println(s);
		String s1 = add(s);
		System.out.println(s1);
		System.out.println(s);
	}
	
	public static String add(String s) {
		return s + "8";
	}
}

```
输出内容：

```
java
java8
java

```
<!--more-->
我们在这里可以看到即使add()返回了 s+"8"，但是原本s的值依然不变。这足以说明已经产生了一个新的对象了，因此String对象是不可变的。
了解了这个机制后，我们在了解一下关于“+”和StringBuilder，先看看一段简单的代码：

```
public class Main {
	public static void main(String[] args) {
		String s = "abc";
		String s1 = s + "def" + "123" + "end";
		System.out.println(s1);
	}
}
```
结合上面讲到的String对象是不可变的，那么每为s增加一段字符，就会产生一个新的对象。可想而知，如果总是这样拼接字符串，会产生多少没用到的对象！尽管有垃圾回收机制，但却会影响性能。
其实，编译器在编译这段代码时会“自作主张”地使用了StringBuilder这个类，因为它会更高效。每当拼接一段字符串时，就会调用StringBuilder的append方法来实现，最终再以toString()的形式输出。那让我们看看使用直接拼接字符串和使用StringBuilder类来拼接字符串在性能上有什么区别：

```
public class Main {
	public static void main(String[] args) {
		StringBuilder builder = new StringBuilder();
		String s = "scau";
		long start = System.currentTimeMillis();
		for (int i = 0; i < 2000; i++) {
			s += "hi";
		}
		System.out.println("直接拼接字符消耗时间：" + (System.currentTimeMillis() - start));
		start = System.currentTimeMillis();
		builder.append("scau");
		for (int i = 0; i < 2000; i++) {
			builder.append("hi");
		}
		System.out.println("StringBuilder拼接字符消耗时间：" + (System.currentTimeMillis() - start));
	}
}
```
输出结果：

```
直接拼接字符消耗时间：16
StringBuilder拼接字符消耗时间：0
```
由此看出，使用StringBuilder基本上没有什么时延。
最后，讲述一下关于“无意识递归”，当我们在String对象后加入this,而this又不是String对象时会出现无限递归的现象，导致爆栈，出现java.lang.StackOverflowError异常：

```
public class Main {
	public static void main(String[] args) {
		System.out.println(new Scau().toString());
	}
}
class Scau {
	@Override
	public String toString() {
		return "scau" + this;
	}
}
```
当返回"scau" + this时，由于this不是String对象，那么它会输出该对象地址，即再次调用toString()方法，从而导致了无限递归...