---
layout: post
title: Android 列表滚动优化之 OverScroller 揭秘
tags: [Android,OverScroller]
mathjax: true
categories: 
- Android
---

# 1. 简介

`OverScroller` 在 Android 系统中承担着为 ListView、RecyclerView、ScrollView 这些滚动控件计算实时滑动位置的任务，这些位置算法直接影响着每一次滚动的体验

众所周知，Android 的动画体验~~远~~不如 iOS，即便如今 Android 已普遍支持 120Hz 高刷，体验起来也不是非常舒服。究其原因已经不是硬件性能限制，而是其中很多动画设计本身就有问题。苹果早在很早之前就发布了 [Designing Fluid Interfaces](https://developer.apple.com/videos/play/wwdc2018/803) 致力于打造一个丝滑流畅的用户体验，反观 Android，对于一个日常使用中使用最多的滑动工具类 `OverScroller` 近几年改进竟然寥寥无几，几乎没有，实在是有点想吐槽

这个系列分为两篇，第一篇主要讲述 Android 实现滚动的核心工具类 `OverScroller` 的使用方法和原理，第二篇我们将探索如何进行改进，希望我们每一次探索都能给用户体验带来提升

<!-- more -->

# 2. 使用介绍

在使用之前，先来看看 OverScroller 能做什么：

 - startScroll：从指定位置滚动一段指定的距离然后停下，滚动效果与设置的滚动距离、滚动时间、插值器有关，**跟离手速度没有关系**。一般用于控制 View 滚动到**指定**的位置
 - fling：从指定位置滑动一段位置然后停下，滚动效果只与**离手速度**以及滑动边界有关，**不能**设置滚动距离、滚动时间和插值器。一般用于触摸抬手后继续让 View 滑动一会
 - springBack：从指定位置回弹到指定位置，一般用于实现拖拽后的回弹效果，不能指定回弹时间和插值器

|startScroll|fling|springBack|
|:-:|:-:|:-:|
| <img style="border: none;" src="http://image.hanschen.site/master/2021-04-09-16-11-34.gif"> | <img style="border: none;" src="http://image.hanschen.site/master/2021-04-09-16-11-51.gif"> |<img style="border: none;" src="http://image.hanschen.site/master/2021-04-09-16-12-02.gif"> |


在代码中使用也很简单：

```java
    // 1. 启动一个滚动
    mOverScroller.startScroll(0, 1600, 0, -1000, 1000);
    // 2. 启动定时刷新任务
    post(new Runnable() {
        @Override
        public void run() {
            // 3. 计算当前最新位置信息
            if (mOverScroller.computeScrollOffset()) {
                // 4. 根据最新位置更新 View 状态
                Log.d("OverScroller", "x=" + mOverScroller.getCurrX() + ", y=" + mOverScroller.getCurrY());
                invalidate();
                // 5. 判断滚动是否停止，没有停止的话启动下一轮刷新任务
                if (!mOverScroller.isFinished()) {
                    postDelayed(this, 16);
                }
            }
        }
    });
```

上面就是启动一个不断滚动并刷新 View 的最小逻辑（当然更工程化的实践也可以把 Runnable 的逻辑放在 View#computeScroll 里再通过 invalidate 触发）。`fling` 和 `springBack` 的启动方式也是一样的，这里就不再进行赘述了。


# 3. 深入 OverScroller 内部

在上面代码中可知，启动一个滚动任务后，是通过不断地调用 computeScrollOffset 来计算位置的，接下来看下代码实现

## 3.1 startScroll

```java

public class OverScroller {

    public void startScroll(int startX, int startY, int dx, int dy, int duration) {
        // 标记当前模式为 SCROLL_MODE
        mMode = SCROLL_MODE;
        // mScrollerX 和 mScrollerY 均是 SplineOverScroller 实例
        // OverScroller 把参数分别传给 mScrollerX 和 mScrollerY，在里面做真正的计算
        mScrollerX.startScroll(startX, dx, duration);
        mScrollerY.startScroll(startY, dy, duration);
    }

    static class SplineOverScroller {
        void startScroll(int start, int distance, int duration) {
            mFinished = false;
            // 标记起始点和结束点
            mCurrentPosition = mStart = start;
            mFinal = start + distance;

            // 标记起始时间和动画时长
            mStartTime = AnimationUtils.currentAnimationTimeMillis();
            mDuration = duration;

            // Unused
            mDeceleration = 0.0f;
            mVelocity = 0;
        }
    }
}
```

startScroll 的逻辑非常的简单，只是根据参数标记了一下开始位置、结束位置、开始时间、动画时长，还没涉及位置计算（因为位置计算是放在 computeScrollOffset 的呀）

再看看定时调用的 computeScrollOffset 逻辑：

```java
public class OverScroller {

    public OverScroller(Context context, Interpolator interpolator, boolean flywheel) {
        // 初始化插值器
        if (interpolator == null) {
            mInterpolator = new Scroller.ViscousFluidInterpolator();
        } else {
            mInterpolator = interpolator;
        }
        ...
    }

    public boolean computeScrollOffset() {
        if (isFinished()) {
            return false;
        }
        switch (mMode) {
            case SCROLL_MODE:
                long time = AnimationUtils.currentAnimationTimeMillis();
                // 计算已过去的时间
                final long elapsedTime = time - mScrollerX.mStartTime;

                final int duration = mScrollerX.mDuration;
                if (elapsedTime < duration) {
                    // 用插值器对时间比做一个变换
                    final float q = mInterpolator.getInterpolation(elapsedTime / (float) duration);
                    mScrollerX.updateScroll(q);
                    mScrollerY.updateScroll(q);
                } else {
                    abortAnimation();
                }
                break;
            break;
        }
        return true;
    }

    static class SplineOverScroller {

        void updateScroll(float q) {
            // 根据比值计算最新位置
            mCurrentPosition = mStart + Math.round(q * (mFinal - mStart));
        }
    }
}
```

逻辑也是很简单，实在没太多可说的……就是把插值器曲线映射到位移曲线，时长如果不指定的话，默认是 250ms，插值器需要通过构造方法传入，如果不指定的话，系统默认会指定一个 `ViscousFluidInterpolator`，下面是这个插值器的曲线，可以看到是一个先缓后快再缓的动画

![ViscousFluidInterpolator](http://image.hanschen.site/master/2021-04-09-17-31-04.png)


## 3.2 fling & springBack

`fling` 和 `springBack` 为什么要一起说呢？因为 `fling` 的动画比较复杂，`springBack` 算是属于 `fling` 的其中一个子状态，考虑以下这个情况：

<table>
    <tr>
        <td><img style="border: none;" src="http://image.hanschen.site/master/2021-04-09-17-44-46.gif" width = "300" /></td>
        <td><img style="border: none;" src="http://image.hanschen.site/master/2021-04-09-18-25-11.png" width = "300" /></td>
    </tr>
</table>

我们看到当以比较大的速度执行 fling 的时候，是很容易碰到边界的，fling 会根据预设的边界值执行越界并回弹，可以把整个动画过程分解成三个阶段：

 - SPLINE：也就是正常滑动阶段
 - BALLISTIC：越界减速阶段
 - CUBIC：回弹阶段

`springBack` 后执行的动画，其实就是 `fling` 的 `CUBIC` 阶段，所以干脆就放在一起说了

这个命名其实也挺有意思，这三个分别是样条曲线、弹道曲线、三次曲线，从命名上大致也可以推断出，三个阶段采用的「时间-位置」曲线是不一样的。

当然了，Android 很多控件在  fling 的时候，都把越界回弹效果取消掉，取而代之的是显示一个 `EdgeEffect`。也就是说执行完 SPLINE 阶段动画后，是看不到 `BALLISTIC` 和 `CUBIC` 的，只能看到一个边缘辉光效果，列表到达顶/底部的时候，往往一下子停在那里了~


### 3.2.1 SPLINE

先来看看启动 fling 的入口函数：

```java
public class OverScroller {

    public void fling(int startX, int startY, int velocityX, int velocityY,
            int minX, int maxX, int minY, int maxY, int overX, int overY) {
        // 标记滚动模式，主要和 startScroll 进行区分
        mMode = FLING_MODE;
        mScrollerX.fling(startX, velocityX, minX, maxX, overX);
        mScrollerY.fling(startY, velocityY, minY, maxY, overY);
    }

    static class SplineOverScroller {

        void fling(int start, int velocity, int min, int max, int over) {
            mOver = over;
            mFinished = false;
            mCurrVelocity = mVelocity = velocity;
            mDuration = mSplineDuration = 0;
            mStartTime = AnimationUtils.currentAnimationTimeMillis();
            mCurrentPosition = mStart = start;

            mState = SPLINE;
            double totalDistance = 0.0;

            if (velocity != 0) {
                // 根据速度计算滑动时长
                mDuration = mSplineDuration = getSplineFlingDuration(velocity);
                // 根据速度计算滑动距离
                totalDistance = getSplineFlingDistance(velocity);
            }

            mSplineDistance = (int) (totalDistance * Math.signum(velocity));
            mFinal = start + mSplineDistance;

            if (mFinal < min) {
                // 如果计算出的滑动距离超过 min 边界，则重新计算到达 min 边界时的滑动时长
                adjustDuration(mStart, mFinal, min);
                mFinal = min;
            }

            if (mFinal > max) {
                // 如果计算出的滑动距离超过 max 边界，则重新计算到达 max 边界时的滑动时长
                adjustDuration(mStart, mFinal, max);
                mFinal = max;
            }
        }

        private double getSplineDeceleration(int velocity) {
            return Math.log(INFLEXION * Math.abs(velocity) / (mFlingFriction * mPhysicalCoeff));
        }

        /**
         * 计算滑动距离
         */
        private double getSplineFlingDistance(int velocity) {
            final double l = getSplineDeceleration(velocity);
            final double decelMinusOne = DECELERATION_RATE - 1.0;
            return mFlingFriction * mPhysicalCoeff * Math.exp(DECELERATION_RATE / decelMinusOne * l);
        }

        /**
         * 计算滑动时长
         */
        private int getSplineFlingDuration(int velocity) {
            final double l = getSplineDeceleration(velocity);
            final double decelMinusOne = DECELERATION_RATE - 1.0;
            return (int) (1000.0 * Math.exp(l / decelMinusOne));
        }
    }
}
```

<img style="border: none;" src="http://image.hanschen.site/master/2021-04-09-19-21-01.png"/>


看代码前先来看看上面的图，图中说明了 start、min、max、over 等位置的意义，这里简要说明一下

 - fling 后最终停下来的位置必须在 min 和 max 区间之间
 - BALLISTIC 越界阶段，不能超过 over 的位置，即 over 是最大越界距离
 - start 可以位于 [min, max] 区间之外，如果处于区间之外，会根据速度方向和大小做一个决策，会有以下三种情况：
    - 速度指向边界外：先执行 BALLISTIC 越界再执行 CUBIC 回弹回到边界
    - 速度指向边界内：如果速度足以越过边界，则按照正常流程执行，先执行 SPLINE
    - 速度指向边界内：如果速度不足以越过边界，直接执行 CUBIC 回到边界


fling 函数主要功能：

 - 根据起始速度计算滑动的时长和距离
 - 如果计算出最终点处于 [min, max] 区间之外，则重新计算到达边界的时长
 - 标记当前状态为 SPLINE 状态


滑动距离和时长的计算公式中，可以把 mPhysicalCoeff 也看做常数，把**时长-速度公式**和图像列出来：

$$
y=1000\cdot \exp \left( \frac{\ln \left( \frac{0.35x}{2140.47} \right)}{1.358} \right)
$$


![2021-04-12-10-26-21](http://image.hanschen.site/master/2021-04-12-10-26-21.png)

再看看**距离-速度公式**：

$$
y=2140.47\cdot \exp \left( 1.74\cdot \ln \left( \frac{0.35\cdot x}{2140.47} \right) \right)
$$
![2021-04-12-10-30-53](http://image.hanschen.site/master/2021-04-12-10-30-53.png)


可以看到距离和时长都是随着速度增大而增大的，只不过时长的增长速度在后期会有一定的收敛，保证动画时长不至于太长

还有一点要注意的是，$ \frac{Distance} {Duration} $ 的比值是一个线性函数，也就是初速度越大，平均速度越大，两者是线性增长的：

$$
y=\frac{2140.47\cdot \exp \left( 1.74\cdot \ln \left( \frac{0.35\cdot x}{2140.47} \right) \right)}{1000\cdot \exp \left( \frac{\ln \left( \frac{0.35x}{2140.47} \right)}{1.358} \right)}
$$

![2021-04-12-10-45-06](http://image.hanschen.site/master/2021-04-12-10-45-06.png)

但是说实话，目前我暂时没想明白这两个公式的物理意义，有明白的大佬求告知~ 难道是利用了对数函数收敛的特性确定了时长公式，然后设定平均速度线性增长后，推导出距离公式？

目前只确定了滑动总距离和时长，那么中间过程是怎么更新位置的呢：

```java
public class OverScroller {

    public boolean computeScrollOffset() {
        if (isFinished()) {
            return false;
        }

        switch (mMode) {
            case FLING_MODE:
                ...
                if (!mScrollerY.mFinished) {
                    // 更新当前阶段的速度和位置
                    if (!mScrollerY.update()) {
                        // 判断是否需要进入下一动画阶段
                        if (!mScrollerY.continueWhenFinished()) {
                            mScrollerY.finish();
                        }
                    }
                }
                break;
        }
        return true;
    }

    static class SplineOverScroller {

        boolean update() {
            final long time = AnimationUtils.currentAnimationTimeMillis();
            final long currentTime = time - mStartTime;

            if (currentTime == 0) {
                return mDuration > 0;
            }
            // 根据动画时长判断当前阶段的动画是否应该结束
            if (currentTime > mDuration) {
                return false;
            }

            double distance = 0.0;
            switch (mState) {
                case SPLINE: {
                    // 把当前时间映射到 100 个采样点的曲线之中
                    final float t = (float) currentTime / mSplineDuration;
                    final int index = (int) (NB_SAMPLES * t);
                    float distanceCoef = 1.f;
                    float velocityCoef = 0.f;
                    if (index < NB_SAMPLES) {
                        final float t_inf = (float) index / NB_SAMPLES;
                        final float t_sup = (float) (index + 1) / NB_SAMPLES;
                        final float d_inf = SPLINE_POSITION[index];
                        final float d_sup = SPLINE_POSITION[index + 1];
                        // 根据映射时间计算样条曲线的当前的斜率，即速度
                        velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
                        // 根据映射时间计算样条曲线的高度，即距离
                        distanceCoef = d_inf + (t - t_inf) * velocityCoef;
                    }
                    // 把样条距离映射回真正的距离
                    distance = distanceCoef * mSplineDistance;
                    // 把样条速递易映射回真正的速度
                    mCurrVelocity = velocityCoef * mSplineDistance / mSplineDuration * 1000.0f;
                    break;
                }

                case BALLISTIC: {
                    // 根据匀减速运动公式计算位置和速度
                    final float t = currentTime / 1000.0f;
                    mCurrVelocity = mVelocity + mDeceleration * t;
                    distance = mVelocity * t + mDeceleration * t * t / 2.0f;
                    break;
                }

                case CUBIC: {
                    // 根据一个自定义的三次曲线计算位置和速度
                    final float t = (float) (currentTime) / mDuration;
                    final float t2 = t * t;
                    final float sign = Math.signum(mVelocity);
                    distance = sign * mOver * (3.0f * t2 - 2.0f * t * t2);
                    mCurrVelocity = sign * mOver * 6.0f * (- t + t2);
                    break;
                }
            }

            mCurrentPosition = mStart + (int) Math.round(distance);

            return true;
        }

        boolean continueWhenFinished() {
            switch (mState) {
                case SPLINE:
                    if (mDuration < mSplineDuration) {
                        // 如果时长小于 mSplineDuration ，说明 mDuration 被重新计算过，即上面说到的到达边界的时间。那么我们需要进入越界状态
                        mCurrentPosition = mStart = mFinal;
                        mVelocity = (int) mCurrVelocity;
                        mDeceleration = getDeceleration(mVelocity);
                        mStartTime += mDuration;
                        onEdgeReached();
                    } else {
                        // SPLINE 停止时未到达边界，结束动画
                        return false;
                    }
                    break;
                case BALLISTIC:
                    // 越界状态结束，进入回弹阶段
                    mStartTime += mDuration;
                    startSpringback(mFinal, mStart, 0);
                    break;
                case CUBIC:
                    // 回弹阶段结束，结束动画
                    return false;
            }

            update();
            return true;
        }
    }
}
```

`SPLINE` 阶段的位置和速度完全是由预设的 `SPLINE_POSITION` 样条曲线决定的，它是一个大小为 101 的数组，里面存储了一条曲线平均采样 100 次的坐标值，初始化如下：

```java
        private static final int NB_SAMPLES = 100;
        private static final float[] SPLINE_POSITION = new float[NB_SAMPLES + 1];
        private static final float[] SPLINE_TIME = new float[NB_SAMPLES + 1];

        static {
            float x_min = 0.0f;
            float y_min = 0.0f;
            for (int i = 0; i < NB_SAMPLES; i++) {
                final float alpha = (float) i / NB_SAMPLES;

                float x_max = 1.0f;
                float x, tx, coef;
                while (true) {
                    x = x_min + (x_max - x_min) / 2.0f;
                    coef = 3.0f * x * (1.0f - x);
                    tx = coef * ((1.0f - x) * P1 + x * P2) + x * x * x;
                    if (Math.abs(tx - alpha) < 1E-5) break;
                    if (tx > alpha) x_max = x;
                    else x_min = x;
                }
                SPLINE_POSITION[i] = coef * ((1.0f - x) * START_TENSION + x) + x * x * x;
            }
            SPLINE_POSITION[NB_SAMPLES] = SPLINE_TIME[NB_SAMPLES] = 1.0f;
        }
```

看代码很难想象它长什么样，直接看看它的图像吧：

![2021-04-12-11-24-04](http://image.hanschen.site/master/2021-04-12-11-24-04.png)


也就是说，SPLINE 和 startScroll 很像，位置曲线都是由一根预置的曲线决定的，把预置曲线映射真实的距离，只是 SPLINE 没有使用插值器曲线，而是使用了一根缓停的样条曲线


### 3.2.2 BALLISTIC

SPLINE 阶段结束后，会通过 continueWhenFinished 进入下一阶段：越界阶段（前提是此时已经到达边界）。越界阶段的原理相对简单，就是一段匀减速运动（直至速度降为 0），默认加速度 a 为 -2000.0f，到达边界进入 BALLISTIC 阶段的初始化逻辑在 `onEdgeReached` 中：


```java
    private void onEdgeReached() {
        final float velocitySquared = (float) mVelocity * mVelocity;
        // 计算速度降为 0 时，运动的距离
        float distance = velocitySquared / (2.0f * Math.abs(mDeceleration));
        final float sign = Math.signum(mVelocity);

        if (distance > mOver) {
                // 如果距离大于最大距离 over，则重新计算加速度，使运动距离恰好为 over
                mDeceleration = - sign * velocitySquared / (2.0f * mOver);
                distance = mOver;
        }

        mOver = (int) distance;
        // 标记动画阶段
        mState = BALLISTIC;
        mFinal = mStart + (int) (mVelocity > 0 ? distance : -distance);
        // 根据初速度和加速度计算总时长
        mDuration = - (int) (1000.0f * mVelocity / mDeceleration);
    }
```

速度：
$$
v_t=v_0+at
$$

距离：
$$
s_t=v_0t + \frac{at^2} {2}
$$

这里有个小吐槽，加速度固定是 2000，这也太小了吧，也就是说，如果越界速度为 10000， 那么需要 5s 之后，速度才能降为 0，一个 5s 的动画童鞋们估计都知道意味着多久吧？


### 3.2.3 CUBIC

上一节内容知道，BALLISTIC 阶段结束时，速度已经降低为 0，我们终于来到最后一段， 从 continueWhenFinished 里会调用 `startSpringback` 作为 CUBIC 的初始化：

```java
    private void startSpringback(int start, int end, int velocity) {
        mFinished = false;
        // 标记动画阶段
        mState = CUBIC;
        mCurrentPosition = mStart = start;
        mFinal = end;
        // CUBIC 阶段运动总距离
        final int delta = start - end;
        mDeceleration = getDeceleration(delta);
        mVelocity = -delta; // only sign is used
        mOver = Math.abs(delta);
        // 计算此阶段的动画时长
        mDuration = (int) (1000.0 * Math.sqrt(-2.0 * delta / mDeceleration));
    }
```

时长计算依然使用匀加速直线运动的逻辑，想象一段初速度为 0，加速度为 a， 距离为 delta 的情况：

$$ 
delta=\frac{(v_0 + v_t)} {2} t
$$

则：
$ v_0=0，v_t=at $，那么 $ delta=\frac{at^2} {2} $，$ t=\sqrt{ \frac{2*delta} {a}}$

在 `update` 方法中，更新 CUBIC 的核心逻辑是：

```java
    case CUBIC: {
        // 根据一个自定义的三次曲线计算位置和速度
        final float t = (float) (currentTime) / mDuration;
        final float t2 = t * t;
        final float sign = Math.signum(mVelocity);
        // 计算运动位置
        distance = sign * mOver * (3.0f * t2 - 2.0f * t * t2);
        // 计算速度，对距离公式求导即可
        mCurrVelocity = sign * mOver * 6.0f * (- t + t2);
        break;
    }
```

核心逻辑是这个 `3.0f * t2 - 2.0f * t * t2`, 这其实是一个比较常用的三次曲线：

![2021-04-12-14-34-02](http://image.hanschen.site/master/2021-04-12-14-34-02.png)


在 [0, 1] 区间内，是一个缓入缓出的曲线。至此，CUBIC 的运动规律也摸清楚了，在固定时间内，把时间映射到 [0, 1] 的区间，再把 y 坐标映射实际的位置

# 4. 小结

目前为止，我们终于把 `startScroll` 和 `fling` 各阶段的曲线看了一遍，至于 `springBack` 和其他一些情况都大同小异，就不细述了。

很多时候 OverScroller 都是只是使用的固定的曲线映射真正的曲线，比如 `startScroll`、`SPLINE` 和 `CUBIC`，那如果想改变效果的话，是不是修改一下曲线形态就可以了呢？但一条曲线是否真的能在不同速度下都有比较好的表现吗？或许我们还要有很多的实践和尝试才能做出一段让用户舒服滑动，而这些探索和尝试，将放在下一篇文章中详细讨论~

