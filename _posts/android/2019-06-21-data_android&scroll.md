---
layout: post
title: Android ScrollTo, ScrollBy
category: Android
tags: Android Scroll 方法
---
* content
{:toc}

>详解Android 中的ScrollTo, ScrollBy方法

 1. 首先看一下网上的一些答案以及说法:
    scrollTo(-x,-y), scrollBy(-x,-y), 如果传入负值就向正方向移动,传入正值向负方向移动
    还有说移动的是里面的内容, 为什么会这样呢.
    总结一句话就是,移动的方向和你传入的正负数相反的方向. 可有没有想过为什么会这样? 本篇将解释为什么会这样. 下面的一些图片更直观的

![avatar](https://github.com/skypx/BlogResource/raw/master/view/scrool1.png)
![avatar](https://github.com/skypx/BlogResource/raw/master/view/scrool2.png)
![avatar](https://github.com/skypx/BlogResource/raw/master/view/scrool3.png)
![avatar](https://github.com/skypx/BlogResource/raw/master/view/scrool4.png)


>源码分析

1. scrollTo, scrollBy 源码

```java
    /**
     * Set the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the x position to scroll to
     * @param y the y position to scroll to
     */
    public void scrollTo(int x, int y) {
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }

    /**
     * Move the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the amount of pixels to scroll by horizontally
     * @param y the amount of pixels to scroll by vertically
     */
    public void scrollBy(int x, int y) {
        scrollTo(mScrollX + x, mScrollY + y);
    }

```
原来scrollBy最终还是调用scrollTo，那我们接着来看scrollTo，scrollTo改变的是View的mScrollX和mScrollY这两个属性，我们来看下文档对这两个属性的解释：

```java
  /**
     * The offset, in pixels, by which the content of this view is scrolled
     * horizontally.
     * {@hide}
     */
    @ViewDebug.ExportedProperty(category = "scrolling")
    protected int mScrollX;
    /**
     * The offset, in pixels, by which the content of this view is scrolled
     * vertically.
     * {@hide}
     */
    @ViewDebug.ExportedProperty(category = "scrolling")
    protected int mScrollY;
```

指的是View内容的偏移量，如果是ViewGroup的话作用的就是它的所有子view，如果是TextView的话则作用的就是TextView的内容。这两个api作用的对象是view的内容而不是view本身。

## scrollTo与scrollBy的区别

从上面源码，注意scrollTo(int x,int y)与scrollBy里的参数都是指偏移量，scrollTo是一步到位直接修改偏移量为x或y，而scrollBy是在当前偏移量加减x或y。这样说好像不是很准确，我们先来看下Android坐标系以及一些参数

![avatar](https://github.com/skypx/BlogResource/raw/master/view/zuobiao.png)


一般情况下View的坐标都是相对父ViewGroup，像以下的api：

1. getTop()
2. getLeft()
3. getRight()
4. getBottom()
5. getX()
6. getY()


是不是发现明明通过scrollTo设置的偏移量是-300，按照正常的逻辑以及android坐标系，应该向上移动，怎么还向下移动了呢，这时候我们就要深入源码来查看究竟了

```java
/**
     * Set the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the x position to scroll to
     * @param y the y position to scroll to
     */
    public void scrollTo(int x, int y) {
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }
```
先来看看onScrollChanged

```java
/**
     * This is called in response to an internal scroll in this view (i.e., the
     * view scrolled its own contents). This is typically as a result of
     * {@link #scrollBy(int, int)} or {@link #scrollTo(int, int)} having been
     * called.
     *
     * @param l Current horizontal scroll origin.
     * @param t Current vertical scroll origin.
     * @param oldl Previous horizontal scroll origin.
     * @param oldt Previous vertical scroll origin.
     */
    protected void onScrollChanged(int l, int t, int oldl, int oldt) {
        notifySubtreeAccessibilityStateChangedIfNeeded();

        if (AccessibilityManager.getInstance(mContext).isEnabled()) {
            postSendViewScrolledAccessibilityEventCallback();
        }

        mBackgroundSizeChanged = true;
        mDefaultFocusHighlightSizeChanged = true;
        if (mForegroundInfo != null) {
            mForegroundInfo.mBoundsChanged = true;
        }

        final AttachInfo ai = mAttachInfo;
        if (ai != null) {
            ai.mViewScrollChanged = true;
        }

        if (mListenerInfo != null && mListenerInfo.mOnScrollChangeListener != null) {
            mListenerInfo.mOnScrollChangeListener.onScrollChange(this, l, t, oldl, oldt);
        }
    }
```

当View注册了OnScrollChangeListener，onScrollChange才会被调用。
接着就调用了postInvalidateOnAnimation

```java
/**
     * <p>Cause an invalidate to happen on the next animation time step, typically the
     * next display frame.</p>
     *
     * <p>This method can be invoked from outside of the UI thread
     * only when this View is attached to a window.</p>
     *
     * @see #invalidate()
     */
    public void postInvalidateOnAnimation() {
        // We try only with the AttachInfo because there's no point in invalidating
        // if we are not attached to our window
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            attachInfo.mViewRootImpl.dispatchInvalidateOnAnimation(this);
        }
    }
```

在来看看dispatchInvalidateOnAnimation的实现

```java
 public void dispatchInvalidateOnAnimation(View view) {
        mInvalidateOnAnimationRunnable.addView(view);
    }
```
```java
final class InvalidateOnAnimationRunnable implements Runnable {
        private boolean mPosted;
        private final ArrayList<View> mViews = new ArrayList<View>();
        private final ArrayList<AttachInfo.InvalidateInfo> mViewRects =
                new ArrayList<AttachInfo.InvalidateInfo>();
        private View[] mTempViews;
        private AttachInfo.InvalidateInfo[] mTempViewRects;

        public void addView(View view) {
            synchronized (this) {
                mViews.add(view);
                postIfNeededLocked();
            }
        }

        public void addViewRect(AttachInfo.InvalidateInfo info) {
            synchronized (this) {
                mViewRects.add(info);
                postIfNeededLocked();
            }
        }

        public void removeView(View view) {
            synchronized (this) {
                mViews.remove(view);

                for (int i = mViewRects.size(); i-- > 0; ) {
                    AttachInfo.InvalidateInfo info = mViewRects.get(i);
                    if (info.target == view) {
                        mViewRects.remove(i);
                        info.recycle();
                    }
                }

                if (mPosted && mViews.isEmpty() && mViewRects.isEmpty()) {
                    mChoreographer.removeCallbacks(Choreographer.CALLBACK_ANIMATION, this, null);
                    mPosted = false;
                }
            }
        }

        @Override
        public void run() {
            final int viewCount;
            final int viewRectCount;
            synchronized (this) {
                mPosted = false;

                viewCount = mViews.size();
                if (viewCount != 0) {
                    mTempViews = mViews.toArray(mTempViews != null
                            ? mTempViews : new View[viewCount]);
                    mViews.clear();
                }

                viewRectCount = mViewRects.size();
                if (viewRectCount != 0) {
                    mTempViewRects = mViewRects.toArray(mTempViewRects != null
                            ? mTempViewRects : new AttachInfo.InvalidateInfo[viewRectCount]);
                    mViewRects.clear();
                }
            }

            for (int i = 0; i < viewCount; i++) {
                mTempViews[i].invalidate();
                mTempViews[i] = null;
            }

            for (int i = 0; i < viewRectCount; i++) {
                final View.AttachInfo.InvalidateInfo info = mTempViewRects[i];
                info.target.invalidate(info.left, info.top, info.right, info.bottom);
                info.recycle();
            }
        }

        private void postIfNeededLocked() {
            if (!mPosted) {
                mChoreographer.postCallback(Choreographer.CALLBACK_ANIMATION, this, null);
                mPosted = true;
            }
        }
    }
```

接着会调用run方法，其中有

```java
 for (int i = 0; i < viewCount; i++) {
                mTempViews[i].invalidate();
                mTempViews[i] = null;
            }
```

遍历所有添加进去的View，然后对每个View调用invalidate()，

```java
    /**
     * Invalidate the whole view. If the view is visible,
     * {@link #onDraw(android.graphics.Canvas)} will be called at some point in
     * the future.
     * <p>
     * This must be called from a UI thread. To call from a non-UI thread, call
     * {@link #postInvalidate()}.
     */
    public void invalidate() {
        invalidate(true);
    }
```

```java
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
        if (mGhostView != null) {
            mGhostView.invalidate(true);
            return;
        }

        if (skipInvalidate()) {
            return;
        }

        if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
            if (fullInvalidate) {
                mLastIsOpaque = isOpaque();
                mPrivateFlags &= ~PFLAG_DRAWN;
            }

            mPrivateFlags |= PFLAG_DIRTY;

            if (invalidateCache) {
                mPrivateFlags |= PFLAG_INVALIDATED;
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            }

            // Propagate the damage rectangle to the parent view.
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                damage.set(l, t, r, b);
                p.invalidateChild(this, damage);
            }

            // Damage the entire projection receiver, if necessary.
            if (mBackground != null && mBackground.isProjected()) {
                final View receiver = getProjectionReceiver();
                if (receiver != null) {
                    receiver.damageInParent();
                }
            }
        }
    }

```

注意这个地方, 这个地方是绘制view面的内容,这也解释了为什么说是用scrollto 或者 scrollby来
移动里面的内容, 而不是这个view本身

```java
if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                damage.set(l, t, r, b);
                p.invalidateChild(this, damage);
            }
```

接着

```java
/**
     * Don't call or override this method. It is used for the implementation of
     * the view hierarchy.
     *
     * @deprecated Use {@link #onDescendantInvalidated(View, View)} instead to observe updates to
     * draw state in descendants.
     */
    @Deprecated
    @Override
    public final void invalidateChild(View child, final Rect dirty) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null && attachInfo.mHardwareAccelerated) {
            // HW accelerated fast path
            onDescendantInvalidated(child, child);
            return;
        }

        ViewParent parent = this;
        if (attachInfo != null) {
            // If the child is drawing an animation, we want to copy this flag onto
            // ourselves and the parent to make sure the invalidate request goes
            // through
            final boolean drawAnimation = (child.mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0;

            // Check whether the child that requests the invalidate is fully opaque
            // Views being animated or transformed are not considered opaque because we may
            // be invalidating their old position and need the parent to paint behind them.
            Matrix childMatrix = child.getMatrix();
            final boolean isOpaque = child.isOpaque() && !drawAnimation &&
                    child.getAnimation() == null && childMatrix.isIdentity();
            // Mark the child as dirty, using the appropriate flag
            // Make sure we do not set both flags at the same time
            int opaqueFlag = isOpaque ? PFLAG_DIRTY_OPAQUE : PFLAG_DIRTY;

            if (child.mLayerType != LAYER_TYPE_NONE) {
                mPrivateFlags |= PFLAG_INVALIDATED;
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            }

            final int[] location = attachInfo.mInvalidateChildLocation;
            location[CHILD_LEFT_INDEX] = child.mLeft;
            location[CHILD_TOP_INDEX] = child.mTop;
            if (!childMatrix.isIdentity() ||
                    (mGroupFlags & ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS) != 0) {
                RectF boundingRect = attachInfo.mTmpTransformRect;
                boundingRect.set(dirty);
                Matrix transformMatrix;
                if ((mGroupFlags & ViewGroup.FLAG_SUPPORT_STATIC_TRANSFORMATIONS) != 0) {
                    Transformation t = attachInfo.mTmpTransformation;
                    boolean transformed = getChildStaticTransformation(child, t);
                    if (transformed) {
                        transformMatrix = attachInfo.mTmpMatrix;
                        transformMatrix.set(t.getMatrix());
                        if (!childMatrix.isIdentity()) {
                            transformMatrix.preConcat(childMatrix);
                        }
                    } else {
                        transformMatrix = childMatrix;
                    }
                } else {
                    transformMatrix = childMatrix;
                }
                transformMatrix.mapRect(boundingRect);
                dirty.set((int) Math.floor(boundingRect.left),
                        (int) Math.floor(boundingRect.top),
                        (int) Math.ceil(boundingRect.right),
                        (int) Math.ceil(boundingRect.bottom));
            }

            do {
                View view = null;
                if (parent instanceof View) {
                    view = (View) parent;
                }

                if (drawAnimation) {
                    if (view != null) {
                        view.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
                    } else if (parent instanceof ViewRootImpl) {
                        ((ViewRootImpl) parent).mIsAnimating = true;
                    }
                }

                // If the parent is dirty opaque or not dirty, mark it dirty with the opaque
                // flag coming from the child that initiated the invalidate
                if (view != null) {
                    if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                            view.getSolidColor() == 0) {
                        opaqueFlag = PFLAG_DIRTY;
                    }
                    if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY) {
                        view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
                    }
                }

                parent = parent.invalidateChildInParent(location, dirty);
                if (view != null) {
                    // Account for transform on current parent
                    Matrix m = view.getMatrix();
                    if (!m.isIdentity()) {
                        RectF boundingRect = attachInfo.mTmpTransformRect;
                        boundingRect.set(dirty);
                        m.mapRect(boundingRect);
                        dirty.set((int) Math.floor(boundingRect.left),
                                (int) Math.floor(boundingRect.top),
                                (int) Math.ceil(boundingRect.right),
                                (int) Math.ceil(boundingRect.bottom));
                    }
                }
            } while (parent != null);
        }
    }
```

会有这样的一行代码：parent = parent.invalidateChildInParent(location, dirty);

```java
 public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
            if(this.mHostView != null) {
                dirty.offset(location[0], location[1]);
                if(this.mHostView instanceof ViewGroup) {
                    location[0] = 0;
                    location[1] = 0;
                    int[] offset = new int[2];
                    this.getOffset(offset);
                    dirty.offset(offset[0], offset[1]);
                    return super.invalidateChildInParent(location, dirty);
                }

                this.invalidate(dirty);
            }

            return null;
        }
```

```java
  /**
     * Mark the area defined by dirty as needing to be drawn. If the view is
     * visible, {@link #onDraw(android.graphics.Canvas)} will be called at some
     * point in the future.
     * <p>
     * This must be called from a UI thread. To call from a non-UI thread, call
     * {@link #postInvalidate()}.
     * <p>
     * <b>WARNING:</b> In API 19 and below, this method may be destructive to
     * {@code dirty}.
     *
     * @param dirty the rectangle representing the bounds of the dirty region
     */
    public void invalidate(Rect dirty) {
        final int scrollX = mScrollX;
        final int scrollY = mScrollY;
        invalidateInternal(dirty.left - scrollX, dirty.top - scrollY,
                dirty.right - scrollX, dirty.bottom - scrollY, true, false);
    }
```

**这个地方都是减去scrollX 和减去scrollY, 当你传入负数的时候就是正值了,所以, 我们就和开篇对应起来了, 传入负数向正方向移动, 传入正数向负方向移动**

参考: <https://www.jianshu.com/p/ce618bebb043>
