---
layout: post
title: 扫盲系列之Retrofit 基本用法
category: 扫盲系列
tags:  Android Retrofit
---
* content
{:toc}

##  1. test

 ### 2. 这是一个

```java
public class OutRecyclerView extends RecyclerView {
    private static final String TAG = "OutRecyclerView";
    private int mInitialTouchX;
    private int mInitialTouchY;

    private static final int INVALID_POINTER = -1;
    private int mScrollPointerId = INVALID_POINTER;

    private int mTouchSlop;
    private ViewConfiguration mViewConfiguration;
    public OutRecyclerView(Context context) {
        this(context, null);
    }

    public OutRecyclerView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public OutRecyclerView(Context context, @Nullable AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        mViewConfiguration = ViewConfiguration.get(context);
        mTouchSlop = mViewConfiguration.getScaledTouchSlop();
    }

    @Override
    public void setScrollingTouchSlop(int slopConstant) {
        super.setScrollingTouchSlop(slopConstant);
        switch (slopConstant) {
            case TOUCH_SLOP_DEFAULT:
                mTouchSlop = mViewConfiguration.getScaledTouchSlop();
                break;
            case TOUCH_SLOP_PAGING:
                mTouchSlop = mViewConfiguration.getScaledPagingTouchSlop();
                break;
            default:
                break;
        }
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent e) {
        final int action = e.getAction();
        Log.d(TAG, getClass().getSimpleName() + "    action : " + action + "   getScrollState: " + getScrollState() + "  ");
        boolean intercepted = false;
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                int state = getScrollState();
                mScrollPointerId = e.getPointerId(0);
                mInitialTouchX = (int) (e.getX() + 0.5f);
                mInitialTouchY = (int) (e.getY() + 0.5f);
                intercepted = super.onInterceptTouchEvent(e);
                if (state == SCROLL_STATE_SETTLING || state == SCROLL_STATE_DRAGGING) {
                    intercepted = false;
                }
                return intercepted;
            case MotionEvent.ACTION_MOVE: {
                final int index = MotionEventCompat.findPointerIndex(e, mScrollPointerId);
                if (index < 0) {
                    return false;
                }
                final int x = (int) (MotionEventCompat.getX(e, index) + 0.5f);
                final int y = (int) (MotionEventCompat.getY(e, index) + 0.5f);
                final int dx = x - mInitialTouchX;
                final int dy = y - mInitialTouchY;
                final boolean canScrollHorizontally = getLayoutManager().canScrollHorizontally();
                boolean startScroll = false;
                if (canScrollHorizontally && Math.abs(dx) > mTouchSlop && (Math.abs(dx) >= Math.abs(dy))) {
                    startScroll = true;
                }
                intercepted = super.onInterceptTouchEvent(e);
                Log.d(TAG, getClass().getSimpleName() +  "   ACTION_MOVE  : intercepted: " + intercepted + "   startScroll: " + startScroll);
                return startScroll && intercepted;
            }
            default:
                return super.onInterceptTouchEvent(e);
        }
    }
}

```

hbnfdghdfgsfhsfdg











































```java
printlin

```
