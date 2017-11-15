---
layout: post
title: CustomView总结
data: 2017/8/28
subtitle: ""
description: ""
thumbnail: "/img/customview.jpeg"
tags:
  - Android#自定义View
categories:
  - Android
---

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fk2tf8bzggj30zk0nqtbj.jpg)

</br>

> ## LayoutParams认知

### Q: 什么是LayoutParams? 它跟view的关系是什么?

> #### 什么是LayoutParams?

我们在XML布局中定义的layout_xx属性,最终都会以Java代码的形式展现出来, 而LayoutParams就是这些layout_xx属性在Java层的映射, 也就是说LayoutParams是view在xml布局中layout_xx的属性容器.

可见, LayoutParams是子View跟父View进行协商的桥梁. 协商的内容可有: 子View宽高, 子View在父View中摆放的位置等.

> #### LayoutParams的继承关系

LayoutParams是作为内部类定义在ViewGroup中,下面是LayoutParams的继承关系:

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fik760lyaqj30s20l8dgp.jpg)

LayoutParams是定义在ViewGroup, LayoutParams只支持 height, width. ViewGroupn内部的一个默认实现是MarginLayoutParams, 该类继承LayoutParams, 扩展了margin属性.接下来, 各个VieGroup的子类内部都有LayoutParams的子类,扩展对应ViewGroup的属性.比如 LinearLayout.LayoutParams, 增加了weight属性.

> #### 在Java代码中获取view的LayoutParams

既然LayoutParms是View对自己在父布局中的属性设置, 那么父View在测量或者布局的时候, 肯定是需要拿到这个LayoutParams对象的.

```java
@ViewDebug.ExportedProperty(deepExport = true, prefix = "layout_")
    public ViewGroup.LayoutParams getLayoutParams() {
        return mLayoutParams;
    }

    /**
     * Set the layout parameters associated with this view. These supply
     * parameters to the <i>parent</i> of this view specifying how it should be
     * arranged. There are many subclasses of ViewGroup.LayoutParams, and these
     * correspond to the different subclasses of ViewGroup that are responsible
     * for arranging their children.
     *
     * @param params The layout parameters for this view, cannot be null
     */
    public void setLayoutParams(ViewGroup.LayoutParams params) {
        if (params == null) {
            throw new NullPointerException("Layout parameters cannot be null");
        }
        mLayoutParams = params;
        resolveLayoutParams();
        if (mParent instanceof ViewGroup) {
            ((ViewGroup) mParent).onSetLayoutParams(this, params);
        }
        requestLayout();
    }
```

上面是View提供的接口, 用于设置和获取View中的LayoutParams.父View在测量和布局时, 就是通过``view.getLayoutParams()``来获得子``View``设置的``LayoutParams``.


> ## 理解MeasureSpec

### Q: 什么是MeasureSpec? 它的工作原理是什么?

> #### 什么是MeasureSpec?

我们都知道, Android体系中, View有三种测量模式,每种测量模式都有对应View的宽高.

一个View可能会被多次测量, 在运行时也可能被动态改变而导致重新测量,布局. Android团队为了减少多次测量带来的对象分配消耗, 将View的测量模式和View的大小打包成一个int类型的值,从而减少了对象分配带来的消耗.

MeasureSpec这个类就是提供了打包和解包方法, 将测量模式和大小打包或者解包成int值.

> #### MeasureSpec的工作原理

在Java中, int类型固定占4字节, 也就是32位.

由于三种测量模式用2位就能够表示, 可分别表示为:``00(UNSPECIFIED)``, ``01(EXACTLY)``, ``10(AT_MOST)``.

既然三种测量模式用2位表示就够, 那么剩下的低30位用来表示大小.

问题来了? 如何将2位的测量模式和30位的大小打包成一个int后, 能够无差错的解包成测量模式和View的大小?

答案: 使用掩码和逻辑或,与运算.

```java
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        /**
         * Measure specification mode: The parent has not imposed any constraint
         * on the child. It can be whatever size it wants.
         */
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;

        /**
         * Measure specification mode: The parent has determined an exact size
         * for the child. The child is going to be given those bounds regardless
         * of how big it wants to be.
         */
        public static final int EXACTLY     = 1 << MODE_SHIFT;

        /**
         * Measure specification mode: The child can be as large as it wants up
         * to the specified size.
         */
        public static final int AT_MOST     = 2 << MODE_SHIFT;
```

MODE_SHIFT为移多少位, MODE_MASK为掩码. 现在我们来算算掩码:3表示为二进制为:11, 所以0x3 << 30 表示为: 1100000000(30个0). 这个掩码的作用就是配合逻辑与,或操作来进行打包和解包的操作.

```java
public static int makeMeasureSpec(int size, int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
            }
        }
```

上面是打包操作, 在``size & ~MODE_MASK``中, ~MODE_MASK为: 0011111(30个1),由于低30为全为1, 那么与size进行&操作时, 低30位就由szie的低三十为决定, 而高2位则为00, ``size & ~MODE_MASK``这个操作就是将size的高2为设置为00, 同理:``(mode & MODE_MASK)``这个操作是将mode低30位设置为0. 最后再进行 | 操作, 这样就得到了高2位为测量模式, 低30位为大小的int.

```java
public static int getMode(int measureSpec) {
            return (measureSpec & MODE_MASK);
        }
```

getMode方法将一个打包的measureSpec解包为mode, 原理是通过&获取measureSpec的高2位. 同理getSize(int measureSpec)也是同种道理.

> ## LayoutParams和MeasureSpec的关系

* MeasureSpec是由父容器中LayoutParams和本身的LayoutParams这两个因素决定的.

* 但是对于顶级View(DecorView),他的MeasureSpec是由屏幕的大小和自身的LayoutParams所决定的.

* 以上两点在阅读源码就可以看出来.

## 从源码角度来理解测量和布局

> ### 测量

为了更好的理解测量过程, 我们需要理解清楚MeasureSpec和LayoutParams的关系, 看MeasureSpec是怎么在LayoutParams的约束下生成的.

测量过程的工作是确定View的宽高.我们先从ViewRootImpl这个类来分析顶级View类是如何开始测量的.

测量,布局,绘制,这三个步骤将View显示到屏幕上,而触发这三个流程的地方是在``performTraversals()``方法中.

在``performTraversals()``方法中,除了测量,布局,绘制这三个阶段外, 其实在存在另外的两个阶段:预测量和窗口布局.

> #### 预测量

ViewRootImpl在进行测量时,会预先进行一次测量,而这次预测量是在``measureHierarchy``方法中进行的. 预测量的目的就是为了在大屏幕的设备上将View更优雅的显示出来.

```java
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
        int childWidthMeasureSpec;
        int childHeightMeasureSpec;
        boolean windowSizeMayChange = false;

        if (DEBUG_ORIENTATION || DEBUG_LAYOUT) Log.v(TAG,
                "Measuring " + host + " in display " + desiredWindowWidth
                + "x" + desiredWindowHeight + "...");

        boolean goodMeasure = false;
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) { //为了让dialog显示得更好, 先根据预先定义好的dialog尺寸, 以此测量出一个包裹dialog的宽度.
            // On large screens, we don't want to allow dialogs to just
            // stretch to fill the entire width of the screen to display
            // one line of text.  First try doing the layout at a smaller
            // size to see if it will fit.
            final DisplayMetrics packageMetrics = res.getDisplayMetrics();
            res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);
            int baseSize = 0;
            if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
                baseSize = (int)mTmpValue.getDimension(packageMetrics);
            }
            if (DEBUG_DIALOG) Log.v(TAG, "Window " + mView + ": baseSize=" + baseSize);
            if (baseSize != 0 && desiredWindowWidth > baseSize) {
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width); //打包成MeasureSpec
                childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec); //开始第一次测量
                if (DEBUG_DIALOG) Log.v(TAG, "Window " + mView + ": measured ("
                        + host.getMeasuredWidth() + "," + host.getMeasuredHeight() + ")");
                if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) { //如果测量出来的大小太小的话, 会再进行测量
                    goodMeasure = true; //对测量结果满意
                } else {
                    // Didn't fit in that size... try expanding a bit.
                    baseSize = (baseSize+desiredWindowWidth)/2;
                    if (DEBUG_DIALOG) Log.v(TAG, "Window " + mView + ": next baseSize="
                            + baseSize); //需要的话, 扩大宽度,进行二次测量
                    childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width); //再次打包成MeasureSpec
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec); //第二次测量
                    if (DEBUG_DIALOG) Log.v(TAG, "Window " + mView + ": measured ("
                            + host.getMeasuredWidth() + "," + host.getMeasuredHeight() + ")");
                    if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                        if (DEBUG_DIALOG) Log.v(TAG, "Good!");
                        goodMeasure = true;
                    }
                }
            }
        }

        if (!goodMeasure) { //最后还是太小的话, 只能妥协,返回窗口的可能改变的标记
            childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
                windowSizeMayChange = true;
            }
        }

        if (DBG) {
            System.out.println("======================================");
            System.out.println("performTraversals -- after measure");
            host.debug();
        }

        return windowSizeMayChange;
    }
```

* 为了让View更优雅的显示出来, 比如dialog在大屏幕的情况下, 如果其内容太大并且测量模式为WRAP_CONTENT的话,有可能会出现这种情况:

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fik7l94nxgj30hi0br3yi.jpg)

* 考虑到上面dialog的宽度有可能太小,会出现下面这种情况:

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fik7n6wvkcj30c809ka9y.jpg)

* 所以在预测量阶段,每次测量后都会根据一个标记``(MEASURED_STATE_TOO_SMALL)``来判断是否要进行二次测量来扩大宽度.

* 如果进行了二次测量的话, 宽度还是大小,就只能妥协,放弃预测量.

* 预测量是针对悬浮窗口而言, 也就是对于非悬浮窗口而言, 是没有预测量阶段的.

> #### 窗口布局

一般测量阶段都会伴随一个布局阶段, 预测量也是如此, 窗口布局就是根据预测量阶段得出的结果来进行布局.

> #### 开始测量

分析完预测量和预布局, 我们来分析``"真正"``的测量阶段

```java
int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

if (DEBUG_LAYOUT) Log.v(TAG, "Ooops, something changed!  mWidth="
                            + mWidth + " measuredWidth=" + host.getMeasuredWidth()
                            + " mHeight=" + mHeight
                            + " measuredHeight=" + host.getMeasuredHeight()
                            + " coveredInsetsChanged=" + contentInsetsChanged);

// Ask host how big it wants to be
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```

* 在View的测量机制中, 是通过给子View传递MeasureSpec来进行的, 因此, 测量阶段会首先进行获取测量规格, 并传递给子View, 我们先来看看顶级View如何进行获取MeasureSpec.

```java
int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
```

* 上面的代码通过调用``getRootMeasureSpec(mWidth, lp.width)``来获取宽度的MeasureSpec, 传递给方法的参数分别为屏幕的窗口大小和顶级View的LayoutParams封装的宽度信息.

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
```

* 如果宽度为``ViewGroup.LayoutParams.MATCH_PARENT``, 直接把将窗口的大小和``MeasureSpec.EXACTLY``打包成一个MeasureSpec; 当宽度为``ViewGroup.LayoutParams.WRAP_CONTENT``情况也是如此.

* 至于``getRootMeasureSpec(mHeight, lp.height)``跟上面的基本一致,这里不多说.

* 分析到这里, 也验证了,``顶级View的MeasureSpec是由窗口的大小和自身的LayoutParams所决定的``

我们接着继续分析``performMeasure()``

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```

* 这个方法中只是简单的调用了View的measure(int, int)方法, 也就是从RootViewImppl调到了View中. 方法中的参数刚刚生成的两个测量规格.可见,测量规格是在父View中生成并传递给子View的.

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        // Suppress sign extension for the low bytes
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

        if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
                widthMeasureSpec != mOldWidthMeasureSpec ||
                heightMeasureSpec != mOldHeightMeasureSpec) {

            // first clears the measured dimension flag
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();

            int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                    mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // flag not set, setMeasuredDimension() was not invoked, we raise
            // an exception to warn the developer
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }
            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }

        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }
```

* 方法略长, 但是核心思路只有两个, 第一: 判断传入的大小和前一次测量的大小是否一样, 如果一样的话, 不进行测量, 如果不一样的话, 就开始测量. 第二:如果传入的大小和前一次测量的大小不一样的话, 会调用onMeasure(int, int),开始测量.

* 对于``onMeasure(int, int)``而言, View和ViewGroup的测量职责是不一样的. ``对于View, 它只需测量自身的大小, 而对于ViewGroup, 它需要测量自己和测量自己的孩子, 一般都是先测量孩子,然后根据孩子的大小来设置自己的大小.``

* 由于``measure()``方法是被final修饰的, 所以``measure(int, int)``是不允许被重写的, 需要我们重写的是onMeasure(int, int). 这样做的优点是:开发者无需关心View测量的其他细节, 只需关心测量View的大小就行, 减轻了开发者的开发难度.

* 既然View和ViewGroup的测量职责不一样, 那么View和ViewGroup中的onMeasure(int, int)的实现肯定不一样, 所以我们开始分情况来讨论

> ##### View#onMeasure(int, int)

我们先从简单的View入手, 进入View的onMeasure(int, int):

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```

* View的onMeasure(int, int)只是简单调用了setMeasureDimension(int size, int measureSpec),这个是保存测量得到的宽高.

* setMeasureDimension(int, int), 其中的宽高是通过``getDefaultSize(int size, int measureSpec)``来获取的.我们进入该方法看看:

```java
public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```

* 首先根据传入的measureSpec来获得测量模式和测量大小, 如果测量模式是``AT_MOST``或者``EXACTLY``的话, 直接返回测量得到的大小.

* 从这个方法我们知道, View的onMeasure(int, int)的默认实现是根据传入的MeasureSpec来获取测量结果.那么这个MeasureSpec是怎么产生的? 前面我们说过, ViewGroup的onMeasure(int, int)中, 是需要测量孩子的, 这个MeasureSpec就是ViewGroup在测量子View时传递给子View的, 换句话说, 这个MeasureSpec是从ViewGroup传递下来的, 通过解包操作, 可以得到MeasureSpec中的大小, 这个大小究竟是ViewGroup的总体大小还是剩余大小? 这个得看具体的ViewGroup的具体实现.

* 上面的分析可能有点难以理解, 不过接下来分析完ViewGroup后, 自然会解开你的迷惑.

> ##### ViewGroup#onMeasure(int, int)

由于ViewGroup的子类对测量都有不同的策略, 因此, ViewGroup并没有重写onMeasure(int, int), 而是让其子类去重写.我们拿比较简单的FrameLayout来分析吧.

```java
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();

        final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();

        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }

        // Account for padding too
        maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
        maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

        // Check against our minimum height and width
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

        // Check against our foreground's minimum height and width
        final Drawable drawable = getForeground();
        if (drawable != null) {
            maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
            maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
        }

        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));
```

* 我抽取了FrameLayout中onMeasure(int, int)的核心代码, 主要思路: 遍历子View,让子View测量自己,也就是触发子View的测量. 接着再根据子View的大小来计算自己的大小.我们先来看看measureChildrenWithMargins(View, int, int).

```java
protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

* 该方法是ViewGroup提供的工具类, 关于View和ViewGroup提供的测量工具类, 后面的专门分析.

* 首先拿到View的LayoutParams, 然后调用getChildMeasureSpec来得到子View的MeasureSpec, 我们看看它是怎么得到的.

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

* 首先进行解包操作, 拿到父View传递下来的测量模式和测量大小, 在xml布局中的顶级View的MeasureSpec是由RootViewImpl传递下来的, 而对于顶级View(DecorView)来说, 其MeasureSpec是由窗口大小和自身的LayoutParams决定的.

* 接着进入switch语句:
 * 当父View的mode为``MeasureSpec.EXACTLY``时, 再根据子View的LayoutParams(也就是子View在xml文件中声明的layout_xx属性的容器), 又分为三种情况:
   1. 当子View在xml文件中声明的宽(用宽来举例子)为确定的值, 那么resultSize为子View在xml文件中声明的宽度(也就是xxdp),resultMode为``MeasureSpec.EXACTLY``,然后再打包为一个MeasureSpec.
   2. 当子View在xml文件中声明的宽为``LayoutParams.MATCH_PARENT``时, 表明子View想要占满父View的宽度, 因此, resultSize设置为父View的大小size, reslutMode为``MeasureSpec.EXACTLY``
   3. 当子View在xml文件中声明的宽为``LayoutParams.WRAP_CONTENT)``时, 表明子View想要根据自己的内容来决定大小, 所以resultSize设置为父View的size,用来表示不能操过这个值, resultMode设置为``MeasureSpec.AT_MOST``

 * 当父View的mode为``MeasureSpec.AT_MOST``时, 根据子View的LayoutParams, 分为三种情况:
   1. 当子View在xml文件中声明的宽为确定的值时, resultSize为子View在xml文件中声明的宽度(也就是xxdp), resultMode为``MeasureSpec.EXACTLY``, 然后再打包为一个MeasureSpec.
   2. 当子View在xml文件中声明的宽为``LayoutParams.MATCH_PARENT``时, 表明子View想要占有父View的宽度, 但是由于父View的测量模式为``MeasureSpec.AT_MOST``(表示,父View也是要根据自身的内容来设定大小), 所以resultMode只能为``MeasureSpec.AT_MOST``(因为父View自身也不知道自己多大), resultSize设置为父Viw的大小.
   3. 当子View在xml文件中声明的宽为``LayoutParams.AT_MOST``时表明子View想要根据自己的内容来决定大小, 所以resultSize设置为父View的size,用来表示不能操过这个值, resultMode设置为``MeasureSpec.AT_MOST``.

* 分析到这里, 我们可以总结出View的MeasureSpec是由哪些因素决定的
 * 对于顶级View的MeasureSpec, 是由窗口的大小和自身的LayoutParams来决定的(可以会看之前分析的代码)
 * 对于非顶级View的MeasureSpec, 是由父View的MeasureSpec(其中的mode)和子View自身的LayoutParams(也就是在xml文件声明的layout_xx的属性)

* 上面我们针对宽度来进行了分析, 对于高度而言, 过程跟宽度一样

* 最后调用子View的``measure(int, int)``, 将在``getChildMeasureSpec``中得到的宽和高对应的MeasureSpec(也就是我们上面以宽度为例子来分析的情况),传递给子View,接下去的过程跟我们前面在分析子View的情况一样.

* 现在我们可以更加确定了这样一个事实: View的onMeasure(int, int)中的widthMeasureSpec和heightMeasureSpec是由父View根据自身的MeasureSpec和子View的LayoutParams产生并传递给子View的.

* 如果你仔细总结的话,会发现:只要子View的的宽或高设置为``LayoutParams``设置为WRAP_CONTENT时, 生成MeasureSpec中的size都是父View的size(一般是父View剩下的size),而mode为AT_MOST.

* 我们现在继续调到子View的measure方法中的onMeasure(), 我们再来看看getDefaultSize()这个方法:

```java
public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```

* 首先解包出由父View传递下来的MeasureSpec, 通过上面的分析, 当子View在xml文件中将layout_width设置为WRAP_CONTENT时, 对应的SpecMode为AT_MOST, 此时进入switch语句, 得到的大小其实是父View的大小, 这也解释了,在自定义View(继承View)时,如果没有重写onMeasure(int, int)时,当这个自定义View的宽度或者高度设置为``WRAP_CONTENT``时,会变成占有父View的全部高度和宽度.

* 分析到这里, 已经基本分析完了测量过程,我们还是总结一下,再进入下一个流程.

> #### 测量过程总结

* onMeasure(int widthSpec, int heightSpec)中的这两个MeasureSpec是由父View传递下来的.MeasureSpec一旦确定了, 在这个方法中就可以确定View的测量宽高了.

* 如何确定MeasureSpec?
 * 对于顶级View的MeasureSpec, 是由窗口的大小和自身的LayoutParams来决定的(可以回看之前分析的代码)
 * 对于非顶级View的MeasureSpec, 是由父View的MeasureSpec(其中的mode)和子View自身的LayoutParams(也就是在xml文件声明的layout_xx的属性)

* 父View的MeasureSpec和子View的LayoutParams如何确定MeasureSpec?
 * 在View测量的时候, 系统会将View的LayoutParams在父View的SpecMode的约束下再次打包为一个传递给子View的MeasureSpec.

* EXACTLY,AT_MOST和LayoutParams的关系
 * EXACTLY: 父View已经检测出子View所需要的大小(通常是子View在xml文件中声明为精确的值,比如20dp),它对应于LayoutParams中的match_parent和具体的数值
 * AT_MOST: 子View不能超过父View给定的大小, 它对应与LayoutParams中的wrap_content.

* 最后来看看流程图加深印象:

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fik8aodsxyj30nq0i2ta8.jpg)

> ### 布局

经过测量阶段后, View已经知道自己的宽和高, 接着就需要在布局阶段确定应该显示在哪个区域,也就是屏幕上的四个点.那我们回到VewiRootImpl中的performLayout()方法, 该方法是触发布局的起点, 在里面调用了View的layout方法:

```java
host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
```

* layout方法中的四个参数分别为屏幕上的四个点, 也就是整个屏幕的区域

* 我们进入View的layout方法:

```java
public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }
```

* 我们主要分析上面的两个核心操作:
 * setFrame(left, top, right, bottom)这个方法主要是保存四个点的位置, 并且判断传入的位置跟之前的是不是一样, 不一样的话,会回调onSizeChange接口
 * 调用onLayout(left, top, right, bottom), 但是这个方法View是一个空方法, 对于ViewGroup来说,具体的子类有其具体的实现方法.因此, ``layout方法主要是view用于确定自己的位置的, 而onLayout是用于ViewGroup循环调用子类的layout方法来对子类进行布局的.``



* 理解了上面的这两点, 其实布局过程就基本没什么了.

* 最后还是看看流程图, 加深理解.

![](http://ww1.sinaimg.cn/large/006VdOYcgy1fik8dhsrxmj30kh0flwfa.jpg)

* 分析到这里, 测量,布局都分析完了, 绘制流程就不讲了,

# 自定义View的分类

> ## 继承View重写onDraw方法和onMeasure方法

这种方法一般用于实现基础控件和组合控件难以达到的效果. 所以需要重写onDraw方法来画出自己的图形, 重写onMeasure方法来支持wrap_content属性,和padding属性.

> ## 继承ViewGrroup派生特殊的Layout

当系统的基础布局容器不能满足我们的需求时, 我们可以继承ViewGroup来自定义一个布局容器. 这种方法需要合适处理ViewGroup的``测量(测量孩子和测量自己)``,布局这两个过程.

> ## 继承基础控件(如TextView)

这种方法一般用于扩展基础控件的功能,相对比较简单. 这种方法不需要自己处理wrap_content和padding.

> ## 继承基础容器(如LinearLayout)

这种方法一般用于组合一些基础控件或者自定义View.

# 自定义View的方法论

> ## 继承View

这种自定义方法,我们处理的主要有两个方法:

* 在onMeasure(int, int)中处理wrap_content和padding

* 在onDraw(Canvas)中绘制你想要的UI和处理padding

> ### onMeasure(int, int)处理

在onMeasure(int, int)中需要处理的有两个: wrap_content和padding这两个属性.

> #### 支持wrap_content

经过前面的分析, 我们知道对于一个继承View的控件, 如果没有重写onMeasure(int, int), 在xml布局中设置layout_width="wrap_content"的时候, 会占满父View的宽度, 其中的原因前面已经分析了, 这里主要将如何支持wrap_content. 接下来我会讲解决这个问题的基本方法和另外一种快捷方法

* 基本方法:

当layout_width或者layout_height为wrap_content, 我们不用默认的高宽, 而是自己根据自己View的内容来决定View的高宽

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
//1.首先拿到宽高的大小和测量模式
int widthMode = MeasureSpec.getMode(widthMeasureSpec);
int widthSize = MeasureSpec.getSize(widthMeasureSpec);
int heightMode = MeasureSpec.getMode(heightMeasureSpec);
int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        //2.接着判断宽高和mode是否为wrap_content, 如果是的话, 我们不用widthSize或者heightSize, 我们自己指定View的大小
        //wrap_content对应MeasureSpec.AT_MOST属性
if (widthMode == MeasureSpec.AT_MOST && heightMode == MeasureSpec.AT_MOST) {
  setMeasuredDimension(300, 300);
  } else if (widthMode == MeasureSpec.AT_MOST) {
    setMeasuredDimension(300, heightSize); //只有width为wrap_content, 所以高直接用heightSize就行
  } else if (heightMode == MeasureSpec.AT_MOST) {
    setMeasuredDimension(widthSize, 300); //只有height为wrap_content, 所以宽直接用widthSize就行
  }
}
```

上面提供了解决wrap_content的基本思路.下面我们介绍另外一种快捷而且屏幕适配更好的方法.在介绍之前,我们先来看看View中的几个有用的方法.

```java
protected int getSuggestedMinimumHeight() {
        return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());

    }
```

这个方法返回一个建议的最小高度.如果View没有设置背景,那么返回值为mMinHeight(也就是在xml布局中声明的属性), 如果有的话, 会返回背景的Drawable对象的高度和mMinHeight中的最大值.

```java
protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```

这个方法和前面的一样,这里不多说.

```java
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
        final int specMode = MeasureSpec.getMode(measureSpec);
        final int specSize = MeasureSpec.getSize(measureSpec);
        final int result;
        switch (specMode) {
            case MeasureSpec.AT_MOST:
                if (specSize < size) {
                    result = specSize | MEASURED_STATE_TOO_SMALL;
                } else {
                    result = size;
                }
                break;
            case MeasureSpec.EXACTLY:
                result = specSize;
                break;
            case MeasureSpec.UNSPECIFIED:
            default:
                result = size;
        }
        return result | (childMeasuredState & MEASURED_STATE_MASK);
    }
```

``resolveSizeAndState(int size, int measureSpec, int childMeasuredState)``这个方法是View本身提供的一个支持wrap_content的一个方法(getDefaultSize()方法不支持wrap_content),这个方法除了支持wrap_content外, 还通过掩码操作添加了一些信息, 如果当size的大小大于父View的高度时, 会通过掩码操作将 MEASURED_STATE_TOO_SMALL和size打包成一个int值.

```java
public static int resolveSize(int size, int measureSpec) {
        return resolveSizeAndState(size, measureSpec, 0) & MEASURED_SIZE_MASK;
    }
```

这个是上个方法的重载版本, childMeasuredState属性传入0,表示不对标志做处理, 最后用MEASURED_SIZE_MASK这个掩码提取出想要是size值.

介绍完了上面的几个方法, 我们开始介绍另外一种快捷的适配wrap_content的方法.

```java
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int maxWidth = getSuggestedMinimumWidth();
        int maxHeight = getSuggestedMinimumHeight();
        //调用resolveSize(int size, int measureSpec), size指的是, 当mode为wrap_content时, 这个方法会为我们返回size值.也就是支持wrap_content
        int width = resolveSize(maxWidth, widthMeasureSpec);
        //调用resolveSize(int size, int measureSpec), size指的是, 当mode为wrap_content时, 这个方法会为我们返回size值.也就是支持wrap_content
        int height = resolveSize(maxHeight, heightMeasureSpec);
        setMeasuredDimension(width, height); //最后设置大小
    }

    @Override
    protected int getSuggestedMinimumWidth() {
        return Math.max(super.getSuggestedMinimumWidth(), mTextSize); //将默认的最小宽度和自己定义的字体大小比较, 取最大值
    }

    @Override
    protected int getSuggestedMinimumHeight() {
        return Math.max(super.getSuggestedMinimumHeight(), mTextSize); //将默认的高度和自己定义的字体大小比较, 取最大值.
    }
```

我们知道,支持wrap_content的实质就是根据内容来决定View的大小.那么我们可以利用``getSuggestedMinimumWidth``和``getSuggestedMinimumHeight``这个两个方法来获取最小的高度和宽度.当mode为wrap_content时, 怎么计算出最小的高度和宽度, 这个看你的自定View情况, 自己灵活选择.

> #### 支持padding

支持padding是需要在onDraw和onMeasure中实现, 思路都很简单, 在onMeasure中的宽高将padding考虑进去, 在onDraw中绘制图形时,除去padding

```java
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int maxWidth = getSuggestedMinimumWidth();
        int maxHeight = getSuggestedMinimumHeight();
        maxWidth = maxWidth + getPaddingLeft() + getPaddingRight(); //加上paddingLeft和paddingRight得到宽度
        maxHeight = maxHeight + getPaddingBottom() + getPaddingTop(); //加上paddingBottom和paddingTop得到高度
        //调用resolveSize(int size, int measureSpec), size指的是, 当mode为wrap_content时, 这个方法会为我们返回size值.也就是支持wrap_content
        int width = resolveSize(maxWidth, widthMeasureSpec);
        //调用resolveSize(int size, int measureSpec), size指的是, 当mode为wrap_content时, 这个方法会为我们返回size值.也就是支持wrap_content
        int height = resolveSize(maxHeight, heightMeasureSpec);
        setMeasuredDimension(width, height); //最后设置大小
    }
```

上面是onMeasure中支持padding, 下面看看onDraw中支持

```java
@Override
    protected void onDraw(Canvas canvas) {
        int wdith = getWidth() - getPaddingLeft() - getPaddingRight();
        int height = getHeight() - getPaddingTop() - getPaddingBottom();
    }
```

在onDraw中, 用到宽高时, 先减去对应的padding就能支持padding了.

> ## 继承ViewGroup

继承ViewGroup的View, 相当于自定义一个布局容器, 需要我们处理的有两个:

* onMeaure(int, int)

* onLayout(int, int, int, int)

> ### 处理onMeasure(int, int)

自定义ViewGroup不同于自定义View, 自定义ViewGroup在onMeasure(int, int)中, 除了测量自己,还需要测量孩子, 通常是遍历孩子并触发孩子的测量方法, 然后根据孩子的宽高来决定自己的宽高. 我们下面来看看ViewGroup给我们提供的方法:

```java
protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

通过传入子View,父View的宽高测量规格, 该方法内部会帮我们调用view的measure方法去测量View

```java
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
```

该方法内部遍历所有子View, 然后调用上一个方法测量View, 也就是帮我们测量了所有子View, 不用我们手动for循环去测量View. 你们可以根据自己的需要去调用其中的方法.

```java
protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

这个方法也是测量子View的方法, 不过这个方法把子View的margin考虑进去了.

下面说说ViewGroup测量的套路

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int maxHeight = getPaddingTop() + getPaddingBottom(); //支持padding
    int maxWidth = getPaddingLeft() + getPaddingRight(); //支持padding
    //循环测量孩子
    for (int i = 0; i < getChildCount(); i++) {
        View child = getChildAt(i);
        measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0); //考虑margin测量孩子
        MarginLayoutParams params = (MarginLayoutParams) child.getLayoutParams();
        maxHeight = maxHeight + child.getMeasuredHeight() + params.topMargin + params.bottomMargin; //将支持margin
        maxWidth = Math.max(maxWidth, child.getMeasuredWidth() + params.leftMargin + params.rightMargin); //将支持margin
    }
    setMeasuredDimension(resolveSize(maxWidth, widthMeasureSpec), resolveSize(maxHeight, heightMeasureSpec)); //测量自己
}
```

上面我模拟了LinearLayout中的vertical布局属性.注意记得考虑padding和margin.

首先考虑padding, 然后循环遍历子View并测量, 最后根据孩子测量得到的宽高和孩子的margin属性和自己的布局属性来进行测量自己的大小.

> ### onLayout(int, int, int, int)

onLayout的职责就是将根据子View测量得到的宽高将其摆放在合适的位置. 注意: 在onLayout阶段, 没有特殊情况的话, 子View的布局要根据其测量得到的宽高来布局, 这样才是符合View的设计规范.

```java
@Override
protected void onLayout(boolean b, int i, int i1, int i2, int i3) {
    int left = getPaddingLeft(); //考虑padding
    int top = getPaddingTop(); //考虑padding
    int bottom;
    for (int j = 0; j < getChildCount(); j++) {
        View child = getChildAt(j);
        MarginLayoutParams params = (MarginLayoutParams) child.getLayoutParams();
        left = left + params.leftMargin;
        top = top + params.topMargin;
        bottom = params.bottomMargin;
        int width = child.getMeasuredWidth();
        int height = child.getMeasuredHeight();
        child.layout(left, top, left + width, top + height + bottom);
        top = top + height + bottom;
    }
}
```

上面是简单的模仿LinearLayout的vertical布局属性, 思路: 遍历所有的子View, 然后确定四个点的位置, 一般都是确定left, top这两个点, 然后对应加上View的宽高再得到另外的right, bottom.注意:这里right和bottom这两个点的确定,如果没有特殊情况, 应该根据view的测量得到的宽高来确定, 不能随便指定特定的值, 这样会导致getMeasureXX和getXX不相等, 如:

```java
child.layout(left, top, left + width + 100, top + height + bottom + 100);
```

如果你按照上面的操作, getWidth会比getMeasureWidth大100, getHeight会比getMeasureHeight大100. 原因就是你布局的时候没有根据测量阶段View的宽高来布局(也就是私自加多了100, 导致两个点相减时会多出100). 因此,如果没有特殊情况, 布局阶段请按照子View测量得到的宽高来布局, 这样才是符合View的设计规范.

> ### 定义你自己的LayoutParams

最后补充一个. 如果你的自定义ViewGroup需要自定义LayoutParams的话, 需要进行下面两个步骤:

```java
public static class CustomLayoutParams extends MarginLayoutParams {

        public int attr;

        public CustomLayoutParams(Context c, AttributeSet attrs) {
            super(c, attrs);
        }

        public CustomLayoutParams(int width, int height) {
            super(width, height);
        }

        public CustomLayoutParams(MarginLayoutParams source) {
            super(source);
        }

        public CustomLayoutParams(LayoutParams source) {
            super(source);
        }

    }
```

在你的类中定义上面的内部类, 属性自己定义.

```java
@Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new CustomLayoutParams(getContext(), attrs);
    }

    @Override
    protected LayoutParams generateLayoutParams(LayoutParams p) {
        return new CustomLayoutParams(p);
    }
```

然后重载上面的方法.

如果理解了上面的方法轮, 那么自定义View也就没多大问题了, 最后剩下的就是绘制了.

剩下的两种情况都不怎么难, 这里多说了.

> ## 自定义View的步骤

1: 首先分析这个View是怎么绘制的? 哪些是需要抽象成参数

2: 将抽象出来的参数定义在attr资源文件

3: 重写onMeasure方法,并让你的View支持wrap_content,或者padding(如果有必要的话)

4: 重写onDraw方法, 主要在onDraw也是要处理padding(如果有必要的话)

5: 暴露接口给外部(例如监听接口, 动态改变属性的接口)

6: 如果存在滑动冲突的话, 需要解决滑动冲突

7: 根据实际情况优化你的View

> ## 自己的自定义View的习惯:

上面是标准的自定义View的步骤, 实际情况中, 不需要按照严格的顺序进行.下面说说我写的时候的套路:

1: 先分析这个自定义View是怎么话的?

2: 抽象出一些参数,定义在View中

3: 先在onDraw中把图形先画出来,

4: 图形出来后, 再重写onMeasure来支持wrap_content.padding等属性

5: 将抽象出来的参数定义在attr资源文件

6: 暴露接口给外部

7: 优化View

## 自定义View的使用场景

> ### 优先考虑继承现有的空间来实现额外的功能

如果一些效果是继承现有控件能够实现的话, 那么优先继承现有的控件, 因为现有的控件都是经过官方多次的优化,性能肯定比我们自己写出来的好.

> ### 当一个View嵌套很多布局时, 考虑自定义View的实现.

如果View嵌套太深, requesLayout触发时, 会导致整个布局都被测量和布局, 这样的消耗太大了, 更好的做法是,自定义一个View, 减少测量和布局阶段的消耗.
