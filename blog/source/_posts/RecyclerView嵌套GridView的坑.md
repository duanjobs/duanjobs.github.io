---
title: RecyclerView嵌套GridView的坑
date: 2017-10-17 11:15:14
categories: Android
keywords: RecyclerView, 复用, GridView
description: RecyclerView嵌套GridView的坑
tags: [Android, RecyclerView, 嵌套GridView]
---
### 遇到的问题
在写一个功能时，不得已采用方案：`RecyclerView`嵌套`GridView`，于是乎，坑来了。

先来看看最原始版本的`ViewHolder`的代码:

```
public class ViewHolder extends RecyclerView.ViewHolder {

        public MeasureHeightGridView gridView;
        public LinearLayout linMore;
        public LinearLayout linContainer;
        public TextView tvSubclassName;
        public TextView tvSubclassIntro;
        public RelativeLayout relBottom;

        public ViewHolder(View itemView) {
            super(itemView);
            gridView = (MeasureHeightGridView) itemView.findViewById(R.id.gv_inside);
            linMore = (LinearLayout) itemView.findViewById(R.id.lin_more);
            linContainer = (LinearLayout) itemView.findViewById(R.id.lin_container);
            tvSubclassName = (TextView) itemView.findViewById(R.id.tv_subclass_name);
            tvSubclassIntro = (TextView) itemView.findViewById(R.id.tv_subclass_intro);
            relBottom = (RelativeLayout) itemView.findViewById(R.id.rel_bottom);
        }
    }
```

然后在 `onBindViewHolder`里给`girdView` `setAdapter`和` setOnItemClickListener()`

**好了，到这问题来了：**

* 1.  **`girdView`点击事件丢失，而且只在滑出屏幕大概两个item的时候，再滑回来出现。**
* 2.  **由于`RecyclerView`的item会有“展示更多”button的伸展操作，在伸展完后，复用时，item高度错误，出现大片空白。**

### 如何解决

看这种情况，明显是复用出了问题啊。

对于问题2. 临时解决方案：不复用这种类型的item（仅仅是临时，因为问题1的解决方案可以解决问题2）

**如何不复用`RecyclerView`的某种`ViewHolder`？一句代码如下**

```
RecyclerView.getRecycledViewPool().setMaxRecycledViews(VIEWHOLDERTYPE, 0);
```

对于问题1. 嗯，思考良久，决定把`Adapter`相关操作放到ViewHolder里，并自定义click事件

```
 public class ViewHolder extends RecyclerView.ViewHolder {

        public MeasureHeightGridView gridView;
        public LinearLayout linMore;
        public LinearLayout linContainer;
        public TextView tvSubclassName;
        public TextView tvSubclassIntro;
        public RelativeLayout relBottom;

        public CourseSubAdapter courseSubAdapter;


        public ViewHolder(View itemView) {
            super(itemView);
            gridView = (MeasureHeightGridView) itemView.findViewById(R.id.gv_inside);
            linMore = (LinearLayout) itemView.findViewById(R.id.lin_more);
            linContainer = (LinearLayout) itemView.findViewById(R.id.lin_container);
            tvSubclassName = (TextView) itemView.findViewById(R.id.tv_subclass_name);
            tvSubclassIntro = (TextView) itemView.findViewById(R.id.tv_subclass_intro);
            relBottom = (RelativeLayout) itemView.findViewById(R.id.rel_bottom);
            courseSubAdapter = new CourseSubAdapter(context);
            gridView.setAdapter(courseSubAdapter);
        }
    }
```

对 `CourseSubAdapter` 的 `getView()`里给 `convertView`设置监听。问题可以解决。

到现在，问题2也可以解决。其实就是复用出了问题。

然后又出现了问题3

* 3. 发现 item 滑动后又混乱了

断点调试发现，`gridView`的`Adapter`也`Notify`了，数据都已更新，但是`gridView` UI并没有更新。只好`gridView.requesetLayout()` 解决。

**在`onViewRecycled`里清空`gridView`的`adapter`数据，并`requesetLayout()`可以，如果这样的话，在`onBindViewHolder`里可以直接notify；**

**也可以每次在`onBindViewHolder` `setData`后直接`requesetLayout()`**

RecyclerView复用机制决定下一篇记录一下~。





