---
title: Android中ListView、CheckBox状态混乱解决方式
date: 2017-09-24 15:11:28
categories: Android
tags: [Android, ListView, CheckBox]
---
### 出现的问题
在开发过程中，发现ListView嵌套CheckBox记录选中状态时，如果滚动列表，则会出现选中状态的混乱。这个问题是很常见的问题了，ListView中如果有图片也会出现类似的情况。主要原因是：ListView的缓存机制：

- ListView会缓存行item。ListView通过adapter的getView函数获得每行的item；
- 滑动过程中，如果某行item已经滑出屏幕，若该item不在缓存内，则放进缓存，若存在，则更新缓存中的数据；
- 获取滑入屏幕的行item之前会先判断缓存中是否有可用的item，如果有，则做为convertView参数传递给adapter的getView。
- 所以说，每个item不同的显示只是改变这个对象的显示值而已，item占用的资源不变。

### 解决方式
1. 不用convertView或者viewHolder。//个人建议不要这样做，数据量大的时候，开销太大了。
2. 用某数据结构记录选中状态。//本文
3. 如果是图片，可以使用setTag（Url），给其一个标识，然后判断加载的url和标识是否一致，此文不再赘述。

### 部分代码
使用HashMap记录选中状态，
`private HashMap<Integer,Boolean> checkRem = new HashMap<Integer,Boolean>();`

记得在Adapter里初始化
```
public void init(){
    for(int i=0;i<mList.size();i++){
        checkRem.put(i,false);
    }
}
```
在getView里设置监听，更新记录状态（此处是为checkBox设置，如果item只有一个checkBox点击事件，可以在OnItemClickListener()中设置监听）。checkBox的初始化，即setChecked要放在监听的后面。

```
box.setOnCheckedChangeListener(new CompoundButton.OnCheckedChangeListener() {
            @Override
            public void onCheckedChanged(CompoundButton compoundButton, boolean b) {
                if(b){
                    checkRem.put(position,true);
                }else{
                    checkRem.put(position,false);
                    }
            }
        });
// 这句要写在监听后面
box.setChecked(checkRem.get(position));
```
