---
title: ViewPager和Fragment懒加载的一些问题
date: 2017-09-24 21:28:56
categories: Android
keywords: ViewPager, Fragment, 懒加载, LazyLoad
tags: [Android, ViewPager, Fragment, 懒加载]
---
#### 懒加载问题
项目中实现滑动页面切换大多用viewpager fragment的方式，为了优化，往往加载方式都是“懒加载”，怎么实现懒加载呢？都知道setUserVisibleHint 中采用如下标识，大体如
下面代码示例：

```
@Override
public void setUserVisibleHint(boolean isVisibleToUser) {
    super.setUserVisibleHint(isVisibleToUser);
    if (getUserVisibleHint()) {
        isVisible = true;
    } else {
        isVisible = false;
    }
    onVisible();
}
```

在`onVisible()` 里加载数据就完了？错
都知道  当fragment被用户可见时，`setUserVisibleHint()`会调用且传入true值，当fragment不被用户可见时，`setUserVisibleHint()`则传入false值。
那`setUserVisibleHint() `何时会被调用呢？
答案是：啥时候都可以。和fragment的生命周期无关。

具体问题具体分析：在 `viewpager` 的`FragmentPagerAdapter`中 ，这个函数在fragment生命周期前就调用多次了。
先来看`FragmentPagerAdapter`源码

```
@Override
public Object instantiateItem(ViewGroup container, int position) {
    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }

    final long itemId = getItemId(position);

    // Do we already have this fragment?
    String name = makeFragmentName(container.getId(), itemId);
    Fragment fragment = mFragmentManager.findFragmentByTag(name);
    if (fragment != null) {
        if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
        mCurTransaction.attach(fragment);
    } else {
        fragment = getItem(position);
        if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
        mCurTransaction.add(container.getId(), fragment,
                makeFragmentName(container.getId(), itemId));
    }
    if (fragment != mCurrentPrimaryItem) {
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
    }

    return fragment;
}
```
从上面代码能看出，在创建Fragment的时候会调用一次。

```
@Override
public void setPrimaryItem(ViewGroup container, int position, Object object) {
    Fragment fragment = (Fragment)object;
    if (fragment != mCurrentPrimaryItem) {
        if (mCurrentPrimaryItem != null) {
            mCurrentPrimaryItem.setMenuVisibility(false);
            mCurrentPrimaryItem.setUserVisibleHint(false);
        }
        if (fragment != null) {
            fragment.setMenuVisibility(true);
            fragment.setUserVisibleHint(true);
        }
        mCurrentPrimaryItem = fragment;
    }
}
```
在设定当前显示页面时候也会调用一次，设置为true。
如果是viewpager的第一个页面，那问题就来了，
** 在fragment还没走生命周期前，`setUserVisibleHint`已经设置为true了。** 
所以如果在本文刚开始的代码中 实现懒加载有如下缺陷：** 如果有参数传入fragment 恰巧这个参数是作为请求数据的参数，那么第一个页面第一次加载数据时是取不到这个参数的。**
解决方法就是：来个标志位,标识生命周期走到哪了。类似于下面的代码：

```
@Override
public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
    View view = inflater.inflate(R.layout.fragment_course_tab, null);
hasCreatView  =true;
if (isVisible && hasCreatView && !isLoaded) {
    requestData();
}
    return view;
}

@Override
public void setUserVisibleHint(boolean isVisibleToUser) {
    super.setUserVisibleHint(isVisibleToUser);
    if (getUserVisibleHint()) {
        isVisible = true;
    } else {
        isVisible = false;
    }
    onVisible();
}
```
这样就可以解决接收不到参数就加载数据的问题。

#### 用哪种Adapter？

那说到`FragmentPagerAdapter` 再来看看里面的源码，我们发现：
`instantiateItem`  `destroyItem`这两个方法
分别调用的是`mCurTransaction.attach(fragment);`
`mCurTransaction.detach((Fragment)object);`
只是attach、detach 并没有remove, 也就是说`FragmentPagerAdapter`里的fragment 不会被销毁，仅仅只是销毁视图，fragment是常驻内存的。那么由此引申：生命周期走到`ondestroryview` 最多咯，所以千万不要在`onDestroy` 里保存啥东西哟，因为走不到哟。
与之对比的是：`FragmentStatePagerAdapter`
用的是add 和remove 那么由此，我们可以得到：在子fragment数量少的时候，`FragmentPagerAdapter`可以胜任，但是多的情况 还是`FragmentStatePagerAdapter`可当大用，
这两者另外一个区别是：
fragment状态的存储，从下面的源码可以看出后者多了状态的保存与销毁。

```
@Override
public Parcelable saveState() {
    Bundle state = null;
    if (mSavedState.size() > 0) {
        state = new Bundle();
        Fragment.SavedState[] fss = new Fragment.SavedState[mSavedState.size()];
        mSavedState.toArray(fss);
        state.putParcelableArray("states", fss);
    }
    for (int i=0; i<mFragments.size(); i++) {
        Fragment f = mFragments.get(i);
        if (f != null && f.isAdded()) {
            if (state == null) {
                state = new Bundle();
            }
            String key = "f" + i;
            mFragmentManager.putFragment(state, key, f);
        }
    }
    return state;
}

@Override
public void restoreState(Parcelable state, ClassLoader loader) {
    if (state != null) {
        Bundle bundle = (Bundle)state;
        bundle.setClassLoader(loader);
        Parcelable[] fss = bundle.getParcelableArray("states");
        mSavedState.clear();
        mFragments.clear();
        if (fss != null) {
            for (int i=0; i<fss.length; i++) {
                mSavedState.add((Fragment.SavedState)fss[i]);
            }
        }
        Iterable<String> keys = bundle.keySet();
        for (String key: keys) {
            if (key.startsWith("f")) {
                int index = Integer.parseInt(key.substring(1));
                Fragment f = mFragmentManager.getFragment(bundle, key);
                if (f != null) {
                    while (mFragments.size() <= index) {
                        mFragments.add(null);
                    }
                    f.setMenuVisibility(false);
                    mFragments.set(index, f);
                } else {
                    Log.w(TAG, "Bad fragment at key " + key);
                }
            }
        }
    }
}
```
其实这两个方法就是讲`mSavedState`和mFragments给保存起来，用于app在意外退出的时候恢复数据的

`saveState`会在程序意外退出时调用，`restoreState`会在程序打开的时候调用，将在`saveState`中持久化的数据通过 Parcelable 对象给还原回来。有点和Activity的`onSaveInstanceState`原理很像。

如果我们打开`ViewPager`的源码会发现下面两个方法会在`onSaveInstanceState()`和`onRestoreInstanceState()`方法中调用。这些方法都是重写View的，说明每一个View都是能力在程序意外退出的时候恢复自己数据.

总结下： ** 懒加载接受参数要注意结合判断生命周期。FragmentPagerAdapter适合少的情况，常驻内存，生命周期能走到哪要注意，不会自己保存view的状态。
FragmentStatePagerAdapter 适合多的情况，fragment会销毁，但是相应的 页面切换的成本相对大了点，会自己保存状态。** 

