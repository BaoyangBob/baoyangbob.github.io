---
layout: post
title: Android内存泄漏总结
category: 技术
tags: Java
catalog:  true
description: 读书笔记
---





TL;DR：本文将介绍 Android 中内存泄漏的原理，分析若干内存泄漏的例子，总结编码中的最佳实践，并介绍相关的内存分析工具。

## Java 内存泄漏

Java 中的自动内存管理机制让我们不需要手动处理内存的分配和回收（参见我的博文[《深入理解Java 虚拟机》读书笔记][1]），我们只要释放不需要的对象的引用，虚拟机就会自动回收该对象。

在  Java 中，内存泄漏就是**存在一些可达但无用的对象**，这些对象有以下两个特点，首先，这些对象是 GC Roots 可达；其次，这些对象是无用的，即程序以后不会再使用这些对象。如果对象满足这两个条件，那么这些对象不会被GC回收，然而它却占用内存，所以就发生了内存泄漏。
此外**，使用 JNI 、native 代码的地方也要防止内存泄漏**。在 JNI 代码中， 调用 NewGlobalRef 创建一个 GlobalReference （全局引用，这种对象如不主动释放，就永远不会被垃圾回收）来保存 Java 对象的引用，会使其永远不被回收;或者调用 native 代码来创建对象，要注意 delete/free。这些地方都需要注意释放内存，不然也会发生内存泄漏。

如果我们的程序出现了 OutOfMemoryException，那就说明有了内存泄漏。

## 常见的内存泄漏场景

内存泄漏的原因一般是以下三点：

- 长期存活的指向 Activity / Context / View / Drawable 等对象的引用，或者 其他一切内部持有所属的 Activity/Context 引用的对象
- 非静态内部类，比如 持有所属 Activity 对象的 Runnable
- 持有对象时间过长的缓存，比如 持有无用元素的容器。


### 容器类型的变量

容器类型的变量，包括Collection接口的实现类和数组。对于内部元素的增、删、改，我们都要很注意，一不小心就会出现内存泄漏。

#### 忘记删去无用成员

容器持有内部元素的引用，如果容器自己还有效，则它们包含的元素也有效，这使得它们的元素不能被回收。
例：

```java
List list = new ArrayList();
for (int i = 1; i < 100; i++) {
    Object o = new Object();
    list.add(o);//添加元素
    o = null;   //清除指向元素的引用，但集合仍持有元素的引用
}
```


#### 修改元素属性后，导致调用remove()方法无效

如果容器的remove()方法依赖于元素的某些属性（如哈希值），而这些属性又被修改过了，导致remove()方法无法找到该元素，该方法失效，导致存在了我们不想要的元素，从而发生内存泄漏。

例： 

```java
Set<Person> set = new HashSet<Person>(); 
Person p1 = new Person("Allen",25); 
Person p2 = new Person("Bill",26); 
Person p3 = new Person("Mark",27); 
set.add(p1); 
set.add(p2); 
set.add(p3); 
System.out.println("size is "+set.size()); //output：size is 3 
p3.setAge(2); //修改p3的年龄,此时p3元素对应的hashcode值发生改变 
set.remove(p3); //remove()的原理是根据哈希值来找到并删除内部的HashMap中的元素，此时哈希值已改变，remove不掉，造成内存泄漏
set.add(p3); //重新添加，居然添加成功 
System.out.println("size is "+set.size()); //output：size is 4 
for (Person person : set) { 
    System.out.println(person); 
} 
```


### static 字段、单例对象

static 字段的生命周期和应用程序一致，它们所引用的对象也不能提前被释放。如果有个变量 mContext 具备 static 的性质,那么给它赋值时就要注意：传入 Application 的 Context，而不能是 Activity 的 Context。
如果传入的是 Activity 的 Context，当对应的 Activity 退出时，由于static 字段持有该 Context 的引用，所以 Context 的生命周期等于整个应用程序的生命周期，所以当前 Activity 退出后也无法释放。而 Application 的生命周期就是整个应用的生命周期，所以传入 Application 的 Context 没有任何问题。

单例对象通常也是存在于整个程序的生命周期中，如果单例对象持有外部对象的引用，那么这个外部对象也只能在应用程序结束时才能被 JVM 正常回收。

### 匿名内部类/非静态内部类和线程

在 Activity、Fragment、View 中，如果使用了匿名内部类/非静态内部类，且被某个线程持有了这个内部类的对象的引用，那么在退出Activity、Fragment、View 时，该线程可能还在执行任务中，此时 Activity、Fragment、View 就会被泄漏了。注意，这里的线程不仅仅可能是worker thread，也可能是主线程（在“Handler 造成的内存泄漏”里提及）。

#### Handler 造成的内存泄漏

Handler 造成的内存泄漏也是“匿名内部类/非静态内部类和线程”中的一种。

Handler、Message 和 MessageQueue 、Looper 都是相关联的，而 Looper 是 TLS(Thread Local Storage) 变量，它会间接持有到 Handler 的引用，所以 Handler 也被线程持有了。

如果用 Handler 来发送延时消息到某个线程中，Handler 发送的 Message 尚未被处理，则该 Message 及发送它的 Handler 对象将被线程的 MessageQueue 一直持有，而 Handler 持有了 Activity、Fragment、View 等外部类的引用，所以外部类就算退出了也会短期内无法释放，就发生内存泄漏了。
例：

```java
public class HandlerTestActivity extends BaseActivity {
    private byte[] bigArray;
    private MyHandler handler = new MyHandler(this);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        //...
        bigArray = new byte[1024 * 1024 * 2];
        handler.sendEmptyMessageDelayed(1, 30000);
    }

    class MyHandler extends Handler {
        private final HandlerTestActivity activity;

        MyHandler(HandlerTestActivity activity) {
            super();
            this.activity = activity;
        }
    }
}
```

### 重复添加Fragment

当 Activity 因为配置发生改变（如屏幕旋转）或者内存不足被系统杀死，重新创建 Activity 时，Fragment 会被保存下来，但是会创建新的 FragmentManager，新的 FragmentManager 会首先会去获取保存下来的 Fragment 队列，重建 Fragment 队列，从而恢复之前的状态。直接不管三七二十一的add 新的 Fragment 实例，会消耗很多内存。（Fragment 的用法可参见[ Android Fragment 你应该知道的一切，by 张鸿洋][2]）

```java
public class MainActivity extends BaseActivity {  
      
    private ContentFragment mContentFragment;   
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);
        mContentFragment = getSupportFragmentManager(). 
                   findFragmentById(R.id.fragment_container);
        if ( mContentFragment == null ) { //错误做法是这里不判断null
        mContentFragment = new MainMenuFragment(); 
            getSupportFragmentManager() 
            .beginTransaction() 
            .add(R.id.fragment_container, mContentFragment, "SOME_TAG_IF_YOU_WANT_TO_REFERENCE_YOUR_FRAGMENT_LATER")
            .commit(); 
    } 
}
```


### 覆盖 finalize() 方法

覆盖 finalize() 方法的类型的对象在 finalize() 方法执行完后才会被垃圾回收。这样的对象被放入FinalizerReference的队列中等待处理。而 finalize() 方法是在低优先级的 finalizer 守护线程中执行，执行的时机和时间都得不到保障。一旦某个对象的 finalize() 方法很耗时，队列也会形成堆积。因而覆盖finalize()方法也可能造成内存泄漏。

在 Java 中也没有太好的办法来提高 finalize() 方法的执行速度，顶多就是遍历所有的线程名，根据名字找到 finalizer 线程， 提高它的优先级。

《Effective Java》第7条：避免使用终结方法。



### 监听器未取消注册

在Android中，用到很多的监听器，如 BroadcastReceiver，ContentObserver，这些监听器必须在实例销毁时取消注册，防止内存泄漏。我们使用时注册，不再需要时要取消注册，一般来说，注册和取消注册的时机应该是对称的生命周期方法，如：

- 在 onCreate() 中 注册，在 onDestroy() 中取消注册;
- 在 onStart() 中 注册，在 onStop() 中取消注册;
- 在 onResume() 中 注册，在 onPause() 中取消注册

不过最好是在 onPause() 中取消注册，因为3.0之后Activity在 onPause() 后随时可能被杀死。

《Effective Java》第6条：

> 如果你实现了一个API，客户端在这个 API 中注册回调，却没有显式地取消注册，那么除非你采取某些动作，否则它们就会积聚。确保回调立即被当做垃圾回收的最佳方法是只保存它们的*弱引用*（weak reference），例如，只将它们保存成 WeakHashMap 中的键。	

### 资源未关闭

一些资源，如数据库，文件句柄，流Stream，套接字Socket，线程，Cursor，Bitmap 都是程序中会用到的不再需要时又要释放的稀有资源，这些资源如果没有被关闭，就会发生内存泄漏。Bitmap 不用了最好调用 recycle() 方法。



## 最佳实践

1. 集合元素要注意及时清除，不能只增不删。
2. 集合元素的修改不能影响元素的equals方法和hashCode方法。
3. 使类的可变性最小化。
4. 消息处理要考虑消息未送达的情况，退出时可用 Handler.removeMessages() 来移除消息。
5. 内部类尽量用静态，避免内部类持有外部类的引用，通过 WeakReference 来持有外部类引用。
6. 静态成员在销毁时置为null。
7. 有注册操作就要有对应的反注册,用 WeakHashMap 来保存注册的回调。
8. 稀缺资源在使用完毕后，必须关闭。

## 分析工具

### 抓取  hprof 文件

hprof 即 heap profiler 堆资料的缩写。

如果使用的是 Android Studio，可以参见[官方指南文档里的Capturing a Heap Dump部分][3]来抓取，AS 里已经集成了抓取工具。

如果不用 AS ，只要有 adb 工具，用以下命令就可以在手机里抓取：

```shell
adb shell	//进入adb shell
am dumpheap com.coderbao.app /sdcard/app.hprof   //抓取包名为com.coderbao.app的应用的hprof文件到某目录
exit	//退出adb shell
adb pull /sdcard/app.hprof d:/heap-original.hprof //将hprof文件从手机中提取到电脑中
hprof-conv heap-original.hprof heap-converted.hprof   //原hprof文件要转换一下才可以打开
```

注意，hprof文件要使用 AndroidSDK/platform 中的hprof-conv工具，转换成可用格式。




### MAT 分析

对于上一步中抓取的 hprof 文件，我们可以用 Eclipse Memory Analyzer 来分析，它会生成清晰的内存占用报表、对象的引用路径。

[Memory Analyzer (MAT)下载地址][4]

分析步骤：IBM的文章详细地介绍了如何[使用 Eclipse Memory Analyzer 进行堆转储文件分析][5]。

### LeakCanary

是 Android 中的一个用于检测内存泄漏的开源类库。
这是[项目主页][6]。
详细用法参见[LeakCanary 中文使用说明][7]、[LeakCanary: 让内存泄露无所遁形][8]。

## 相关文章

IBM的[Java 的内存泄漏][9]，介绍了另一个分析工具 Optimizeit 的基本功能和工作原理。
[Android 内存泄漏总结][10]
[Dalvik虚拟机 Finalize 方法执行分析][11]


[1]: http://coderbao.com/2016/08/15/Notes-of-Understanding-JVM/
[2]: http://blog.csdn.net/lmj623565791/article/details/42628537
[3]: https://developer.android.com/studio/profile/investigate-ram.html#HeapDump
[4]: http://www.eclipse.org/mat/
[5]: http://www.ibm.com/developerworks/cn/opensource/os-cn-ecl-ma/index.html
[6]: https://github.com/square/leakcanary
[7]: http://www.liaohuqiu.net/cn/posts/leak-canary-read-me/
[8]: http://www.liaohuqiu.net/cn/posts/leak-canary/
[9]: http://www.ibm.com/developerworks/cn/java/l-JavaMemoryLeak/
[10]: https://yq.aliyun.com/articles/3009
[11]: http://blog.csdn.net/kai_gong/article/details/24188803