---
layout: post
title: 代理模式和Java中的动态代理机制
category: 技术
tags: Java
description: 代理模式 动态代理
---


## 代理模式

### 定义


为实际对象提供一个代理，以控制对实际对象的访问。代理类负责为委托类预处理消息，过滤消息并转发消息，以及进行消息被委托类执行后的后续处理。（by [IBM developerworks][1]）

### 代理模式的UML图
![DelegatePattern.png-18.3kB][2]

代理类和委托类通常会实现相同的接口，以保证两者能处理相同的消息，在访问者看来两者没有丝毫的区别，是透明的。通过代理类这一中间层，能有效控制对委托类对象的直接访问，也可以很好地隐藏和保护委托类对象，为实施不同的控制策略预留了空间，从而在设计上获得了更大的灵活性。

### 静态代理

代理类（如下的ProxySubject类）是在编译时就实现好的。

```java
package com.coderbao.reflection;

public interface Subject {
	void doSomething();
}
```

```java
package com.coderbao.reflection;

public class RealSubject implements Subject {
	public void doSomething() {
		System.out.println("RealSubject doSomething()");
	}
}
```
```
package com.coderbao.reflection;

public class ProxySubject implements Subject{
	
	private Subject subject;

	public ProxySubject(Subject subject) {
		this.subject=subject;
	}
	
	@Override
	public void doSomething() {
		
		System.out.println("PreProcess the message");
		
		this.subject.doSomething();
		
		System.out.println("PostProcess the message");
		
	}
}
```

```java
package com.coderbao.reflection;

public class DynamicProxyDemo {
	public static void main(String[] args) {
		Subject subject = new RealSubject();
		new ProxySubject(subject).doSomething();
	}
}
```

Output:

```
PreProcess the message

RealSubject doSomething()

PostProcess the message

```

### 生活中的代理模式
    一些在线第三方购票网站，可以在上面买票、退票。代售网站卖的机票也是从各大航空公司获取，此时这些网站就是代理，顾客发起买票请求-->网站向航空公司发起请求-->航空公司发出响应，是否购买成功-->代售网站告诉顾客。这中间的过程里，网站向航空公司发起请求时可能会拖延一会儿来赚差价，这就是代理自己对消息的预处理。如果你想退票，航空公司买的票是可以退票的，而第三方网站不支持退票，这就是拦截了你的调用。
    浏览国外网站时翻墙使用的代理。用户的网络请求发给代理-->代理把请求发给服务器-->服务器把响应发给代理-->代理把响应发给用户。
    

## Java中的动态代理机制
    在运行时创建接口的实现（create dynamic implementations of interfaces at runtime）
	
### 相关的API

#### java.lang.reflect.Proxy
- static InvocationHandler getInvocationHandler(Object proxy) ：获得代理对象对应的调用处理器对象
- static Class getProxyClass(ClassLoader loader, Class[] interfaces) ：根据类加载器和接口创建代理类
- static boolean isProxyClass(Class cl) ：动态代理类是否已被创建
- static Object newProxyInstance(ClassLoader loader, Class[] interfaces, InvocationHandler h):有三个参数，1、用来加载动态代理类的类加载器（ClassLoader）。 2、代理类要实现的接口的数组。 3、InvocationHandler实例， 把所有方法的调用都转到代理上，并指定代理的具体做法
#### java.lang.reflect.InvocationHandler
所有对动态代理对象的方法调用都会被转向到 InvocationHandler 接口的实现上

- Object invoke(Object proxy, Method method, Object[] args) throws Throwable
    - 传入 invoke()方法中的 proxy 参数是实现要代理接口的动态代理对象。通常你是不需要他的。
    - invoke()方法中的 Method 对象参数代表了被动态代理的接口中要调用的方法，从这个 method 对象中你可以获取到这个方法名字，方法的参数，参数类型等等信息。
    - Object 数组参数包含了被动态代理的方法需要的方法参数。注意：原生数据类型（如int，long等等）要传入等价的包装对象（如Integer， Long等等）


### 使用方式

```
// Step1、实现InvocationHandler接口创建自定义处理器
InvocationHandler handler = new InvocationHandler() {
	@Override
	public Object invoke(Object proxy, Method method, Object[] args)
			throws Throwable {
		//在调用委托对象的方法之前，可以执行一些预处理
		System.out.println("PreProcess the message");
		
		//调用委托对象的方法
		subject.doSomething();
		subject.doSomething();
		
        //在调用委托对象的方法之后，可以执行一些收尾处理
        System.out.println("PostProcess the message");
		return null;
	}
};

// 创建动态代理类的实例，封装了下节中的Step2、3、4
Subject proxy = (Subject) Proxy.newProxyInstance(
		Subject.class.getClassLoader(), new Class[] { Subject.class },
		handler);
proxy.doSomething();
```

Output:

```
PreProcess the message

RealSubject doSomething()

RealSubject doSomething()

PostProcess the message
```

### 源码实现

#### 动态类的实例生成

Proxy 的重要静态变量

```java
   /** maps a class loader to the proxy class cache for that loader 
   * 映射表：用于维护类加载器对象到其对应的代理类缓存
   */
    private static Map<ClassLoader, Map<List<String>, Object>> loaderToCache
        = new WeakHashMap<>();

    /** marks that a particular proxy class is currently being generated
    * 标记一个动态代理类正在被创建中
    */
    private static Object pendingGenerationMarker = new Object();

    /** set of all generated proxy classes, for isProxyClass implementation
    * 同步表：记录已经被创建的动态代理类类型，供方法 isProxyClass 进行相关的判断
    */
    private static Map<Class<?>, Void> proxyClasses =
        Collections.synchronizedMap(new WeakHashMap<Class<?>, Void>());
        
    /** 所有代理类名字的前缀，均为$Proxy */
    private final static String proxyClassNamePrefix = "$Proxy";
    
    /** 用于每个独特的代理类的名字生成的下一个数字*/
    private static long nextUniqueNumber = 0;
        
    /**
     * 代理类实例关联的调用处理器引用
     * @serial
     */
    protected InvocationHandler h;
```

Proxy的带参构造函数

```
    protected Proxy(InvocationHandler h) {
        doNewInstanceCheck();
        this.h = h;
    }
```

Proxy的静态方法newProxyInstance,生成动态代理类的实例

```java
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        if (h == null) {
            throw new NullPointerException();
        }

        final SecurityManager sm = System.getSecurityManager();
        //对这组接口interfaces进行安全检查,检查接口类对象是否对类加载器可见，接口长度限制，数组中是否均为接口，是否有重复接口等
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, interfaces);
        }

        /*
         * Step2、根据指定类装载器和一组接口，生成代理类的类对象，或从缓存中查找
         */
        Class<?> cl = getProxyClass0(loader, interfaces);

        /*
         * Step3、通过反射获取上文提及的构造函数对象
         */
        try {
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            //Step4、生成代理类的实例
                return newInstance(cons, ih);//即return cons.newInstance(new Object[] {h} )；
            //-------------------------------------------
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString());
        }
    }

```

#### 动态类的代码生成

getProxyClass0(loader, interfaces)方法调用ProxyGenerator的 generateProxyClass方法生成代理类的代码

```java
    private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        //确定代理类的名字，为包名+$ProxyN，N为0，1，2，3，...                                   
        long num;
        synchronized (nextUniqueNumberLock) {
            num = nextUniqueNumber++;
        }
        String proxyName = proxyPkg + proxyClassNamePrefix + num;
		/*
		 * 生成代理类
		 */
		byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
			proxyName, interfaces);
		try {
			proxyClass = defineClass0(loader, proxyName,
				proxyClassFile, 0, proxyClassFile.length);
		} catch (ClassFormatError e) {
			throw new IllegalArgumentException(e.toString());
		}
	｝	
```

sun.misc.ProxyGenerator为我们代劳了写些套路化的代码（OpenJDK源码下载点[这里][3]，导入需要的ProxyGenerator.Java文件）

```java
    public static byte[] generateProxyClass(final String name,Class<?>[] interfaces) {
        return generateProxyClass(name, interfaces, (ACC_PUBLIC | ACC_FINAL | ACC_SUPER));
    }

    public static byte[] generateProxyClass(final String name,
                                            Class<?>[] interfaces,
                                            int accessFlags)  {  
       ProxyGenerator gen = new ProxyGenerator(name, interfaces);  
       //动态生成代理类的字节码  
       final byte[] classFile = gen.generateClassFile();  
  
    // 如果saveGeneratedFiles为true，则会把生成的代理类的字节码保存到硬盘上  
       if (saveGeneratedFiles) {  
           java.security.AccessController.doPrivileged(  
           new java.security.PrivilegedAction<Void>() {  
               public Void run() {  
                   try {  
                       FileOutputStream file =  
                           new FileOutputStream(dotToSlash(name) + ".class");  
                       file.write(classFile);  
                       file.close();  
                       return null;  
                   } catch (IOException e) {  
                       throw new InternalError(  
                           "I/O exception saving generated file: " + e);  
                   }  
               }  
           });  
       }  
  
    // 返回代理类的字节码  
       return classFile;  
   }  
```

#### 获取动态类的字节码

我们导入sun.misc.ProxyGenerator.java后，调用generateProxyClass可以生成动态类的字节码，并将其保存到硬盘上。
这里可以自定义生成类的名字为DynamicProxyType。

```java
package com.coderbao.reflection;

import java.io.FileOutputStream;
import java.io.IOException;

import sun.misc.ProxyGenerator;

public class ProxyGeneratorUtils {
    /** 
     * 把代理类的字节码写到硬盘上 
     * @param path 保存路径 
     */  
    public static void writeProxyClassToDisk(String path) {  
          
        // 获取代理类的字节码  
        byte[] classFile = ProxyGenerator.generateProxyClass("DynamicProxyType", RealSubject.class.getInterfaces());  
          
        FileOutputStream out = null;  
          
        try {  
            out = new FileOutputStream(path);  
            out.write(classFile);  
            out.flush();  
        } catch (Exception e) {  
            e.printStackTrace();  
        } finally {  
            try {  
                out.close();  
            } catch (IOException e) {  
                e.printStackTrace();  
            }  
        }  
    }  
}
```

```java
public class DynamicProxyDemo {
	public static void main(String[] args) {
        //在F盘根目录下生成了Proxy1.class，拖到AndroidStudio中反编译即可
		ProxyGeneratorUtils.writeProxyClassToDisk("F:/DynamicProxyType.class");  
	}	
}
```

#### 生成的动态类的代码

```java
import com.coderbao.reflection.Subject;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class DynamicProxyType extends Proxy implements Subject {
        private static Method m1;
        private static Method m0;
        private static Method m3;
        private static Method m2;

        //构造方法，参数就是一开始实例化的InvocationHandler接口的实例 
        public $Proxy1(InvocationHandler var1) throws  {
                super(var1);
        }

        public final boolean equals(Object var1) throws  {
                try {
                        return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
                } catch (RuntimeException | Error var3) {
                        throw var3;
                } catch (Throwable var4) {
                        throw new UndeclaredThrowableException(var4);
                }
        }

        public final int hashCode() throws  {
                try {
                        return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
                } catch (RuntimeException | Error var2) {
                        throw var2;
                } catch (Throwable var3) {
                        throw new UndeclaredThrowableException(var3);
                }
        }

        public final void doSomething() throws  {
                try {
                        super.h.invoke(this, m3, (Object[])null);
                } catch (RuntimeException | Error var2) {
                        throw var2;
                } catch (Throwable var3) {
                        throw new UndeclaredThrowableException(var3);
                }
        }

        public final String toString() throws  {
                try {
                        return (String)super.h.invoke(this, m2, (Object[])null);
                } catch (RuntimeException | Error var2) {
                        throw var2;
                } catch (Throwable var3) {
                        throw new UndeclaredThrowableException(var3);
                }
        }

        //static代码块加载Method对象
        static {
                try {
                        m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
                        m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
                        m3 = Class.forName("com.coderbao.reflection.Subject").getMethod("doSomething", new Class[0]);
                        m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
                } catch (NoSuchMethodException var2) {
                        throw new NoSuchMethodError(var2.getMessage());
                } catch (ClassNotFoundException var3) {
                        throw new NoClassDefFoundError(var3.getMessage());
                }
        }
}
```

由代码可见，调用动态代理类的实例的方法时，会转发给InvocationHandler实现类的invoke方法处理。InvocationHandler实现类的invoke方法会代理Object类的equals、hashCode、toString方法和指定接口的所有方法。
#### 

### 动态代理的性能消耗和利弊

#### 性能消耗

原因： There is probably some performance cost because of dispatching methods reflectively instead of using the built-in virtual method dispatch（使用反射来分发方法）

[Debunking myths: proxies impact performance ][4]（50倍）

[Java theory and practice: Decorating with dynamic proxies][5]（less than a factor of two）

[Benchmarking the cost of dynamic proxies][6]（factor of 1.63 in raw use）

总结：

- 如果被代理的对象要执行重量级的耗时操作（数据库或文件读写或事务管理），那么动态代理增加的性能消耗可以忽略
- 如果的确需要优化性能，可使用字节码生成工具（a byte code weaving approach），如AspectJ
- 动态代理的操作次数不宜过多

#### 不足

从设计上看动态代理类要继承Proxy类，而Java中没有多继承，所以只能对接口创建动态代理类，不能代理抽象类。此外，还有一些历史遗留的类，它们将因为没有实现任何接口而从此与动态代理永世无缘。

####优点

可以充当接口的Decorator或Proxy，减少书写重复代码

### 实际用途

#### Database Connection and Transaction Management（数据库的连接与事务管理）

Spring 框架中有一个事务代理可以让你提交/回滚一个事务。它的具体原理在 [Advanced Connection and Transaction Demarcation and Propagation][7] 一文中有详细描述，，方法调用序列的大意如下：

```
  web controller --> proxy.execute(...);
  proxy --> connection.setAutoCommit(false);
  proxy --> realAction.execute();
  realAction does database work
  proxy --> connection.commit();
```

#### Dynamic Mock Objects for Unit Testing（单元测试中的动态Mock对象）

[Butterfly Testing工具][8]利用动态代理来实现dynamic stub，mock 和代理类，从而进行单元测试。在测试类A的时候，如果类A用到了类B（B实际上是接口），你可以传一个B 接口的 mock 实现给 A ，来代替实际的 B 接口实现。所有对接口B的方法调用都会被记录，你可以自己设置 B 的 mock 中方法的返回值。 
而且 Butterfly Testing 工具允许你在 B 的 mock 中包装真实的 B 接口实现，这样所有调用 mock 的方法都会被记录，然后把调用转发到真实的 B 接口实现。这样你就可以检查B中方法真实功能的调用情况。例如：你在测试 DAO 时你可以把真实的数据库连接包装到 mock 中。DAO 可以如常地在数据库中读写数据，因为mock 会把所有对数据库的调用都转发给数据库，你可以通过 mock 来检查 DAO 是不是以正确的方式来使用数据库连接，比如是否调用了 `connection.close()`方法,这种情况不能通过DAO 方法的返回值来判断。

####  Adaptation of DI Container to Custom Factory Interfaces（依赖注入容器到自定义工厂接口的适配器）

依赖注入容器 [Butterfly Container][9] 有个强大的特性可以让你把整个容器注入到这个容器生成的 bean 中。但是，如果你不想依赖这个容器接口，这个容器可以按你需要地把自己适配成一个自定义的工厂接口。你只需要写接口，不必实现它。这样这个工厂接口和你的类看起来就像这样：

```java
public interface IMyFactory {
  Bean   bean1();
  Person person();
  ...
}

public class MyAction{

  protected IMyFactory myFactory= null;

  public MyAction(IMyFactory factory){
    this.myFactory = factory;
  }

  public void execute(){
    Bean bean = this.myFactory.bean();
    Person person = this.myFactory.person();
  }

}
```

当 MyAction 类调用通过容器注入到构造方法中的 IMyFactory 实例的方法时，这个方法调用实际先调用了 IContainer.instance()方法，这个方法可以让你从容器中获取实例。这样这个对象可以把 Butterfly Container 容器在运行期当成一个工厂使用，比起在创建这个类的时候进行注入，这种方式显然更好。而且这种方法没有依赖到 Butterfly Container 中的任何接口。

####  AOP-like Method Interception（AOP中的方法拦截）

如果某个bean实现了某些接口，那么Spring 框架就能拦截对bean 的方法调用。Spring 框架使用动态代理来包装 bean，所有对 bean 中方法的调用都会被代理拦截。代理可以决定是否要调用其他对象的方法来做预处理/拦截/后续处理。

####  Retrofit中自定义的网络请求到OkHttp.Call的适配

使用`GitHub github = retrofit.create(GitHub.class);`时；我们就创建了一个我们自定义的接口GitHub的动态代理类，当我们发出消息时（调用github.contributors().execute()时），代理类会按照Retrofit先前配置的逻辑来处理我们发出的消息：比如交由okhttp3.Call来进行网络请求。Retrofit完成的是封装客户网络请求的高效工作，而真正的网络请求的工作是委托给了OkHttp来完成。

[Sample][10]

```java
package com.example.retrofit;

import java.io.IOException;
import java.util.List;
import retrofit2.Call;
import retrofit2.converter.gson.GsonConverterFactory;
import retrofit2.Retrofit;
import retrofit2.http.GET;
import retrofit2.http.Path;

public final class SimpleService {
  public static final String API_URL = "https://api.github.com";

  public static class Contributor {
    public final String login;
    public final int contributions;

    public Contributor(String login, int contributions) {
      this.login = login;
      this.contributions = contributions;
    }
  }
  
  public interface GitHub {
    @GET("/repos/{owner}/{repo}/contributors")
    Call<List<Contributor>> contributors(
        @Path("owner") String owner,
        @Path("repo") String repo);
  }

  public static void main(String... args) throws IOException {
    // 创建一个指向GitHub API接口的REST适配器
    Retrofit retrofit = new Retrofit.Builder()
        .baseUrl(API_URL)
        .addConverterFactory(GsonConverterFactory.create())
        .build();

    // 创建GitHub API接口的实例
    GitHub github = retrofit.create(GitHub.class);

    // 获取contributors，返回一个Call<List<Contributor>>对象，根据注解和方法参数配置好了url和请求参数
    Call<List<Contributor>> call = github.contributors("square", "retrofit");

    // execute()发起真正的请求，同步
    List<Contributor> contributors = call.execute().body();
    for (Contributor contributor : contributors) {
      System.out.println(contributor.login + " (" + contributor.contributions + ")");
    }
  }
}
```

Retrofit.create

```java
public final class Retrofit {
      public <T> T create(final Class<T> service) {
        return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
            new InvocationHandler() {
              private final Platform platform = Platform.get();
    
              @Override public Object invoke(Object proxy, Method method, Object... args)
                  throws Throwable {
                // method来自Object.class则不做自定义处理
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                //默认处理，即根据平台类型（Android、Java8、IOS）做处理
                if (platform.isDefaultMethod(method)) {
                  return platform.invokeDefaultMethod(method, service, proxy, args);
                }
                ServiceMethod serviceMethod = loadServiceMethod(method);
                OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
                return serviceMethod.callAdapter.adapt(okHttpCall);
              }
            });
      }
}
```

## References

[Java Reflection - Dynamic Proxies][11]（一个不错的的Java知识点学习网站）

[Java 动态代理机制分析及扩展，第 1 部分][12]（IBM啊）

[Java动态代理机制详解（JDK 和CGLIB，Javassist，ASM）][13]

[retrofit][14]


  [1]: http://www.ibm.com/developerworks/cn/java/j-lo-proxy1/
  [2]: http://static.zybuluo.com/Jacoburg/xd2t2dqf6wnn53ifo0fipu0v/DelegatePattern.png
  [3]: http://download.java.net/openjdk/jdk8/
  [4]: http://blog.springsource.com/2007/07/19/debunking-myths-proxies-impact-performance/
  [5]: http://www.ibm.com/developerworks/java/library/j-jtp08305.html
  [6]: http://ordinaryjava.blogspot.com/2008/08/benchmarking-cost-of-dynamic-proxiescom/developerworks/java/library/j-jtp08305.html
  [7]: http://tutorials.jenkov.com/java-persistence/advanced-connection-and-transaction-demarcation-and-propagation.html
  [8]: https://github.com/jjenkov/butterfly-testing-tools
  [9]: https://github.com/jjenkov/butterfly-di-container
  [10]: https://github.com/square/retrofit/tree/master/samples
  [11]: http://tutorials.jenkov.com/java-reflection/dynamic-proxies.html
  [12]: http://www.ibm.com/developerworks/cn/java/j-lo-proxy1/
  [13]: http://blog.csdn.net/luanlouis/article/details/24589193
  [14]: https://github.com/square/retrofit