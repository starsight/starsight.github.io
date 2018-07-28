---
layout: post
title: "翻页效果的RecyclerView"
date: 2018-07-20 20:37:00 +0800
comments: true
categories: 2018-07
tags: [android,recycleView]
---

实现一个自定义LayoutManager，默认只有一个抽象方法，实现的重点在于onLayoutChildren对页面的布局和滑动操作，当然还有缓存。

### 属性定义

mPosition是当前item的位置信息，mPositionOffset是偏移信息，mMinVy是最低的y方向的速度，这个需要根据不同屏幕尺寸来定。

```java
private static final int MIN_VY = 300;
private int mPosition = 0;
private int mPositionOffset = 0;
private int mMinVy = 0;
private Context mContext;
```

### 布局流程

为了实现翻页效果，每次滑动都是有前景页和背景页。当翻页时，如果flip没有超过一半，当前页是primary（前景页），下一页是secondary（背景页）；当超过一半当前页已经是“下一页”（背景页）了，而刚才的当前页变成了上一页（前景页）。<!--more-->值得注意的是，需要把需要把item布局文件的**背景设置成白色**，不然会有重叠效果。翻页效果在FlipCard中实现。

```java
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {

    //如果没有item，直接返回
    if (getItemCount() <= 0) {
        return;
    }
    // 跳过preLayout，preLayout主要用于支持动画
    if (state.isPreLayout()) {
        return;
    }
    fill(recycler, state);
}

private void fill(RecyclerView.Recycler recycler, RecyclerView.State state) {

        checkPosition(state);

        View primary;
        View pre = null;
        View next = null;

        detachAndScrapAttachedViews(recycler);

    	// 当前页
        primary = recycler.getViewForPosition(mPosition);

        if (mPosition + 1 < state.getItemCount()) {
            // 下一页
            next = recycler.getViewForPosition(mPosition + 1);
        }
        if (mPosition - 1 >= 0) {
            // 上一页
            pre = recycler.getViewForPosition(mPosition - 1);
        }
        View secondary = null;

    	// 根据偏移计算是需要上一页还是下一页
        if (mPositionOffset > 0) {
            secondary = next;
        } else if (mPositionOffset < 0) {
            secondary = pre;
        }
        if (mPositionOffset != 0 && secondary != null) {
            // 存在背景页就添加
            addView(secondary);
            measureChildWithMargins(secondary, 0, 0);
            layoutDecorated(secondary, 0, 0, getWidth(), getHeight());
        }

    	// 回收另一页的信息
        if (pre != null && secondary !=pre) {
            recycler.recycleView(pre);
        }
        if (next != null && secondary !=next) {
            recycler.recycleView(next);
        }

    	// 添加当前页作为前景页
        addView(primary);
        measureChildWithMargins(primary, 0, 0);
        layoutDecorated(primary, 0, 0, getWidth(), getHeight());

        if (primary instanceof FlipCard && (secondary == null || secondary instanceof FlipCard)) {
            final float percent = (float) mPositionOffset / getItemHeightPositon();
            // 计算滑动比例，得到flip效果
            Log.d(TAG, "fill: "+percent);
            if (secondary != null) {
                ((FlipCard) secondary).setState(false, percent);
                ((FlipCard) primary).setState(true, percent);
            } else {
                // 当在第一页或最后一页时不存在背景页（留白/刷新等）
                ((FlipCard) primary).setState(true, percent);
            }
        } else {
            throw new IllegalStateException("view should be FlipCard");
        }
    }

    private void checkPosition(RecyclerView.State state) {
        // 页面最多能滑动到 -2/5*itemHeight ~ itemHeight*(state.getItemCount()-1) + itemHeight*2/5
        final int itemHeight = getItemHeightPositon();
        final int current = mPosition * itemHeight + mPositionOffset;
        final int max = itemHeight * (state.getItemCount() - 1) + itemHeight * 2 / 5;

        int pos = Math.max(-itemHeight * 2 / 5, Math.min(current, max));
        mPosition = Math.round(pos / (float)itemHeight);
        mPosition = mPosition >= 0 ? mPosition : 0;
        mPositionOffset = pos - mPosition * itemHeight;
    }
```

为了使翻页效果流畅，设置每一页的高度都是实际高度的2/3，这样避免完全滑满一个屏幕才完成翻页效果（不考虑惯性滑动）。

```java
// 这样把一个item的高度设为了原来的2/3，直观的结果就是只要滑动2/3就相当于过了一个item
private int getItemHeightInPositon() {
    return getHeight() * 2 / 3;
}
```

### 滑动

完成上述步骤自然还不能滑动

```java
@Override
public boolean canScrollVertically() {
	// 纵向可滑
    return true;
}

@Override
public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
    final int before = mPosition * getItemHeightPositon() + mPositionOffset;

    mPositionOffset += dy;
    checkPosition(state);
    final int after = mPosition * getItemHeightPositon() + mPositionOffset;

    final int ans = after - before;
    fill(recycler, state);

    return ans;
}
```

```java
@Override
public void scrollToPosition(int position) {
    mPosition = position;
    mPositionOffset = 0;
    requestLayout();
    Log.d(TAG, "scrollToPosition " + position + " position " + mPosition + " positionOffset " + mPositionOffset);
}

// 平稳滑动
@Override
public void smoothScrollToPosition(RecyclerView recyclerView, RecyclerView.State state, int position) {
    Log.d(TAG, "smoothScrollTo " + position + " position " + mPosition + " positionOffset " + mPositionOffset);

    FlipScroller scroller = new FlipScroller(recyclerView.getContext());
    scroller.setTargetPosition(position);
    startSmoothScroll(scroller);

}

private class FlipScroller extends LinearSmoothScroller {
    private static final String TAG = "FlipScroller";

    public FlipScroller(Context context) {
        super(context);
    }

    @Override
    protected void onStop() {
        super.onStop();
        Log.d(TAG, "onStop: ");
    }

    @Override
    public int calculateDyToMakeVisible(View view, int snapPreference) {
        final int position = getPosition(view);
        final int now = mPositionOffset + mPosition * getItemHeightPositon();
        final int to = position * getItemHeightPositon();
        Log.d(TAG, "calculateDyToMakeVisible: position " + position + " ans " + (to - now));
        return (now - to);
    }

    @Override
    public int calculateDxToMakeVisible(View view, int snapPreference) {
        return 0;
    }

    @Override
    protected int calculateTimeForScrolling(int dx) {

        Log.d(TAG, "calculateTimeForScrolling: ");
        int time = super.calculateTimeForScrolling(dx*5);

        return time;
    }

}
/*
以下四个方法是为了滑动时的对齐操作，不至于滑了半页然后就停在那
详细可看：https://www.jianshu.com/p/e54db232df62
*/
@Override
public PointF computeScrollVectorForPosition(int targetPosition) {
    int dir = 0;
    int now = mPosition * getItemHeightPositon() + mPositionOffset;
    int to = targetPosition * getItemHeightPositon();
    if (now > to) {
        dir = -1;
    } else if (now < to) {
        dir = 1;
    }
    Log.d(TAG, "computeScrollVector " + dir + " now " + mPosition + " target " + targetPosition);

    return new PointF(0, dir);
}
public int calculateDistance(View view) {
    int pos = getPosition(view);
    final int now = getItemHeightPositon() * mPosition + mPositionOffset;
    final int to = getItemHeightPositon() * pos;

    return to - now;
}

public int findTargetPosition(int vY) {
    int ans = mPosition;
    Log.d(TAG, "findTargetPosition: "+vY+"~"+mPositionOffset);
    int absV = vY > 0 ? vY : -vY;
    if (absV > mMinVy) {
        if (vY * mPositionOffset > 0) {
            // 速度与位置偏移同向
            int d = vY > 0 ? 1 : -1;
            ans += d;
        } else {
            ans = mPosition;
        }
    } else {
        ans = mPosition;
    }

    int count = getItemCount();
    if (count == 0) {
        return 0;
    }

    ans = Math.min(count - 1, Math.max(0, ans));
    return ans;
}

public View findSnapView() {
    Log.d(TAG, "findSnapView: "+getChildCount()+"~"+getItemCount());
    for (int i = 0;i < getChildCount(); i ++ ){
        View child = getChildAt(i);
        if (getPosition(child) == mPosition) {
            return child;
        }
    }
    return null;
}
```

```java
// 滑动时的对齐操作
// 惯性滑动先根据findTargetSnapPosition()计算到TargetSnapView，再根据calculateDistanceToFinalSnap()计算到TargetSnapView与对齐位置的剩余距离。普通滑动就是等到滑动停止，findSnapView()找到需要对齐的View即SanpView，再calculateDistanceToFinalSnap()计算得到额外的滑动距离。
public class MySnap extends SnapHelper {
    private static final String TAG = "MySnap";

    @Nullable
    @Override
    public int[] calculateDistanceToFinalSnap(@NonNull RecyclerView.LayoutManager layoutManager, @NonNull View targetView) {
        if (layoutManager instanceof CustomLayoutManager) {
            return new int[]{0, ((CustomLayoutManager) layoutManager).calculateDistance(targetView)};
        } else {
            throw new RuntimeException();
        }
    }

    @Nullable
    @Override
    public View findSnapView(RecyclerView.LayoutManager layoutManager) {
        CustomLayoutManager flipLayoutManager = (CustomLayoutManager) layoutManager;

        return flipLayoutManager.findSnapView();
    }

    @Override
    public int findTargetSnapPosition(RecyclerView.LayoutManager layoutManager, int velocityX, int velocityY) {
        if (layoutManager instanceof CustomLayoutManager) {
            return ((CustomLayoutManager) layoutManager).findTargetPosition(velocityY);
        } else {
            throw new RuntimeException();
        }
    }
}
```

### 翻页View

```java
public class FlipCard extends FrameLayout {

    private static final String TAG = "FlipCard";

    private Paint mScrimPaint;

    private Camera mCamera;
    private Matrix mMatrix;

    private boolean mIsForground;
    private float mPercent;

    public FlipCard(Context context) {
        this(context, null);
    }

    public FlipCard(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public FlipCard(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);

        mScrimPaint = new Paint();
        mCamera = new Camera();
        mMatrix = new Matrix();
    }

    public void setState(boolean isForground, float percent) {
        mIsForground = isForground;
        mPercent = percent;
    }

    @Override
    public void draw(Canvas canvas) {

        if (mPercent == 0) {
            super.draw(canvas);
            return;
        }

        final int height = canvas.getHeight();
        final int width = canvas.getWidth();
        final float percent = mPercent > 0 ? mPercent : -mPercent;

        if (mIsForground) {
            // clip card effect for forground view
            // draw part1
            int save1 = canvas.save();
            if (mPercent > 0) {
                canvas.clipRect(0, 0, width, height / 2);
            } else {
                canvas.clipRect(0, height / 2, width, height);
            }
            super.draw(canvas);
            canvas.restoreToCount(save1);

            // draw part2
            if (mPercent < 0) {
                canvas.clipRect(0, 0, width, height / 2);
            } else {
                canvas.clipRect(0, height / 2, width, height);
            }
            mCamera.save();
            mCamera.setLocation(0f, 0f, -80);
            mCamera.rotateX(mPercent * 180);
            mCamera.getMatrix(mMatrix);
            mCamera.restore();
            mMatrix.preTranslate(-width / 2, -height / 2);
            mMatrix.postTranslate(width / 2, height / 2);
            canvas.concat(mMatrix);
            super.draw(canvas);

            mScrimPaint.setColor(0x08000000);
            canvas.drawRect(0, 0, width, height, mScrimPaint);
        } else {
            // 作为背景，不需要有flip效果，只需要切出一半view展示出来即可
            // draw shadow for underground view
            final int scrimColor = (int) (0xff * (1 - percent * 2)) << 24;
            mScrimPaint.setColor(scrimColor);

            if (mPercent < 0) {
                canvas.clipRect(0, 0, width, height / 2);
            } else {
                canvas.clipRect(0, height / 2, width, height);
            }

            super.draw(canvas);
            canvas.drawRect(0, 0, width, height, mScrimPaint);
        }
    }

}
```

### 使用

```java
customLayoutManager = new CustomLayoutManager(this);
recyclerView.setLayoutManager(customLayoutManager);
adapter = new CustomAdapter(myData,MainActivity.this);
recyclerView.setAdapter(adapter);

MySnap snap = new MySnap();
snap.attachToRecyclerView(recyclerView);
```
### 缓存

翻页效果带来了本身其滑动范围有限，依次只能上下翻页，因为对于缓存，首先稍微设置mCacheViews大一些，默认为2，由于每次翻页会绑定下一页，因此我设置为3来保证由本页（例：1）翻到下一页（例：2）时，不会因为加载再下一页（例：3）而导致原来的上一页（例：0）被回收。

另一项策略时预加载Prefetch，对于自定义LayoutManager，需要重写[LayoutManager.collectAdjacentPrefetchPositions()](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.LayoutManager.html#collectAdjacentPrefetchPositions%28int,%20int,%20android.support.v7.widget.RecyclerView.State,%20android.support.v7.widget.RecyclerView.LayoutManager.LayoutPrefetchRegistry%29)方法，可以参考LinearLayoutManager实现。

包含完整源码的项目：<https://github.com/starsight/Gank>

### 参考链接

[让你明明白白的使用RecyclerView——SnapHelper详解](https://www.jianshu.com/p/e54db232df62)

https://medium.com/google-developers/recyclerview-prefetch-c2f269075710

[Android ListView 与 RecyclerView 对比浅析--缓存机制](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653578065&idx=2&sn=25e64a8bb7b5934cf0ce2e49549a80d6&chksm=84b3b156b3c43840061c28869671da915a25cf3be54891f040a3532e1bb17f9d32e244b79e3f&mpshare=1&scene=1&srcid=0626Dl0bjG3664qJlKb7RSlq#rd)