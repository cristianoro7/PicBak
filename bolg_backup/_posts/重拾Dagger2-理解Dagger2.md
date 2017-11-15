---
layout: post
title: 重拾Dagger2-理解Dagger2
data: 2017/3/30
subtitle: "Understanding Dagger2"
description: "Understanding  Dagger2"
thumbnail: "/img/dagger2/understanddagger2.jpg"
tags:
  - Android#Dagger2
categories:
  - Android
---

### 回顾
[上一篇](https://cristianoro7.github.io/2017/03/29/%E9%87%8D%E6%8B%BEDagger2-%E4%BD%BF%E7%94%A8Dagger2/)我们主要介绍了如何用Dagger2在Android应用中进行依赖注入, 在这一篇中, 我们主要来理解Dagger2中的原理, 毕竟知己知彼后才用得踏实

### API关系
* Provider<T>: provider<T>是一个接口, 它的作用是包装被依赖的类
* Factory<T>: 继承于Provider, 作用是创建依赖的对应实例
* MembersInjector<T>:  也是一个接口, 作用是将依赖注入到需要依赖的地方, 其中的T为注入地点

理解上面的接口的作用后, 我们来理解Dagger2不会太难啦

### Inject注入解读
我们利用上篇中的Demo来解读Inject注入. Dagger2生成的源代码可以在app目录下的build目录中的apt目录找到.

既然Component是注入依赖的桥梁, 那我们就先来看看它是怎么出入的吧. 回顾一下上次我们在StudentActivity注入的代码

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    //连接依赖方和被依赖方
    DaggerStudentComponent.builder()
            .build()
            .inject(this);

    //之后就可以mStudent实例了
    Toast.makeText(this, mStudent.name(), Toast.LENGTH_SHORT).show();
}
```

在onCreate方法中进行了注入, 仔细看注入的代码, 我们可以大概猜测是用Builder模式创建StudentComponenet实例, 既然这样, 我们先进入build()方法研究一波

```java
public static final class Builder {
  private Builder() {}

  public StudentComponent build() {
    return new DaggerStudentComponent(this);
  }
}
```

在build()中, 实例了DaggerStudentComponent, DaggerStudentComponent是StudentComponent具体的实现类, 我们顺着思路, 进入DaggerStudentComponent看看

```java
public final class DaggerStudentComponent implements StudentComponent {
  private MembersInjector<StudentActivity> studentActivityMembersInjector;

  private DaggerStudentComponent(Builder builder) {
    assert builder != null;
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  public static StudentComponent create() {
    return new Builder().build();
  }

  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {

    this.studentActivityMembersInjector =
        StudentActivity_MembersInjector.create(Student_Factory.create());
  }

  @Override
  public void inject(StudentActivity activity) {
    studentActivityMembersInjector.injectMembers(activity);
  }

```

上面的代码中, 主要调用了initialize方法进行初始化, 在其中, 创建了StudentActivity_MemberInjector实例

还记得我们在StudentActivity中调用了build()方法后, 还会调用inject(this)吗?  调用该方法后, 会调用我们刚刚在initialize方法中创建出来的StudentActivity_MemberInjector的injectMembers(activity), 我们看看对应的方法都干了什么事情

```java
@Override
public void injectMembers(StudentActivity instance) {
  if (instance == null) {
    throw new NullPointerException("Cannot inject members into a null reference");
  }
  instance.mStudent = mStudentProvider.get();
}
```

看到上面的代码, 有没有一种熟悉的感觉? instance.mStudent正是我们在StudentActivity中需要注入的成员变量, 而给它赋值的又是谁?

mStudentProvider变量是我们实例化StudentActivity_MemberInjector传进来的, 我们自然进入对应的类探究探究

```java
public final class Student_Factory implements Factory<Student> {
  private static final Student_Factory INSTANCE = new Student_Factory();

  @Override
  public Student get() {
    return new Student();
  }

  public static Factory<Student> create() {
    return INSTANCE;
  }
}
```

StudentFactory中的get()方法返回了对应的Student实例

分析到这里, Inject注解已经完成了一次依赖注入, 因此我们来串联一下各个步骤吧

首先提供依赖的是Stduent_Factory类, 它是Factory的具体实现类, 给我们需要依赖的地方new出实例,

StudentActivity_MemberInjector, 它是MemberInjector的具体实现类, 它在injectMembers(StudentActivity中, 拿到我们需要依赖的实例, 并给被Inject标注的成员变量赋值, 而实例的来源正是我们刚才分析的

讲到这里, 我想大家都对Component的作用加深了, Component就像一座桥梁, 将提供依赖的工厂类, 和需要依赖的地方联系起来, 从而完成依赖注入

### Moduley依赖注入
Module依赖注入的讲解, 我们也是用Student的Demo来讲解

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    //连接依赖方和被依赖方
    DaggerStudentComponent.builder()
            .build()
            .inject(this);

    //之后就可以mStudent实例了
    Toast.makeText(this, mStudent.name(), Toast.LENGTH_SHORT).show();
}
```

我们按照来老套路, 进入build()方法

```java
public static final class Builder {
  private StudentModule studentModule;

  private Builder() {}

  public StudentComponent build() {
    if (studentModule == null) {
      this.studentModule = new StudentModule();
    }
    return new DaggerStudentComponent(this);
  }

  public Builder studentModule(StudentModule studentModule) {
    this.studentModule = Preconditions.checkNotNull(studentModule);
    return this;
  }
}
```

还是熟悉的套路, 只不过这次多了一个StduentModule变量, StudentModule是我们事先写好用于提供依赖的类, 我们可以看到在build()方法中, 实例化了StudentModule和DaggerStduentComponent, 又是老套路, 进入DaggerStduentComponent看看

```java
public final class DaggerStudentComponent implements StudentComponent {
  private Provider<Student> providesStudentProvider;

  private MembersInjector<StudentActivity> studentActivityMembersInjector;

  private DaggerStudentComponent(Builder builder) {
    assert builder != null;
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  public static StudentComponent create() {
    return new Builder().build();
  }

  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {

    this.providesStudentProvider =
        StudentModule_ProvidesStudentFactory.create(builder.studentModule);

    this.studentActivityMembersInjector =
        StudentActivity_MembersInjector.create(providesStudentProvider);
  }

  @Override
  public void inject(StudentActivity activity) {
    studentActivityMembersInjector.injectMembers(activity);
  }
```

跟我们上面分析Inject的代码差不多, 不同的是这次在initialize中创建的是StudentModule_ProvidesStudentFactory, 接下来的流程和Inject都差不多, 我们现在主要来分析StudentModule_providesStduentFactory怎么拿到依赖实例

```java
@Override
public Student get() {
  return Preconditions.checkNotNull(
      module.providesStudent(), "Cannot return null from a non-@Nullable @Provides method");
}
```

在get()方法中, 利用module.providesStudent() 提供实例, 这个方法是我们事先在StudentModule中定义的

总体来说Module依赖注入的方式和Inject差不多, 只是提供依赖时生成的工厂方法不一样而已

分析完了Inject和Module注入, 我们来看看Name注解是如何解决依赖迷失,



```java
@Module
public class StudentModule {

    @Named("student1")
    @Provides
    Student providesStudent() {
        return new Student();
    }

    @Named("student2")
    @Provides
    Student proStudnet() {
        return new Student();
    }
}
```

我们make一下工程, 再到app->build->source->apt->debug目录下, 可以看看这两个类

```java
public final class StudentModule_ProvidesStudentFactory implements Factory<Student> {
  private final StudentModule module;

  public StudentModule_ProvidesStudentFactory(StudentModule module) {
    assert module != null;
    this.module = module;
  }

  @Override
  public Student get() {
    return Preconditions.checkNotNull(
        module.providesStudent(), "Cannot return null from a non-@Nullable @Provides method");
  }

  public static Factory<Student> create(StudentModule module) {
    return new StudentModule_ProvidesStudentFactory(module);
  }

  /** Proxies {@link StudentModule#providesStudent()}. */
  public static Student proxyProvidesStudent(StudentModule instance) {
    return instance.providesStudent();
  }
}
```

```java
public final class StudentModule_ProStudnetFactory implements Factory<Student> {
  private final StudentModule module;

  public StudentModule_ProStudnetFactory(StudentModule module) {
    assert module != null;
    this.module = module;
  }

  @Override
  public Student get() {
    return Preconditions.checkNotNull(
        module.proStudnet(), "Cannot return null from a non-@Nullable @Provides method");
  }

  public static Factory<Student> create(StudentModule module) {
    return new StudentModule_ProStudnetFactory(module);
  }

  /** Proxies {@link StudentModule#proStudnet()}. */
  public static Student proxyProStudnet(StudentModule instance) {
    return instance.proStudnet();
  }
}
```

相信聪明的你,已经知道Name注解解决的思路. 很简单, 就是每一个方法对应生成一个工厂类提供对应的依赖, 至于类的命名规则我就不说啦, 自己看

### 迷之Scope
Scope是作用域, Singleton是被Scope包装的一个注解, 在Dagger2中如果被Singleton标注的话, 该依赖可以在一定程度上实现单例 只不过这种单例是需要条件的, 这个条件是什么? 代码是最好的老师, 我们还是来看看被Sington标注的依赖会生成什么代码

```java
public final class DaggerStudentComponent implements StudentComponent {
  private Provider<Student> proStudnetProvider;

  private MembersInjector<StudentActivity> studentActivityMembersInjector;

  private DaggerStudentComponent(Builder builder) {
    assert builder != null;
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  public static StudentComponent create() {
    return new Builder().build();
  }

  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {

    this.proStudnetProvider =
        DoubleCheck.provider(StudentModule_ProStudnetFactory.create(builder.studentModule));

    this.studentActivityMembersInjector =
        StudentActivity_MembersInjector.create(proStudnetProvider);
  }

  @Override
  public void inject(StudentActivity activity) {
    studentActivityMembersInjector.injectMembers(activity);
  }

  public static final class Builder {
    private StudentModule studentModule;

    private Builder() {}

    public StudentComponent build() {
      if (studentModule == null) {
        this.studentModule = new StudentModule();
      }
      return new DaggerStudentComponent(this);
    }

    public Builder studentModule(StudentModule studentModule) {
      this.studentModule = Preconditions.checkNotNull(studentModule);
      return this;
    }
  }
}
```

仔细观察上面代码, 我们看到唯一不同的provider赋值的时候, 多了一个DoubleCheck, 其中的奥妙

```java

public final class DoubleCheck<T> implements Provider<T>, Lazy<T> {
  private static final Object UNINITIALIZED = new Object();

  private volatile Provider<T> provider;
  private volatile Object instance = UNINITIALIZED;

  private DoubleCheck(Provider<T> provider) {
    assert provider != null;
    this.provider = provider;
  }

  @SuppressWarnings("unchecked") // cast only happens when result comes from the provider
  @Override
  public T get() {
    Object result = instance;
    if (result == UNINITIALIZED) {
      synchronized (this) {
        result = instance;
        if (result == UNINITIALIZED) {
          result = provider.get();
          /* Get the current instance and test to see if the call to provider.get() has resulted
           * in a recursive call.  If it returns the same instance, we'll allow it, but if the
           * instances differ, throw. */
          Object currentInstance = instance;
          if (currentInstance != UNINITIALIZED && currentInstance != result) {
            throw new IllegalStateException("Scoped provider was invoked recursively returning "
                + "different results: " + currentInstance + " & " + result + ". This is likely "
                + "due to a circular dependency.");
          }
          instance = result;
          /* Null out the reference to the provider. We are never going to need it again, so we
           * can make it eligible for GC. */
          provider = null;
        }
      }
    }
    return (T) result;
  }

  /** Returns a {@link Provider} that caches the value from the given delegate provider. */
  public static <T> Provider<T> provider(Provider<T> delegate) {
    checkNotNull(delegate);
    if (delegate instanceof DoubleCheck) {
      /* This should be a rare case, but if we have a scoped @Binds that delegates to a scoped
       * binding, we shouldn't cache the value again. */
      return delegate;
    }
    return new DoubleCheck<T>(delegate);
  }

  /** Returns a {@link Lazy} that caches the value from the given provider. */
  public static <T> Lazy<T> lazy(Provider<T> provider) {
    if (provider instanceof Lazy) {
      @SuppressWarnings("unchecked")
      final Lazy<T> lazy = (Lazy<T>) provider;
      // Avoids memoizing a value that is already memoized.
      // NOTE: There is a pathological case where Provider<P> may implement Lazy<L>, but P and L
      // are different types using covariant return on get(). Right now this is used with
      // DoubleCheck<T> exclusively, which is implemented such that P and L are always
      // the same, so it will be fine for that case.
      return lazy;
    }
    return new DoubleCheck<T>(checkNotNull(provider));
  }
}
```

DoubleCheck是Provider的具体体现类, 在provider方法中, 返回了DoubleCheck实例, DoubleCheck的精华主要是在get方法中

```java
SuppressWarnings("unchecked") // cast only happens when result comes from the provider
@Override
public T get() {
  Object result = instance;
  if (result == UNINITIALIZED) {
    synchronized (this) {
      result = instance;
      if (result == UNINITIALIZED) {
        result = provider.get();
        /* Get the current instance and test to see if the call to provider.get() has resulted
         * in a recursive call.  If it returns the same instance, we'll allow it, but if the
         * instances differ, throw. */
        Object currentInstance = instance;
        if (currentInstance != UNINITIALIZED && currentInstance != result) {
          throw new IllegalStateException("Scoped provider was invoked recursively returning "
              + "different results: " + currentInstance + " & " + result + ". This is likely "
              + "due to a circular dependency.");
        }
        instance = result;
        /* Null out the reference to the provider. We are never going to need it again, so we
         * can make it eligible for GC. */
        provider = null;
      }
    }
  }
  return (T) result;
}
```

get()方法中的代码有没有很熟悉?? 咋一看, 我还以为是DCL单例模式, 其实不然, 它是利用DCL的思想, 将依赖对象缓存到DoubleCheck中, 下次提供依赖时, 直接从缓存拿. 说了这么多, 你可能会问: 究竟Singleton怎么实现单例啊? 其实这个问题在get()方法中已经体现出来了

既然后一次提供依赖时会从缓存拿, 那我们只要保证DoubleCheck之会被初始化一次, 那么每次用到这个依赖时, 都是从缓存拿的, 也就变相的实现了单例, 为了保证DoubleCheck只会被初始化一次, 我们可以在Appication中实现注入, 那就保证整个应用中DoubleCheck只会被初始化一次.  

细心的你可能会注意到, 有被Singleton标注的依赖都会在Component实例中被DoubleCheck包装, 而没有的依赖则会直接被工厂类创建出来. 那么我们可以自定义一个注解PerActivity, 那么它就可以管理依赖对象在需要注入依赖的Activity实现局部单例

看完Dagger2这个实现单例的思想瞬间膜拜.

总体来说, Scope注解还是很有用的, 利用它我们可以有效的管理依赖对象的生命周期, 让我们不用再去写一堆的单例类

### 最后
本篇主要讲了Dagger注入依赖的原理, 重点讲了Scope实现单例的原理, 对于Component和SubComponent我想留到下一篇讲, 下篇我会针对Component和SubComponent来分析, 并利用他们有效的组织依赖注入. 今天就到这里啦...
