---
title: View的绘制流程源码分析
date: 2021-01-31
categories: Android
tags: 
- Android
- 源码分析
---

### 1. Activity启动后从哪里开始开启View的绘制

在ActivityThread中，当Activity被创建完成后，然后通过ASM会调用到`ActivityThread#handleResumeActivity`方法，在这个方法内，会通过WindowManager的addView方法将DecorView添加到window中，具体实现在WindowManagerImpl中

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

这里的`mGlobal`是`WindowManagerGlobal`单例对象，在`WindowManagerGlobal`的addView方法中，主要是创建了`ViewRootImpl`,并将`ViewRootImpl`和`DecorView`建立关联。

```java
root = new ViewRootImpl(view.getContext(), display);
try {
    root.setView(view, wparams, panelParentView);
} catch (RuntimeException e) {
    // ...
    throw e;
}
```

<!--more-->

`ViewRootImpl`可以看做是WindowManager和DecorView的纽带，WindowManager对View的控制就是通过ViewRootImpl来实现的，View的measure、layout、draw流程就是通过ViewRootImpl来完成的。

具体流程就是从`ViewRootImpl`的`setView`方法开始，调用到`requestLayout()`方法，再到`scheduleTraversals()`中

```java
final class TraversalRunnable implements Runnable {
  	@Override
  	public void run() {
    	doTraversal();
  	}
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 1-1 mTraversalRunnable是一个Runnable，通过postCallback触发
        mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
        		scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }
				// 1-2 开始绘制View的地方
        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

然后在`scheduleTraversals()`中 1-1 代码处通过`Choreographer`的postCallback触发`TraversalRunnable`，接着执行`doTraversal()`，在`doTraversal()`中，会执行到 1-2 的`performTraversals()`，这里就是View开始执行绘制过程的地方。

### 2. performTraversals流程

```java
private void performTraversals() {
    // ... 
    
    if (mFirst || windowShouldResize || insetsChanged ||
            viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
        // ...
        if (!mStopped || mReportNextDraw) {
            boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                    (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
            if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                    || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                    updatedConfiguration) {
                // 2-1 开始measure流程
                // Ask host how big it wants to be
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                
                if (measureAgain) {
                    if (DEBUG_LAYOUT) Log.v(mTag,
                            "And hey let's measure once more: width=" + width
                            + " height=" + height);
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                }
                layoutRequested = true;
            }
        }
    } else {
        maybeHandleWindowMove(frame);
    }

    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    boolean triggerGlobalLayoutListener = didLayout
            || mAttachInfo.mRecomputeGlobalAttributes;
    if (didLayout) {
        // 2-2 开始layout流程
        performLayout(lp, mWidth, mHeight);
        // ...
    }

    // ... 
    if (!cancelDraw && !newSurface) {
        if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).startChangingAnimations();
            }
            mPendingTransitions.clear();
        }
        // 2-3 开始draw流程
        performDraw();
    } else {
        if (isViewVisible) {
            // Try again
            scheduleTraversals();
        } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).endChangingAnimations();
            }
            mPendingTransitions.clear();
        }
    }

    mIsInTraversal = false;
}
```

`performTraversals()`方法比较长，我们简化一下，可以看到，在2-1，2-2，2-3处分别是`performMeasure`、`performLayout`、`performDraw`，也就是对应着View绘制的三大流程：测量、布局、绘制。在`performMeasure`会调用`view.measure(int widthMeasureSpec, int heightMeasureSpec)`方法来触发view的测量， 然后在`performLayout`中会调用view.layout(int l, int t, int r, int b)来布局，`performDraw`的传递过程是在draw方法中通过dispatchDraw来实现的。

我们接着到View中一步一步看measure、layout、draw流程。

### 3.  测量过程 View#measure -> onMeasure

#### 3.1 MeasureSpec测量模式

View的measure方法接收的是两个32位int类型的参数，这个32位int类型的参数包含两部分，一部分是高2位，代表着测量模式。另一部分则是低30位，存储着size信息，这个就是View的MeasureSpec信息。

对于顶级View（DecorView）来说，它的MeasureSpec是由窗口的尺寸和自身的LayoutParams共同确定的。具体逻辑如下代码所示

```java
// ViewRootImpl中的measureHierarchy方法内 确定DecorView的MeasureSpec
childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

// 获取DecorView的MeasureSpec的具体实现
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

所以，对于DecorView来说，它的MeasureSpec规则就是通过它自身LayoutParams来划分：

> 1.  LayoutParams.MATCH_PARENT : MeasureSpec.EXACTLY，大小就是窗口的size
> 2. LayoutParams.WRAP_CONTENT :  MeasureSpec.AT_MOST，大小不确定，但最大不能超过窗口size
> 3. 固定大小（如200dp）: MeasureSpec.EXACTLY，大小为LayoutParams指定的值

#### 3.2 MeasureSpec和LayoutParams的配合过程

对于普通View来说，MeasureSpec是ViewGroup从`measureChildWithMargins`方法中传递下来的，具体生成的规则我们可以看`getChildMeasureSpec`方法

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
  	// 通过spec获取到测量模式和size
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
		// 需要减去父布局的padding
    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // 父布局规定了固定的尺寸
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
          	// 子控件LayoutParams设置了固定大小，那么对应的测量模式为EXACTLY，大小为子控件LayoutParams的大小
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
          	// 子控件LayoutParams为MATCH_PARENT，那么对应的测量模式为EXACTLY，大小为父控件传递下来的大小
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
          	// 子控件LayoutParams为WRAP_CONTENT，那么对应的测量模式为AT_MOST，大小不能超过父控件传递下来的大小
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // 父布局规定了最大尺寸
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
          	// 子控件LayoutParams设置了固定大小，那么对应的测量模式为EXACTLY，大小为子控件LayoutParams的大小
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
           	// 子控件LayoutParams为MATCH_PARENT，那么对应的测量模式为AT_MOST，大小为父控件传递下来的大小
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
          	// 子控件LayoutParams为WRAP_CONTENT，那么对应的测量模式为AT_MOST，大小不能超过父控件传递下来的大小
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // 父布局没规定尺寸，看子布局要展示多大
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
          	// 子控件LayoutParams设置了固定大小，那么对应的测量模式为EXACTLY，大小为子控件LayoutParams的大小
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
           	// 子控件LayoutParams为MATCH_PARENT，那么对应的测量模式为UNSPECIFIED，大小为0
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
           	// 子控件LayoutParams为WRAP_CONTENT，那么对应的测量模式为UNSPECIFIED，大小为0
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

从以上代码可以清晰的看到获取子控件测量模式的规则，子控件的测量模式需要父控件的测量模式、父控件的size以及子控件自身设置的LayoutParams共同来决定。

#### 3.3 View的具体测量过程

了解了测量模式以及其生成规则后，我们继续到View中看具体的测量过程，`measure()`方法是一个被final修饰的方法，不能被子类覆写，然后在`measure()`中调用了`onMeasure`方法，我们来看看`onMeasure`方法。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
  	// 3-1 设置View的大小
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

getDefaultSize和getSuggestedMinimumWidth方法具体就不细看，主要是通过MeasureSpec和View的一些其他限制规则(如设置了最小宽度之类的)获取到一个真正的size，然后通过setMeasuredDimension方法设置。

```java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;

        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
    }
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;

    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}
```

最终将值设置到 `mMeasuredWidth`、 `mMeasuredHeight`上，通过这一系列的测量过程，就最终确定下来了该View的大小。这也是我们通过`View#getMeasuredWidth()`和`View#getMeasuredHeight()`一定要在onMeasure之后才能获取到正确值的原因。

### 4.  布局流程 View#layout -> onLayout

layout过程相比measure过程来说是简单一些，主要作用就是确定子View的位置。

#### 4.1 通过setFrame来设定View四个顶点的位置

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
    // 4-1 实际都是调用setFrame对mLeft、mRight、mTop、mBottom进行赋值
    // 返回的结果代表view的位置是否发生过变化，来判定是否需要重新进行layout
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    // 4-2 需要重新layout的情况
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        // 4-3 调用onLayout对子View进行布局
        onLayout(changed, l, t, r, b);

        // ...
    }

    // ...
}
```

在4-1处，可以知道 setOpticalFrame 也是通过setFrame()确定当前View上下左右顶点的位置，然后与之前的位置对比，看是否需要重新layout

```java
private boolean setOpticalFrame(int left, int top, int right, int bottom) {
    Insets parentInsets = mParent instanceof View ?
            ((View) mParent).getOpticalInsets() : Insets.NONE;
    Insets childInsets = getOpticalInsets();
    return setFrame(
            left   + parentInsets.left - childInsets.left,
            top    + parentInsets.top  - childInsets.top,
            right  + parentInsets.left + childInsets.right,
            bottom + parentInsets.top  + childInsets.bottom);
}
```

```java
protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;
				// ... 位置发送变化，重新设置顶点位置，确定当前View需要摆放的位置
    }
    return changed;
}
```

在4-2处，判断当前View的位置发生了变化，当当前View的位置发生了变化，就会在4-3调用到onLayout方法中去，重新摆放子view。

#### 4.2 调用onLayout方法，旨在确定子元素的位置

```java
// View中的 onLayout
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}

// ViewGroup 中的 onLayout
@Override
protected abstract void onLayout(boolean changed, int l, int t, int r, int b);
```

#### 4.3 onLayout方法由具体子类去实现，达到不同layout不同的功效

可以看到View中的onLayout是个空方法，而ViewGroup中的onLayout方法是被abstract修饰的。在View和ViewGroup中，onLayout方法没做任何处理，而是交由子类去处理，而ViewGroup的实现类必须实现onLayout方法，决定子View摆放的规则。

这里可以转到LinearLayout中去查看onLayout的具体实现，此文不做详细介绍了。

### 5.  绘制过程 View#draw -> onDraw

绘制过程的代码注释写的清楚明了，从注释来看，整个绘制过程被分为了6步，并且说明了第二步和第五步可以跳过，那我们对其余的四个步骤进行查看。

```java
public void draw(Canvas canvas) {
    // ...

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */

    // Step 1, draw the background, if needed
  	// 5-1 绘制背景
    int saveCount;
    if (!dirtyOpaque) {
        drawBackground(canvas);
    }

    // skip step 2 & 5 if possible (common case)
    // ...

    // Step 2, save the canvas' layers
    // ...
    saveCount = canvas.getSaveCount();
    int solidColor = getSolidColor();
    if (solidColor == 0) {
        if (drawTop) {
            canvas.saveUnclippedLayer(left, top, right, top + length);
        }

        if (drawBottom) {
            canvas.saveUnclippedLayer(left, bottom - length, right, bottom);
        }

        if (drawLeft) {
            canvas.saveUnclippedLayer(left, top, left + length, bottom);
        }

        if (drawRight) {
            canvas.saveUnclippedLayer(right - length, top, right, bottom);
        }
    } else {
        scrollabilityCache.setFadeColor(solidColor);
    }

    // Step 3, draw the content
  	// 5-2 绘制自己本身内容
    if (!dirtyOpaque) onDraw(canvas);

    // Step 4, draw the children
  	// 5-3 绘制子View
    dispatchDraw(canvas);

    // Step 5, draw the fade effect and restore layers
    final Paint p = scrollabilityCache.paint;
    final Matrix matrix = scrollabilityCache.matrix;
    final Shader fade = scrollabilityCache.shader;

    if (drawTop) {
        matrix.setScale(1, fadeHeight * topFadeStrength);
        matrix.postTranslate(left, top);
        fade.setLocalMatrix(matrix);
        p.setShader(fade);
        canvas.drawRect(left, top, right, top + length, p);
    }
    // ... 

    // Step 6, draw decorations (foreground, scrollbars)
  	// 5-4 绘制装饰
    onDrawForeground(canvas);

}
```

#### 5.1  绘制背景 background draw(canvas)

```java
private void drawBackground(Canvas canvas) {
    // 获取到设置的背景 （android:background、setBackgroundColor、setBackgroundResource）
    final Drawable background = mBackground;
    if (background == null) {
        return;
    }

    // 设置绘制背景区域
    setBackgroundBounds();

    // ...

    // 通过Drawable的draw()方法来完成绘制
    final int scrollX = mScrollX;
    final int scrollY = mScrollY;
    if ((scrollX | scrollY) == 0) {
        background.draw(canvas);
    } else {
        canvas.translate(scrollX, scrollY);
        background.draw(canvas);
        canvas.translate(-scrollX, -scrollY);
    }
}
```

#### 5.2 绘制自己 onDraw

```java
/**
 * Implement this to do your drawing.
 *
 * @param canvas the canvas on which the background will be drawn
 */
protected void onDraw(Canvas canvas) {
}
```

onDraw是个空方法，ViewGroup也没有覆写该方法，这是因为每个View绘制自己的内容都各不相同，具体的绘制内容就交由具体子类去实现。

#### 5.3 绘制children dispatchDraw

```java
// View # dispatchDraw
/**
 * Called by draw to draw the child views. This may be overridden
 * by derived classes to gain control just before its children are drawn
 * (but after its own view has been drawn).
 * @param canvas the canvas on which to draw the view
 */
protected void dispatchDraw(Canvas canvas) {

}

// ViewGroup # dispatchDraw
protected void dispatchDraw(Canvas canvas) {
    boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
    final int childrenCount = mChildrenCount;
    final View[] children = mChildren;
    // ...

    for (int i = 0; i < childrenCount; i++) {
        while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
            final View transientChild = mTransientViews.get(transientIndex);
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                    transientChild.getAnimation() != null) {
                // 分发到每个子View去绘制
                more |= drawChild(canvas, transientChild, drawingTime);
            }
            transientIndex++;
            if (transientIndex >= transientCount) {
                transientIndex = -1;
            }
        }
        // ...
    }
    // ... 

    // Draw any disappearing views that have animations
    if (mDisappearingChildren != null) {
        final ArrayList<View> disappearingChildren = mDisappearingChildren;
        final int disappearingCount = disappearingChildren.size() - 1;
        // Go backwards -- we may delete as animations finish
        for (int i = disappearingCount; i >= 0; i--) {
            final View child = disappearingChildren.get(i);
            // 分发到每个子View去绘制
            more |= drawChild(canvas, child, drawingTime);
        }
    }
    // ...
}
```

可以看到，dispatchDraw在View中是一个空方法，即没有子View了，就不需要往下分发draw事件了。而对应的在ViewGroup中，是会遍历所有子View，通过drawChild方法调用到子View的draw方法，完成子View的绘制。

```java
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    return child.draw(canvas, this, drawingTime);
}
```

#### 5.4 绘制装饰 onDrawScrollBars

```java
public void onDrawForeground(Canvas canvas) {
    onDrawScrollIndicators(canvas);
    onDrawScrollBars(canvas);

    final Drawable foreground = mForegroundInfo != null ? mForegroundInfo.mDrawable : null;
    if (foreground != null) {
        if (mForegroundInfo.mBoundsChanged) {
            mForegroundInfo.mBoundsChanged = false;
            final Rect selfBounds = mForegroundInfo.mSelfBounds;
            final Rect overlayBounds = mForegroundInfo.mOverlayBounds;

            if (mForegroundInfo.mInsidePadding) {
                selfBounds.set(0, 0, getWidth(), getHeight());
            } else {
                selfBounds.set(getPaddingLeft(), getPaddingTop(),
                        getWidth() - getPaddingRight(), getHeight() - getPaddingBottom());
            }

            final int ld = getLayoutDirection();
            Gravity.apply(mForegroundInfo.mGravity, foreground.getIntrinsicWidth(),
                    foreground.getIntrinsicHeight(), selfBounds, overlayBounds, ld);
            foreground.setBounds(overlayBounds);
        }
        foreground.draw(canvas);
    }
}
```

最后一步，就是对View的一些装饰进行绘制，如滚动条之类的，这里就不继续深入分析了。

### 总结

- View的绘制过程主要包含measure、layout、draw三个具体步骤
- 测量过程是MeasureSpec和LayoutParams配合工作，测量出子View和父View所需空间的大小
- View的layout就是用于确定自身摆放的位置，而ViewGroup要根据其作用和特性确定子View如何进行摆放
- View的draw是绘制自身的过程，ViewGroup会遍历子View，通知子View进行draw