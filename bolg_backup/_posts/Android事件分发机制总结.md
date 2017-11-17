* 分发的对象
* 设计思序

> ## 分发对象

在Android中, 点击手势被封装成``MotionEvent``对象. 因此对于点击事件的分发, 实质上是对``MotionEvent``对象的分发.

对于单点触控, 有下面两种情况的手势对应``MotionEvent``的状态:

* 点击 -> 松开 : ``ACTION_DOWN`` -> ``ACTION_UP``

* 点击 -> 移动 -> 移动 -> ...-> 松开 :　``ACTION_DOWN`` -> ``ACTION_MOVE`` -> ``ACTION_MOVE`` -> ... -> ``ACTION_UP``

> ## 事件的传递顺序

点击事件发生后, 事件的传递顺序为: ``Activity`` -> ``PhoneWindow`` -> ``DecorView`` -> ``ViewGroup`` .... -> ``View``

> ## 设计思想

``Android``事件分发机制的设计思想是基于责任链模式. 事件在传递的过程中, 先从上层传递到下层, 在这个过程中, 如果该事件你需要处理的话, 可以拦截下来处理, 如果不需要的话, 交给下层处理. 当事件传递到底层且底层不处理事件的话, 事件会从底层往上层传递. 如果上传的过程中, 事件没有被消费的话, 最终由``Activity``消费.

> ## 分发过程中涉及的方法

事件分发的过程中, 事件需要被传递, 被分发, 分消费, 这些操作涉及到下面的方法:

``public boolean dispatchTouchEvent(MotionEvent ev)`` : 用来分发事件

``public boolean onInterceptTouchEvent(MotionEvent ev)`` : 用来判断是否拦截事件, 如果当前``View``拦截了某个事件, 那么同一个事件序列当中, 此方法不会被调用. 换句话说, 该方法不是每次都会被调用.

``public boolean onTouchEvent(MotionEvent ev)`` : 用来处理点击事件.

上面的三个方法的关系可以用下面的伪代码来表示:

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
  boolean isConsume = false;

  if (onInterceptTouchEvent(ev)) { //判断是否要拦截
      isConsume = onTouchEvent(ev); //是的话, 自己处理点击事件
  } else {
      isConsume = child.dispatchTouchEvent(ev); //不是的话, 传递给下一层处理
  }
  return isConsume;
}
```

> 注意: 事件的消费与否是通过``boolean``来标识的, ``true``表示消费事件, ``false``表示不消费事件. 事件的消费与否与是否使用了``MotionEvent``对象无关.

前面说过, 事件的传递顺序为 : ``Activity`` -> ``PhoneWindow`` -> ``DecorView`` -> ``ViewGroup`` .... -> ``View``. 下面我们来看看``Activity``, ``ViewGroup``和``View``中是否有上面的三个方法:

| 类型 | 方法 | Activity | ViewGroup | View |
| ------| ------ | ------ | ------ | ------ |
| 事件分发 | dispatchTouchEvent | 存在 | 存在 | 存在 |
| 事件拦截 | onInterceptTouchEvent | 不存在 | 存在 | 不存在 |
| 事件消费 | onTouchEvent | 存在 | 存在 | 存在 |

> 从上表可以看出: ``Activity``和``View``是不存在事件拦截的.

> ## 事件分发的总体流程

一个点击时事件总是先到达``Activity``, 然后传给``Window``, 接着``Window``传给顶级``View``, 最后按照事件分发机制去分发. 分发的流程图如下:

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fll2pjny8dj30jb0u3jtj.jpg)

> ## 监听器的优先级

当一个``View``设置了``OnTouchListener``, 那么``OnTouchListener``的``onTouch``会先被调用. 如果``OnTouch``返回``true``的话, 事件被消费. 如果返回``false``的话, 当前``View``的``onTouEvent``会被调用. 而在``onTouchEvent``中, 会根据事件类型来调用``OnLongClickListener``或者``OnClickListener``监听器的监听方法.

换句话说, 监听器的优先级为: ``OnTouchListener`` -> ``OnLongClickListener`` -> ``OnClickListener``.

> ## View默认的消费事件行为

如果``View``的``clickable``属性为``true``的话, 该``View``会默认消费事件.

如果给``View``设置了``onClickListener``、``onLongClickListener``、``OnContextClickListener``其中一个监听器, 或者``android:clickable="true"``的话, 该``View``是可点击的, 也就是``clickable``为``true``.

有些控件默认是可点击的, 比如:``Button``. 不可点击的控件: ``TextView``.

> 注意: ``View``的``enable``属性并不会影响事件的分发.

> ## 事件分发小总结

* 正常情况下, 一个事件序列, 只能被一个``View``拦截并且消费. 因为如果一个``View``拦截了一个事件, 那么后续的事件将交给它处理.

* 如果一个``View``不消费``ACTION_DOWN``事件的话, 后续的事件就不会传递给它. 事件将交给父元素处理, 即使父元素的``OnTouchEvent``会被调用.

* ``ViewGroup``默认不拦截任何事件.

* 事件总是先传给父元素, 再传给子元素. 子元素可以通过``requestDisallowInterceptTouchEvent``来干预父元素的拦截, 但是对于``ACTION_DOWN``事件, 子元素不能干预父元素的拦截.

> ## View的滑动冲突

常见的滑动冲突有三种:

1. 外部和内部的滑动方向不一样

2. 外部和内部的滑动方向一样.

3. 上面两种情况的嵌套.

> ### 处理规则

场景1: 当用户左右滑动时, 让外部的``View``拦截点击事件; 当用户上下滑动时, 让内部的``View``拦截点击事件. 可以根据``View``滑动的水平距离和垂直距离差或角度差来判断用户是左右还是水平滑动.

场景2: 这种情况只能根据业务逻辑来拦截事件.

场景3: 跟场景2一样.

> ### 方法论

处理冲突的方法可以分为两种:

* 外部拦截法: 事件总是先经过父元素, 如果父元素需要的话, 就拦截下来.

* 内部拦截法: 父容器默认不拦截事件, 所有事件先交给子元素处理, 如果子元素需要的话, 就消费掉, 不需要的话, 交给父元素处理.

下面是这两种方法的模板代码:

> 外部拦截法:

```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
    boolean intercepted = false;
    int x = (int) ev.getX();
    int y = (int) ev.getY();

    switch (ev.getAction()) {
        case MotionEvent.ACTION_DOWN:
            intercepted = false;
            break;
        case MotionEvent.ACTION_MOVE:
            if (父容器需要当前点击事件) {
                intercepted = true;
            } else {
                intercepted = false;
            }
            break;
        case MotionEvent.ACTION_UP:
            intercepted = false;
            break;
        default:
            break;
        mLastXIntercepted = x;
        mLastYIntercepted = y;
        return intercepted;
    }
}
```

> 内部拦截法

```java
//子元素
public boolean dispatchTouchEvent(MotionEvent ev) {
    int x = (int) ev.getX();
    int y = (int) ev.getY();

    switch (ev.getAction()) {
        case MotionEvent.ACTION_DOWN:
            getParent().requestDisallowInterceptTouchEvent(true);
            break;
        case MotionEvent.ACTION_MOVE:
            if (父容器需要此点击事件) {
                getParent().requestDisallowInterceptTouchEvent(false);
            }
            break;
        case MotionEvent.ACTION_UP:
            break;
        default:
            break;
    }
    mLastX = x;
    mLastY = y;
    return super.dispatchTouchEvent(ev);
}

//父元素:
public boolean onInterceptTouchEvent(MotionEvent ev) {
    int action = ev.getAction();
    if (action == MotionEvent.ACTION_DOWN) {
        return false;
    } else {
        return true;
    }
}
```
