---
title: RecyclerView的ViewHolder复用机制总结
date: 2017-10-18 14:14:14
categories: Android
keywords: RecyclerView, 复用机制, ViewHolder
description: RecyclerView的ViewHolder复用机制总结
tags: [Android, RecyclerView, ViewHolder]
---
### 前文

在上一篇出现的问题总结起来就是：**复用问题**，其实布局没那么嵌套复杂，也不会出现，主要是嵌套的`GridView`导致这么多问题。如果是单纯的ImageView什么的，点击事件或者显示混乱什么的都很好解决。

### 正文

所以，写个总结来总结下`RecyclerView`的`ViewHolder`复用机制吧。先来个表格，如下：


|         |回调onCreateViewHolder|回调onBindViewHolder|生命周期|备注|
| :-------| :--------| :-------| :--------|:-------|
|mAttachedScrap|否|否|onLayout函数周期内|屏幕内ItemView重用，具体指执行remove的ItemView(匹配postion)|
|mCachedViews|否|否|与mAdapter一致，否则存入mRecyclePool|默认缓存屏幕外2个(匹配postion)|
|mViewCacheExtension||||不直接使用，由用户实现|
|mRecyclerPool|否|是|与自身生命周期一致，不被引用时释放|默认缓存5个，可以RecyclerView之间共享|

对应表格的简略版代码

```
final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
        ArrayList<ViewHolder> mChangedScrap = null;

final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
private int mRequestedCacheMax = DEFAULT_CACHE_SIZE;
int mViewCacheMax = DEFAULT_CACHE_SIZE;

RecycledViewPool mRecyclerPool;

private ViewCacheExtension mViewCacheExtension;
```

由表中可以看到`RecyclerView`其实是四级缓存：

* 1. `mAttachedScrap`用于复用屏幕内的`itemView`，源码中具体存储的其实是正在执行`remove`操作的`ItemView`。（一级缓存）
* 2. `mCachedViews` 用于缓存屏幕外的`itemView`，默认缓存两个，当滑回来的时候，并不会执行`onBindViewHolder`，更别提`onCreateViewHolder`了。（二级缓存）
* 3. `mViewCacheExtension` 需要我们自己实现，默认是没有的，（三级缓存）
* 4. `mRecyclerPool` 四级缓存，默认缓存五个，可以由多个RecyclerView共享，其实`mRecyclerPool`的内部维护了一个`Map`，里面以不同的`viewType`为Key存储了各自对应的`ViewHolder`集合。可以通过提供的方法来修改内部缓存的`Viewholder`。可以根据`viewType`得到对应的`itemView`，如下代码，

```
public ViewHolder getRecycledView(int viewType) {
            final ScrapData scrapData = mScrap.get(viewType);
            if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
                final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
                return scrapHeap.remove(scrapHeap.size() - 1);
            }
            return null;
        }
```

也可以为 某个特定的`viewType`设置缓存数量,方法`setMaxRecycledViews`，即可,不缓存某种就设`max`为0

```
        public void setMaxRecycledViews(int viewType, int max) {
            ScrapData scrapData = getScrapDataForType(viewType);
            scrapData.mMaxScrap = max;
            final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
            if (scrapHeap != null) {
                while (scrapHeap.size() > max) {
                    scrapHeap.remove(scrapHeap.size() - 1);
                }
            }
        }
```



#### 怎么从缓存中读？

依次从这四级缓存中取，如果都没有的话，只能重走`onCreateViewHolder`和`onBindViewHolder`了。

#### 怎么写入缓存？

`mAttachedScrap`用于复用屏幕内的itemView，源码中具体存储的其实是正在执行remove操作的ItemView。这个前文说过。

正常滚动的写入逻辑是：首先判断集合mCachedViews是否满了，

如果已满就从mCachedViews集合中移出一个到mRecyclerPool集合中去，再把新的ItemView添加到mCachedViews集合；

如果不满就将ItemView直接添加到mCachedViews集合。 

#### 缓存的是啥
**View + ViewHolder(避免每次createView时调用findViewById) + flag(标识状态)；**

RecyclerView中通过postion获取的是viewholder，即pos ——> (view，viewHolder，flag)；标志flag的作用是判断view是否需要重新bindView，这也是RecyclerView实现局部刷新的一个核心.

其他的也不多说了，源码啥的文中也没贴，太长。








