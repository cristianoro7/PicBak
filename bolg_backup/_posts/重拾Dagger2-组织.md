---
title: 重拾Dagger2-组织
data: 2017/3/30
subtitle: "goodbye Dagger2"
description: "goodbye  Dagger2"
thumbnail: "/img/dagger2/goodbyedagger2.png"
tags:
  - Android#Dagger2
categories:
  - Android
---

### 回顾
在[上一篇](https://cristianoro7.github.io/2017/03/30/%E9%87%8D%E6%8B%BEDagger2-%E7%90%86%E8%A7%A3Dagger2/)中, 我们从源码的角度分析了Dagger的工作原理, 在这一篇中, 我们来重点讲解Dagger2依赖组织的方式, 在讲解之前, 我们先来理解Component dependencies和SubComponent

### Component dependencies VS SubComponent
为了提高一个Component中代码的复用度, 我们可以利用 Component dependencies 和 SubComponent来获取Component中的依赖, 那么这两种方式有什么区别?

####  Component dependencies
我们还是用前两篇中的Demo来讲解, 首先我们定义一个AppComponent和AppModule, 并且在AppModule提供一个Student的实例

```java
@Singleton
@Component(modules = {AppModule.class})
public interface AppComponent {

    Application providesApp();

    Student providesStudent();
}
```

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
    Student providesStudent() {
        return new Student();
    }
}
```

现在我们有另外一个StudentComponent和StudentModule, 其中StudentModule需要Student实例, 由于我们在AppModule已经提供了Student实例, 我们可以在StudentComponent中定义dependencies, 这样就可以实现复用Student实例了

```java
@PerActivity
@Component(dependencies = AppComponent.class, modules = StudentModule.class)
public interface StudentComponent {
    void inject(StudentActivity activity);
}

```

```java
@Module
public class StudentModule {

    @Provides
    @PerActivity
    Data providesData() {
        return new Data();
    }
}
```

这就是Component dependencies的使用

#### SubComponent
SubComponent也可以实现Component的代码复用, 我们具体来看看怎么使用

```java
@Singleton
@Component(modules = {AppModule.class})
public interface AppComponent {

    StudentComponent getComponent();


    Application providesApp();

//    Student providesSyudent();
}
```

```java
@PerActivity
@Subcomponent(modules = StudentModule.class)
public interface StudentComponent {
    void inject(StudentActivity activity);
}
```

#### 两者区别?
既然两者都可以实现Component代码的复用, 那么他们的区别的是什么?
* Component dependencies方式的代码复用, 父Component必须显式暴露依赖给 dependencies的Component, 如上面的AppComponent

```java
  Student providesStudent();
```

显式暴露了Student依赖给dependencies的Component, 而对于SubComponent不用显式暴露, 直接在父Component定义一个返回SubComponent的方法, 并在对应的类标注SubComponent注解

* 从设计的角度来看, SubComponent是为了让两个Component更具有内聚性. 而Component dependencies则是让两个Component彼此独立

### 源码读解
为什么 Component dependencies方式必须要显式暴露? 且听我道来

```java
@Override
public Application providesApp() {
  return providesApplicationProvider.get();
}

@Override
public Student providesSyudent() {
  return providesStudentProvider.get();
}
```

上面两个方法是AppComponent具体实现类的重写的两个方法, 在AppComponent暴露方法的主要目的是在对应的实现类提供依赖, 而在依赖于AppComponent的StudentComponent中,

```java
public final class DaggerStudentComponent implements StudentComponent {
  private Provider<Student> providesSyudentProvider;

  private Provider<Data> providesDataProvider;

  private MembersInjector<StudentActivity> studentActivityMembersInjector;

  private DaggerStudentComponent(Builder builder) {
    assert builder != null;
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {

    this.providesSyudentProvider =
        new dagger.internal.Factory<Student>() {
          private final AppComponent appComponent = builder.appComponent;

          @Override
          public Student get() {
            return Preconditions.checkNotNull(
                appComponent.providesSyudent(),
                "Cannot return null from a non-@Nullable component method");
          }
        };

    this.providesDataProvider =
        DoubleCheck.provider(StudentModule_ProvidesDataFactory.create(builder.studentModule));

    this.studentActivityMembersInjector =
        StudentActivity_MembersInjector.create(providesSyudentProvider, providesDataProvider);
  }

  @Override
  public void inject(StudentActivity activity) {
    studentActivityMembersInjector.injectMembers(activity);
  }

  public static final class Builder {
    private StudentModule studentModule;

    private AppComponent appComponent;

    private Builder() {}

    public StudentComponent build() {
      if (studentModule == null) {
        this.studentModule = new StudentModule();
      }
      if (appComponent == null) {
        throw new IllegalStateException(AppComponent.class.getCanonicalName() + " must be set");
      }
      return new DaggerStudentComponent(this);
    }

    public Builder studentModule(StudentModule studentModule) {
      this.studentModule = Preconditions.checkNotNull(studentModule);
      return this;
    }

    public Builder appComponent(AppComponent appComponent) {
      this.appComponent = Preconditions.checkNotNull(appComponent);
      return this;
    }
  }
}
```

我们看看内部类Builder中, 有AppComponent成员变量, 因为我们依赖了AppComponent, 所以在创建StudentComponent的实例时, Builder强制要求我们调用 appComponent(AppComponent)方法时把StudentComponent依赖的AppComponent实例传递进来, 接下来, Student实例会通过传递进来的appComponent实例, 进行赋值

我们来看看SubComponent为什么不用显式暴露

```java
private final class StudentComponentImpl implements StudentComponent {
  private final StudentModule studentModule;

  private Provider<Data> providesDataProvider;

  private MembersInjector<StudentActivity> studentActivityMembersInjector;

  private StudentComponentImpl() {
    this.studentModule = new StudentModule();
    initialize();
  }

  @SuppressWarnings("unchecked")
  private void initialize() {

    this.providesDataProvider =
        DoubleCheck.provider(StudentModule_ProvidesDataFactory.create(studentModule));

    this.studentActivityMembersInjector =
        StudentActivity_MembersInjector.create(
            DaggerAppComponent.this.providesStudentProvider, providesDataProvider);
  }

  @Override
  public void inject(StudentActivity activity) {
    studentActivityMembersInjector.injectMembers(activity);
  }
}
```

上面的StudentComponentImpl是StudentComponent具体实现类, 它是AppComponent实现类的内部类, 这样要实现复用就不用显式暴露接口了, 利用内部类的特性直接访问

经过我们上面分析, 我们也可以回答设计的区别了,  因为SubComponent对应的实现类会作为父Component的内部类, 这样两个Component的内聚性也就增强了, 而dependencies则会生成两个对应的分开的两个类, 通过暴露接口和传递实例进行沟通, 从而达到两个Component独立分离的目的

### 两者的使用场景
* 如果你想让两个Component的内聚性更强, 你应该用SubComponent
* 如果你想让两个Component彼此独立分离, 你应该用dependencies

### 组织方式

#### 利用Singleton标注AppComponent
Singleton实现单例必须保证Component只会被初始化一次, 那么我们可以把需要定义成单例的类都定义在AppModule, 并且在AppComponent暴露对应单例给依赖的Component, 并在Application初始化AppComponent, 这样就保证了AppComponent只初始化一次

#### 自定义Scope标注Activity
为了更好的组织项目结构, 我们可以定义PerAc(名字随意)注解, 标注基类ActivityComponent并且依赖与AppComponent, 接下来, 每个Activity可以定义对应的XXActivityComponent, 都让每个XXXActivity继承基类ActivityComponent, 并且依赖于AppComponent

#### SubComponent
我们的Activity可以能会有Fragment, 那Fragment对应的Component是要依赖于AppComponent还是做为对应ActivityComponent的SubComponent?
由于Fragment是包含在Activity中的, 更好的做法是将FragmentComponent作为对应的ActivtyComponent的SubComponent

### 告别Dagger2
写了三篇Dagger2的文章, 总算对Dagger2了解多了, 现在用得了很踏实.  所以该暂时告别Dagger2的学习了, 不过, 最后我还准备了一个Dagger2的使用的Demo, 这个Demo准备用MVP + Dagger2 + RxJava实现一个知乎日报, 嘻嘻, 如果你喜欢的话, 手抖给个star咯,

Demo在项目地址的RxJava+MVP+Dagger2分支中

[项目地址](https://github.com/cristianoro7/Daily/tree/RxJava+MVP+Dagger2)

### 参考资料
> https://guides.codepath.com/android/Dependency-Injection-with-Dagger-2#setup
> http://jellybeanssir.blogspot.jp/2015/05/component-dependency-vs-submodules-in.html
> http://stackoverflow.com/questions/29587130/dagger-2-subcomponents-vs-component-dependencies
> https://google.github.io/dagger/users-guide.html
