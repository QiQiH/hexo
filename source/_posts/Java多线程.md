---
title: Java多线程
date: 2016-01-21 17:45
tags: Java
categories: Java
---
前言：线程是java的重要功能之一，多线程能使程序反应更快，交互性更强，执行效率更高。
## 一、任务和线程的创建 ##
任务就是对象。为任务创建线程可以简单分为以下两种：1.继承Thread类 2.实现Runnable接口。
示例程序片段：
1.继承Thread类

```
class Task extends Thread{
	@Override
	public void run() {
		super.run();
	}
}

public class Main {
	public static void main(String[] args) {
		Task task = new Task(); //创建任务
		task.start();    //告诉java虚拟机该线程准备运行，然后java虚拟机通过调用run()方法执行任务
	}
}
```
2.实现Runnable接口

```
class Task implements Runnable{
	@Override
	public void run() {
	}
}
public class Main {
	public static void main(String[] args) {
		Task task = new Task(); //创建任务
		Thread thread = new Thread(task);
		thread.start();    //告诉java虚拟机该线程准备运行，然后java虚拟机通过调用run()方法执行任务
	}
}
```
推荐使用方法2，原因如下：
1.由于java不能实现多继承而可以实现多个接口，如果一个类继承了其他类，就导致无法继续继承Thread类。
2.使用1方法会是任务和运行任务混在一起。

附：
1.Thread类
Thread类除了上述为任务创建线程之外，还有许多控制线程的方法。Thread类同时也实现了Runnable接口。

```
+isAlive() : boolean  //测试当前任务是否正在运行
+setPriority(p :int) : void  //设置线程优先级（1-10）   
+join() : void  //使当前等待线程结束
+sleep(ms : long) //线程休眠时长
+yield() : void  //使线程暂停并允许执行其他线程
+interrupt() : void //中断线程
```
2.线程状态
![](http://img.blog.csdn.net/20160121174258378)
该图片转自http://blog.csdn.net/huang_xw/article/details/7316354
<!--more-->
## 二、线程池 ##
当一个程序有大量任务且需要为每个任务都创建一个新线程时，创建线程池会是个好方法。以下为两个重要接口。
Executor接口：执行线程池中的任务。
ExecutorService接口：管理和控制任务，是Executor的子接口。
有以下主要方法：

```
+execute(Runnable r) : void  //执行任务
+shutdown() : void //关闭执行器，不再接收新任务，但会完成当前未完成的任务。
+shutdownNow() : List<Runable> //关闭执行器，不再接收新任务，当前未完成的任务停止工作并返回未完成的任务表。
+isShutdown() : boolean //执行器已关闭返回ture
+isTerminated() : boolean //如果线程池中的任务都终止，返true
```
为了创建执行器，即Executor对象，需要使用Executors类中的静态方法

```
+newFixedThreadPool(numOfThread : int) : ExecutorService //创建一个固定线程数目的线程池，如果任务完成，则会重新使用以执行另一线程。
+newCachedThreadPool() : ExecutorService //为每个等待的任务创建线程。
```
线程池示例：

```
public class Main {
	public static void main(String[] args) {
		ExecutorService executor = Executors.newFixedThreadPool(3); //创建固定线程数目的线程池
		//ExecutorService executor = Executors.newCachedThreadPool();//该方法按需创建线程数目
		executor.execute(new Task()); //执行任务
		executor.execute(new Task());
		executor.execute(new Task());
		executor.shutdown(); //关闭执行器，不再接收任务
	}
}
```
## 三、线程同步 ##
当多个线程同时访问同一资源时，可能会导致数据破坏。如下示例：

```
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
/*此程序为该类的num添加数值，创建100个线程，每个线程为num加1*/
class Number { 
	private int num = 0;
	
	public void addNum(int n) {
		try {
			int newNum = num + n;
			Thread.sleep(5);
			num = newNum;
		} catch (InterruptedException e) {
			e.printStackTrace();
		} 
	}
	
	public int getNum() {
		return num;
	}
}

public class Main {
	public static Number number = new Number();
	
	public static void main(String[] args) {
		ExecutorService executor = Executors.newCachedThreadPool();
		for (int i = 1; i <= 100; i++) {
			executor.execute(new AddNum()); //创建100个线程，每个线程为num加1
		}
		executor.shutdown();
		
		while (!executor.isTerminated()) { 
		}
		//任务完成
		System.out.println(number.getNum());
	}
	
	
	static class AddNum implements Runnable{
		@Override
		public void run() {
			number.addNum(1);
			
		}
	}
}
```
该程序输出的最终值不确定（或2或3或其他），并非100。该程序因为多个线程访问同一资源而使数据破坏。当线程1执行到int newNum = num + n但还未执行num = newNum时，线程2可能执行到nt newNum = num + n，导致此处的num仍然是之前的num，并没有发生改变，从而导致数据发生错误。这中问题成为竞争状态，存在线程安全问题。 
解决方案：线程同步
1.使用synchronized关键字（隐式加锁）

```
public synchronized void addNum(int n) { //被同步的代码块称为同步块
	//...
}
```
或者

```
public void addNum(int n) {
		synchronized(this) {
			try {
				int newNum = num + n;
				Thread.sleep(5);
				num = newNum;
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		} 
	}
```

当线程1进入同步块时，会对number对象进行加锁，线程执行完成后释放锁。当线程1的number对象加锁时，其他线程的对象就只能等待线程1释放锁才能对number对象加锁。通过此种方式有效地避免了竞争状态，实现了线程安全。

2.利用加锁同步（显式加锁）
Lock接口：定义了可以加锁和释放锁的方法。

```
+lock() : void //加锁
+unlock() : void //释放锁
```
可以通过创建ReentrantLock类的对象来创建锁。

```
+ReentrantLock() //与ReentrantLock(false)等价
+ReentrantLock(b : boolean) //创建一个具有公平策略的锁,true则等待时间最长的任务获得锁，false则线程执行没有特定顺序
```
加锁示例：

```
class Number { 
	private int num = 0;
	public void addNum(int n) {
		lock.lock();	//加锁
		try {
			int newNum = num + n;
			Thread.sleep(5);
			num = newNum;
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock(); //释放锁
		}	
	}
	public int getNum() {
		return num;
	}
}

```
附：线程同步之单例模式
在程序设计中，往往一个类只能有一个对象，会采用单例模式。示例如下：

```
public class Model {
	private static Model model = null;
	private Model() { //外界不能实例化
	}
	
	public static Model getInstance() { //通过此获得对象，只能有一个对象
		if (model == null) {
			model = new Model();
		} 
		return model;
	}
}
```
在多线程中，如果每个线程都要对Model的对象进行操作，就会出现数据错误。假设线程1执行到if (model == null)但却未执行到model = new Model()时线程2也进入该方法，此时model对象仍然为空，会进入if的语句里，当执行完毕后就产生了两个不同model对象，导致线程安全问题。因此，在设计单例模式时同时要考虑线程安全问题。可以通过同步加锁来解决。例如：

```
public static synchronized Model getInstance(){
	//...
}
```
但是在方法头使用synchronized来锁住对象会明显占用资源，因为每当程序调用该方法时都需要给对象加锁，可以如下改进：

```
public static Model getInstance() {
		if (model == null) {
			synchronized(model) {
				if (model == null)
					model = new Model();
			}
		} 
		return model;
	}
```

## 五、线程死锁 ##
当两个或多个线程需要在几个共享资源上获得锁时，就可能产生死锁。如下示例：

```
/*线程1*/
synchronized (object o1) {
	//...
	synchronized （object o2）{ //等待线程2释放object2的锁
	//...
	}
}
```

```
/*线程2*/
synchronized (object o2) {
	//...
	synchronized （object o1）{ //等待线程1释放object1的锁
	//...
	}
}
```
上述情况会导致死锁，两个程序都无法继续运行。因此在设计程序中应确保每个线程都能获得对应的锁，避免死锁。