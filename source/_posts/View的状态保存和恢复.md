---
title: View的状态保存和恢复
date: 2017-01-22 22:05:43
tags: Android
categories: Android
---
>在写自定义控件的时候发现一个问题，在控件中设置了变量来控制控件的状态，但是在屏幕旋转的时候，activity经历了销毁重启，此时自定义控件中的状态也随之重置了。
>
>为了保证在屏幕旋转的时候，控件的状态保持不变，可以通过重写**View#onSaveInstanceState**和**View#onRestoreInstanceState**来保证自定义控件的状态在重启的时候得到保留。
>
>通过这个小问题的解决，对View的保存和恢复状态有了一个了解。本篇文章只针对于View的保存和恢复。

<!--more-->
## 一、保存状态流程
###  Activity#onSaveInstaceState
```java
protected void onSaveInstanceState(Bundle outState) {
	//将view树的状态保存在bundle中
	outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
	Parcelable p = mFragments.saveAllState();
		if (p != null) {
		outState.putParcelable(FRAGMENTS_TAG, p);
	}
	getApplication().dispatchActivitySaveInstanceState(this, outState);
}
```
通过WINDOW\_HIERARCHY\_TAG这个key来保存view树的数据，这里Window的实现类是PhoneWindow.

### PhoneWindow#saveHierarchyState
```java
@Override
public Bundle saveHierarchyState() {
	Bundle outState = new Bundle();
	if (mContentParent == null) {
		return outState;
	}
		
	SparseArray<Parcelable> states = new SparseArray<Parcelable>();
	mContentParent.saveHierarchyState(states);
	outState.putSparseParcelableArray(VIEWS_TAG, states);
	// save the focused view id
	View focusedView = mContentParent.findFocus();
	if (focusedView != null) {
		if (focusedView.getId() != View.NO_ID) {
			outState.putInt(FOCUSED_ID_TAG, focusedView.getId());
		} else {
			if (false) {
				Log.d(TAG, "couldn't save which view has focus because the focused view "
						+ focusedView + " has no id.");
			}
		}
	}
	// save the panels
	SparseArray<Parcelable> panelStates = new SparseArray<Parcelable>();
	savePanelState(panelStates);
	if (panelStates.size() > 0) {
		outState.putSparseParcelableArray(PANELS_TAG, panelStates);
	}
	if (mDecorContentParent != null) {
		SparseArray<Parcelable> actionBarStates = new SparseArray<Parcelable>();
		mDecorContentParent.saveToolbarHierarchyState(actionBarStates);
		outState.putSparseParcelableArray(ACTION_BAR_TAG, actionBarStates);
	}
	return outState;
}
```
这里的SparseArray类似HashMap，接着调用了mContentParent的saveHierarchyState()方法，并把结果放进outState中并返回。这里的mContentParent是DecorView的子元素或者其自身，这里可以把mContentParent看做整个View树的顶层视图，由于mContentParent是一个ViewGroup，但是ViewGroup没有重写saveHierarchyState方法,那么这里调用的便是

### ViewGroup#saveHierarchyState:
```java
public void saveHierarchyState(SparseArray<Parcelable> container) {
	dispatchSaveInstanceState(container);
}
```
分发到 各个子view，去保存各个子view的数据
### View#saveHierarchyState:
```java
protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
	if (mID != NO_ID && (mViewFlags & SAVE_DISABLED_MASK) == 0) {
		mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
		Parcelable state = onSaveInstanceState();
		if ((mPrivateFlags & PFLAG_SAVE_STATE_CALLED) == 0) {
			throw new IllegalStateException(
					"Derived class did not call      super.onSaveInstanceState()");
		}
		if (state != null) {
			// Log.i("View", "Freezing #" + Integer.toHexString(mID)
			// + ": " + state);
			container.put(mID, state);
		}
	}
}
```

这里首先判断当前View是不是有一个ID以及saveEnable属性，接着下面便调用到了View#onSaveInstanceState()方法，也即我们的自定义View需要重写的方法，这个方法返回Parcelable对象，即可序列化对象，最后把该Parcelable对象放进了SparseArray内，key是该View的id。


>由此可知，如果一个View需要保存状态，那么至少需要以下两个条件：
>
1.  必须给view设置一个id
2.  view的saveEnable属性必须是true

## 二、恢复状态流程
### Activity#onRestoreInstanceState
```java
protected void onRestoreInstanceState(Bundle savedInstanceState) {
	if (mWindow != null) {
		Bundle windowState = savedInstanceState.getBundle(WINDOW_HIERARCHY_TAG);
		if (windowState != null) {
			mWindow.restoreHierarchyState(windowState);
		}
	}
}
```
先是根据WINDOW\_HIERARCHY\_TAG这个key获取Bundle对应的数据，即View树数据，接着调用mWindow.restoreHierarchyState方法，我们继续看

### PhoneWindow#restoreHierarchyState：
```java
@Override
	public void restoreHierarchyState(Bundle savedInstanceState) {
	SparseArray<Parcelable> savedStates
			= savedInstanceState.getSparseParcelableArray(VIEWS_TAG);
	if (savedStates != null) {
		mContentParent.restoreHierarchyState(savedStates);
	}
	//省略...
}
```
从Bundle里面根据VIEWS\_TAG来获取SparseArray，这个之前我们也说过了，这就是View树数据所对应的SparseArray，接着调用mContentParent.restoreHierarchyState，到这里我们也知道接下来应该是调用**View#restoreHierarchyState**方法，而就如保存状态一样，恢复状态也需要把事件分发给ViewGroup的所有子View，所以在restoreHierarchyState方法里面又调用到了 

### ViewGroup#dispatchRestoreInstanceState:
```java
@Override
protected void dispatchRestoreInstanceState(SparseArray<Parcelable> container) {
	//调用View的dispatchRestoreInstanceState，目的是恢复ViewGroup自身的状态
	super.dispatchRestoreInstanceState(container);
	final int count = mChildrenCount;
	final View[] children = mChildren;
	//遍历所有子View，逐个恢复它们的状态
	for (int i = 0; i < count; i++) {
		View c = children[i];
		if ((c.mViewFlags & PARENT_SAVE_DISABLED_MASK) != PARENT_SAVE_DISABLED) {
			c.dispatchRestoreInstanceState(container);
		}
	}
}
```
可以看出，代码和dispatchOnSaveInstanceState方法基本类似，接着我们看 
  
### View#dispatchRestoreInstanceState:
```java
protected void dispatchRestoreInstanceState(SparseArray<Parcelable> container) {
	if (mID != NO_ID) {
		Parcelable state = container.get(mID);
		if (state != null) {
			mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
			onRestoreInstanceState(state);
		}
	}
}	
```
这里根据View的Id在SparseArray中获得对应的Parcelable对象，即视图数据，接着调用了**View#onRestoreInstanceState(Parcelable)**方法，交给每一个View来自行恢复数据，至此，View树的数据恢复解析完毕。

>一般对于自定义View来说，我们会重写**onSaveInstanceState()**和**onRestoreInstanceState(Parcelable)**方法，来处理我们需要恢复的数据。

## 参考文章：
>1. [Android 视图树&View状态保存](http://www.jianshu.com/p/4c1a6d382a85)
>2. [Android View状态保存 ](http://blog.csdn.net/hp910315/article/details/51890813)
>3. [Android状态保存与恢复流程 完全解析](http://www.jianshu.com/p/58579627f70a)