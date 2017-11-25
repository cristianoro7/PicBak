---
title: java反射基础
data: 2017/11/24
tags:
  - Java#基础
categories:
  - Java
---

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fltld849k4j30m80b47gy.jpg)

> ### 什么是反射

在``Java``中, 每一个对象都对应有一个``Class``对象, 这个``Class``对象记录着对象的类型信息, 也就是类的内部结构.

我们知道, 我们编写的``.java``文件, 是需要被编译成``.class``文件, 然后才能被虚拟机加载执行. 正常情况下, ``.class``是在编译期生成并且被``JVM``识别. 而反射机制则将``.class``文件的打开和检查推迟到运行时.

简单的说, 反射机制能让你在运行时操作一个类, 这些操作可以是: 实例化该类的对象, 调用对象方法和修改对象的属性值. 至于操作的类的``.class``文件可以是编译期未知.

> ### 反射的概述

一个类的元素可以被拆分为下面几个元素:

* ``Class``: 一个类对象

* ``Constructor``: 构造器对象, 可以通过``Class``对象获得

* ``Method``: 方法对象, 可以通过``Class``对象获得

* ``Fields``: 字段对象, 可以通过``Class``对象获得

* ``Annotation``: 注解对象, 可以通过``Class``对象获得

从上面的元素可以看出, 我们如果要进行反射的话, 首先得获得类的``Class``对象, 进而才可以操作类的字段, 方法和构造器等.

> ### 获取Class对象

要进行反射, 我们首先要获得需要反射的类的``Class``对象. 而获取的方法有下面三种:

* ``Class class2 = Class.forName("com.desperado.reflaction.Bean")``: 调用``Class``类的``forName(String)``方法, 参数为类的全限定符号. 如果运行时找不到该类的话, 会报``ClassNotFoundException``异常. 这种方法一般用于编译期不能获得``.class``文件的情形.

* ``Class class1 = Bean.class``: 通过类字面量来获取``Class``对象.

* ``Bean bean = new Bean(); Class class3 = bean.getClass();``: 通过对象来获取``Class``对象.

> ### 获取Constructor

得到``Class``对象后, 我们可以利用这个``Class``对象来获取类的构造函数, 以此来创建实例.

获取构造方法的方法有:

* ``getConstructors()``: 获取所有被``public``修饰的构造方法.

* ``getConstructor(Class<?>... parameterTypes)``: 返回指定参数类型的构造方法, 并且构造方法是被``public``修饰的.

* ``getDeclaredConstructors()``: 获取所有的构造方法, 与权限修饰符无关.

* ``getDeclaredConstructor(Class<?>... parameterTypes)``: 获取指定类型的构造方法, 与权限修饰符无关.

> #### 创建实例

```java
Class beanClass = Bean.class;
Bean  b = (Bean) beanClass.newInstance();
```

``newInstance()``这个方法会调用类无参的构造函数, 如果类没有无参构造函数的话, 会报异常.

```java
public class Human {

    private String name;

    public int age;

    public Human() {}

    private Human(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public Human(String name) {
        this.name = name;
    }
}

Class<?> c = Human.class;
Constructor constructor = c.getConstructor(String.class);
Human human = (Human) constructor.newInstance("xiaohong");

Constructor priCon = c.getDeclaredConstructor(String.class, int.class);
priCon.setAccessible(true);
Human h = (Human) priCon.newInstance("xiaohong", 18);
```

利用``getConstructor``方法, 调用了``Human``中带有``String``参数的构造方法.``getDeclaredConstructor``可以获取私有构造方法.

> ### 获取Method

利用``Class``对象, 我们可以获得``Method``来调用Class的方法. 获得``Method``的方法有:

* ``getMethods()``: 获取所有被``public``修饰的方法, 包含父类和接口中的方法

* ``getMethod(String name, Class<?>... parameterTypes)``: 获取指定``name``的``public``方法. 包含父类的方法.

* ``getDeclaredMethods()``:  获取所有的方法, 包括``private``. 但是不包括父类的方法

* ``getDeclaredMethod(String name, Class<?>... parameterTypes)``: 获取指定的方法, 包括``private``. 但是不包括父类的方法.

> #### getMethods和getMethod

```java
public class Human {

    private String name;

    private int age;

    public Human() {}

    private Human(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public Human(String name) {
        this.name = name;
    }

    public void humanPubMethod(){
        System.out.println("humanPubMethod");
    }

    private void humanPriMethod() {
        System.out.println("humanPriMethod");
    }
}

public class XiaoHong extends Human {

    private String name = "xiaohong";

    public XiaoHong(String name) {
        this.name = name;
    }

    private void priMethod() {
        System.out.println("priMethod");
    }
}

public static void main(String[] args) throws IllegalAccessException,
              InstantiationException, NoSuchMethodException,InvocationTargetException {

    Class<?> c = XiaoHong.class;
    Constructor constructor = c.getConstructor(String.class);
    XiaoHong xiaoHong = (XiaoHong) constructor.newInstance("xiaohong");

    Method method = c.getMethod("humanPubMethod", null);
    method.invoke(xiaoHong, null); //调用方法

    Method[] methods = c.getMethods();
    for (Method m : methods) { //遍历方法
        System.out.println(m.getName());
    }
}

```

> #### getDeclaredMethod和getDeclaredMethods

```java
public static void main(String[] args) throws IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {

    Class<?> c = XiaoHong.class;
    Constructor constructor = c.getConstructor(String.class);
    XiaoHong xiaoHong = (XiaoHong) constructor.newInstance("xiaohong");

    Method method = c.getDeclaredMethod("priMethod", null);
    method.setAccessible(true);
    method.invoke(xiaoHong, null);

    Method[] methods = c.getDeclaredMethods();
    for (Method m : methods){
        System.out.println(m.getName());
    }
}
```

> ### 获取Fields

利用``Class``对象, 我们同样可以获得类的字段, 并且可以``set``和``get``字段的值. 我们可以通过下面的方法获取``Field``对象

* ``getFields()``: 获取类的所有被``public``修饰的字段, 包括父类中的.

* ``getField(String name)``: 获取类的指定被``public``修饰的字段, 包括父类中的.

* ``getDeclaredFields()``: 获取类的所有字段, 包括被``private``修饰的字段. 但是不包括父类中的.

* ``getDeclaredField(String name)``: 获取类的指定字段, 包括被``private``修饰的字段. 但是不包括父类中的.

> #### getFields和getField

```java
public static void main(String[] args) throws IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {

    Class<?> c = XiaoHong.class;
    Constructor constructor = c.getConstructor(String.class);
    XiaoHong xiaoHong = (XiaoHong) constructor.newInstance("xiaohong");

    Field field = c.getField("pubName");
    String pubName = (String) field.get(xiaoHong);
    System.out.println(pubName);

    Field[] fields = c.getFields();
    for (Field f : fields) {
      System.out.println(f.getName());
    }
}
```

> #### getDeclaredFields和getDeclaredField

```java
public static void main(String[] args) throws IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {

    Class<?> c = XiaoHong.class;
    Constructor constructor = c.getConstructor(String.class);
    XiaoHong xiaoHong = (XiaoHong) constructor.newInstance("xiaohong");

    Field field = c.getDeclaredField("name");
    field.setAccessible(true);
    System.out.println(field.get(xiaoHong));

    Field[] fields = c.getDeclaredFields();
    for (Field f : fields) {
      System.out.println(f.getName());
    }
}
```

> ### 获得Annotation

由于注解可以用来标注类, 方法, 字段, 参数, 所以, ``Annotation``可以通过这些来获取.下面以``Class``对象为例子. 可以通过下面两个方法获取:

* ``getAnnotation(Class<A> annotationClass)``: 获取类指定的注解

* ``getAnnotations()``: 获取类标注的所有注解

> #### getAnnotations和getAnnotation(Class<A> annotationClass)

```java
public static void main(String[] args) throws IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException, NoSuchFieldException {

    Class<?> c = XiaoHong.class;
    Constructor constructor = c.getConstructor(String.class);
    XiaoHong xiaoHong = (XiaoHong) constructor.newInstance("xiaohong");

    BindView bindView = c.getAnnotation(BindView.class);
    if (bindView != null) {
        System.out.println(bindView.value());
    }

    Annotation[] annotations = c.getAnnotations();
    for (Annotation a : annotations) {
        if (a.annotationType().equals(BindView.class)) {
            System.out.println("equal");
        }
    }
}
```

> ### 利用反射处理数组

``Java``反射机制通过``java.lang.reflect.Array``这个类来处理数组。

#### 创建数组

```java
int[] ints = (int[]) Array.newInstance(int.class, 3);
```

上面是一个创建数组的例子, ``newInstance``方法的第一个参数为: 数组存放的类型, 第二个为数组的长度.


#### 访问数组

```java
int a = Array.getInt(ints, 0);
Array.setInt(ints,1, 3);
```

对于基本类型, ``Array``中提供了``setXX``和``getXX``方法来访问数组, 对于普通对象, 可以调用``get``和``set``方法.

#### 数组Class对象

对于获取数组的``Class``对象, 可以先实例化对于的数组, 再得到它的``Class``对象

```java
Class aClass = Class.forName("com.desperado.reflaction.XiaoHong"); //先获得class对象
Class classArray = Array.newInstance(aClass, 0).getClass(); //再创建class对象数组, 最后获取数组class对象
```

> ### 反射和泛型

由于泛型会在编译期被擦除, 所以我们是不能在运行时获得泛型类的参数类型的. 但是我们却可以通过使用了参数类型的字段或者返回了带有类型参数的方法来获得一个泛型类的类型信息.

```java
public class TClass {

    private List<String> stringList = new ArrayList<>();

    public List<String> getList() {
        return stringList;
    }

    public void setList(List<String> list) {
        this.stringList = list;
    }
}
```

#### 获取方法返回值的参数类型

```java
Method method = TClass.class.getMethod("getList", null);
Type type = method.getGenericReturnType();
if (type instanceof ParameterizedType) {
    ParameterizedType parameterizedType = (ParameterizedType) type;
    Type[] ts = parameterizedType.getActualTypeArguments();
    for (Type t : ts) {
        Class c = (Class) t;
        System.out.println(c.getName());
    }
}
```

#### 获取方法参数的参数类型

```java
Field field = TClass.class.getDeclaredField("stringList");
Type t = field.getGenericType();
if (t instanceof ParameterizedType) {
    ParameterizedType pt = (ParameterizedType) t;
    Type[] types = pt.getActualTypeArguments();
    for (Type ty : types) {
        Class c = (Class) ty;
        System.out.println(c.getName());
    }
}
```

#### 获取字段的参数类型

```java
Method method = TClass.class.getMethod("setList", List.class);
Type[] type = method.getGenericParameterTypes();
for (Type parT : type) {
    if (parT instanceof ParameterizedType) {
        ParameterizedType parameterizedType = (ParameterizedType) parT;
        Type[] types = parameterizedType.getActualTypeArguments();
        for (Type ty: types) {
            Class c = (Class) ty;
            System.out.println(c.getName());
        }
    }
}
```

> ### 参考资料:

### Java核心编程思想

### http://ifeve.com/java-reflection-9-generics/
