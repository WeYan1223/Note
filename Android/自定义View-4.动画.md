# 1. 属性动画

> 一段时间内，通过不断更新 View 的属性，并进行重绘，让 View 表现出动画的效果。



## 1.1 ValueAnimator

> 一句话描述 `ValueAnimator` 的作用：屏幕刷新信号到来时，计算该时间点对应的属性值，并将属性值的更新通知到观察者们。

### 1.1.1 基本用法

````kotlin
class MainActivity : AppCompatActivity() {
    // 1.实例化ValueAnimator
    val mValueAnimator = ValueAnimator()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        
        // 3.监听属性值的更新
        mValueAnimator.addUpdateListener { valueAnimator ->
            val h = valueAnimator.animatedValue as Int
            // 4.更新View的属性
            mCellLayout.updateLayoutParams {
                height = h
            }
        }
        
        ...
    }
    
    // 2.在合适的时机启动动画
    private fun startAnimate() {
        mValueAnimator.setIntValues(0, 100)
        mValueAnimator.start()
    }
}
````

### 1.1.2 源码分析

#### 1.1.2.1 ofInt()

通常使用 `ValueAnimator.ofXxx()` 来实例化 `VlaueAnimator`，这里以 `ofInt()` 为例

```java
// ValueAnimator.java

// 保存所有的属性值
PropertyValuesHolder[] mValues;

// 用于支持通过属性名来获取属性值，空间换时间
HashMap<String, PropertyValuesHolder> mValuesMap;

public static ValueAnimator ofInt(int... values) {
    ValueAnimator anim = new ValueAnimator();
    anim.setIntValues(values);
    return anim;
}

// 唯一的构造方法，啥也没干
public ValueAnimator() { }

public void setIntValues(int... values) {
    if (values == null || values.length == 0) {
        return;
    }
    if (mValues == null || mValues.length == 0) {
        // 首次设值，将int类型的数据封装为PropertyValuesHolder类型
        setValues(PropertyValuesHolder.ofInt("", values));
    } else {
        // 非首次设值，复用数组的第0号位置
        PropertyValuesHolder valuesHolder = mValues[0];
        valuesHolder.setIntValues(values);
    }
    mInitialized = false;
}

public void setValues(PropertyValuesHolder... values) {
    int numValues = values.length;
    mValues = values;
    mValuesMap = new HashMap<>(numValues);
    for (int i = 0; i < numValues; ++i) {
        PropertyValuesHolder valuesHolder = values[i];
        mValuesMap.put(valuesHolder.getPropertyName(), valuesHolder);
    }
    mInitialized = false;
}
```

这里出现的 `PropertyValuesHolder`，是真正负责存储动画属性值的容器

```java
// ValueAnimator.java

public Object getAnimatedValue(String propertyName) {
    PropertyValuesHolder valuesHolder = mValuesMap.get(propertyName);
    if (valuesHolder != null) {
        // 动画的属性值最终也是通过PropertyValuesHolder来获取
        return valuesHolder.getAnimatedValue();
    } else {
        return null;
    }
}
```

这里对 `PropertyValuesHolder` 有如下认知即可：

* 实例化 `PropertyValuesHolder` 需要传入一个或多个属性值，每个属性值可理解为动画执行时的关键帧
* 实例化 `PropertyValuesHolder` 需要传入 `TypeEvaluator` 用于将动画执行百分比转换为我们想到的值
* 一个 `PropertyValuesHolder` 对应一种属性，如果需要同时执行多个动画，可以设置多个 `PropertyValuesHolder`



#### 1.1.2.2 start()

调用 `start()` 启动动画

````java
// ValueAnimator.java

private void start(boolean playBackwards) {
    // 省略大部分成员变量的初始化代码...
    
    addAnimationCallback(0);

    if (mStartDelay == 0 || mSeekFraction >= 0 || mReversing) {
        // 内部也是做一些初始化工作
        startAnimation();
    }
}

private void addAnimationCallback(long delay) {
    // 注意这里传入的参数是this，是ValueAnimator对AnimationHandler.AnimationFrameCallback的实现
    // 从方法名来看，该实现会在某个时候被回调
    getAnimationHandler().addAnimationFrameCallback(this, delay);
}

public AnimationHandler getAnimationHandler() {
    return mAnimationHandler != null ? mAnimationHandler : AnimationHandler.getInstance();
}
````

> 这里出现了一个新类 `AnimationHandler`，而且是一个单例类
>
> 一句话描述 `AnimationHandler` 的作用：

````java
// AnimationHandler.java

// 保存设置了延迟执行，且还未执行的AnimationFrameCallback
private final ArrayMap<AnimationFrameCallback, Long> mDelayedCallbackStartTime = new ArrayMap<>();
// 保存所有未结束动画的AnimationFrameCallback
private final ArrayList<AnimationFrameCallback> mAnimationCallbacks = new ArrayList<>();
private final ArrayList<AnimationFrameCallback> mCommitCallbacks = new ArrayList<>();

public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
    if (mAnimationCallbacks.size() == 0) {
        // mProvider默认实现是MyFrameCallbackProvider
        // 内部会通过Choreographer向底层注册监听下一个屏幕刷新信号事件
        getProvider().postFrameCallback(mFrameCallback);
    }
    // 将传入的AnimationFrameCallback缓存起来
    if (!mAnimationCallbacks.contains(callback)) {
        mAnimationCallbacks.add(callback);
    }
    if (delay > 0) {
        mDelayedCallbackStartTime.put(callback, (SystemClock.uptimeMillis() + delay));
    }
}
````

当屏幕刷新信号到来时，回调 `Choreographer.FrameCallback#doFrame(frameTimeNanos)`

````java
// AnimationHandler.java

private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        doAnimationFrame(getProvider().getFrameTime());
        if (mAnimationCallbacks.size() > 0) {
            // mAnimationCallbacks的size大于0，说明还有动画需要执行
            // 所以需要再次注册监听下一个屏幕刷新信号事件
            getProvider().postFrameCallback(this);
        }
    }
};

private void doAnimationFrame(long frameTime) {
    long currentTime = SystemClock.uptimeMillis();
    final int size = mAnimationCallbacks.size();
    for (int i = 0; i < size; i++) {
        final AnimationFrameCallback callback = mAnimationCallbacks.get(i);
        // 动画结束，会将自身在mAnimationCallbacks中的引用赋值为null
        if (callback == null) {
            continue;
        }
        // 若可以执行动画，则为ture（不能执行动画的情况为设置了延迟执行且还没到执行时间）
        if (isCallbackDue(callback, currentTime)) {
            callback.doAnimationFrame(frameTime);
        }
    }
    // 移除mAnimationCallbacks中引用为null的项
    cleanUpList();
}
````

 刷新信号到达后，调用了 `callback.doAnimationFrame(frameTime)`，又回到了 `ValueAnimator`

````java
// ValueAnimator.java

public final boolean doAnimationFrame(long frameTime) {
    final long currentTime = Math.max(frameTime, mStartTime);
    // 属性动画的核心，根据当前时间计算对应的属性值，并通知所有观察者 
    boolean finished = animateBasedOnTime(currentTime);
    if (finished) {
        // 动画结束，将自身在mAnimationCallbacks中的引用赋值为null
        endAnimation();
    }
    return finished;
}
````

以上流程的时序图如下：

![](https://raw.githubusercontent.com/WillisNotFound/Pic/master/Android/%E8%87%AA%E5%AE%9A%E4%B9%89View-%E5%B1%9E%E6%80%A7%E5%8A%A8%E7%94%BB-%E6%BA%90%E7%A0%81%E6%97%B6%E5%BA%8F%E5%9B%BE1.png)

从 `animateBaseOnTime()` 开始，是属性动画的核心流程

````java
// ValueAnimator.java

boolean animateBasedOnTime(long currentTime) {
    boolean done = false;
    if (mRunning) {
        // 省略了前面一堆逻辑：计算动画执行时间百分比判断动画是否结束
        // 最终得到动画执行时间的百分比currentIterationFraction
        animateValue(currentIterationFraction);
    }
    return done;
}

void animateValue(float fraction) {
    // 根据插值器将动画执行时间百分比转换为动画真正执行进度的百分比
    fraction = mInterpolator.getInterpolation(fraction);
    mCurrentFraction = fraction;
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
        // 根据动画执行百分比，映射为我们需要的值
        mValues[i].calculateValue(fraction);
    }
    // 通知动画进度更新
    if (mSeekFraction >= 0 || mStartListenersCalled) {
        callOnList(mUpdateListeners, AnimatorCaller.ON_UPDATE, this, false);
    }
}
````



## 1.2 ObjectAnimator

`ObjectAnimator` 继承自 `ValueAnimator`，提供相关方法来简化使用动画所需编写的代码

````kotlin
ObjectAnimator.ofFloat(mCellLayout, "translationY", 0F, 100F)
	.setDuration(1000)
	.start()
````



## 1.3 ViewPropertyAnimator

`ObjectAnimator` 内部实现用的也是 `ValueAnimator`，相比于 `ObjectAnimator`，最少只需一行代码即可启动动画：

* 通过 `View#animate()` 获取与该 View 关联的 `ViewPropertyAnimator` 对象
* 再调用相关的 API 即可实现简单的动画效果

````kotlin
val viewPropertyAnimator = animate()
viewPropertyAnimator.setDuration(500L) // 设置动画时长
	.translationYBy(500F) // 向下平移500像素
	.translationXBy(50F) // 向右平移50像素
````

更多 API 自行查阅[ViewPropertyAnimator](https://developer.android.com/reference/android/view/ViewPropertyAnimator)



# 参考资料

* [属性动画 ValueAnimator 运行原理全解析](https://www.cnblogs.com/dasusu/p/8595422.html)