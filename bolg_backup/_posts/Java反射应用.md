---
title: java反射的应用
data: 2017/11/25
tags:
  - Java#基础
categories:
  - Java
---

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fluguov7y5j30m80b4wux.jpg)

> ### 什么是代理模式

代理模式是在不改变被代理类的代码的情况下, 对被代理的方法进行扩展, 这些扩展可以是打印日志, 控制访问等.

代理模式又分为静态代理和动态代理. 静态代理是指代理类在编译时, 就能够生成字节码被``JVM``识别. 而动态代理则是在运行时, 运用反射生成代理类字节码,最后被``JVM``识别.

#### 静态代理

静态代理比较简单, 下面给出一个例子, 然后分析它的缺点.

```java
public interface Student {

    void goToSchool();

}

public class XiaoMing implements Student {

    @Override
    public void goToSchool() {
        System.out.printf("I am XiaoMing, I`am going to school!");
    }
}

public class XiaoMingProxy implements Student {

    private Student student;

    public XiaoMingProxy(Student student) {
        this.student = student;
    }

    @Override
    public void goToSchool() {
        System.out.println("goodbye mom!");
        student.goToSchool();
    }
}

public class Main {

    public static void main(String[] args) {
        XiaoMingProxy proxy = new XiaoMingProxy(new XiaoMing());
        proxy.goToSchool();
    }
}
```

``XiaoMingProxy``代理对象``XiaoMing``, 在他去上学之前打印日志.

上面就是一个简单的静态代理. 静态代理, 有一个很明显的缺点: 必须我们自己手动写代理类, 并且``代理类的复用行不强``. ``代理类复用性不强``指的是: 本来打印日志这个功能是通用的, 但是静态代理的代理类不能被其他类复用. 比如现在我们添加一个教师类, 我们也要给她添加一个打印日志的功能, 那么我们必须重新写代理类, 不能复用之前的代理类

```java
public interface Teacher {

    void goToSchool();
}

public class YoungTeacher implements Teacher {


    @Override
    public void goToSchool() {
        System.out.println("go to school!");
    }
}

public class TeacherProxy implements Teacher {

    private Teacher teacher;

    public TeacherProxy(Teacher teacher) {
        this.teacher = teacher;
    }

    @Override
    public void goToSchool() {
        System.out.println("goodbye mon!");
        teacher.goToSchool();
    }
}
```

原本一样的逻辑, 我们却得再次写代理类.

#### 动态代理

动态代理可以解决静态代理的缺点.  它的原理是: 在运行时, 运用反射自动帮你生成实现了接口的代理类, 并且将扩展方法的逻辑都分发到了一个``InvocationHandler``中的``invoke``方法. 这样就能实例代理方法的复用. 下面我们运用动态代理来解决之前静态代理出现的问题.

```java
public class LoggerHandler implements InvocationHandler {

    private Object target;

    @SuppressWarnings("unchecked")
    public <T> T bind(Object tar) {
        this.target = tar;
        return (T) Proxy.newProxyInstance(tar.getClass().getClassLoader(),
                tar.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("goodbye mom!");
        return method.invoke(target, args);
    }
}

public class Main {

    public static void main(String[] args) {
        LoggerHandler handler = new LoggerHandler();

        Student student = handler.bind(new XiaoMing());
        student.goToSchool();

        Teacher teacher = handler.bind(new YoungTeacher());
        teacher.goToSchool();
    }
}
```

上面是一个动态代理的例子, 我们将打印日志的功能放在``LoggerHandler``中, 这样其他的代理类就可以通过复用这个``LoggerHandler``来复用打印日志的功能.

#### 动态代理原理解析

动态代理之所以能克服静态代理的缺点, 主要是: 它是在运行时通过反射创建代理类(这个代理类实现了我们被代理类的接口), 然后返回这个代理类的引用, 每个代理类都会绑定一个``InvocationHandler``的实例. 接着每次对这个代理类的方法调用, 都会被分发到``InvocationHandler``中的``invoke``方法来统一处理. 因此``InvocationHandler``常常是我们写扩展逻辑的地方. 在``LoggerHandler``中, 它就是扩展了打印日志的功能. 这样不同被代理类如果需要扩展这个打印日志的功能的话, 就可以共享这个``LoggerHandler``来实现.

![](http://ww1.sinaimg.cn/large/006VdOYcgy1flubhp8380j30nx0bz3zp.jpg)

> ### 利用反射来获取注解

反射另外一个应用是: 在运行时获取注解信息. 下面是利用反射实现的一个``View``的绑定.

```java
@Target({ElementType.FIELD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface BindView {
    int value() default -1;
}

public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity";

    @BindView(R.id.main_tv_hello_world)
    private TextView mTvextView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ViewInjection.inject(this);
        Log.d(TAG, "onCreate: " + mTvextView.getText().toString());
    }
}

public class ViewInjection {

    public  static <T extends Activity>  void inject(T activity) {
        Class activityClass = activity.getClass();

        Field[] fields = activityClass.getDeclaredFields();
        for (Field f : fields) {
            BindView bindView = f.getAnnotation(BindView.class);
            if (bindView != null) {
                int id = bindView.value();
                View view = activity.findViewById(id);
                try {
                    f.setAccessible(true);
                    f.set(activity, view);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

> ### 反射的缺点

反射能够在运行时调用一个类的方法,或者访问其字段等, 包括被``private``修饰的字段或者方法, 这一定程度上说明了反射是具有一定的破坏性.

#### 利用反射绕开编译时的泛型限定

``Java``泛型是在编译期由编译器检查和实现的. 如果我们运用反射绕开编译期的泛型限定的话, 我们可以破坏泛型机制.

```java
public class Main {

    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        List<String> strings = new ArrayList<>();
        strings.add("xixi");
        Method method = strings.getClass().getDeclaredMethod("add", Object.class);
        method.invoke(strings, 1);
        for (String s : strings) {
            System.out.println(s);
        }
    }
}
```
在``main``方法中, 我们在运行时, 利用反射给``strings``添加了一个``int``类型, 这样就绕开了泛型的限定. 所以当遍历``strings``时,会报异常.

#### 反射破坏单例

既然反射能够访问``private``字段和``private``方法, 那么我们就可以利用反射来破坏单例.

```java
public class Singleton {

    private static volatile Singleton INSTANCE = null;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (INSTANCE == null) {
            synchronized (Singleton.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}

public class Main {

    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException, InstantiationException {
        Singleton singleton = Singleton.getInstance();
        System.out.println(singleton.hashCode());

        Class c = Singleton.class;
        Constructor constructor = c.getDeclaredConstructor(null); //获取Singleton的私有构造方法
        constructor.setAccessible(true);
        Singleton newInstance = (Singleton) constructor.newInstance(null); //然后再实例化另一个
        System.out.println(newInstance.hashCode());
    }
}
```

运行上面的代码, 最后打印出来的``HashCode``不一样, 说明单例被破坏了.
