---
layout: post
title: Dagger 2 思路学习
category: 技术
date:   2017-03-16 22:22:11
catalog:  true
tags:
    - Dagger2
description: Dagger2
---



## 引言

 本文是在熟悉 Dagger 2 用法的基础上，解读 Dagger 2 的生成代码，根据代码来说明官方文档中的有些约定、概念的起源，暂不研究如何生成代码的 compiler 部分。

### 框架与注解

现在有很多框架均借助于注解和注解处理工具，如 大名鼎鼎的 Butterknife、Dagger 2、Retrofit ，或者是在 Java 代码之外补充完善元数据，或者是把注解当作一种标记来生成满足特定需求的模式化代码。

Dagger 2 的主要特点是：使用了一系列如`@Module`、`@Component` 、`@Provides`等注解，表明程序中的对象依赖关系，代替我们手写、生成能表示这种关系的代码，使得我们对依赖注入关系的校验可以从运行时前移到编译期。

### Dagger 2 中的几个概念

- Binding

Dagger2 中把依赖类型为 binding，比如下文中的 D1、D2 类型

- Module

为对象依赖图提供所依赖的 binding 类型的实例的类。

可以认为 Module 类的角色是上游的依赖生产工厂，不直接对需求依赖的下游用户代码开放。

在代码中体现为用`@Module`注解 的类，`@Module`注解有两个注解元素：`includes` 和 `subcomponents` ，注解元素 `includes`后面跟组成这个 Module 类的其他 Module 类的 Class 值，注解元素 `subcomponents` 后面跟这个 Module 类将被安装到的 component 的子 component的 Class 值。

举个例子，下面的 ModuleA  提供了类型分别为D1、D2的两种类型的实例 d1、d2：

```java
@Module
public class ModuleA {
    public ModuleA() {}

    @Provides
    D1 provideD1(){
        return d1;
    }

    @Provides
    D2 provideD2() {
        return d2;
    }
}
```



- Component

如果说 Module 是依赖的工厂，Component 就像是代理经销商，Module 中提供的依赖要想提供给下游用户使用，一是必须在 Component 这里展示“货物”（即写出 返回值类型为依赖类型的 provision方法），二是必须在下游的用户代码中引入 Component 实例并由 Component 注入依赖（即调用 Component 的 Members-injection methods方法）。

举个例子，下面的 ComponentA 接口 把 ModuleA 中提供的 D1类型的依赖提供给用户c，D2类型则未提供：

```java
@Component(dependencies = {ComponentB.class}, modules = ModuleA.class)
public interface ComponentA {

    D1 provideD1();	//a provision method
  	void inject(Client c);	//a members-injection method for injecting dependencies to client c
}
```

 写好后 build 一下，Dagger2 就会在 gen 目录下自动生成 Dagger前缀+该 Component 接口名所组成的实现类 DaggerComponentA。



- `@Inject`注解

用在两种地方：

1. 依赖源：依赖类型没有使用 provideXxx 的工厂方法来提供依赖，而是通过构造函数提供，那就要在构造函数上加该注解
2. 依赖注入点：也就是加在客户代码中的一些需要DI 框架来注入依赖实例的 field上。





## 生成代码的解读

### Module 和 Component 的简单实现

假设我们需要一个最简单的依赖关系， ModuleA 和 ComponentA 提供依赖给用户代码，那么我们会写出这样的代码：

```java
//依赖类型 D1
public class D1 {}
```

提供 D1类型的 ModuleA.java:

```java
@Module
public class ModuleA {
    public ModuleA() {}

    @Provides
    D1 provideD1(){
        return d1;
    }
}
```

向用户暴露 D1 类型的 ComponentA.java

```java
@Component(modules = ModuleA.class)
public interface ComponentA {

    D1 provideD1();	//a provision method，Component 中的两种方法之一
  	void inject(Client c);	//a members-injection method for injecting dependencies to client c，，Component 中的两种方法之二
}
```

需求 D1 类型对象的用户代码 ClientA.java：

```java
class Client {
 	@Inject//表示这个预变量需要 DI 框架来注入实例
	D1 d;
  
	void methodHook(){
		//这个方法可以是个 hook method，好比 Java 中的 main 方法 或 Android 中的 Activity.onCreate方法
		DaggerComponentA componentImplA = DaggerComponentA.moduleA(new ModuleA()).build();
		componentImplA.inject(this);
      	d.toString();//现在 d 就已经被注入了 ModuleA 中提供的实例了，不是 null 了。
}
```

Dagger 2 在幕后做了哪些工作，使得我们可以轻松地完成依赖注入呢？

在开始使用 Dagger 时，我们在 app 的 build.gradle 文件中添加了如下语句(作用见注释)

```groovy
 apply plugin: 'com.neenbedankt.android-apt' //gradle 插件，辅助那些用到注解处理器的框架，作用有两点：1，框架分成了 compiler 部分和 api 部分，compiler 部分的代码不想加入最终的 app 中，这个插件提供了第6、8行的 apt 功能；2，框架用 annotation processor 生成了我们项目中要用到的源代码，不过这些代码通常在 gen 目录下，IDE 不会自动识别这个目录，会报错，所以这个插件会配置源码路径，使得 IDE 能识别它们
 ...
 dependencies {
    ...
    compile 'com.google.dagger:dagger:2.6'//compile 的包要加入最终 app 中
    apt 'com.google.dagger:dagger-compiler:2.6'//Dagger 中处理注解、生成代码的部分，这个依赖中的代码不加入到最终的 apk 中
    //compile 'com.jakewharton:butterknife:8.5.1' //butterknife 的做法也是类似的
    //apt 'com.jakewharton:butterknife-compiler:8.5.1'
 }
```

接着我们编译后，会生成如下代码（删去无关代码，流程简化，后续代码也如此）：

 ModuleA 对应于 ModuleA_ProvideD1Factory:

```java
public final class ModuleA_ProvideD1Factory implements Factory<D1> {//Factory接口继承了 Provider 接口
  private final ModuleA module;

  public ModuleA_ProvideD1Factory(ModuleA module) {
    this.module = module;
  }

  @Override
  public D1 get() {//建立了Factory=ModuleA_ProvideD1Factory->D1的引用关系
    return module.provideD1();
  }

  public static Factory<D1> create(ModuleA module) {//采用 static factory 方法 来实例化此类
    return new ModuleA_ProvideD1Factory(module);
  }
}
```

 ComponentA 对应于 DaggerComponentA:

```java
public final class DaggerComponentA implements ComponentA {
  private Provider<D1> provideD1Provider;
  
  private MembersInjector<Client1> client1MembersInjector;

  private DaggerComponentA(Builder builder) {
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  public static ComponentA create() {
    return builder().build();
  }

  private void initialize(final Builder builder) {
    this.provideD1Provider = ModuleA_ProvideD1Factory.create(builder.moduleA);//建立了DaggerComponentA 实例 到 Provider 实例 再到 moduleA 实例的引用关系
    this.client1MembersInjector = Client1_MembersInjector.create(provideD1Provider);//建立了 DaggerComponentA 实例 到 Client1_MembersInjector 实例的引用关系
  }

  @Override
  public D1 provideD1() {
    return provideD1Provider.get();
  }

  @Override
  public void inject(Client1 c) {//用MembersInjector来给客户注入依赖
    client1MembersInjector.injectMembers(c);
  }

  public static final class Builder {//因为可能有多个参数，采用 builder pattern 来生成 Component 实现类
    private ModuleA moduleA;

    private Builder() {}

    public ComponentA build() {
      if (moduleA == null) {
        this.moduleA = new ModuleA();
      }
      return new DaggerComponentA(this);
    }

    public Builder moduleA(ModuleA moduleA) {
      this.moduleA = moduleA;
      return this;
    }
  }
}


```



```java
/**
* 当调用 injectXxx 方法时，会试图将自己所有的 Provider 对应的 binding 实例注射到 Client 中
**/
public final class Client1_MembersInjector implements MembersInjector<Client1> {
  private final Provider<D1> d1Provider;

  public Client1_MembersInjector(Provider<D1> d1Provider) {
    this.d1Provider = d1Provider;
  }

  //该方法将 MembersInjector 与 Provider 关联
  public static MembersInjector<Client1> create(Provider<D1> d1Provider) {
    return new Client1_MembersInjector(d1Provider);
  }

  @Override
  public void injectMembers(Client1 instance) {//注入所有依赖
    instance.d1 = d1Provider.get();
  }

  public static void injectD1(Client1 instance, Provider<D1> d1Provider) {//注入单个依赖类型
    instance.d1 = d1Provider.get();
  }
}
```



这里的DaggerComponentA 是ComponentA 接口的实现类，它主要有三个作用：

1. 通过 DaggerComponentA 实例 到 ModuleA_ProvideD1Factory 实例 再到 ModuleA 实例的 引用关系，串联起 Component ——> Provider ——> Module 这条线

2. Component.inject(Client)时，就是通过 DaggerComponentA 实例 到 Client1_MembersInjector 实例 再到  Client1实例 的引用关系，串联起 Component ——> MembersInjector ——> Client 这条线

3. 有了上述两条线，自然就可以将 Module 中的依赖类型提供给 Client 了，但是1和2两步中框架只知道实例化 Provider 和 MembersInjector 的方式（用 static factory ），却不知道如何实例化 Module 和其他 Component，所以 Dagger 需要我们通过 builder 模式传入已经实例化好了的 module、component 参数来构建 Component 实例、配置参数。如果某个 module 或 component 是用默认无参构造函数来初始化，那框架就能自己实例化那个参数，我们可以省去不写，举例如下：

   ```java
      public static void main(String[] args) {
        OtherComponent otherComponent = ...;
        MyComponent component = DaggerMyComponent.builder()
            // required because component dependencies must be set
            .otherComponent(otherComponent)
            // required because FlagsModule has constructor parameters
            .flagsModule(new FlagsModule(args))
            // may be elided because a no-args constructor is visible
            .myApplicationModule(new MyApplicationModule())
            .build();
      }
    
   ```

   ​

   ​

### Component 之间的关系

在官方文档中，Component 之间的关系有两种，分别是：

- [Subcomponents](https://google.github.io/dagger/api/latest/dagger/Module.html#subcomponents)

> Subcomponents are declared by listing the class in the `Module.subcomponents()` attribute of one of the parent component's modules. This binds the [Subcomponent.Builder](https://google.github.io/dagger/api/latest/dagger/Subcomponent.Builder.html) within the parent component.
>
> Subcomponents may also be *declared via a factory method on a parent component or subcomponent*

```java
   //这是 parent Component
   @Singleton  @Component
   interface ApplicationComponent {
     // component methods...

     RequestComponent newRequestComponent(RequestModule requestModule);
   }

  //这是 subcomponent
  @Subcomponent(modules = RequestModule.class)
  interface RequestComponent {
    RequestHandler requestHandler();

    @Subcomponent.Builder
    interface Builder {
      Builder requestModule(RequestModule module);
      RequestComponent build();
    }
  }
```

- Component dependencies（ 举例为下文 ComponentB）

> While subcomponents are the simplest way to compose subgraphs of bindings, subcomponents are tightly coupled with the parents; they *may use any binding defined by their ancestor component and subcomponents*. As an alternative, components *can use bindings only from another **component interface** by declaring a [component dependency](https://google.github.io/dagger/api/latest/dagger/Component.html#dependencies). When a type is used as a component dependency, each [provision method](https://google.github.io/dagger/api/latest/dagger/Component.html#provision-methods) on the dependency is bound as a provider. Note that **only** the bindings exposed as provision methods are available through component dependencies.*

概括下来就是：Subcomponent 是组合 binding 的最简洁的方式，但与父 component 的耦合度高。为什么这么说呢？因为 subcomponent 可以直接使用它的父或子 component 里的任意 binding 类型。 而component dependencies 只能使用被它的 provision method 暴露的 binding 类型。

为什么会有这么奇怪的设定呢？原因就在下面的代码里。

#### Component dependencies

现在我们再引进一个以 component dependencies 方式和 ComponentA 相联系的 ComponentB，来看看代码是如何实现的：

```java
@Component(modules = ModuleB.class,dependencies = ComponentA.class)
public interface ComponentB {
    D2 provideD2();
  
    void inject(Client1 c);
}

@Module
public class ModuleB {
    @Provides
    D2 provideD2(){
        return new D2();
    }
}

public class D2 {}
```

生成如下的代码：

```java
public final class DaggerComponentB implements ComponentB {
  private Provider<D2> provideD2Provider;//D2 类型的提供者
  private MembersInjector<Client1> client1MembersInjector;//注射器

  private DaggerComponentB(Builder builder) {
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  private void initialize(final Builder builder) {

    this.provideD2Provider = ModuleB_ProvideD2Factory.create(builder.moduleB);
    this.client1MembersInjector = Client1_MembersInjector.create(provideD2Provider);
  }

  @Override
  public D2 provideD2() {
    return provideD2Provider.get();
  }
  
  @Override
  public void inject(Client1 c) {//将依赖注入用户中
    client1MembersInjector.injectMembers(c);
  }

  public static final class Builder {
    private ModuleB moduleB;

    private ComponentA componentA;

    private Builder() {}

    public ComponentB build() {
      if (moduleB == null) {
        this.moduleB = new ModuleB();
      }
      if (componentA == null) {
        throw new IllegalStateException(ComponentA.class.getCanonicalName() + " must be set");
      }
      return new DaggerComponentB(this);
    }

    public Builder moduleB(ModuleB moduleB) {
      this.moduleB = Preconditions.checkNotNull(moduleB);
      return this;
    }

    public Builder componentA(ComponentA componentA) {
      this.componentA = Preconditions.checkNotNull(componentA);
      return this;
    }
  }
}
```

注意，这里由于我们没有在 ComponentB 接口 中声明 类型 D1的 provision method，所以没有生成注入 D1 类型的方法，这也符合官方文档中的描述。

我们在 ComponentB 中添加一个 D1类型的 provision method,则生成的 DaggerComponentB 有以下变化：

```java
  private void initialize(final Builder builder) {
    //差异点，多出了 D1 类型的提供者
    this.provideD1Provider =
        new Factory<D1>() {
          private final ComponentA componentA = builder.componentA;

          @Override
          public D1 get() {
            return Preconditions.checkNotNull(
                componentA.provideD1(), "Cannot return null from a non-@Nullable component method");
          }
        };

    this.provideD2Provider = ModuleB_ProvideD2Factory.create(builder.moduleB);
    
    this.client1MembersInjector = Client1_MembersInjector.create(provideD1Provider);
  }

  @Override
  public D1 provideD1() {//差异点，多出了注入 D1 类型的方法
    return provideD1Provider.get();
  }
```

可以看出 ComponentB 实例是间接持有 ComponentA 的实例，从而可以使用 ComponentA 暴露的binding 类型 D1。

####  Subcomponents

现在我们再引进 ComponentC，使之与 ComponentA 为 Subcomponents 关系。

```java
@Subcomponent(modules = ModuleC.class)
public interface ComponentC {
  	D3 provideD3();
	void inject(Client1 c);
}

@Module
public class ModuleC {
    @Provides
    D3 provideD3() {
        return new D3();
    }
}

public class D3 {
}
```

ComponentA 修改为：

```java
@Component(modules = ModuleA.class)
public interface ComponentA {
    D1 provideD1();
    void inject(Client1 c);

    ComponentC newComponentC(ModuleC moduleC);
}
```

此时生成的 ComponentC 的实现类 ComponentCImpl 竟然是 ComponentA 实现类的内部类!很明显，此时 ComponentCImpl 可以直接持有 ComponentA 中的注射器 MembersInjector，从而拿到 ComponentA 能提供的所有 binding 类型，所以不需要在 ComponentC 中写出`D1 provideD1()`这样的 provision method 了。

```java
public final class DaggerComponentA implements ComponentA {
  private Provider<D1> provideD1Provider;

  private MembersInjector<Client1> client1MembersInjector;

  private DaggerComponentA(Builder builder) {
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  public static ComponentA create() {
    return builder().build();
  }

  private void initialize(final Builder builder) {

    this.provideD1Provider = ModuleA_ProvideD1Factory.create(builder.moduleA);

    this.client1MembersInjector = Client1_MembersInjector.create(provideD1Provider);
  }

  @Override
  public D1 provideD1() {
    return provideD1Provider.get();
  }

  @Override
  public void inject(Client1 c) {
    client1MembersInjector.injectMembers(c);
  }

  @Override
  public ComponentC newComponentC(ModuleC moduleC) {
    return new ComponentCImpl(moduleC);
  }

  public static final class Builder {
    private ModuleA moduleA;

    private Builder() {}

    public ComponentA build() {
      if (moduleA == null) {
        this.moduleA = new ModuleA();
      }
      return new DaggerComponentA(this);
    }

    public Builder moduleA(ModuleA moduleA) {
      this.moduleA = Preconditions.checkNotNull(moduleA);
      return this;
    }
  }

  /**存在于 DaggerComponentA 内部的 ComponentCImpl，两者的注射器和 Provider 都对对方开放**/
  private final class ComponentCImpl implements ComponentC {
    private final ModuleC moduleC;

    private Provider<D3> provideD3Provider;

    private ComponentCImpl(ModuleC moduleC) {
      this.moduleC = Preconditions.checkNotNull(moduleC);
      initialize();
    }

    private void initialize() {
      this.provideD3Provider = ModuleC_ProvideD3Factory.create(moduleC);
    }

    @Override
    public D3 provideD3() {
      return provideD3Provider.get();
    }

    @Override
    public void inject(Client1 c) {
      DaggerComponentA.this.client1MembersInjector.injectMembers(c);
    }
  }
}
```

Component 两种关系的实现方式的总结如下：

采用 subcomponent 模式时，假设C——>B——>A为从下到上的有“继承”关系的三个 Component ，则它们的 Component 实现类的关系是嵌套的内部类的关系，即ComponentCImpl 是 ComponentBImpl 的 private final 内部类，ComponentBImpl 又是ComponentAImpl 的 private final 内部类。我们知道，Java 中的内部类和嵌套类之间有这样一种相互关系：不管对方的成员变量和方法的修饰符是什么，哪怕是 private，也可以获取到那个成员变量和方法。所以 ComponentCImpl 可以直接获取到A和B中的成员 MembersInjector，也就是能拿到 binding，不需要显式的在A和B中写 provision method。
而采用 component dependencies 方式的话，A、B、C的实现类分别是3个没有嵌套关系的类， 它们之间是组合的关系，Dagger 需要我们显式地说明需要的类型才会生成相关代码，所以不像 subcomponent 方式 先天地就能获取到父子 Component 中的 binding类型。



### Subcomponents 和 scope

我们知道 Dagger 2 中有 作用域注解，比如 `@Singleton`以及其它我们自定义的作用域注解。

当我们在不添加作用域注解时，想向 ClientA 对象 中的两个 D1类型的成员变量注入值时（如下代码）：

```java
class Client {
 	@Inject//表示这个预变量需要 DI 框架来注入实例
	D1 da;
  
   	@Inject//表示这个预变量需要 DI 框架来注入实例
	D1 db;
  
	void methodHook(){
		//这个方法可以是个 hook method，好比 Java 中的 main 方法 或 Android 中的 Activity.onCreate方法
		DaggerComponentA componentImplA = DaggerComponentA.moduleA(new ModuleA()).build();
		componentImplA.inject(this);
      	assert da ！= db;//现在 d 就已经被注入了 ModuleA 中提供的实例了，不是 null 了。
}
```

我们会发现变量 da 和 db 是不同的对象。

但是在 Component 上使用了作用域注解后，那么对于每一个 Component 实例，它每次注入的实例都是同一个对象了。

我们稍微修改下上面的 ComponentA 和 ModuleA，给它们都加上`@Singleton`注解，那么 da 和 db 就是同一个对象了。

修改点：

```java
@Singleton//这里加上一行
@Component(modules = ModuleA.class)
public interface ComponentA {
    D1 provideD1();
    void inject(Client1 c);

    ComponentC newComponentC(ModuleC moduleC);
}

@Module
public class ModuleA {
    public ModuleA() {}

    @Singleton//这里加上一行
    @Provides
    D1 provideD1(){
        return new D1();
    }
}
```

框架是怎么实现作用域的呢？

Dagger 2 在发现作用域注解后，生成的 Provider 对象有所不同了：

```java
//差异点，DaggerComponentA 中initialize 方法中的 Provider<D1> 对象的实现
  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {

    this.provideD1Provider = DoubleCheck.provider(ModuleA_ProvideD1Factory.create(builder.moduleA));//差异点，这个工厂使用 DoubleCheck 来生成 D1类型的对象

    this.client1MembersInjector = Client1_MembersInjector.create(provideD1Provider);
  }
```

我们可以看下 DoubleCheck 类是如何实现的。

```java
public final class DoubleCheck<T> implements Provider<T>, Lazy<T> {
  private static final Object UNINITIALIZED = new Object();

  private volatile Provider<T> provider;
  private volatile Object instance = UNINITIALIZED;

  private DoubleCheck(Provider<T> provider) {
    assert provider != null;
    this.provider = provider;
  }

  @Override
  public T get() {
    Object result = instance;
    if (result == UNINITIALIZED) {
      synchronized (this) {
        result = instance;
        if (result == UNINITIALIZED) {
          result = provider.get();
          Object currentInstance = instance;
          if (currentInstance != UNINITIALIZED && currentInstance != result) {
            throw new IllegalStateException("Scoped provider was invoked recursively returning "
                + "different results: " + currentInstance + " & " + result);
          }
          instance = result;
          provider = null;
        }
      }
    }
    return (T) result;
  }

  /** Returns a {@link Provider} that caches the value from the given delegate provider. */
  public static <T> Provider<T> provider(Provider<T> delegate) {
    checkNotNull(delegate);
    return new DoubleCheck<T>(delegate);
  }
```

如果你看过 《Effective Java》第二版中的第71条讲解的 double-check idiom，那么你就很明显地看出上面的代码是在实现懒初始化的单例模式。将之前无作用域版本的 Provider 对象 传入 DoubleCheck 对象中并作为真正发挥功能的 provider，直到 get() 被调用时才懒初始化生成 T 类型的对象 instance，置空释放 provider 对象，之后缓存复用 instance。DoubleCheck 和 ModuleA_ProvideD1Factory 的关系也体现了 EJ 第16条 “组合优先于继承”的思想。

当然，对于同一个 Component 类型的不同实例，作用域也不能保证它们提供的依赖是单例的。因为单例模式只是保证 同一个 Component 对象里对依赖对象的复用。换句话说，所谓的作用域跟 Component 的生命周期一致的，而单例是局限于 Component 生命周期里的概念。