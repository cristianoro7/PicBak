---
layout: post
title: Java注解基础
data: 2017/11/22
subtitle: ""
description: ""
tags:
  - Java#基础
categories:
  - Java
---

![](http://ww1.sinaimg.cn/large/006VdOYcgy1flr4xe3496j30m80b47mn.jpg)

> ### 什么是注解

注解是一种元数据, 它能够让我们在代码中添加信息, 使得我们可以在稍后的某个时刻非常方便地使用这些数据.

> ### 使用场景

* 编译时生成一些配置文件或者部署文件

* 可以根据注解中的数据来生成模板代码, 从而减轻编写模板代码的负担

> ### Java内置的3个注解

``Java``内置了3个注解, 他们是:

* ``Override``: 表示当前方法定义将覆盖父类的方法. 如果你不小心拼写错误的话, 编译器会报警提示.

* ``Deprecated``: 如果使用了被该注解标注的元素的话, 编译器会发出警告.

* ``SuppressWarning``: 关闭编译器的报警提示.

上面的3个注解是``Java``内置的注解, 主要是帮助编译器来检查和规范代码.

> ### 自定义注解

``Java``内置的注解一般是不满足我们平常开发的需求的, 因此我们需要自己自定义注解. 下面说说用于描述注解的``元注解``, 接着再谈谈注解元素支持的类型.

> #### 元注解

``元注解``是描述注解的注解, 他们负责定义创建的注解的一些信息:

* ``@Target``: 表示该注解是用于什么地方的, 可选的字段有:
 1. ``CONSTRUCTOR``: 用于注解构造器
 2. ``FIELD``: 用于注解实例域
 3. ``METHOD``: 用于注解方法
 4. ``PACKAGE``: 用于注解包
 5. ``PARAMETER``: 用于注解参数
 6. ``TYPE``: 用于注解类, 接口(包括注解类型)或者枚举.

* ``@Retention``: 用于描述注解的声明周期, 或者说是需要在什么时候保留注解信息, 什么时候丢弃注解信息. 可选的字段有:
 1. ``SOURCE``: 注解在源文件内有效, 但是会被编译器丢弃.
 2. ``CLASS``: 注解在``.class``文件内有效, 但是会被``VM``丢弃.
 3. ``RUNTIME``: 注解在运行时有效, 因此可以在运行时通过反射来获取注解的信息.

* ``@Document``: 将此注解包含在``Javadoc``中.

* ``@Inherit``: 允许子类继承父类拥有的注解, 注意: 这里并不是说注解支持继承.

> #### 注解元素

注解内的元素可用的类型如下:

* 所有基本类型.

* ``String``

* ``Class``

* ``enum``

* ``Annotation``

* 以上类型的所有数组形式.

> #### 例子

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.CLASS)
public @interface Author {
    String date() default "";
    String[] name() default {};
    String describe() default "";
}

@Author(
        name = {"xiaoming, xiaohong"},
        describe = "注册器",
        date = "2017/11/22"
)
public class RegisterController extends HttpServlet {
    ...
}

```

``Author``是一个自定义的注解. 定义注解的语法跟定义接口很像, 只不过多了在``interface``之前加``@``符号的这个步骤. 在``Author``中, ``@Target``的元素值为:``ElementType.TYPE``和``ElementType.METHOD``, 表示该注解用于标注类或者方法. ``@Retention``的元素值为: ``RetentionPolicy.CLASS``表示在``.class``文件中仍然保持注解, 但是在运行时, 注解会被丢弃.

``Author``中的元素值都有被``default``标注, 这个关键字可以为元素值设置一个默认值. 需要注意的是, 注解是不支持不确定的值的, 所以在使用注解时, 要么给出注解的确定值, 要么在定义注解给定默认值. 注解不支持``null``. 在定义注解时, 最好都给每个元素标注要默认值, 用于表示特殊的状态. 例如, ``int``类型可以用``-1``来表示默认值; ``String``类型可以用``""``空字符串来表示默认值.

> ### 最后

定了注解后, 我们会运用反射或者注解处理器去获取注解元素的值, 然后生成一些配置文件或者模板代码.

反射是在运行时获取注解的元素值, 会比较耗性能. 所以在移动端会比较少用反射. 后端用得比较多.

注解处理器则是在编译时, 通过扫描源文件来获取注解元素的值, 不存在性能的问题, 最多是增加编译的时长. 因此, 注解处理器在移动端会用得比较多.
