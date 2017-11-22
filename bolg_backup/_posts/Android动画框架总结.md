---
layout: post
title: Android动画框架总结
data: 2017/11/21
subtitle: ""
description: ""
tags:
  - Android#自定义View 
categories:
  - Android
---

![](http://ww1.sinaimg.cn/large/006VdOYcgy1flpsxet776j30m80b4tf9.jpg)

> ### 概述

在``Android``中, 动画分为两种: ``Animation``和``Transition``. 其中``Animation``分为``View Animation``和``Property Animation``.

``Transition``是用于``Activity``和``Fragment``的转场动画.

``View Animation``的作用对象是整个``View``对象, 它只支持4种动画效果: 平移, 缩放, 旋转和透明度动画. 另外, ``View Animation``的一种特殊表现为: 逐帧动画. 它是通过缓慢的播放一组图像来到达一种动画效果. 需要注意的是: 由于逐帧动画是播放一组图像, 如果图片太大的话, 容易导致``OOM``.

``Property Animation``中文意思为: 属性动画. 它是``Android 3.0``推出的全新动画. 它主要是为了弥补``View Animation``的缺陷. ``View Animation``只能支持``View``的四中动画. 除了这四种外的动画, ``View Animation``都无能为力. 例如: 颜色的渐变动画.

下面主要总结一下``Android``中的属性动画.

> ### 动画的简单原理

假设现在有一个``View``, 它的``X``坐标为0. 如果我们想让它移动到``X``坐标为100的地方. 我们可以调用``view.setTranslationX()``. 但是出来的效果是: ``View``在一瞬间跳跃到了100那里, 给人一种非常突然的感觉. 基于这种情况, 我们要让``View``移动到100的地方这个过程更符合人类的感官. 所以我们可以在一定的时间内, 每一帧改变``View``的位置, 当最后一帧的时候, 让``View``处于最终位置. 这样会比较符合人类的感官.

简单的来讲, 动画的整个过程可以这么概括: 给定一个动画完成的时间``S``, 动画每一帧的时间``Sa``, 根据经过的帧数来计算出动画完成的时间度(n*Sa / S). 接着根据这个时间完成度, 计算出动画完成度``D``. 最后根据``D``来计算出此时对应的属性为多少, 然后根据这个值来重新绘制.

时间完成度通过一个``TimeInterpolator``来计算出来动画的完成度. 而动画完成度通过``TypeEvaluator``计算出此刻的属性.


> ### 属性动画API

在使用属性动画时, 我们可以用这三个类:

* ``ValueAnimator``: 最基础的动画类, 它只负责根据时间完成度来计算出动画完成度. API最难使用, 但是最为灵活.

* ``ObjectAnimator``: ``ValueAnimator``的子类, 对``ValueAnimator``进一步的封装. 内部会根据传入的属性值来调用其对应的``set``和``get``方法来更新界面. API比较简单, 但是灵活性不如``ValueAnimator``.

* ``ViewPropertyAnimator``: 针对``Android View``控件的动画的封装. 该类的使用场景: 多个动画需要同时进行. 该类内部会只调用一个重绘操作. 这也是该类的一个主要优化. 它的API调用最为简单. 但是只支持``View``的四种动画, 灵活性不大.

> ### ViewPropertyAnimator

#### 使用

``ViewPropertyAnimator``的使用最为简单:

* 执行单个动画:

```java
view.animate().translationX(50);  //向右平移50
```
* 多个动画同时执行:

```java
view.animate()
    .setDuration(400)
    .setInterpolator(new LinearInterpolator())
    .translationX(50)
    .scaleX(0.5f);
```

由于``ViewPropertyAnimator``在播放多个动画时, 只支持同时播放, 所以播放同时播放多个动画时, 各个动画是共享``Interpolator``和播放时间的.

#### 设置监听器

``ViewPropertyAnimator``支持设置三种类型的监听器:

* ``ViewPropertyAnimator.setListener()``

* ``ViewPropertyAnimator.setUpdateListener()``

* ``ViewPropertyAnimator.withStartAction/EndAction()``

```java
propertyAnimator.setListener(new Animator.AnimatorListener() {
                            @Override
                            public void onAnimationStart(Animator animation) {
                              //动画开始时被回调
                            }

                            @Override
                            public void onAnimationEnd(Animator animation) {
                              //动画完成的时候被回调, 注意: 当动画被取消后, 该方法也会回调.
                            }

                            @Override
                            public void onAnimationCancel(Animator animation) {
                              //动画被取消的时候被会回调
                            }   

                            @Override
                            public void onAnimationRepeat(Animator animation) {
                              //动画重复执行的时候被回调
                            }
                        });

propertyAnimator.setListener(new AnimatorListenerAdapter() {
                          //如果不想重写那么多方法的话, 可以利用这个适配器
                          //根据需要重写对应的方法
                        });
```

```java
propertyAnimator.setUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                          @Override
                          public void onAnimationUpdate(ValueAnimator animation) {
                          //动画更新时被调用, 默认更新的时间为10ms
                          }});
```

```java
propertyAnimator.withEndAction(new Runnable() {
                            @Override
                            public void run() {
                                //动画结束的时候被调用(被取消的时候不会被调用), 该监听器是一次性的, 只要被调用过一次的话, 后面就不会再被调用了
                            }
                        });

                        propertyAnimator.withEndAction(new Runnable() {
                            @Override
                            public void run() {
                              //动画开始的时候被调用, 该监听器是一次性的, 只要被调用过一次的话, 后面就不会再被调用了
                            }
                        });
```

#### 适用场景

``ViewPropertyAnimator``的动画只支持``View``的四种动画. 还有只支持多个动画同时播放. 所以它的适用场景:

* 需要的动画类型为: ``View``的四种动画.(因此API调用简单)

* 需要的是``View``动画类型, 并且多个动画同时播放. (因此内部会只进行一次重绘的优化)

> ### ObjectAnimator

如果你有一个自定义``View``, 它是一个自定义的圆形进度条. 你想对它的进度进行动画, 但是进度的类型为``float``, 因此, ``ViewPropertyAnimator``不能胜任. 此时就要用``ObjectAnimator``了.

``ObjectAnimator``支持对任何类型值进行动画. 但是有一个前提: 在自定义``View``的内部添加``setter``和``getter``方法.

```java
ObjectAnimator animator = ObjectAnimator.ofFloat(view, "progress", 0, 75);
animator.setInterpolator(new LinearInterpolator());
animator.start();
```

#### Interpolator

前面说过, ``Interpolator``是计算动画完成度的依据. 它本质上是一个算法, 动画的完成度由时间完成度根据这个算法算出来的.

系统自带了它的很多子类. 这里不多说, 可以参考这[篇文章](http://hencoder.com/ui-1-6/)

#### 监听器

``ObjectAnimator``支持三种监听器

* ``ObjectAnimator.addListener()``

* ``ObjectAnimator.addUpdateListener()``

* ``ObjectAnimator.addPauseListener()``

```java
animator.addListener(new Animator.AnimatorListener() {
                    @Override
                    public void onAnimationStart(Animator animation) {
                      //动画开始时被回调
                    }

                    @Override
                    public void onAnimationEnd(Animator animation) {
                      //动画完成的时候被回调, 注意: 当动画被取消后, 该方法也会回调.
                    }

                    @Override
                    public void onAnimationCancel(Animator animation) {
                      //动画被取消的时候被会回调
                    }

                    @Override
                    public void onAnimationRepeat(Animator animation) {
                      //动画重复执行的时候被回调
                    }
                });

animator.addListener(new AnimatorListenerAdapter() {
                  //如果不想重写那么多方法的话, 可以利用这个适配器
                  //根据需要重写对应的方法
                });
```

```java
animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                    @Override
                    public void onAnimationUpdate(ValueAnimator animation) {
                      //动画更新时被调用, 默认更新的时间为10ms
                    }
                });
```

```java
animator.addPauseListener(new Animator.AnimatorPauseListener() {
                    @Override
                    public void onAnimationPause(Animator animation) {
                        //动画被暂停的时候回调
                    }

                    @Override
                    public void onAnimationResume(Animator animation) {
                        //动画被恢复的时候调用
                    }
                });
```

#### 适用场景

* 当动画类型不是``View``的四种动画类型, 并且你有权利给你的``View``添加``setter``和``getter``

但是如果你没有权利给``View``添加``setter``和``getter``方法呢? 比如你正在用一个第三方的``View``. 这时候有两种选择:

* 用一个类来包装你的``View``, 为包装``View``添加``setter``和``getter``方法.

* 使用``ValuesAnimator``来自己手动更新.


> ### ValuesAnimator

``ValuesAnimator``是``ViewPropertyAnimator``和``ObjectAnimator``的父类, 这两个类都是对``ValuesAnimator``的进一步封装, 以此来提供更为便利的``API``调用. 因此这两个类的灵活性也下降了. 当上面的两个类都不能满足你的需求时, 你可以使用``ValuesAnimator``, 让它帮你计算每一帧后的属性值, 然后你监听这个属性的变化, 最后设置对应的属性, 并且重绘``View``.

```java
ValueAnimator valueAnimator = ValueAnimator.ofInt(0, 100);
valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
              @Override
              public void onAnimationUpdate(ValueAnimator animation) {
                        int value = (int) animation.getAnimatedValue();
                        //设置 value属性, 并且重绘View
              }
});
valueAnimator.start();
```

从上面的代码, 我们发现, ``ValueAnimator``的``API``调用相比前两个类, 要复杂一点, 但是它最为灵活. 前两个类只不过是对``ValueAnimator``的封装, 以便提供更为方便的``API``.

> ### TypeEvaluator

``TypeEvaluator``为估值器, 前面说到, 算出动画的完成度后, 我们可以根据这个动画完成度来计算出当前属性的值. 那怎么算呢? ``TypeEvaluator``就是来完成这个计算工作的.

在``ObjectAnimator``中, 我们常用的是``ofInt``和``ofFloat``方法, 但是如果你的属性是一个对象呢? 比如``PointF``对象, 这时, 应该使用``ofObject``方法.

下面以官方提供的一个``PontFEvaluator``为例.

```java
ObjectAnimator.ofObject(view, "point", new PointFEvaluator(), new PointF(0, 0),
                       new PointF(100, 100));

public class PointFEvaluator implements TypeEvaluator<PointF> {

  private PointF mPoint;


  public PointFEvaluator() {}

  public PointFEvaluator(PointF reuse) {
      mPoint = reuse;
  }

  @Override
  public PointF evaluate(float fraction, PointF startValue, PointF endValue) {
      float x = startValue.x + (fraction * (endValue.x - startValue.x)); //根据动画完成度算出X坐标
      float y = startValue.y + (fraction * (endValue.y - startValue.y)); //根据动画完成度算出Y坐标

      if (mPoint != null) {
          mPoint.set(x, y);
          return mPoint; //返回对应的新对象
      } else {
          return new PointF(x, y);
      }
  }
}
```

> ### 多个属性的动画同时执行

``ViewPropertyAnimator``调用多个属性进行动画时, 是同时进行的. 那么对于``ObjectAnimator``呢? 它如何让多个属性进行配合? 答案是使用``PropertyValuesHolder``. 用法如下:

```java
PropertyValuesHolder holder1 = PropertyValuesHolder.ofFloat("scaleX", 1.5f);
PropertyValuesHolder holder2 = PropertyValuesHolder.ofFloat("scaleY", 1.5f);
PropertyValuesHolder holder3 = PropertyValuesHolder.ofFloat("alpha", 0.4f);

ObjectAnimator objectAnimator = ObjectAnimator.ofPropertyValuesHolder(view, holder1, holder2, holder3);
objectAnimator.start();
```

> ### 多个动画配合执行

有时候, 我们需要的效果可能是多个动画配合执行, 这时候, 我们可以用``AnimatorSet``.

```java
ObjectAnimator animator1 = ObjectAnimator.ofInt(...);
ObjectAnimator animator2 = ObjectAnimator.ofInt(...);
ObjectAnimator animator3 = ObjectAnimator.ofInt(...);

AnimatorSet set = new AnimatorSet();

set.playSequentially(animator1, animator2, animator3); //按顺序播放
set.start();

set.playTogether(animator1, animator2); //同时播放
set.start();

set.play(animator1).with(animator2);
set.play(animator2).after(animator3); //进行精准控制播放
set.start();
```

> ### 总结

* 当你的动画类型是``View``四种动画的其中一种或者两种以上的话, 优先使用``ViewPropertyAnimator`` 因为``API``调用很简单.

* 当``ViewPropertyAnimator``不能满足你的需求时, 使用``ObjectAnimator``.

* 如果使用的``View``的属性没有对应的``setter``和``getter``方法的话, 可以使用``ValueAnimator``.

> ### 参考资料

[HenCoder](http://hencoder.com/ui-1-6/)

Android开发艺术探索
