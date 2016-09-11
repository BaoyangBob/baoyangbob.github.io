---
layout: post
title: Android内存泄漏总结
category: 技术
tags: Java
catalog:  true
description: 读书笔记
---





本文将介绍 Android 中内存泄漏的基础知识，分析若干内存泄漏的例子，总结编码中的最佳实践，并介绍相关的内存分析工具。

## Java 内存泄漏

Java 中的自动内存管理机制让我们不需要手动处理内存的分配和回收（参见我的博客[《深入理解Java 虚拟机》读书笔记][1]），我们只要释放不需要的对象的引用，虚拟机就会自动回收该对象。

在 Java 中，内存泄漏就是**存在一些可达但无用的对象**，这些对象有以下两个特点，首先，这些对象是GC Roots可达；其次，这些对象是无用的，即程序以后不会再使用这些对象。如果对象满足这两个条件，那么这些对象不会被GC回收，然而它却占用内存，所以就发生了内存泄漏。
此外**，使用 JNI 的地方也要防止内存泄漏**。在 JNI 代码中， 调用NewGlobalRef创建一个GlobalReference （全局引用，这种对象如不主动释放，就永远不会被垃圾回收）来保存 Java 对象的引用，会使其永远不被回收;或者调用 native 代码来创建对象，要注意 delete/free。这些地方都需要注意释放内存，不然也会发生内存泄漏。

## 常见的内存泄漏场景

内存泄漏的原因一般是以下三点：

- 长期存活的指向 Activity / Context / View / Drawable 等对象的引用，或者 其他一切内部持有所属的 Activity/Context 引用的对象
- 非静态内部类，比如 持有所属 Activity 对象的 Runnable
- 持有对象时间过长的缓存，比如 持有无用元素的容器。


### 容器类型的变量

容器类型的变量，包括Collection接口的实现类和数组。对于内部元素的增、删、
改，我们都要很注意，一不小心就会出现内存泄漏。

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

static 字段的生命周期和应用程序一致，它们所引用的对象也不能提前被释放。如果有个变量mContext 具备 static 的性质,那么给它赋值时就要注意：传入 Application 的 Context，而不能是 Activity 的 Context。
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

当 Activity 因为配置发生改变（如屏幕旋转）或者内存不足被系统杀死，重新创建 Activity时，Fragment会被保存下来，但是会创建新的 FragmentManager，新的 FragmentManager 会首先会去获取保存下来的 Fragment 队列，重建 Fragment 队列，从而恢复之前的状态。直接不管三七二十一的add 新的 Fragment 实例，会消耗很多内存。（Fragment的用法可参见[ Android Fragment 你应该知道的一切，by 张鸿洋][2]）

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


### 覆盖finalize()方法

覆盖 finalize() 方法的类型的对象在 finalize() 方法执行完后才会被垃圾回收。这样的对象被放入FinalizerReference的队列中等待处理。而 finalize() 方法是在低优先级的 finalizer 守护线程中执行，执行的时机和时间都得不到保障。一旦某个对象的 finalize() 方法很耗时，队列也会形成堆积。因而覆盖finalize()方法也可能造成内存泄漏。

在 Java 中也没有太好的办法来提高 finalize() 方法的执行速度，顶多就是遍历所有的线程名，根据名字找到 finalizer 线程， 提高它的优先级。



### 监听器未取消注册

在Android中，用到很多的监听器，如 BroadcastReceiver，ContentObserver，这些监听器必须在实例销毁时取消注册，防止内存泄漏。我们使用时注册，不再需要时要取消注册，一般来说，注册和取消注册的时机应该是对称的生命周期方法，如：

- 在 onCreate() 中 注册，在 onDestroy() 中取消注册;
- 在 onStart() 中 注册，在 onStop() 中取消注册;
- 在 onResume() 中 注册，在 onPause() 中取消注册

不过最好是在 onPause() 中取消注册，因为3.0之后Activity在 onPause() 后随时可能被杀死。

### 资源未关闭

一些资源，如数据库，文件句柄，流Stream，套接字Socket，线程，Cursor，Bitmap 都是程序中会用到的不再需要时又要释放的稀有资源，这些资源如果没有被关闭，就会发生内存泄漏。Bitmap 不用了最好调用 recycle() 方法。



## 最佳实践

1. 集合元素要注意及时清除，不能只增不删。
2. 集合元素的修改不能影响元素的equals方法和hashCode方法。
3. 消息处理要考虑消息未送达的情况，退出时可用Handler.removeMessages来移除消息。
4. 内部类尽量用静态，避免内部类持有外部类的引用，通过WeakReference来持有外部类引用。
5. 静态成员在销毁时置为null。
6. 有注册操作就要有对应的反注册。
7. 稀缺资源在使用完毕后，必须关闭。

## 分析工具

### 抓取 HProf

详细用法参见[官方指南文档里的Capturing a Heap Dump部分][3]。


### MAT 分析

[Memory Analyzer (MAT)下载地址][4]
详细用法参见IBM的文章[使用 Eclipse Memory Analyzer 进行堆转储文件分析][5]

### LeakCanary

是一个用于检测内存泄漏的开源类库。
[项目主页][6]。
详细用法参见[LeakCanary 中文使用说明][7]、[LeakCanary: 让内存泄露无所遁形][8]。

## 相关文章

IBM的[Java 的内存泄漏][9]，介绍了Optimizeit的基本功能和工作原理。
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