---
layout: post
title: 理解 Java 中的四种引用
category: 技术
tags: Java
catalog:  true
description: 引用
---





垃圾回收（Garbage Collection）机制是Java 区别于C/C++ 的一大特点。Java 让我们无须进行显式的内存管理。
Java 中使用的是引用，而不是指针，这一点有助于垃圾回收：

- 系统可以分辨出对象的引用和其他类型的数据的不同。系统可以掌控所有被引用的对象，掌控何时清理、如何清理对象占用的内存，且不会出现 难以分辨一个真正的指针和一个可能是指针而实际是一个数值的东西 这样的情况。
- 我们无法访问引用本身，无法对引用进行算术运算。如果对象在内存中被移动了，这对于我们来说也是透明的，我们的引用仍旧可用。

JVM会持有对象直到它们不再被任何客户端或容器可达。
Java中有四种强度不同的引用，从强到弱依次是，强引用>弱引用>软引用>虚引用。它们均为java.lang.ref.Reference抽象类的实现类，实现不同则垃圾回收的处理策略不同。

如果指向一个实例的所有引用中最强的引用是强/软/弱引用，则称该实例处于强/软/弱引用可达（strongly/softly/weakly reachable）的状态（下文也用该方式描述）。一个对象的引用可达性状态会影响GC回收的表现。

### 强引用(Strong Reference)

举例：

```java
Book book = new Book();
```

强引用是日常使用最多的。上面创建了一个Book对象，并将一个指向这个对象的强引用存到变量book中。只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象，也就是说，**GC不回收强引用可达（strongly reachable）的对象。**如果你不想让你正在使用的对象被回收，就用强引用。

### 软引用([SoftReference][1])

用来描述一些还有用但并非必须的对象。
对于软引用关联的对象，**如果GC后发现内存仍不足，则会把这些对象列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。JVM保证在抛出OutOfMemoryError之前已经回收了所有的软引用指向的软引用可达（softly reachable）的对象。**软引用经常用来实现内存敏感型的场景，如缓存Cache。



### 弱引用([WeakReference][2])

也是用来描述非必需对象的，但是它的强度比软引用更弱一些。被弱引用关联的对象只能生存到下一个垃圾收集之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被软引用关联的对象。
在对象尚未被清除时，软/弱引用的get()方法会返回对象。在对象被标记为垃圾时会返回null。在使用对象前需要检查返回值以防NPE。

```java
WeakReference<Cacheable> weakData = new WeakReference<Cacheable>(data);
if(weakData.get()!=null){
//do what you like
}
```



**弱引用可达（weakly reachable）的对象，下次GC时会被回收。**
弱引用在对象被认为可回收、对象的finalize或GC完成之前插入引用队列。


举例：
假设我们有个书架bookShelf，现在要给书架上的书登记编号，那么可以用以下代码表示：

```java
Map bookShelf = new HashMap();
bookShelf.put(book1,id1);
bookShelf.put(book2,id2);
...
book1=null;
//一组键值对已经无用了，但不会被清理
```

如果图书book1被拿走了，那么我们的书架上不应该有关于这本书的信息了，这个条目应当从map中移除，否则可能导致内存泄漏。在Java中，我们不需要手动移除条目，使用弱引用可以自动回收条目。

[WeakHashMap.Entry][3]

```java
public class WeakHashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V> {

	/**
     * 监控无用Entry的引用队列
     */
    private final ReferenceQueue<Object> queue = new ReferenceQueue<>();
	
	/**
     * 存入键值对
     */
    public V put(K key, V value) {
        Entry<K,V>[] tab = getTable();
		//存入键值对，WeakHashMap内部会创建新Entry，并放入tab数组中
        tab[i] = new Entry<>(k, value, queue, h, e);
        //...
        return value;
    }
	
	 /**
     * 移除无用entry后返回table
     */
    private Entry<K,V>[] getTable() {
        expungeStaleEntries();
        return table;
    }
	
	private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
		Entry(Object key, V value,
			  ReferenceQueue<Object> queue,//引用队列，用于跟踪引用的状态
			  int hash, Entry<K,V> next) {
			//父类Reference构造器中，关联引用队列queue，当key不可用时，将Entry插入队列中
			super(key, queue);
			this.value = value;
			this.hash  = hash;
			this.next  = next;
		}
			
		/**
		 * 移除无用entry
		 */
		private void expungeStaleEntries() {
			for (Object x; (x = queue.poll()) != null; ) {
				synchronized (queue) {
					Entry<K,V> e = (Entry<K,V>) x;
					int i = indexFor(e.hash, table.length);

					Entry<K,V> prev = table[i];
					Entry<K,V> p = prev;
					//遍历Entry单向链表
					while (p != null) {
						Entry<K,V> next = p.next;
						if (p == e) {
						//将链表中无用Entry前后的节点相连接，移除无用Entry
							if (prev == e)
								table[i] = next;//无用Entry为第一个节点时
							else
								prev.next = next;//为后续节点时
							e.value = null; // 清除指向value对象的引用，方便回收
							size--;
							break;
						}
						prev = p;
						p = next;
					}
				}
			}
		}
}
```

WeakHashMap类 与 HashMap类的不同之处就是，WeakHashMap 中的键值对 Entry内部类 继承了 WeakReference，配合引用队列来跟踪key的状态、及时移除无用的entry。
如果key被标记为垃圾了，那么对应的 entry 也会被自动地从Map中移除。
实现这一效果的方法为`expungeStaleEntries()`，当key为null 时，对应的entry 会被加入引用队列中，将entry置为null释放，从Map中移除entry。resize(),getTable(),put(),putAll(),get(),remove()等方法在正常操作前均直接或间接调用了`expungeStaleEntries()`。






### 虚引用 （[PhantomReference][4]）

虚引用的get()方法永远返回null，这是因为虚引用的设计目的不是获取对象实例，而是作为对象已清除（finalized）、GC准备回收内存（reclaim memory）的信号。
虚引用要配合ReferenceQueue使用，虚引用构造时必须传入一个ReferenceQueue对象。

虚引用在对象从内存中完全移除后插入引用队列。通过ReferenceQueue.remove()可以获得有新引用对象插队的通知，灵活地进行最后的清理工作。（不推荐使用终结器finalize()方法来清理，执行时间不可靠、笨重，在终结器中创建一个强引用指向正在终结的对象会使对象复活，相关文章[Finalization and Phantom References][5]、[Dalvik虚拟机 Finalize 方法执行分析][6]）。

举例：
[FileCleaningTracker.Tracker][7]

### 引用队列([ReferenceQueue][8])

引用队列可以很容易地跟踪引用的状态。在构造Reference时传入一个ReferenceQueue对象，当该引用指向的对象被标记为垃圾时，这个引用对象会插入到引用队列里。接下来，你就可以处理传入的引用对象，比如做一些清理工作。


### finalize()方法

要宣告一个对象真正死亡，至少要经历两次标记过程：如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选的条件是此对象是否有必要执行finalize()方法。当对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。
如果这个对象被判定为有必要执行finalize()方法，那么这个对象将会被放置在一个叫做F-Queue的队列中，并在稍后由一个由虚拟机自动建立的、低优先级的Finalizer线程去执行它。这里所谓的“执行”是指虚拟机会触发这个方法，但并不承诺会等待它运行结束，这样做的原因是，如果一个对象在finalize()方法中执行缓慢，或者发生了死循环，将很可能会导致F-Queue队列中的其它对象永久处于等待，甚至导致整个内存回收系统崩溃。finalize()方法是对象逃脱死亡命运的最后一次机会，稍后GC将对F-Queue中的对象进行第二次小规模的标记，如果对象要在finalize()中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，比如把自己（this关键字）复制给某个类变量或者对象的成员变量，那在第二次标记时它将被移除出“即将回收”的集合;如果对象这时候还没有逃脱，那基本上它就真的被回收了。
任何一个对象的finalize()方法都只会被系统自动调用一次，如果对象面临下一次回收，它的finalize()方法不会被再次执行。
避免使用finalize()方法。

### 内存泄漏

Java 程序中的内存泄漏不是因为程序员忘了释放某段内存，而是因为程序员不再需要某个对象时忘记了另一个对象仍持有指向该对象的引用。当我们不再需要某个对象时，如果多处存在指向它的引用，我们就要确保清除所有指向这个对象的引用。如果其中的某个引用仍然存在，那么对象就会存在，占据着原本可以被重用的空间，导致可用空间越来越少。

### 如何使用弱引用

我们假设一个场景:A为一个Activity，我们使用了内部类MyHandler(A)来处理内部逻辑。因为Thread类型的主线程对象持有Looper对象的引用，Looper持有Handler，Handler持有Activity实例的引用，则形成了Thread-Looper-Handler的引用链。如果Thread的生命周期比Activity的更长，退出Activity时就无法回收Activity从而发生内存泄漏。

解决方案：使用static内部类避免持有外部类引用，通过弱引用来获取外部类引用

具体代码如下：

```
//方案1，当static内部类有必须继承的基类时使用
class A {
	private MyHandler mHandler；
	
	void main(){
		Object key = new Object();
		mHandler = new MyHandler(key);
	}

	static class MyHandler extends Handler{
		private WeakReference wr;
		
		MyHandler(Object key){
			wr = new WeakReference(key);
		}
		
		void handle(){
			Object key = wr.get();
			if(key != null) {...}
		}
	}
}
//方案2，当static内部类无必须继承的基类时使用
class A {
	private MyHandler mHandler；
	
	void main(){
		Object key = new Object();
		mHandler = new MyHandler(key);//=> new WeakReference(key),当key为null时，下次GC会回收key所指向的对象
	}

	static class MyHandler extends WeakReference{
		private WeakReference wr;
		
		MyHandler(Object key){
			super(key);
		}
		
		void handle(){
			Object key = get();
			if(key != null) {...}
		}
	}
}
```

在方案2的基础上，如果不仅要回收key，还要回收内部类的实例，那么可以参考[WeakHashMap.Entry][9]关联[ReferenceQueue][10]的做法。

### 推荐阅读

《深入理解Java虚拟机》CH2,CH3

《ThinkingInJava》CH2,CH3

[《深入理解Java 虚拟机》读书笔记][15]

[Java Weak Reference][11]

[Java References: From Strong to Soft to Weak to Phantom][12]

[Java Reference Objects][13]

[Garbage Collection，Chapter 9 of Inside the Java Virtual Machine][14]


  [1]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/lang/ref/SoftReference.java#SoftReference
  [2]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/lang/ref/WeakReference.java#WeakReference
  [3]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/util/WeakHashMap.java#WeakHashMap.Entry
  [4]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b27/java/lang/ref/PhantomReference.java#PhantomReference
  [5]: https://dzone.com/articles/finalization-and-phantom
  [6]: http://blog.csdn.net/kai_gong/article/details/24188803
  [7]: http://grepcode.com/file/repo1.maven.org/maven2/commons-io/commons-io/1.4/org/apache/commons/io/FileCleaningTracker.java#FileCleaningTracker.Tracker
  [8]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b27/java/lang/ref/ReferenceQueue.java#ReferenceQueue
  [9]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/util/WeakHashMap.java#WeakHashMap.Entry
  [10]: http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/lang/ref/ReferenceQueue.java#ReferenceQueue
  [11]: http://javapapers.com/core-java/java-weak-reference/
  [12]: https://www.rallydev.com/blog/engineering/java-references-strong-soft-weak-phantom
  [13]: http://www.kdgregory.com/index.php?page=java.refobj
  [14]: http://www.artima.com/insidejvm/ed2/gcP.html
  [15]: http://coderbao.com/2016/08/15/Notes-of-Understanding-JVM/