---
layout: post
title: 重拾Dagger2-使用Dagger2
data: 2017/3/29
subtitle: "Hey, Dagger2"
description: "Hey, Dagger2"
thumbnail: "/img/dagger2/dagger2.jpeg"
tags:
  - Android#Dagger2
categories:
  - Android
---


## 常用注解解释
在学习Dagger2之前,我们最好就是将Dagger2中的常用的注解的含义捋清一遍, 这样上手Dagger2就不会显得那么难,所以下面我准备介绍Dagger2中常用的注解的含义,这些注解包括

* Inject
* Provides
* Module
* Component
* Qualifiers
* Scope

### Inject
Inject的中文意思是注射,在Dagger2中它在不同的地方代表不同的含义．当Inject是标注在类的属性域时,那么它代表Dagger会给我们提供被Inject标注的实例; 当Inject标注的是类的构造方法时,它表示Dagger会帮我们实例化这个类并提供给需要这个类的地方

### Provides
Provides提供一种通过注解方法来提供依赖的机制, 它主要是用来弥补Inject的缺陷, 如果类的方法被Provides标注后, 它相当于告诉Dagger, 通过方法可以提供项目中所需要的依赖

### Modue
Module的作用是来管理被Provides标注的方法, 有点类似工厂

### Component
通过上面介绍的三个注解, 我们可以归纳出: 在Dagger2中可以有两种方式进行依赖注入,一种是利用Inject来进行注解, 另一种是通过在被Module标注的类中,将Provides注解标注方法,并在方法中返回需要被依赖的实例．Component的作用就是将被依赖的对象提供给需要依赖的对象，也就是说Component其实就是连接依赖方和被依赖方的桥梁,只有通过Component，依赖方和被依赖方才能联系在一起．

### Qualifiers
在Dagger2中, 如果在被Module标注的类中, 有两个方法都是返回相同类型的实例时，此时，Dagger并不知道要调用哪个方法才好，因此Dagger干脆就不干了，直接在编译时就报错．这种情况叫做依赖注入迷失．而Qualifiers的作用正是来解决依赖注入迷失的问题, 具体的看代码咯

### Scope
Scope代表作用域，Scope常常用于管理依赖对象的生命周期，Dagger中提供的Singleton注解被Scope标注，用于实现单例，不过这个单例的实现跟我们平时所写的单例不一样,具体的我会在下篇中讲到．

其实只看上面的几个注解可能会让你很蒙, 那下面我们来结合代码开始来使用Dagger2

### Inject注入
我们先来看看如何用Inject实现注入

* 首先新建Student类, 并在其构造方法添加Inject注解
```java
public class Student {

    private String name = "xiaoming";

    @Inject
    public Student(){}

    public String name() {
        return name;
    }
}
```

* 接着新建StudentComponent, 用于连接依赖方和被依赖方
```java
@Component
public interface StudentComponent {
    void inject(StudentActivity activity); //表示需要被注入依赖的地方为StudentActivity
}
```

* 最后实现依赖注入
```java
public class StudentActivity extends AppCompatActivity{

    @Inject
    Student mStudent;

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
}
```

经过这三个步骤我们就完成了一次 依赖注入．在StudentActivity中，我们需要用到Stduent类，因此我们用Inject注解标注Student属性域, 接着在Student的构造函数上标注Inject，表示当需要Student依赖时，Dagger会帮我们实例化出Student类, 现在依赖方和被依赖方都有了，就剩下最后连接彼此，而连接彼此需要在StudentComponent中添加需要注入到哪个类中,在这个例子中, 需要被注入的类是StudentActivity, 最后调用inject方法实现注入．

### Inject注入的缺陷
使用Inject注入有一个缺点: 当我们需要注入的是第三方库的类时，我们无法直接在其构造方法标注Inject，最简单的解决办法就是新建一个类并继承我们需要的第三方库中的类．但是这种解决方法太笨拙了，如果我们需要注入的类有很多的话，我们岂不是要新建很多类? 显然这种体力活不适合程序员．关于这个问题，Dagger早就帮我们考虑好了，Dagger针对这种情况给我们提供了Provides注解来解决这种问题．

### Provides实现注入
我们来看看Provides怎么解决Inject的缺陷

* 首先定义StudentModule类, 用来管理提供的依赖
```java
@Module
public class StudentModule {

    @Provides
    Student providesStudent() {
        return new Student();
    }
}
```

* 接着在StudentComponent中添加moule = StudentModule.class, 相当于告诉Dagger. 可以到该类中找依赖

```java
@Component(modules = StudentModule.class)
public interface StudentComponent {
    void inject(StudentActivity activity);
}
```

* 最后连接彼此

```java
public class StudentActivity extends AppCompatActivity{

    @Inject
    Student mStudent;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //连接依赖方和被依赖方
        DaggerStudentComponent.builder()
                .studentModule(new StudentModule())
                .build();

        //之后就可以mStudent实例了
        Toast.makeText(this, mStudent.name(), Toast.LENGTH_SHORT).show();
    }
}
```

* 在StudentModule中, 我们通过Provides标注告诉Dagger, 这个方法返回的实例是被依赖的, 而我们在StudentComponent中添加modules = StudentModule.class, 相当于在StduentComponent注入StudentActivity时, 让Dagger去StudentModule类中找提供依赖的方法．

### Qualifiers解决依赖迷失
如果我们在StudentModule中提供两个返回类型相同的实例方法:

```java
@Module
public class StudentModule {

    @Provides
    Student providesStudent() {
        return new Student();
    }

    @Provides
    Student proStudnet() {
        return new Student();
    }
}
```

上面的代码在编译时会报错, 原因是我们之前讲的依赖注入迷失. 为了解决这个问题, 我们可以用Qualifiers注解. Dagger2中已经给我们提供了一个Named注解, Named注解是被Qualifiers修饰的一个注解, 使用Named就可以解决依赖迷失, 我们先来看看Named源码:

```java
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Named {
    /** The name. */
    String value() default "";
}
```

我们可以根据我们业务需求仿照Named来自定义注解. 看完Named注解,  我们来瞧瞧怎么用, 由于使用比较简单, 我只贴代码咯

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

```java
public class StudentActivity extends AppCompatActivity{

    @Named("student1")
    @Inject
    Student mStudent;

    @Named("student2")
    @Inject
    Student mStudent2;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //连接依赖方和被依赖方
        DaggerStudentComponent.builder()
                .studentModule(new StudentModule())
                .build();

        //之后就可以mStudent实例了
        Toast.makeText(this, mStudent.name(), Toast.LENGTH_SHORT).show();
    }
}
```

### 依赖注入优先级
试想一下, 如果同时提供了Inject和Module依赖注入, Dagger会优先注入哪个? 答案是 Module. 在依赖注入时, Dagger会优先从Module查找, 如果没的话, 在从Inject构造函数, 两者只有其中一者会提供依赖

### Dagger依赖查找策略
当我们需要依赖注入时, Dagger会采取以下策略来查找依赖:
1. 首先从Module中查找, 如果查找得到的并且Provides方法中没有参数的话, 直接提供依赖返回, 如果Module中查找不到时, 如果被依赖的对象的构造函数有Inject标注并且没有参数的话, 提供依赖并且返回
2. 步骤一的都是在Provides标注的方法或Inject标注的构造函数中没有参数的情况下,  现在讨论有参数的情况
3. 首先先从Module查找, 如果Module有提供对应的依赖并且方法中有参数, Dagger会按照先Module后Inject的顺序去查找参数的依赖, 然后重复步骤一递归查找, 查不到的话再从Inject查, 如果Inject标注的构造函数有参数的话, 也是递归步骤1进行查找

### Scope实现生命周期的管理
Dagger中采用Scope来实现被依赖对象的生命周期管理, 最直接的一个标注就是Singleton. 被Singleton标注的对象可以实现单例, 但是这种单例跟我们平常写的不太一样, 具体实现下篇我会分析, 现在我们把重点放在怎么使用上

* 首先在Component标注Singleton

```java
@Singleton
@Component(modules = {AppModule.class})
public interface AppComponent {
}
```

* 接着在Modue中的Provides方法中标注Singleton

```java

@Module
public class AppModule {
    private final Application application;

    public AppModule(Application application) {
        this.application = application;
    }

    @Provides
    @Singleton
    Application providesApplication() {
        return application;
    }

    @Provides
    @Singleton
    Student providesData() {
        return new Student();
    }
}
```

* 最后在Application实现注入就可以实现单例了, 具体代码不贴了.
* Scope由于比较特殊, 关于它的实现原理和具体是使用, 准备留到源码篇和组织篇的时候再分析

#### 注意
如果Modue中的Provides有标注作用域的话, 必须和对应的Component标注的作用域相同, 不然编译时会报错.

### 最后
本篇文章主要讲Dagger2的最基本的使用, 目的指在熟悉Dagger2中的基本使用, 注入规则以及需要注意的地方, 关于Dagger2的实现原理和Dagger的组织方式, 准备开两篇来讲..嗯...今天就到这里了哈
