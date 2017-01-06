---
title: 浅析ClassLoader
date: 2016-10-01 00:41
tags: JVM
categories: JVM
---

类加载过程中加载阶段是最可控的一个阶段，可以由系统的类加载器来加载，也可以用户自定义加载，目的都是为了将二进制字节流读入内存中。在加载类时，是通过ClassLoader的loadClass()方法来实现的。
##**ClassLoader的简单使用**
那系统提供的类加载器与用户自定义的类加载器加载的类有什么区别呢？那么尝试自己手动创建一个类加载器来加载类：

```
public class ClassLoaderTest {
	public static void main(String[] args) {
		ClassLoader loader = new MyClassLoader();
		try {
			Object object = loader.loadClass("com.scau.ClassLoaderTest").newInstance();
			System.out.println(object.getClass());
			//通过instanceof关键字来判断系统加载和自定义加载是否相同
			System.out.println(object instanceof com.scau.ClassLoaderTest);
		} catch (InstantiationException e) {
			e.printStackTrace();
		} catch (IllegalAccessException e) {
			e.printStackTrace();
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		}
	}
	
}
<!--more-->

//自定义加载器，重写loadClass方法
class MyClassLoader extends ClassLoader {
	@Override
	public Class<?> loadClass(String name) throws ClassNotFoundException {
		String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
		InputStream is = getClass().getResourceAsStream(fileName);
		if (is == null) {
			return super.loadClass(name);
		}
		try {
			byte[] b = new byte[is.available()];
			is.read(b);
			return defineClass(name, b, 0, b.length);
		} catch (IOException e) {
			e.printStackTrace();
		}
		return super.loadClass(name);
	}
}
```

结果：

>class com.scau.MyClassLoader
>false


会发现名字上虽然一样，但是在类属性比较的时候却是不同的。因为此时虚拟机中存在了两个MyClassLoader类，一个有系统加载器加载，而另一个有用户自定义加载器加载，是两个独立的类。

##**双亲委派模型**
Java虚拟机中本身存在着两种不同的类型的加载器：一种是启动类加载器(Bootstrap ClassLoader)，由C++实现，而一种是有Java实现的所有其他类加载器，它们都继承自ClassLoader类。
往细里分，可以把类加载器分为3类：
>1.启动类加载器：这个类加载器负责将JAVA_HOME / lib目录中的类加载到虚拟机内存中。无法被Java程序引用。
>2.扩展类加载器：这个类负责加载JAVA_HOME / lib / ext 中或是java.ext.dirs指定的类库。可以直接使用这个加载器。
>3.应用程序类加载器：也称系统类加载器，由getSystemClassLoader方法返回，负责加载ClassPath上指定的类库。

###**类加载的层级关系**
![这里写图片描述](http://img.blog.csdn.net/20161001001955391)
上图展示了类加载器的层级关系，称为类加载器的**双亲委派模型**。该模型要求除了顶级类加载器外，其他的类加载器都应该有父类加载器。
###**双亲委派模式的工作过程**
可以先看看java.lang.ClassLoader是如何实现这个模型的，找到其中的loadClass方法：

```
/**
* @params resolve true:保证类已经装载，而且已经连接了。false:仅仅是去装载这个类，不关心是否连接。
*/
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
	    //获取对象锁
        synchronized (getClassLoadingLock(name)) {
            //首先，检查类是否被加载过
            Class<?> c = findLoadedClass(name);
            if (c == null) {
	            //没有加载过，进入条件语句
                long t0 = System.nanoTime();
                try {
	                //判断是否有父类加载器
                    if (parent != null) {
	                    //委托父类加载器去加载
                        c = parent.loadClass(name, false);
                    } else {
	                    //到达顶级，没有父类加载器
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                }

                if (c == null) {
                    // 仍然找不到，委托findClass方法加载
                    long t1 = System.nanoTime();
                    c = findClass(name);
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```
从上面的代码可以看到，当一个类收到了类加载的请求，首先会委托父类加载器去加载，那么所有的类加载都会最终传到顶级启动类加载器去处理。只有当父类加载器无法处理时，子类才会尝试自己加载。这读起来就有点**责任链模式**的味道。