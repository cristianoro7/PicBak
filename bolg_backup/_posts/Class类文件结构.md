---
layout: post
title: 类文件结构
data: 2017/5/11
subtitle: ""
description: ""
thumbnail: "/img/jvm/内存分配策略.jpg"
tags:
  - Java#JVM
categories:
  - Java
---

# 类文件结构

> ## 目录

* Class类文件结构
 * 特殊字符串概念
 * 魔数与Class文件的版本
 * 常量池
 * 访问标志
 * 类索引,父类索引与接口索引集合
 * 字段表集合
 * 方法表集合
 * 属性表集合

> ### 类文件结构

Class文件结构只有两种数据类型:无符号和表.
无符号数属于基本的数据类型,以u1,u2,u4,u8分别代表1个字节,2个字节,4个字节和8个字节的无符号数,它用来描述数字,索引引用,数量值或者UTF-8编码构成字符串值.
表是由多个无符号数或者其他表作为数据项构成的复合数据类型,所有表都习惯以"_info" 结尾
整个class文件本质就是一张表:

| 类型 | 名称 | 数量 |
| ------ | ------ | ------ |
| u4 | magic | 1 |
| u2 | minor_version | 1 |
| u2 | major_version | 1 |
| u2 | constant_pool_count | 1 |
| cp_info | constant_pool | constant_pool_count |
| u2 | access_flag | 1 |
| u2 | this_class | 1 |
| u2 | super_class | 1 |
| u2 | interface_count | 1 |
| u2 | interface | interface_count |
| u2 | fields_count | 1 |
| field_info | fields | fields_count |
| u2 | methods_count | 1 |
| method_info | methods | methods_count |
| u2 | attributes_count | 1 |
| attribute_info | attributes | attributes_count |

#### 特殊字符串的概念

* 全限定名: 把类的全名中的"."替换成"/",最后再加上";"
* 简单名称: 没有类型和参数修饰的方法或者字段名称.
* 方法和字段的描述符: 描述符的作用是用来描述字段的数据类型,方法参数列表和返回值.
* 描述符标识字符含义:

| 标识字符 | 含义 |
| ------ | ------ |
| B | 基本类型byte |
| C | 基本类型char |
| D | 基本类型double |
| F | 基本类型float |
| I | 基本类型int |
| J | 基本类型long |
| S | 基本类型short |
| Z | 基本类型boolean |
| V | 特殊类型void |
| L | 对象类型 |

#### 魔数与Class文件版本

* 每个Class文件的头4个字节称为魔数,它的唯一作用是确定这个文件是否为一个能被虚拟机接受的class文件.
* 紧接魔数后面的4个字节储存的是Class文件的版本号: 第5,6个字节是此版本号(Minor Version),第7和第8个字节是主版本号.JDK的版本号是从45开始的.

#### 常量池

* 常量池可以理解为Class文件之中的资源仓库,它同时是Class文件关联其他项目最多的数据类型,也是占用Class文件空间最大的数据项目之一.
* 由于常量池中的常量是不固定的,所以需要在常量池入口放置一项u2类型的数据,代表常量池容量计数值.
* 常量池中主要储存两大类常量:字面量和符号引用．字面量比较接近Java语言层面的常量概念,如字符串,声明为final常量值等. 而符号引用则属于编译原理方面的概念,主要包含了下面三类常量:
 * 类和接口的全称限定名
 * 字段的名称和描述符
 * 方法的名称和描述符
* 常量池的项目类型:

| 类型 | 标志 | 描述 |
| ------ | ------ | ------ |
| CONSTANT_Utf_info | 1 | UTF-8编码的字符串 |
| CONSTANT_Integer_info | 3 | 整型字面量 |
| CONSTANT_Float_info | 4 | 浮点型字面量 |
| CONSTANT_Long_info | 5 | 长整类型字面量 |
| CONSTANT_Double_info | 6 | 双精度浮点型字面量 |
| CONSTANT_Class_info | 7 | 类或接口的符号引用 |
| CONSTANT_String_info | 8 | 字符串类型字面量 |
| CONSTANT_Fieldref_info | 9 | 字段的符号引用 |
| CONSTANT_Methodref_info | 10 | 类中方法的符号引用 |
| CONSTANT_InterfaceMethodref_info | 11 | 接口中方法的符号引用 |
| CONSTANT_NameAndType_info | 12 | 字段或者方法的部分符号引用 |
| CONSTANT_MethodHandle_info | 15 | 表示方法句柄 |
| CONSTANT_MethodType_info | 16 | 标识方法类型 |
| CONSTANT_InvokeType_info | 18 | 表示一个动态方法调用点 |

#### 访问标志

* access_flags用于识别一些类或者接口层次的访问信息.例如:这个class是类还是接口;是否定义为public类型;是否定义为abstract类型等. 具体标记:

| 标志名称 | 标志值 | 含义 |
| ------ | ------ | ------ |
| ACC_PUBLIC | 0x0001 | 是否为public类型 |
| ACC_FINAL | 0x0010 | 是否被声明为final,只有类可设置 |
| ACC_SUPER | 0x0020 | 是否允许使用invokespecial字节码指令心新语意 |
| ACC_INTERFACE | 0x0200 | 标识这是一个接口 |
| ACC_ABSTRACT | 0x0400 | 是否为abstact类型, 对于接口或者抽象类来说, 这个标志为真,其他值为假 |
| ACC_SYNTHETIC | 0x1000 | 标识这个类并非由用户代码生成 |
| ACC_ANNOTION | 0x2000 | 标识这是一个注解 |
| ACC_ENUM | 0x4000 | 标识这个一个枚举 |

#### 类索引,父类索引与接口索引集合

* 类索引用于确定这个类的全限定名
* 父类索引用于确定这个类的父类的全限定名
* 接口索引集合用于描述这个类实现了哪些接口

#### 字段表集合

* 字段表集合用于描述接口或者类中声明的变量.
* 字段包括类级变量和实例级变量, 但不包括在方法内的局部变量

#### 方法表集合

* 方法表结构:

| 类型 | 名称 | 数量 |
| ------ | ------ | ------ |
| u2 | access_flags | 1 |
| u2 | name_index | 1 |
| u2 | descriptor | 1 |
| u2 | attributes_count | 1 |
| attribute_info | attributes | attributes_count |

* 方法表集合结构跟字段表结构大致相同
* 如果父类分方法没有被子类重写,方法表集合中就不会出现父类方法的信息.

#### 属性表集合

* 虚拟机规范预定义的属性

| 属性名称 | 使用位置| 含义 |
| ------ | ------- | ------- |
| Code | 方法表 | Java代码编译成的字节码指令 |
| ConstantValue | 字段表 | final关键字定义的常量值 |
| Deprecated | 类,方法表字段表 | 被声明为deprecated的方法和字段 |
| Exceptions | 方法表 | 方法抛出的异常 |
| EbcloseingMethod | 类文件 | 仅当一个类为局部类或者匿名类时才能拥有这个属性,这个属性用于标识这个类所在的外围方法. |
| InnerClasses | 类文件 | 内部类列表 |
| LineNumberTable | Code属性 | Java源码的行号与字节码指令对应的关系 |
| LocalVariableTable | Code属性 | 方法的局部属性描述 |
| StackMapTable | Code属性 | JDK1.6中新增的属性, 供新的类型检查验证器检查和处理目标方法的局部变量和操作数栈所需要的类型是否匹配. |
| Signature | 类,方法表,字段表 |

待续...
