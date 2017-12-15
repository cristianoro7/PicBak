---
layout: post
title: Java内部类整理
data: 2017/12/15
subtitle: ""
description: ""
tags:
  - Java基础
categories:
  - Java
---

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fmhu5d40r1j30m80b41ck.jpg)

> ### 内部类的定义

将一个类置于另外一个类内部, 这就是内部类.

> ### 内部类存在的意义

* 提供一种代码隐藏机制: 因为它允许将逻辑相关的类组织到一起, 并且可以控制内部类的可见性.

* 有效地实现了"多重继承": 每个内部类都能独立地继承一个普通类, 抽象类或者实现一个接口.

> ### 内部类的特性:

* 内部类可以拥有多个实例, 每个实例都拥有自己的状态信息.

* 在单个外围类内, 可以让多个内部类以不同的方式实现同一个接口或者继承同一个类.

> ### 内部类的分类

* 成员内部类

* 静态内部类

* 匿名内部类

* 局部内部类

> #### 成员内部类

成员内部类指的是: 像定义成员变量地定义一个类,例如:

```java
public class OuterClass {
    public class InnerClass {

    }
}
```

下面通过一个例子来说明创建成员内部类的两种方法以及它的一些特性:

```java
public interface Selector {

    boolean end();

    Object current();

    void next();

}

public class Sequence {

    private Object[] items;

    private int next = 0;

    public Sequence(int size) {
        items = new Object[size];
    }

    public void add(Object x) {
        if (next < items.length) {
            items[next++] = x;
        }
    }

    public class SequenceSelector implements Selector {

        private int i = 0;

        @Override
        public boolean end() {
            return i == items.length;
        }

        @Override
        public Object current() {
            return items[i];
        }

        @Override
        public void next() {
            if (i < items.length) {
                i++;
            }
        }
    }
}

public class Main {

    public static void main(String[] args) {
        // write your code here
        Sequence sequence = new Sequence(10);
        for (int i = 0; i < 10; i++) {
            sequence.add(i);
        }
        Selector selector = sequence.new SequenceSelector();
        while (!selector.end()) {
            System.out.println(selector.current().toString() + "");
            selector.next();
        }
    }
}
```

这个例子是一个简单的迭代器, ``SequenceSelector``作为内部类实现了``Selector``接口. 然后在``Main``中创建了内部类. 这个例子我们需要关注的有两点:

* ``SequenceSelector``内部类具有访问其外围类的成员信息的权限.

* 创建一个成员内部类时, 必须要先拥有外围类的实例才能创建.

成员内部类之所以能访问外围类的信息是: 因为在创建内部类时, 编译器会将外围类实例作为参数传入内部类中, 这个过程对程序员是不可见的. 这也解释了为什么创建内部类时, 需要先获得外围类的实例.

由于成员内部类是建立在外围类的层次上的, 所以, 成员内部类不能含有``static``字段或者``static``方法.

创建成员内部类的方式: xxx.new InnerClass. 但是这样的方式不怎么优雅, 这种方法使得外部还是可以感知内部类的存在. 更好的做法是:

```java
public class Sequence {

    private Object[] items;

    private int next = 0;

    public Sequence(int size) {
        items = new Object[size];
    }

    public void add(Object x) {
        if (next < items.length) {
            items[next++] = x;
        }
    }

    private class SequenceSelector implements Selector {

        private int i = 0;

        @Override
        public boolean end() {
            return i == items.length;
        }

        @Override
        public Object current() {
            return items[i];
        }

        @Override
        public void next() {
            if (i < items.length) {
                i++;
            }
        }
    }

    public Selector selector() {
        return new SequenceSelector();
    }
}

public class Main {

    public static void main(String[] args) {
        // write your code here
        Sequence sequence = new Sequence(10);
        for (int i = 0; i < 10; i++) {
            sequence.add(i);
        }
        Selector selector = sequence.selector();
        while (!selector.end()) {
            System.out.println(selector.current().toString() + "");
            selector.next();
        }
    }
}
```

上面改动的地方有两处: 将内部类的权限改为``private``. 添加``selector()``方法来创建内部类.

经过上面的改动, 外部不能感知``SequenceSelector``的存在, 这样就防止了外部对其进行依赖, 并且完全隐藏了实现的细节.

> #### 静态内部类

与成员内部类相反, 静态内部类指的是将内部类声明为``static``, 它与外围类之间没有什么联系, 因此, 静态内部类不能访问外围类的非``static``字段或者方法.

静态内部类与成员内部类的另外一个区别: 静态内部类可以含有``static``字段或者``static``方法. 下面是一个例子

```java
public interface Contents {
    int value();
}

public class Parcel {

    private static class ParcelContents implements Contents {

        private int v = 11;

        @Override
        public int value() {
            return v;
        }
    }

    public Contents contents() {
        return new ParcelContents();
    }
}
```


在``ParcelContents``不需要对外围类进行引用, 所以将``ParcelContents``可以声明为静态内部类.

> #### 匿名内部类

匿名内部类是指创建一个继承或者实现XX的类并且它是没有名字的, 例如:

```java
public class Test {

    public Contents contents() {
        return new Contents() { //匿名内部类
            private int i = 11;

            @Override
            public int value() {
                return i;
            }
        };
    }
}
```

由于匿名类是没有名字的, 所以如果我们需要做一些构造器行为的时候便无能为力(因为匿名内部类没有名字). 这时候, 我们可以用实例初始化块来解决这个问题:

```java
public abstract class Base {
    public Base(int i) {
        System.out.println("Base constructor: " + i);
    }

    public abstract void func();
}

public class AnonymousConstructor {

    public static Base getBase(int i) {
        return new Base(i) {

            {
                System.out.println("run after");
            }

            @Override
            public void func() {
                System.out.println("func");
            }
        };
    }

    public static void main(String[] args) {
        Base base = getBase(11);
        base.func();
    }
}

输出:
Base constructor: 11
run after
func
```

> #### 局部内部类

局部内部类是指: 将一个类定义在一个方法或者是方法内的一个代码块.

局部内部类不能有访问说明符, 因为它不是外围类的一部分. 但是它可以访问当前代码块内的``常量``以及外围类的所有成员.

```java
public class Test {

    private int x = 1;

    public  Base fun1() {
        final int aFinal = 1;
        class LocalClass extends Base{

            public LocalClass(int i) {
                super(i);
            }

            @Override
            public void func() {
                System.out.println(aFinal); //访问方法内的常量
            }
        }
        return new LocalClass(x); //访问成员变量
    }
}
```

> ### 内部类的继承

因为成员内部类会默认持有外部类的引用, 因此, 当继承一个内部类时, 编译器强制要求继承内部类的类中必须要有一个构造器, 这个构造器必须将外部类作为参数, 并且调用外部类的``super``方法

```java
public class Test {

    public class ExBase extends Base {

        public ExBase(int i) {
            super(i);
        }

        @Override
        public void func() {

        }
    }
}

public class ExtendsTest extends Test.ExBase {

    public ExtendsTest(Test sequence) {
        sequence.super(11);
    }
}
```

> ### 内部类是不能被覆盖的

当继承一个类时, 被继承的类的内部类是不能被覆盖的, 两个类中的两个内部类是完全独立的两个实体, 各自在自己的命名空间内.

```java
public class Egg {

    private Yolk yolk;

    protected class Yolk {
        public Yolk() {
            System.out.println("egg.yolk");
        }
    }

    public Egg() {
        System.out.println("new egg");
        yolk = new Yolk();
    }
}

public class BigEgg extends Egg {

    public class Yolk {
        public Yolk() {
            System.out.println("BigEgg.yolk");
        }
    }

    public static void main(String[] args) {
        new BigEgg();
    }
}

输出:
new egg
egg.yolk
```

这个例子说明被继承的类的内部类是不能被覆盖的, 两个类中的两个内部类是完全独立的两个实体.
