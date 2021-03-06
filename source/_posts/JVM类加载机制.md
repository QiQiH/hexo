---
title: JVM类加载机制
date: 2016-09-30 20:54
tags: JVM
categories: JVM
---
虚拟机把描述类的数据从class文件加载到内存，并进行数据校验，转化解析和初始化，最终形成被虚拟机直接使用得Java类型，这个过程就是体现了虚拟机的类加载机制。
###**类加载经历了哪些过程？**
类从被加载到虚拟机内存中开始，到被卸载出内存为止，总共经历了一下这些阶段：***加载、验证、准备、解析、初始化、使用、卸载***。其中验证、准备、解析三个部分统称为连接，如下图所示：
![这里写图片描述](http://img.blog.csdn.net/20160930194613661)
<!--more-->
其中，加载、验证、准备、初始化和卸载这5个的顺序是固定的，而解析可以在初始化之后开始，原因在于Java的动态绑定机制。
关于加载这一阶段什么时候执行，虚拟机并没有强制约束。但对于初始化阶段，有四种情况必须立即对类进行初始化：
>1.遇到new、getstatic、putstatic、invokestatic这四条指令码时，必须先初始化。比较常见的场景有，用new来实例化一个对象，或者是读取设置一个类的静态字段（final修饰除外，因为在编译器就把结果放入常量池）。
2.通过反射机制调用类的时候
3.含有main()方法的主类
4.初始化类时，发现有父类，先初始化父类

上述四种情况即对一个类的主动引用，与之对应的为被动引用，被动引用不会触发类的初始化。写几个简单的示例：

```
/**
 * 父类
 */
public class SuperClass {
	public static int value = 1;
	
	static {
		System.out.println("SuperClass created");
	}
}

/**
* 子类
*/
class SubClass extends SuperClass{
	static {
		System.out.println("SubClass created");
	}
}

```

```
public class Client {
	public static void main(String[] args) {
		System.out.println(SubClass.value);
	}
}
```
结果为：

> SuperClass created
1

由于被动引用的存在，子类并没有被初始化，但是却触发了父类的初始化。
另一种情况：

```
//将上述代码的value用final修饰
public final static int value = 1;
```
结果：
>1

会发现父类也没有被初始化，原因在于此时value的值已经存到NoInitlization类的常量池中。对于SubClass.value的引用实际上都转换为NoInitlization类对自身常量池的引用。
关于接口的初始化与类的初始化也有所不同，接口初始化时，并不需要其父接口都实现初始化，只有真正用到父接口时才会初始化。

###**类加载过程分析**
####**1.加载**
加载阶段，虚拟机需要完成三件事：
>1.获取类的二进制字节流
>2.将这个字节流转换为方法区的运行时数据结构
>3.在Java堆中生成一个代表这个类的Class对象，作为方法区数据的访问入口。

加载阶段是开发期可控性最强的阶段，这个阶段可以使用系统提供的类加载器区完成，也可以用户自定义加载器来完成。加载完成后，上述三点就实现了。加载阶段和连接阶段交叉进行，加载阶段尚未完成，连接阶段就有可能已经开始。

####**2.验证**
这一阶段是为了确保Class文件的字节流信息符合虚拟机要求，不会危害虚拟机自身安全，其中验证包含以下四种验证过程：
>1.文件格式验证，检查字节流是否符合Class文件格式规范，并且能被虚拟机处理。
>2.元数据验证，对语意进行分析，如是否继承了final类，是否有父类（除Object类外都应该有父类）等。
>3.字节码验证：该阶段验证的主要工作是进行数据流和控制流分析，以保证被校验的类的方法在运行时不会做出危害虚拟机安全的行为。
>4.符号引用验证：它发生在虚拟机将**符号引用**转化为**直接引用**的时候。可以看作是是对类自身以外的信息（常量池中的各种符号引用）进行匹配性的校验。

####**3.准备**
准备阶段是正式为类分配内存并设置类初始化变量的时候，这些内存都将在方法区分配。这个时候进行内存分配的只是static修饰的变量，而不包括实例变量。
考虑下面这种情况：

```
public static int value = 1;
```
value在准备阶段后初值为0，而不是1。赋值为1则是在初始化阶段才进行的。关于各种数据类型在准备阶段的初始值也可以参考下图：
![这里写图片描述](http://img1.imgtn.bdimg.com/it/u=1994714320,1734740313&fm=21&gp=0.jpg)

####**4.解析**
  解析动作主要针对类或接口、字段、类方法、接口方法四类符号引用进行，分别对应于常量池中的CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info、CONSTANT_InterfaceMethodref_info四种常量类型。
  先了解两个知识点：
  

> 符号引用：用一组符号来描述所引用的目标，符号可以是任意字面量。
> 直接引用：可以是指向目标的指针，相对偏移量或是能够定位到目标的句柄。

解析分一下几个过程：

> 1.类或接口解析：判断所要转化成的直接引用是对数组类型，还是普通的对象类型的引用，从而进行不同的解析。
> 2.字段解析：查找类中是否包含有简单名称和字段描述符都与目标相匹配的字段，如果有，结束查找，否则，继续往父类查找。
> 3.类方法解析：与字段解析类似，需要先解析出方法所属的类或接口的符号引用。
> 4.接口方法解析：解析出接口方法中所属的类或接口的符号引用。否则，向上查找。

（有点难懂T_T）

####**4.初始化**
>参考自 http://blog.csdn.net/ns_code/article/details/17881581

(以下方法都有<>包围，MarkDown语法跪了。。)

初始化是类加载过程的最后一步，到了此阶段，才真正开始执行类中定义的Java程序代码。在准备阶段，类变量已经被赋过一次系统要求的初始值，而在初始化阶段，则是根据程序员通过程序指定的主观计划去初始化类变量和其他资源，或者可以从另一个角度来表达：初始化阶段是执行类构造器<clinit>()方法的过程。
   这里简单说明下client()方法的执行规则:
    1、client()方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句中可以赋值，但是不能访问。
    2、client()方法与实例构造器init()方法（类的构造函数）不同，它不需要显式地调用父类构造器，虚拟机会保证在子类的client()方法执行之前，父类的client()方法已经执行完毕。因此，在虚拟机中第一个被执行的client()方法的类肯定是java.lang.Object。
    3、client()方法对于类或接口来说并不是必须的，如果一个类中没有静态语句块，也没有对类变量的赋值操作，那么编译器可以不为这个类生成client()方法。
    4、接口中不能使用静态语句块，但仍然有类变量（final static）初始化的赋值操作，因此接口与类一样会生成client()方法。但是接口鱼类不同的是：执行接口的client()方法不需要先执行父接口的client()方法，只有当父接口中定义的变量被使用时，父接口才会被初始化。另外，接口的实现类在初始化时也一样不会执行接口的client()方法。
    5、虚拟机会保证一个类的client()方法在多线程环境中被正确地加锁和同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的client()方法，其他线程都需要阻塞等待，直到活动线程执行client()方法完毕。如果在一个类的client()方法中有耗时很长的操作，那就可能造成多个线程阻塞，在实际应用中这种阻塞往往是很隐蔽的。


以上则是类加载过程的概述。