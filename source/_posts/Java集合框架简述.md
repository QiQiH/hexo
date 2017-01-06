---
title: Java集合框架简述
date: 2015-12-19 16:16
tags: Java
categories: Java
---
Java集合框架实现了线性表、链表和哈希表这几类数据结构，为我们在程序开发带来了许多便捷。Java集合框架分为两部分：1.集合，用于存数一个元素集合；2.图，用来存储键值对。该文主要对JDK中Collection和Map两个接口中进行简述。
一、Collection接口
Java集合框架中主要支持三种类型的集合：1.Set（规则集） 2.List（线性表） 3.Queue（队列），其层级关系如图：

<!--more-->
附：
Iterator接口
Iterator接口提供了为不同类型集合中的元素进行遍历的统一方法。通过Collection接口中的iterator方法可以返回一个Iterator接口的实例（迭代器）。该实例可以通过hasNext()方法检测迭代器中是否有更多的元素，next()方法顺序访问集合的元素等。

注：java集合框架中的所有具体类中都实现了Cloneable和Serializable接口，因此它们的实例都是可复制且可序列化的。

1、规则集Set
Set接口扩展于Collection接口,在一个实现Set的类中必须确保该规则集没有相同的元素。Set接口中有三个具体类：散列集HashSet、链式散列集LinkedHashSet和树形集TreeSet。
HashSet
HashSet扩展于Set接口，可以用来储存互不相同的元素。当程序向HashSet的实例中添加多个相同的元素时，只有一个元素会被存储，因为规则集中只能存储不同的元素。此外，HashSet实例中存储的元素没有特定的顺序，并不会按照插入顺序进行排序。
主要构造方法：
[java] view plain copy
+HashSet()  //无参构造，创建一个HashSet实例  
+HashSet(Collection<? extends E> c)  //从集合c中创建HashSet实例  
+HashSet(int size) //创建容量为size的HashSet实例  

LinkedHashSet
LinkedHashSet用一个链表实现来扩展HashSet类，因此依旧无法添加重复的元素。在HashSet中，我们无法对其实例的元素进行排序，而当我们需要对元素插入的顺序进行排序时，LinkedHashSet是一个可用的选择。
主要构造方法：
[java] view plain copy
+LinkedHashSet()  //创建一个无参实例  
+LinkedHashSet(Collection<? extends E> c) //从集合c中创建LinkedHashSet实例  
+LinkedHashset(int size) //创建一个容量为size的实例  
TreeSet
TreeSet是SortSet接口中的一个具体子类，其中SortSet为Set的子接口。在LinkedHashSet中，可以通过元素插入的顺序对元素排序，但是有时候需要自定义元素排序的顺序，在TreeSet中，只要对象可比较，即可添加进树形集中，并且可通过以下两种方式进行排序：
1.使用Comparable接口实现。当插入的对象为Comparable实例（如String,Date等）时，就可以通过接口中的compareTo对对象进行排序。此种排序方式为自然顺序。
2.使用比较器接口Comparator实现。有时我们可能需要自定义元素排序的顺序，或者说对象不是Comparable的实例，就可以通过比较器中的compare(object e1, object e2)方法来实现自定义的排序。此种排序方式为比较器顺序。
主要构造方法：
[java] view plain copy
+TreeSet()  //创建一个无参实例  
+TreeSet( Collection<? extends E> c) //从集合c中创建LinkedHashSet实例  
+TreeSet(int size) //创建一个容量为size的实例  
以下为实现比较器顺序的一个案例（以学生信息为例，可通过学号排序，也可通过成绩排序）：
[java] view plain copy
import java.io.Serializable;  
import java.util.Comparator;  
  
public class StudentComparator implements Comparator<Student>, Serializable{ //Java所有集合中都实现了序列化，为了能成功排序，必须实现该接口  
  
    @Override  
    public int compare(Student o1, Student o2) {  
        if (o1.getGrade() > o2.getGrade())  
            return 1;  
        else if (o1.getGrade() == o2.getGrade())  
            return 0;  
        else   
            return -1;    
    }  
      
}  
[java] view plain copy
<pre name="code" class="java">import java.util.Iterator;  
import java.util.TreeSet;  
  
public class TestTreeSet {  
    public static void main(String[] args) {  
        TreeSet<Student> set = new TreeSet<>(new StudentComparator());  
        set.add(new Student(1, 80));  
        set.add(new Student(2, 50));  
        set.add(new Student(3, 78));  
  
        Iterator<Student> iterator = set.iterator();  //使用迭代器遍历元素  
        while (iterator.hasNext()) {  
            Student student = (Student) iterator.next();  
            System.out.println(student.getId() + " " + student.getGrade());  
        }  
    }  
}  
  
/*以下为输出结果 
2 50 
3 78 
1 80*/  

2、线性表List
在程序开发中，往往我们需要向集合中添加相同的元素，线性表实现了添加重复元素的功能，此外，线性表也允许通过下标访问集合中指定位置的元素。线性表是一个有序允许重复的集合。
List接口中方法有：
[java] view plain copy
+add(int index) //指定下标添加元素  
+addAll(int index, Collection<? extends E> c) //指定下标处添加c中所有元素  
+get(int index) //返回指定下标元素  
+lastIndexOf(Object o) //返回相同元素的下标  
+listIterator() //返回遍历列表的迭代器  
+listIterator(int startIndex) //返回从startIndex开始的所有元素的迭代器  
+remove(int index) //删除指定下标的元素  
+set(int index, E element) //设置指定下标的元素  
+subList(int fromIndex, int toIndex) //返回从fromIndex到toIndex元素子列表   
List接口中有两个具体实现类：数组线性表ArrayList和链表LinkedList。
ArrayList
ArrayList使用数组来存储元素，这个数组是动态创建的，当插入的元素超过数组的长度时，就会创建更大的数组，并把当前数组的元素复制到新的数组当中。因此相对于数组来说ArrayList更具有灵活性。
构造方法：
[java] view plain copy
+ArrayList() //创建一个空列表  
+ArrayList(Collection<? extends E> c) //从集合c中创建实例  
+ArrayList(int size) //创建一个大小为size的空列表  
此外，ArrayList中有trimToSize()方法可以将ArrayList的容量缩小到当前列表大小。
在Java2之前引入了向量类Vector，其使用方式与ArrayList类似，但Vector实现了现成同步，以避免多线程访问数据时引起数组损坏。关于线程同步，会在后续文章中提及。
LinkedList
LinkedList是一个双向链表，除了实现List接口的方法还实现了添加和删除表头表尾元素的方法。
构造方法及其他方法：
[java] view plain copy
+LinkedList() //创建一个空链表  
+LinkedList(int size) //创建一个容量为size的空链表  
+addFirst(E e) //添加表头元素  
+addLast(E e) //添加表尾元素  
+getFirst() //返回表头元素  
+getLast() //返回表尾元素  
+removeFirst() //移除表头元素  
+removeLast() //移除表尾元素  

3、队列Queue
Queue，队列，是一种先进先出的数据结构。新增的元素会插在队列的末尾。在优先队列中，优先级高的元素会首先出队。Queue接口中主要有以下方法：

[java] view plain copy
+offer(E e) //添加元素  
+poll() // 返回并删除队头元素，否则返回null  
+remove() //返回并删除队头元素，否则抛出异常  
+peek() // 返回队头元素，否则返回null  
+element() //返回队头元素，否则抛出异常  

Queue中有两个具体实现类：链表LinkedList和优先队列PriorityQueue。

LinkedList
在上述的List接口中也提到过LinkedList，它同时扩展自List接口和Deque接口。双向队列Deque接口扩展自Queue接口，支持在队列的两端在两端插入或删除数据。具体方法可参考上述内容。

PriorityQueue
此类实现了优先队列，在默认情况下，该队列的初始容量为11。其实例所存储的元素默认以自然顺序排列，因此自然顺序下最小的元素会优先出队。队列中可能出现对个优先级相同的元素，那么拥有相同优先级的元素会有其中任意一个优先出队。在讲述TreeSet时提到过使用Comparator接口来实现比较器顺序，在优先队列中依然可行。
构造方法：
[java] view plain copy
+PriorityQueue() //创建一个默认的优先队列  
+PriorityQueue(int size) //创建一个容量为size的优先队列  
+PriorityQueue(Collection<? extends E> c) //从集合c中创建优先队列  
+PriorityQueue(int size, Comparator<? super E>) //创建一个容量为size且拥有比较器顺序的优先队列  


二、Map接口
Java集合框架中支持三种类型的图：散列图HashMap,链式散列图LinkedHashMap和树形图TreeMap。其层级关系与规则集Set类似。
Map----
         |
         |----SortMap----TreeMap
         |
         |----HashMap----LinkedHashMap

图是按照键值存储数据的容器，这一点与List有相似之处，键值类似于List中的下标，不同的是Map的键值可以是任意对象，同时键值不能重复。图中每个键值对应着一个值，键与值一起存储在图中。
Map接口中有如下方法：
[java] view plain copy
+clear(); //删除图中所有条目  
+containsKey(Object key) //图中如果包含指定键值返回true  
+containsValue(Object value) //图中如果包含指定值返回true  
+get(Object key) //获得指定键值对应的值  
+entrySet() //返回包含图中条目的规则集  
+isEmpty() //判断是否空  
+keySet() //返回图中包含键值的一个规则集  
+put(Object key, Object value) //添加键值对  
+putAll( ) //将指定实例中的键值对添加到当前实例中  
+remove(Object key) //删除指定键值对应的值  
+size() //键值对个数  
+values() //返回图中包含的集合  

HashMap
与HashSet相类似，HashMap实例存储的键值对是无序的，不会根据插入的顺序将键值对进行排序，并且存储顺序是随机的。

LinkedHashMap
该类扩展自HashMap，实现了插入的键值对的排序。其排序方式有两种，一种根据插入顺序排序（插入顺序），一种根据最后一次被访问的顺序排序（访问顺序）。通过以下构造方法说明：
[java] view plain copy
+LinkedHashMap(int size, float loadFactor, boolean Order) //创建一个容量为size的图，客座率为loadFactor（即存储的键值对超过该值，会自动增加图的容量，一般为0.75f），true代表访问顺序，false代表插入顺序  

TreeMap
与TreeSet类似，可通过两种方式来实现对图中的键值进行排序。

上述三种具体类都可以通过构造方法从其他图中进行创建。以下为一些
[java] view plain copy
import java.util.HashMap;  
import java.util.LinkedHashMap;  
import java.util.Map;  
import java.util.TreeMap;  
  
public class TestMap {  
    public static void main(String[] args) {  
        Map<String, Integer> hashmap = new HashMap<>();  
        hashmap.put("s1", 10);  
        hashmap.put("s2", 6);  
        hashmap.put("s3", 7);  
        System.out.println("HashMap:");  
        System.out.println(hashmap);  
          
        Map<String, Integer> linkedHashMap = new LinkedHashMap<>(3, 0.75f, true);  
        linkedHashMap.put("s1", 10);  
        linkedHashMap.put("s2", 6);  
        linkedHashMap.put("s3", 7);  
        System.out.println("LinkedHashMap:");  
        System.out.println(linkedHashMap);  
          
        Map<String, Integer> treeMap = new TreeMap<>();  
        treeMap.put("s1", 10);  
        treeMap.put("s2", 6);  
        treeMap.put("s3", 7);  
        System.out.println("TreeMap:");  
        System.out.println(treeMap);  
    }  
      
}  
/*输出结果 
HashMap: 
{s3=7, s1=10, s2=6} 
LinkedHashMap: 
{s1=10, s2=6, s3=7} 
TreeMap: 
{s1=10, s2=6, s3=7}*/  

总结
HashSet与HashMap都不能存储相同的元素。其判断是否存在相同元素通过hashcode()和equals()两个方法来实现，当插入一个新元素或键值时，会先判断集合或图中是否存在hashcode()值相同的元素，伪代码如下：
[java] view plain copy
if (两个元素hashcode值相等) {  
    if (两个元素相等)   
        return true; //不添加  
    else   
        return false; //添加  
}  

由此可知，hashcode相等的两个元素未必相同，但两个相同的元素hashcode必定相同。
在Set接口的实例中如果不需要维护元素插入的顺序，则使用HashSet，因为HashSet更高效率。Map接口中的HashMap也是如此。
在List接口中，ArrayList在尾部提取和插入元素比较高效，LinkedList在任意位置删除和插入元素较高效。
如果在应用程序中不需要添加重复的元素，那么规则集会是最高效的集合。