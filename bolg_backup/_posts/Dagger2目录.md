---
layout: post
title: 重拾Dagger2
data: 2017/3/29
subtitle: "meet with dagger2"
description: "meet with dagger2"
thumbnail: "/img/dagger2/startdagger2.jpeg"
tags:
  - Android#Dagger2
categories:
  - Android
---

最近项目没什么bug,也没什么新的需求, 闲得有点慌(其实一直很闲...), 既然那么闲, 那就重新深入学习Dagger2吧. 之前虽然有学过Dagger2, 但是并没研究其中的原理, 用起来总感觉很不踏实, 于是借此机会研究了一波Dagger2, 并准备分为三篇来讲, 分别为: 使用篇, 原理篇和组织篇. 本篇的目的就指在介绍Dagger2

### Dagger2是什么?
Dagger2是之前由Square公司开源的Dagger的分支, 它运行在Android或者Java上的编译时注入依赖的框架, 关于什么是依赖注入,这里就不多说啦

### 为什么要使用Dagger2
回想一下, 在我们的项目中, 什么代码是有用的? 什么代码是没什么用但我们必须要写的? 这个问题真的值得思考.举个例子, 平常我们实现网络请求的功能, 我们为了解耦, 会写一堆工厂类将网络请求的各个组件连接起来, 在这个场景中,我们真正关注的是网络请求, 而那些将网络请求连接起来的工厂类我们其实对它并不感兴趣, 它只是我们实现网络请求的一个工具而已. 面对这样的问题, Dagger利用依赖注入的模式来替代工厂类, 将我们从编写一堆的工厂类中解放出来,从而让我们把精力放在我们感兴趣的代码

### 使用Dagger2的好处
利用Dagger2, 能让我们从一堆的引用模板代码中解放出来, 将更多的精力放在业务需求上; 由于Dagger2会帮我们自动生成需要依赖的代码, 这能极大的减少我们的工作量. 更有趣的是:Dagger2帮我们管理依被赖对象的生命周期, 比如使用Singleton注解就能实现单例, 不过这个单例实现是有条件的, 通过Dagger2管理被依赖的对象能让我们的项目结构更清晰

### 计划
接下来, 我会写一下系列的Dagger2文章

> [重拾Dagger2-使用Dagger2](https://cristianoro7.github.io/2017/09/29/%E9%87%8D%E6%8B%BEDagger2-%E4%BD%BF%E7%94%A8Dagger2/)
> [重拾Dagger2-理解Dagger2](https://cristianoro7.github.io/2017/09/29/%E9%87%8D%E6%8B%BEDagger2-%E7%90%86%E8%A7%A3Dagger2/)
> [重拾Dagger2-组织依赖注入](https://cristianoro7.github.io/2017/09/29/%E9%87%8D%E6%8B%BEDagger2-%E7%BB%84%E7%BB%87/)
