---
layout: post
title: "Android Fragment使用小结及介绍"
date: 11/10/2017 3:36:58 PM 
comments: true
tags: 
	- 技术 
	- Android
	- Fragment
---
---
偶记得第一次接触Fragment，觉得好牛叉的组件，可以做许多Activity可以做的事，辅助Activity让功能可以做得更加强大;一次编写，可以多个地方可以使用，解放了Activity。在这里，本篇文章主要是总结fragment的两种添加方式，add和replace。
### 一、什么是Fragment
简单来说，Fragment其实可以理解为一个具有自己生命周期的控件，只不过这个控件又有点特殊，它有自己的处理输入事件的能力，有自己的生命周期，又必须依赖于Activity，能互相通信和托管。

使用Fragment还有这么几个方面优势：

- 代码复用。特别适用于模块化的开发，因为一个Fragment可以被多个Activity嵌套，有个共同的业务模块就可以复用了，是模块化UI的良好组件。
- Activity用来管理Fragment。Fragment的生命周期是寄托到Activity中，Fragment可以被Attach添加和Detach释放。
- 可控性。Fragment可以像普通对象那样自由的创建和控制，传递参数更加容易和方便，也不用处理系统相关的事情，显示方式、替换、不管是整体还是部分，都可以做到相应的更改。
- Fragments是view controllers，它们包含可测试的，解耦的业务逻辑块，由于Fragments是构建在views之上的，而views很容易实现动画效果，因此Fragments在屏幕切换时具有更好的控制。

### 二、Fragment的生命周期
Fragment的生命周期类似Activity,如下图，Activity生命周期与Fragment生命周期对比图：
<!-- more -->
![这里写图片描述](http://img.blog.csdn.net/20171109172319334?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQXdlbnlpbmk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


### 三、Fragment的两种添加方式(add&replace)
Fragment添加与FragmentManager与FragmentTransaction息息相关。add和replace都是FragmentTransaction的方法。除这两个方法，其中还有remove,hide和show方法。

FragmentManager与FragmentTransaction的获取：
```java
FragmentManager frgmentManager = getFragmentManager() // v4中，getSupportFragmentManager
FragmentTransaction transaction = frgmentManager.benginTransatcion();
```
**1.项目中多个Fragment，add方式添加**

i.添加代码
```java
    public void add(BaseLibFragment fragment, int id,String tag){
        FragmentManager fragmentManager = ((BaseLibActivity)mContext).getSupportFragmentManager();
        FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
        //优先检查，fragment是否存在，避免重叠
        BaseLibFragment tempFragment = (BaseLibFragment)fragmentManager.findFragmentByTag(tag);
        if(EmptyUtils.isNotEmpty(tempFragment)){
            fragment = tempFragment;
        }
        if(fragment.isAdded()){
            addOrShowFragment(fragmentTransaction,fragment,id,tag);
        }else{
            if(currentFragment!=null&&currentFragment.isAdded()){
                fragmentTransaction.hide(currentFragment).add(id, fragment,tag).commit();
            }else{
                fragmentTransaction.add(id, fragment,tag).commit();
            }
            currentFragment = fragment;
        }
    }
    /**
     * 添加或者显示 fragment
     *
     * @param fragment
     */
    private void addOrShowFragment(FragmentTransaction transaction, BaseLibFragment fragment, int id,String tag) {
        if(currentFragment == fragment)
            return;
        if (!fragment.isAdded()) { // 如果当前fragment未被添加，则添加到Fragment管理器中
            transaction.hide(currentFragment).add(id, fragment,tag).commit();
        } else {
            transaction.hide(currentFragment).show(fragment).commit();
        }
        currentFragment.setUserVisibleHint(false);
        currentFragment =  fragment;
        currentFragment.setUserVisibleHint(true);
    }
```
ii.添加顺序

- 第一次添加，先hide(隐藏)currentFragment，再add(添加)新Fragment。生命周期会按正常流程走，onCreate->onResume
- 第二次添加，先hide(隐藏)currentFragment，在show(显示)老Fragment。生命周期不会重新走，会调用onHiddenChanged()，展示fragment的显示状态，我们可以在此做一些刷新数据操作。

iii.add方式Fragment重叠BUG解决方案
为fragment设置Tag，通过findFragmentByTag查找是否存在，然后再添加
```java
//优先检查，fragment是否存在，避免重叠
        BaseLibFragment tempFragment = (BaseLibFragment)fragmentManager.findFragmentByTag(tag);
        if(EmptyUtils.isNotEmpty(tempFragment)){
            fragment = tempFragment;
        }
```

**2.项目中多个Fragment，replace方式添加**
i.添加代码
```java
    public void replace(BaseLibFragment fragment, int id){
        FragmentManager fragmentManager = ((BaseLibActivity)mContext).getSupportFragmentManager();
        FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
        fragmentTransaction.replace(id, fragment);
        fragmentTransaction.commit();
    }
```
ii.添加方式
添加方式比较直接，直接替换。在这过程中因为是替换，第一和第二次添加没啥区别，生命周期都要重新执行一次

### 四、两种添加方式性能比较
标准的四大金刚模式。底部四个Item，通过Fragment内容切换，此种方式add与replace性能对比，如下两图：

**add方式**

![这里写图片描述](http://img.blog.csdn.net/20171109193642486?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQXdlbnlpbmk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

**replace方式**
![这里写图片描述](http://img.blog.csdn.net/20171109193725890?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQXdlbnlpbmk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

由以上两图知：replace方式内存波动比较大，网络请求消耗大；add方式则反之。



