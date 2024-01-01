# 下拉刷新组件

源码：https://github.com/LiYaokun97/ui-framework

建议跟着源码来理解下面的内容



## 需要具备的知识

- GestureDetector 

https://cloud.tencent.com/developer/article/1641523

- onScroller 

https://blog.csdn.net/guolin_blog/article/details/48719871

- android 事件分发机制 

https://www.bilibili.com/video/BV1d54y1d7y7/?p=1&spm_id_from=pageDriver 

事件分发机制，有助于你理解cancel事件



## 下拉刷新组件的设计

让我们从一个简单的下拉刷新组件需要做到什么，来分析该如何设计下拉刷新组件。

当一个列表滑动到顶部时，我们下拉一段距离X，

1. 如果X大于某个固定距离X0，用户松手后，先回弹到固定距离X0，然后触发刷新，刷新结束后，回到初始状态
2. 如果X小于该固定距离X0，用户松手后，直接回到初始状态，未触发刷新

**后面我们如果称情况1，就是指这里的1，情况2就是指这里的2**

根据上面简单的分析，整个过程分为两个部分：

   A. 下拉刷新组件随着手指的下滑，跟着下滑 

   B. 松手后，回弹（回弹有两种情况，分别是上面的1 2两种情况）

A B两个部分在代码中也分别对应两个部分，A部分我们需要使用GestureDetector 来使我们的组件跟着用户的下滑而下滑，B部分我们则需要使用onScroller来实现回弹效果。

如果说得详细一些，那就是在下拉刷新组件的dispatchTouchEvent中，当用户未松开手之前，控制权限交由GestureDetector，用户松手之后，控制权限则交由onScroller。

### 术语规范

为了方便理解，先解释一下后面会出现的术语

- HiRfreshLayout：我们要实现的下拉刷新组件，可以看UML图
- header ： 也就是下拉后出现的view，它是HiRfreshLayout 的 index为0 的子view
- child：可滑动的view，可以是RecyclerView，可以是ListView，是HiRefreshLayout的index 为1的子view
-  mHiOverView.mPullRefreshHeight ： 也就是上面说的X0，用户滑动超过这个距离就会refresh



### UML类图



![](hi_refresh-uml.png)





### 状态设计

我们一共设计了5中状态，来跟踪上述的过程，分别是

```
public enum HiRefreshState {
    /**
     * 初始态
     */
    STATE_INIT,
    /**
     * Header展示的状态，实际实现中，只要用户下拉了就会由state_init变为state_visible，
     * 因为设计的header底部可能是空白的，
     * 实际上header已经可见了，但是因为header底部是空白的，你依然看不见，这种情况下，状态也是state_visible
     */
    STATE_VISIBLE,
    /**
     * 超出可刷新距离的状态, 只用来设置HiOverView，在HiRfreshLayout并不会设置这个状态
     */
    STATE_OVER,
    /**
     * 刷新中的状态
     */
    STATE_REFRESH,
    /**
     * 超出刷新位置松开手后的状态
     */
    STATE_OVER_RELEASE
}
```

如果用户的操作是上面说的1，那么HiRfreshLayout中mState 的变化是：

null -> STATE_INIT -> STATE_VISIBLE -> STATE_OVER_RELEASE -> STATE_REFRESH -> STATE_INIT

这里如果是首次刷新，mState是null，如果不是首次触发刷新，则是从STATE_INIT开始的（设计得有点不好，应该首次触发也是STATE_INIT) ；另外，最后从state_refresh -> state_init，也就是刷新完回到初始状态这个阶段，中间是可以插入一个state_visible状态的，但是这会让系统更加复杂，这里就直接设置为init状态了。

如果用户的操作是上面说的2，那么HiRfreshLayout中mState 的变化是：

null -> STATE_INIT -> STATE_VISIBLE -> STATE_INIT

实际上用户的滑动还可以是更复杂的操作：先下滑超过距离X0，不放手，然后再上滑，上滑到距离顶部小于X0的距离，这个时候松手，不应该触发refresh操作，这个过程mState的变化：

null -> STATE_INIT -> STATE_VISIBLE -> STATE_INIT



建议下载源码，自行通过打Log 体验一下mState的变化



### dispatchTouchEvent 方法

先从dispatchTouchEvent 开始讲起

```java
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        //事件分发处理
        if (!mAutoScroller.isFinished()) {
            return false;
        }

        View head = getChildAt(0);
        // 注释1
        if (ev.getAction() == MotionEvent.ACTION_UP || ev.getAction() == MotionEvent.ACTION_CANCEL) {//松开手
            HiLog.it(TAG + " dispatchTouchEvent", "release");
            if (head.getBottom() > 0) {
                HiLog.it(TAG + " dispatchTouchEvent", "bottom > 0");
                if (mState != HiRefreshState.STATE_REFRESH) {//非正在刷新
                    HiLog.it(TAG + " dispatchTouchEvent", "recover");
                    recover(head.getBottom());
                    return false;
                }
            }
            mLastY = 0;
        }
        
        
        // 注释2 
        boolean consumed = mGestureDetector.onTouchEvent(ev);
        HiLog.it(TAG + " dispatchTouchEvent", "gesture consumed：" + consumed);
        if ((consumed || (mState != HiRefreshState.STATE_INIT && mState != HiRefreshState.STATE_REFRESH)) && head.getBottom() != 0) {
            HiLog.it(TAG + " dispatchTouchEvent", "send cancel to parent");
            ev.setAction(MotionEvent.ACTION_CANCEL);//让父类接受不到真实的事件
            return super.dispatchTouchEvent(ev);
        }

        if (consumed) {
            return true;
        } else {
            return super.dispatchTouchEvent(ev);
        }
    }

```



注释1的代码的if，时判断用户是否松手了，如果进入了这if，就说明用户已经松手了，最终会调用recover函数，里面会将滑动控制权交给onScroller

看到注释2，如果没有走进注释1的if，就说明用户还没有松手，这个时候，控制权交由mGestureDetector



```java

    private void init() {
        mGestureDetector = new GestureDetector(getContext(), hiGestureDetector);
        mAutoScroller = new AutoScroller();
    }

	HiGestureDetector hiGestureDetector = new HiGestureDetector() {
        @Override
        public boolean onScroll(MotionEvent e1, MotionEvent e2, float disX, float disY) {
            if (Math.abs(disX) > Math.abs(disY) || mHiRefreshListener != null && !enableRefresh) {
                //横向滑动，或刷新被禁止则不处理
                return false;
            }
            if (disableRefreshScroll && mState == HiRefreshState.STATE_REFRESH) {//刷新时是否禁止滑动
                return true;
            }

            View head = getChildAt(0);
            View child = HiScrollUtil.findScrollableChild(HiRefreshLayout.this);
            if (HiScrollUtil.childScrolled(child)) {
                //如果列表发生了滚动则不处理
                return false;
            }
            //没有刷新或没有达到可以刷新的距离，且头部已经划出或下拉
            
            // 注释3
            if ((mState == null || mState == HiRefreshState.STATE_INIT || mState == HiRefreshState.STATE_VISIBLE)
                    && (head.getBottom() > 0 || disY <= 0.0F)) {
                //还在滑动中
                int height;
                //阻尼计算
                if (child.getTop() < mHiOverView.mPullRefreshHeight) {
                    height = (int) (mLastY / mHiOverView.minDamp);
                } else {
                    height = (int) (mLastY / mHiOverView.maxDamp);
                }
                //如果是正在刷新状态，则不允许在滑动的时候改变状态
                boolean bool = moveDown(height);
                mLastY = (int) (-disY);
                return bool;
            } else {
                HiLog.it(TAG + " hiGestureDetector", "return false");
                return false;
            }
        }
    };
```



看上面的代码，mGestureDetector 定义时，设置了OnGestureListener，也就是hiGestureDetector，它的onScroll会在系统检测到滑动操作时被调用

注释3前的代码，是对特殊情况的处理，

- 如果时横向滑动，不应该响应
- 如果设置了禁止滑动，则直接返回true，不做任何操作，这样就起到了禁止滑动的效果
- 如果列表发生了滚动则不处理

注释3处代码，由 状态设计 的分析，我们可以得出结论 null -> STATE_INIT -> STATE_VISIBLE，这个过程中的每一个状态，用户手指都还没松开，所以注释3的if判断了是否等于这三个状态。

阻尼计算：用户下滑100个单位，我们的下拉刷新组件应该只滑动也许60个单位，也许40个单位，组件实际下滑的高度，就是通过mHiOverView.minDamp 和mHiOverView.maxDamp 来控制的

接下来，看到moveDown 函数

```java

    private void updateStateWhenDown(int childTop) {
        if (childTop <= mHiOverView.mPullRefreshHeight) {
            setRefreshState(HiRefreshState.STATE_VISIBLE);
            if (mHiOverView.getState() != HiRefreshState.STATE_VISIBLE) {
                mHiOverView.onVisible();
                mHiOverView.setState(HiRefreshState.STATE_VISIBLE);
            }
        } else {
            //超出刷新位置
            if (mHiOverView.getState() != HiRefreshState.STATE_OVER) {
                mHiOverView.onOver();
                mHiOverView.setState(HiRefreshState.STATE_OVER);
            }
        }
    }

    /**
     * 根据偏移量移动header与child
     *
     * @param offsetY 偏移量
     * @return
     */
    private boolean moveDown(int offsetY) {
        HiLog.it(TAG + " moveDown", "offsetY:" + offsetY);
        View head = getChildAt(0);
        View child = getChildAt(1);
        int childTop = child.getTop() + offsetY;

        HiLog.it(TAG + " moveDown", "current state: " + mState);
        HiLog.it(TAG + " moveDown", "mPullRefreshHeight: " + mHiOverView.mPullRefreshHeight);
        HiLog.it(TAG + " moveDown", "moveDown head-bottom: " + head.getBottom() + ", child.getTop():" + child.getTop() + ", offsetY:" + offsetY);

        if (mState == null) {
            setRefreshState(HiRefreshState.STATE_INIT);
        }

        // 注释4
        switch (mState) {
            case STATE_INIT:
            case STATE_VISIBLE:
                //移动head与child的位置，到原始位置
                head.offsetTopAndBottom(offsetY);
                child.offsetTopAndBottom(offsetY);
                updateStateWhenDown(childTop);
                break;

            default:
                HiLog.it(TAG + " moveDown", "default");
                setRefreshState(HiRefreshState.STATE_INIT);
                break;
        }

        if (mHiOverView != null) {
            mHiOverView.onScroll(head.getBottom(), mHiOverView.mPullRefreshHeight);
        }

        return true;
    }

```

看到注释4，这实际上就是一个状态机，因为这个函数只会在 state为null -> STATE_INIT -> STATE_VISIBLE 三种情况下进入，所以switch只需要管着三种情况就行了，在STATE_INIT 和 STATE_VISIBLE，我们更新了head和child的位置，也就是把它们滑动到对应的位置。

moveDown会一直被调用，直到用户手指不动（还在屏幕上 未拿起）或者用户已经拿起手指

当用户拿起手指，up事件将被触发，又回到了dispatchTouchEvent的注释1，这个时候我们会走入这个if中，这个时候mState还没有被更新，还是处在STATE_VISIBLE状态下，故而可以走入这个if

```java
if (mState != HiRefreshState.STATE_REFRESH) {//非正在刷新
                    HiLog.it(TAG + " dispatchTouchEvent", "recover");
                    recover(head.getBottom());
                    return false;
                }
```

将调用recover，代码如下

```java
    private void recover(int dis) {//dis =200  200-100
        if (dis > mHiOverView.mPullRefreshHeight) {
            mAutoScroller.recover(dis - mHiOverView.mPullRefreshHeight); // 注释5
            setRefreshState(HiRefreshState.STATE_OVER_RELEASE);
            // 恢复到刷新位置
//            getHandler().postAtTime(() -> refreshFinished(), System.currentTimeMillis() + refreshTime);
            postDelayed(()-> refreshFinished(), refreshTime); // 注释6
        } else {
            refreshFinished();
        }
    }

```

会调用我们自定义的AutoScroller 中的recover 函数，见下

在这个函数中，我们调用startScroll开始滑动（mScroller.startScroll并不会真的开始滑动，真的滑动是在run函数里的moveUpWhenRelease 实现的），并通过post 把这个Runnable放入队列中执行。而在run中则会不断调用post 把自身放入队列中，直到滚动完成，

这样下拉刷新组件最终就能滚动recover方法中传入的 dis 距离

看到注释5，注释5滚动的距离是 dis - mHiOverView.mPullRefreshHeight，也就是超过固定距离X0的距离大小，这样下拉刷新组件将在滑动到该位置后开始刷新



```java
    /**
     * 借助Scroller实现视图的自动滚动
     * https://juejin.im/post/5c7f4f0351882562ed516ab6
     */
    private class AutoScroller implements Runnable {
        private Scroller mScroller;
        private int mLastY;
        private boolean mIsFinished;

        AutoScroller() {
            mScroller = new Scroller(getContext(), new LinearInterpolator());
            mIsFinished = true;
        }

        @Override
        public void run() {
            if (mScroller.computeScrollOffset()) {//还未滚动完成
                moveUpWhenRelease(mLastY - mScroller.getCurrY());
                mLastY = mScroller.getCurrY();
                post(this);
            } else {
                removeCallbacks(this);
                mIsFinished = true;
            }
        }

        void recover(int dis) {
            if (dis <= 0) {
                return;
            }
            removeCallbacks(this);
            mLastY = 0;
            mIsFinished = false;
            mScroller.startScroll(0, 0, 0, dis, 300);
            post(this);
        }

        boolean isFinished() {
            return mIsFinished;
        }

    }
```



接下来让我们看到moveUpWhenRelease

```java
    /**
     * 根据偏移量移动header与child
     *
     * @param offsetY 偏移量
     * @return
     */
    private boolean moveUpWhenRelease(int offsetY) {
        HiLog.it(TAG + " moveUpWhenRelease", "offsetY:" + offsetY);
        View head = getChildAt(0);
        View child = getChildAt(1);
        int childTop = child.getTop() + offsetY;

        HiLog.it(TAG + " moveUpWhenRelease", "current state: " + mState);
        HiLog.it(TAG + " moveUpWhenRelease", "mPullRefreshHeight: " + mHiOverView.mPullRefreshHeight);
        HiLog.it(TAG + " moveUpWhenRelease", "moveDown head-bottom: " + head.getBottom() + ", child.getTop():" + child.getTop() + ", offsetY:" + offsetY);
        if (childTop <= 0) {//异常情况的补充
            HiLog.it(TAG  + " moveUpWhenRelease", "case 1");
            offsetY = -child.getTop();
            //移动head与child的位置，到原始位置
            head.offsetTopAndBottom(offsetY);
            child.offsetTopAndBottom(offsetY);
            if (mState != HiRefreshState.STATE_REFRESH) {
                setRefreshState(HiRefreshState.STATE_INIT);
            }
        } else if (childTop <= mHiOverView.mPullRefreshHeight) {//还没超出设定的刷新距离
            // 注释7
            HiLog.it(TAG  + " moveUpWhenRelease", "case 3");
            head.offsetTopAndBottom(offsetY);
            child.offsetTopAndBottom(offsetY);
            if (childTop == mHiOverView.mPullRefreshHeight && mState == HiRefreshState.STATE_OVER_RELEASE) {
                HiLog.it(TAG + " moveUpWhenRelease", "refresh，childTop：" + childTop);
                refresh();
            }
        } else {
            HiLog.it(TAG  + " moveUpWhenRelease", "case 4"); // 注释8
            head.offsetTopAndBottom(offsetY);
            child.offsetTopAndBottom(offsetY);
        }
        if (mHiOverView != null) {
            mHiOverView.onScroll(head.getBottom(), mHiOverView.mPullRefreshHeight);
        }
        return true;
    }

```

这里没有用状态机来实现，逻辑有点混乱，可以试着自己重写一下这个函数

如果是情况1，会先走到注释8，然后不断走到注释7，在这里当child的top与X0相等时（回弹到刷新位置），就会开始刷新

再看到注释6（注释6在函数recover中），这里postDelayed了一个 refreshFinished 函数，它负责将组件从刷新位置恢复到初始位置，可以自行看下代码

```java

    public void refreshFinished() {
        final View head = getChildAt(0);
        HiLog.i(this.getClass().getSimpleName(), "refreshFinished head-bottom:" + head.getBottom());
        mHiOverView.onFinish();
        mHiOverView.setState(HiRefreshState.STATE_INIT);
        final int bottom = head.getBottom();

        if (mHiRefreshListener != null) {
            mHiRefreshListener.onRefreshFinished();
        }

        if (bottom > 0) {
            //下over pull 200，height 100
             //  bottom  =100 ,height 100
            mAutoScroller.recover(bottom);
        }
        setRefreshState(HiRefreshState.STATE_INIT);
    }

```



最终，就回到了初始位置。

情况2 可以自行分析一下。



