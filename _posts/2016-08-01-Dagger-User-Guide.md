---
layout: post
title: Dagger2使用指南
date:   2016-08-01 02:10:10
catalog:  true
tags:
    - 依赖注入
    - Dagger
description: 依赖注入 Dagger
---


## Dagger2使用指南

本文基于个人对[Dagger官方说明文档][1]的翻译，加上个人理解整理而成。
转载请注明出处：http://coderbao.com/2016/08/01/Dagger-User-Guide/

### 引言

在程序中，最棒的类往往都是功能实现类，像 `BarcodeDecoder`（条形码解码器）,`KoopaPhysicsEngine`（某某物理引擎）,`AudioStreamer`（音频流）这类干实事的。这些类往往有依赖类，可能是`BarcodeCameraFinder`（条形码扫描器）,`DefaultPhysicsEngine`（默认物理引擎）,`HttpStreamer`（Http流）。

相反， 程序中最糟糕的类是辅助类，占了空间又没做什么，比如: `BarcodeDecoderFactory`（条形码解码器工厂）， `CameraServiceLoader`（相机服务加载器），和 `MutableContextWrapper`（可变上下文包装器）。这些辅助类就像笨拙的胶带，将那些重要的功能类连接起来。

Dagger 就是用来取代这些`Factory`类来实现依赖注入设计模式([dependency injection][2])，不再需要写臃肿、公式化的代码，让你可以把精力集中在有意思的工作上。

#### 使用依赖注入框架的好处

每个类都**易于测试**。高层代码只依赖于接口，而不需要关心低层的实现类的实例化和生命周期，这些都由依赖注入框架搞定。你不再需要写一大堆代码，仅仅是为了把 `RpcCreditCardService` 换成 `FakeCreditCardService`。

模块**可复用、可移植**。你可以在你的所有程序中共享同一个模块 `AuthenticationModule` （认证模块），你也可以在开发和生产阶段分别使用`DevLoggingModule`和`ProdLoggingModule `模块。

### Dagger 2的不同之处

为什么要重复发明轮子呢？Dagger 2 与以前的[依赖注入][5]框架的区别在于： Dagger 2 是第一个使用生成代码来实现整个依赖注入技术栈的（implement the full stack with generated code）。Dagger 2 的 API简单明了，生成代码很像手写代码；易于调试追踪，完整的代码调用流程均有迹可循；高性能，图验证、配置等都是在编译时完成，未使用反射。想知道更多设计细节，请移步观看[+Gregory Kick][6] 的[油管演讲][7]。


### 使用Dagger

接下来， 我们通过[咖啡机的例子][8]来阐述 依赖注入和 Dagger。

#### 声明依赖———@Inject注解

Dagger 会自动构造实例并且满足其所需的依赖。它通过[javax.inject.Inject]注解来确定它感兴趣的构造器（constructor）和字段（field）。

想让Dagger调用哪个构造器来创建实例，就使用 @Inject 注解哪个构造器。当一个新实例被需求时，Dagger 会获取所需参数值、调用这个构造器。

```java
class Thermosiphon implements Pump {
  private final Heater heater;

  //@Inject注解构造器,heater参数来自对象图中已有的heater对象
  @Inject
  Thermosiphon(Heater heater) {
    this.heater = heater;
  }

  ...
}
```

Dagger可以直接注入字段中。本例中，它会为heater字段获取Heater实例，为pump字段获取Pump实例。

```java
class CoffeeMaker {
  @Inject Heater heater;
  @Inject Pump pump;

  ...
}
```

如果你的类中有 @Inject 注解的字段但没有对应的 `@Inject` 注解的构造器，那么Dagger会在被请求时尝试注入那些字段，但无法创建新的实例。添加一个 `@Inject` 注解的无参构造器就可以指示Dagger创建实例了。

如果`@Inject`注解的是XXX类中需要依赖的字段，那么这个字段不能用`private`关键字修饰，不然生成代码中的XXX_MembersInjector类无法拿到XXX类中的该字段，从而无法给字段赋值。

Dagger也支持方法注入（method injection），然而构造器或字段注入（constructor or field injection）的优先级更高。

#### 满足依赖———@Provides注解和@Module注解

默认情况下，Dagger采用上面描述的方式————通过构造出被请求类型的实例来满足依赖，当你需求`CoffeeMaker`类时，Dagger会通过调用`new CoffeeMaker()`和设置它的可注入的字段来获取一个实例。

但是 `@Inject`在以下场景用不了：

- 接口，因为无法被构造实例
- 第三方类库，因为无法添加注解到构造器上
- 必须配置参数的对象

在这些`@Inject`不适用的场景中，可以使用[@Provides][9]注解的方法来满足依赖。该方法的返回值定义了它被用来满足哪个依赖。

例如，`provideHeater()`方法在一个`Heater`实例被需求时调用：

**DripCoffeeModule.java**

```java
@Provides
static Heater provideHeater(){
    return new ElectricHeater();
}
```

`@Provides`方法自身也可以有依赖。当需求`Pump`实例时，`ProvidePump`方法返回的是一个`Thermosiphon`实例。

`@Provides`方法必须从属于一个模块（module）。module类是用[`@Module`][10]注解的类，如`DripCoffeeModule `。

**DripCoffeeModule.java**

```java
@Module
class DripCoffeeModule {
    @Provides 
    static Heater provideHeater() {
        return new ElectricHeater();
    }
```

**PumpModule.java**

```java
@Module
class PumpModule {
    @Provides 
    static Pump providePump(Thermosiphon pump) {
        return pump;
    }
}
```

按照命名习惯，`@Provides`方法名带有`provide`前缀，module类名带有`Module`后缀。

#### 建立对象图———@Component注解

`@Inject`和`@Provides`注解的类，通过依赖关系连结成一张对象图。`main()`方法或Android的 [Application类][11] 中调用代码可以通过一些定义良好的根节点来获取对象图，也就是Dagger 2 中的 component 接口。component接口的约束为：接口使用`@Component`注解，`@Component`注解的`modules`参数为提供依赖的module的类型，接口方法也有约束。编译时，Dagger 2会生成这种约束的实现类。

**component方法**（Component methods）

接口中的方法只有两种，一种是provision method（供应方法），没有参数、返回值类型为被需求类的类型，**有了此方法则依赖才会对外暴露**，例如：

```java
   SomeType getSomeType();
   Set<SomeType> getSomeTypes();
   @PortNumber int getPortNumber();
   Provider<SomeType> getSomeTypeProvider();
   Lazy<SomeType> getLazySomeType();
```

只有当component显式暴露了它引入的module中的依赖类型时，即通过provision methods暴露（即返回值类型为依赖类型的方法），这些依赖才算是对外暴露了，否则它们将无法用于依赖注入。

另一种是members-injection（成员注入方法），参数为需求依赖的类型T的对象,**此方法调用则依赖被注入给对象T**，例如：

```java
   void injectSomeType(SomeType someType);
   SomeType injectAndReturnSomeType(SomeType someType);
```

component中调用`getSomeTypeMembersInjector()`再`MembersInjector.injectMembers(T)`等价于`injectT(T)`，会完成相同的注入工作。

```java
   MembersInjector<SomeType> getSomeTypeMembersInjector();
```



```java
@Component(modules = DripCoffeeModule.class)
interface CoffeeShop {
    CoffeeMaker maker();
}
```

**component命名规范**

component 实现类的名字是 `Dagger`前缀+component接口名。调用实现类的`builder()`方法返回builder，接着使用builder设置依赖，再调用`build()`就生成了新的component实例。如component接口名为CoffeeShop，则实现类名为DaggerCoffeeShop。

```java
CoffeeShop coffeeShop = DaggerCoffeeShop.builder()
.dripCoffeeModule(new DripCoffeemodule)
```

注意：如果component接口位于某个类的内部，不在最外层，那么生成的component实现类的名字就是`Dagger`前缀+外部类名+`_`+component接口名。

```java
class Foo {
    static class Bar {
        @Component
        interface BazComponent {}
    }
}
```

上面这段代码生成的实现类叫做`DaggerFoo_Bar_BazComponent`。

使用component builder的一些注意点：

- 只要module有外部可达的默认构造器（accessible default constructor），就可以省去不写这个module，因为builder会在什么都不设置时自动构造出一个实例。

- 如果module的所有`@Provides`方法都是static修饰的，component实现类就不需要module的实例。

- 如果component的所有依赖都不需要用户手动创建实例（指没有component依赖，而且所有的module都有可见的无参构造器），那么生成的实现类会多出一个`create()`方法，该方法等价于`.builder().build()`方法，也可以创建component实现类的实例。

例如：
```java
public static void main(String[] args) {
    OtherComponent otherComponent = ...;
    MyComponent component = DaggerMyComponent.builder()
        // 必要，因为component依赖必须设置
        .otherComponent(otherComponent)
        // 必须，因为FlagsModules的构造器有参数
        .flagsModule(new FlagsModule(args))
        // 可省，因为MyApplicationModule的无参构造器可见
        .myApplicationModule(new MyApplicationModule())
        .build();
}
```


```java
CoffeeShop coffeeShop = DaggerCoffeeShop.create();
```

现在，我们的`CoffeeApp`可以很方便地使用Dagger生成的实现类CoffeeShop来获得已注入好依赖的`CoffeeMaker`实例。

```java
public class CoffeeApp {
    public static void main(String[] args) {
        //注入入口点
        CoffeeShop coffeeShop = DaggerCoffeeShop.create();
        coffeeShop.maker().brew();        
    }
}
```

现在对象图已经被创建、入口点被注入了，我们可以跑通coffee maker项目了。

```shell
$ java -cp ... coffee.CoffeeApp
~ ~ ~ heating ~ ~ ~
=> => pumping => =>
 [_]P coffee! [_]P
```

注意：

- 当调用`injectSelf(Self instance)`方法且传入参数为Child类型的实例时，仍然只能注入a和b，因为`injectSelf(Self instance)`内部调用的是形如`supertypeInjector.injectMembers(instance);//注入父类的字段
    instance.field = get();//注入本类的字段`这样的代码，只会注入本类型的字段和它的父类型的字段。

```java
   class Parent {
     @Inject A a;
   }

   class Self extends Parent {
     @Inject B b;
   }

   class Child extends Self {
     @Inject C c;
   }
```

#### 图中的粘合器（binding）

上面的例子展示了如何用一些典型的Binding来构建component，但将binding提供给图的方式有很多，接下来这些都可以当作依赖、构成component：

- 那些声明了@Module注解、包含@Provides方法的module，通过`@Component.modules`直接被引用或`@Module.includes`间接被引用

- 包含`@Inject`构造器的类，而且要无作用域或作用域与component的作用域一致

- [component 依赖][12]（即通过`@Component.dependencies`引入的component）的 [provision方法][13]

- component自身

- 被引入的 [subcomponent][14] 的未加限定符的 [builders][15]

- 上面提及的bindings的 `Provider` 或 `Lazy` 包装类

- 上面提及的bindings的 `Lazy` 、`Provider`
嵌套的包装类（如`Provider<Lazy<CoffeeMaker>>`）

- 任意类型的 `MembersInjector` 

#### Singleton 和 作用域绑定（Scoped Binding）

在一个 `@Provides` 方法或可注入类上使用 [@Singleton][16]作用域 ，图就会对所有的客户端提供同一个实例。

```java
@Provides 
@Singleton
static Heater provideHeater() {
    return new ElectricHeater();
}
```

在可注入类上使用的 `@Singleton` 注解也会出现在 [documentation][17] 中，以提醒用户这个类可能被多个线程共享。

```java
@Singleton
class CoffeeMaker {
    ...
}
```

既然 Dagger 2 将依赖图中有作用域的实例和component实现类的实例关联起来了，component自己也需要声明它想代表的作用域。比如，在同一个 component 共存中的 `@Singleton` binding类 （全局作用域）和 `@RequestScoped` binding类（自定义的作用域，标志着“使得对象的生命周期与所在请求的生命一致，实现每个请求内的单例”） 没有意义，因为这两个作用域有不同的生命周期，因此也应该存在于不同生命周期的 component 中。想将component与某个作用域关联，只需要给 component 接口加上作用域的注解。

```java
@Component(modules = DripCoffeeModule.class)
@Singleton
interface CoffeeShop {
    CoffeeMaker maker();
}
```

component 可以有多个作用域注解，这表明它们是同一个作用域的别名，因而 component 可以引入 那些和component声明的作用域一致的 有作用域的binding类（scoped bindings）。


#### Reusable 作用域

在Android环境中，分配空间的代价高昂。有时我们想要限制 实例化（`@Inject`注解构造器的类被实例化或 `@Provides` 方法被调用）的次数，又不需要保证特定component生命周期中的单例，对于有这种需求的 binding 类型，我们可以用 [@Reusable][18] 作用域。

binding类 使用 `@Reusable` 作用域的效果和使用其它作用域的效果不同， `@Reusable` 并不是把自身和某个 component 关联，而是让每个使用 binding 类的 component  都缓存 `@Provides` 方法返回 或 `@Inject` 构造器实例化的 
 对象。

这意味着如果 component 中有一个 `@Reusable` binding，但只有一个子component实际使用了 binding，那么只有那个子component会缓存binding对象。如果有两个不同宗的component各使用了binding，他们俩各缓存各的对象，互不相干。如果component的祖先已经缓存了对象，那么子component会复用该对象。

我们不能确保component只会调用binding一次，所以对那些返回可变对象的binding 或 需要返回同一个实例的场景 使用 `@Reusable` 很危险。安全使用 `@Reusable` 的场景应该是：你不在乎它们被分配多少次空间、没加作用域的不变对象。

```java
@Reusable // 用来搅拌咖啡的小匙。我们不在乎用了多少个小匙，但也不要浪费。
class CoffeeScooper {
  @Inject CoffeeScooper() {}
}

@Module
class CashRegisterModule {
  @Provides
  @Reusable // 错误做法。现金出纳机类。我们当然在乎把钱放到哪个出纳机里了。
            // 使用一个特定的作用域来代替。
  static CashRegister badIdeaCashRegister() {
    return new CashRegister();
  }
}

  
@Reusable // 错误做法。咖啡滤纸类。我们每次都要用一张新的滤纸，所以这里不该加作用域。 
class CoffeeFilter {
  @Inject CoffeeFilter() {}
}
```


#### 延迟注入（Lazy injections）

有时，我们需要一个延迟到使用时才初始化的对象。对于任意的 binding 类 `T`，我们可以创建 [Lazy<T>][19] 对象，它会把实例化延迟到第一次调用 `Lazy<T>.get()` 方法的时候。如果`T`是 singleton提供方式，那么在对象图中的注入的所有`Lazy<T>`都是同一个实例。如果`T`不是 singleton，每个注入`Lazy<T>`的点，都是不同的实例。

```java
//带研磨功能的咖啡机
class GridingCoffeeMaker {
  @Inject Lazy<Grinder> lazyGrinder;

  // 冲泡功能
  public void brew() {
    while (needsGrinding()) {
      // 第一次调用 .get() 时创建，之后的调用都使用缓存值
      lazyGrinder.get().grind();
    }
  }
}
```

#### Provider 注入(Provider injections)

有时我们需要返回多个实例，而不是注入单一的值。我们有多种做法（如Factory,Builder 等），其中一种做法是注入一个`Provider<T>`而不是`T`。每次调用`Provider<T>.get()`时都会调用binding逻辑。如果binding逻辑是 `@Inject` 构造器，就会创建新实例，而`@Provides`方法不一定。

```java
class BigCoffeeMaker {
  @Inject Provider<Filter> filterProvider;

  public void brew(int numberOfPots) {
  ...
    for (int p = 0; p < numberOfPots; p++) {
      maker.addFilter(filterProvider.get()); //new filter every time.
      maker.addCoffee(...);
      maker.percolate();
      ...
    }
  }
}
```

注意：注入`Provider<T>`可能创造易混淆的代码，但在有些场景也有奇效（例如，servlet从设计上看是单例，但只在请求相关数据的上下文中有用）。（Injecting Provider<T> has the possibility of creating confusing code, and may be a design smell of mis-scoped or mis-structured objects in your graph. Often you will want to use a factory or a Lazy<T> or re-organize the lifetimes and structure of your code to be able to just inject a T. Injecting Provider<T> can, however, be a life saver in some cases. A common use is when you must use a legacy architecture that doesn’t line up with your object’s natural lifetimes (e.g. servlets are singletons by design, but only are valid in the context of request-specfic data)）

#### 修饰符（Qualifiers）

有时，仅仅是类型并不能确定依赖，比如，一个复杂的咖啡制造程序也许需要不同的加热器，用来加热水和热食。

这种情况下，我们添加 **修饰符注解**（**qualifier annotation**）。这种注解自身使用了 `@Qualifier` 注解。这里是 `javax.inject`包中的一个修饰符注解 `@Named`的定义：

```java
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Named {
    String value() default "";
}
```

你可以创建你自己的修饰符注解，或者就用`@Named`。根据需求来给变量或参数添加修饰符注解，类型和修饰符注解会被一起使用来确定依赖。

```java
class ExpensiveCoffeeMaker {
    @Inject 
    @Named("water")
    Heater waterHeater;
    
    @Inject 
    @Named("hot plate")
    Heater hotPlateHeater;
}
```

把修饰符注解和它的值加到对应的`@Provides`方法上。比如：

```java
@Provides
@Named("hot plate")
static Heater provideHotPlateHeater {
    return new ElectricHeater(70);
}

@Provides
@Named("water")
static Heater provideWaterHeater {
    return new ElectricHeater(93);
}
```

通常情况下，依赖没有多个修饰符注解。

### 编译时校验（Compile-time Validation）

Dagger 的注解处理器 很严格，会在任何 binding 无效或不完整时编译出错。比如，下面的module被加入了component中，但 component 中缺乏 `Executor` 的依赖提供：

```java
@Module
class DripCoffeeModule {
    @Provides
    static Heater provideHeater(Executor executor) {
        return new CpuHeater(executor);
    }
}
```

编译时会报如下错误：

```
[ERROR] COMPILATION ERROR :
[ERROR] error: java.util.concurrent.Executor cannot be provided without an @Provides-annotated method.
```

解决问题的方法就是，在component中的任意一个module中添加 `Executor`类的`@Provides` 方法。

### 编译时的代码生成（Compile-time Code Generation）

Dagger 的注解处理器可能会生成像````的源文件，这些文件是Dagger实现的细节。虽然在调试单步注入时使用到了，但你不应该直接使用它们。你的代码中唯一应该使用的类是以Dagger开头的component实现类。

### 在编译脚本中使用Dagger

#### Gradle配置

在应用的运行时引入`dagger-2.2.jar`，在编译时引入`dagger-compiler-2.2.jar`。具体操作如下：

build.gradle （项目的根目录中）

```
dependencies {
        ...
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
}
```

build.gradle （Android/Java module 中）

```
//配置注解处理器，以生成代码和配置编译时的代码路径
apply plugin: 'com.neenbedankt.android-apt'

dependencies {
    //编译时引入compiler库以生成代码
    apt 'com.google.dagger:dagger-compiler:2.0'
    //运行时只需引入dagger库
    compile 'com.google.dagger:dagger:2.0'
    ...
}
```

#### Maven配置

```xml
<dependencies>
  <dependency>
    <groupId>com.google.dagger</groupId>
    <artifactId>dagger</artifactId>
    <version>2.2</version>
  </dependency>
  <dependency>
    <groupId>com.google.dagger</groupId>
    <artifactId>dagger-compiler</artifactId>
    <version>2.2</version>
    <optional>true</optional>
  </dependency>
</dependencies>
```

版权声明：本文为博主原创文章，未经博主允许不得转载。

  [1]: http://google.github.io/dagger/users-guide.html
  [2]: http://en.wikipedia.org/wiki/Dependency_injection
  [3]: http://docs.oracle.com/javaee/7/api/javax/inject/package-summary.html
  [4]: https://jcp.org/en/jsr/detail?id=330
  [5]: http://en.wikipedia.org/wiki/Dependency_injection
  [6]: https://google.com/+GregoryKick/
  [7]: https://www.youtube.com/watch?v=oK_XtfXPkqw&feature=youtu.be
  [8]: https://github.com/google/dagger/tree/master/examples/simple/src/main/java/coffee
  [9]: http://google.github.io/dagger/api/latest/dagger/Provides.html
  [10]: http://google.github.io/dagger/api/latest/dagger/Module.html
  [11]: http://developer.android.com/reference/android/app/Application.html
  [12]: http://google.github.io/dagger/api/latest/dagger/Component.html#dependencies%28%29
  [13]: http://google.github.io/dagger/api/latest/dagger/Component.html#provision-methods
  [14]: http://google.github.io/dagger/api/latest/dagger/Subcomponent.html
  [15]: http://google.github.io/dagger/api/latest/dagger/Subcomponent.Builder.html
  [16]: http://docs.oracle.com/javaee/7/api/javax/inject/Singleton.html
  [17]: http://docs.oracle.com/javase/7/docs/api/java/lang/annotation/Documented.html
  [18]: http://google.github.io/dagger/api/latest/dagger/Reusable.html
  [19]: http://google.github.io/dagger/api/latest/dagger/Lazy.html