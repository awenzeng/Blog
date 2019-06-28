---
layout: post
title: "Activity布局加载流程源码分析(III)"
date: 6/28/2019 10:09:39 AM  
comments: true
tags: 
	- 技术 
	- Android
	- Android框架源码解析
---
---
在[Activity布局加载流程源码分析(I)](http://blog.csdn.net/awenyini/article/details/78934390)、[Activity布局加载流程源码分析(II)](https://blog.csdn.net/awenyini/article/details/78964353)、[DecorView绘制流程源码分析](https://blog.csdn.net/awenyini/article/details/78983463)与[View绘制三大流程源码分析](https://blog.csdn.net/awenyini/article/details/79006432)等四篇文章中，已经很详细分析了Acitivity的布局加载过程及布局的绘制过程。但在[Activity布局加载流程源码分析(I)](http://blog.csdn.net/awenyini/article/details/78934390) 中，对于setContentView("资源文件")怎么转化View的，没有细说，本篇博文主要想梳理一下这块内容（**ps:面试的时候，被面试官问到，既然答不上来，所以决定对这部分知识也好好梳理一下**）。

在开始分析之前，我们需要了解一些概念，如：

- **PhoneWindow：** 是Window类具体实现类，Activity中布局加载逻辑主要就是在此类中完成的。
- **LayoutInflater：** 是布局填充类，主要就是将我们的layout转化为View。
- **XmlPullParser：** 是XML解析器，主要是解析xml文件也即layout.xml文件。

# 一、源码分析
从[Activity布局加载流程源码分析(I)](http://blog.csdn.net/awenyini/article/details/78934390)文中，我们知道，在Activity的onCreate()中setContentView()后，最后也是调用PhoneWindow中的setContentView()方法。源码如下：
~~~java
   //Activity中
  public void setContentView(int layoutResID) {
        getWindow().setContentView(layoutResID);//核心代码
        initActionBar();
    }
~~~
~~~java
    //PhoneWindow中
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);//核心代码
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
~~~
这里我们主要来看核心代码，也即 mLayoutInflater.inflate(layoutResID, mContentParent)；其中mLayoutInflater的初始化主要是在PhoneWindow中构造方法中初始化的：
~~~java
  public PhoneWindow(Context context) {
        super(context);
        mLayoutInflater = LayoutInflater.from(context);
    }
~~~
所以，重点我们还是来看一下LayoutInflater类：
~~~java
public abstract class LayoutInflater {
     .......
    /**
     * Obtains the LayoutInflater from the given context.
     */
    public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }
    .......
  //填充方法1
  public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }

    /**
     * 填充方法2
     * Inflate a new view hierarchy from the specified xml node. Throws
     * {@link InflateException} if there is an error. *
     */
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root) {
        return inflate(parser, root, root != null);
    }

    /**
     * 填充方法3
     * Inflate a new view hierarchy from the specified xml resource. Throws
     * {@link InflateException} if there is an error.
     */
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        .......
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }

    /**
     * 填充方法4
     * Inflate a new view hierarchy from the specified XML node. Throws
     * {@link InflateException} if there is an error.
     * <p>
     */
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                .........
                final String name = parser.getName();
                ..........
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;
                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " + root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }
            ........
            return result;
        }
    }
    ..........
}
~~~
从上面代码我们知道，mLayoutInflater 的初始化，主要是从context.getSystemService(Context.LAYOUT_INFLATER_SERVICE)中获取的。在LayoutInflater的源码中，还有多个inflate方法中，这里我们先来看看填充方法2：
~~~java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        .......
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
~~~
由代码易知，这里主要是从资源Resources中获取布局xml，通过res.getLayout(resource)得到一个Xml资源解析器XmlResourceParser，然后再调用填充方法4，这里的XmlResourceParser，我们也可以来看一下源码：
~~~java
public interface XmlResourceParser extends XmlPullParser, AttributeSet, AutoCloseable {
    /**
     * Close this interface to the resource.  Calls on the interface are no
     * longer value after this call.
     */
    public void close();
}
~~~
由上易知，XmlResourceParser也是一个接口，这里主要继承了三个类XmlPullParser（XML文件解析器）、AttributeSet（自定义属性值）、AutoCloseable ，主要是了后面解析服务。所以这里我们重点来看看Resource中的getLayout()方法：
~~~java
public class Resources {
    ........
    public XmlResourceParser getLayout(@LayoutRes int id) throws NotFoundException {
        return loadXmlResourceParser(id, "layout");
    }
    ........
    
    /*package*/ XmlResourceParser loadXmlResourceParser(int id, String type)
            throws NotFoundException {
        synchronized (mAccessLock) {
            TypedValue value = mTmpValue;
            if (value == null) {
                mTmpValue = value = new TypedValue();
            }
            getValue(id, value, true);
            if (value.type == TypedValue.TYPE_STRING) {
                return loadXmlResourceParser(value.string.toString(), id,
                        value.assetCookie, type);
            }
            throw new NotFoundException(
                    "Resource ID #0x" + Integer.toHexString(id) + " type #0x"
                    + Integer.toHexString(value.type) + " is not valid");
        }
    }
    
    /*package*/ XmlResourceParser loadXmlResourceParser(String file, int id,
            int assetCookie, String type) throws NotFoundException {
        if (id != 0) {
            try {
                // These may be compiled...
                synchronized (mCachedXmlBlockIds) {
                    // First see if this block is in our cache.
                    final int num = mCachedXmlBlockIds.length;
                    for (int i=0; i<num; i++) {
                        if (mCachedXmlBlockIds[i] == id) {
                            //System.out.println("**** REUSING XML BLOCK!  id="
                            //                   + id + ", index=" + i);
                            return mCachedXmlBlocks[i].newParser();
                        }
                    }

                    // Not in the cache, create a new block and put it at
                    // the next slot in the cache.
                    XmlBlock block = mAssets.openXmlBlockAsset(
                            assetCookie, file);
                    if (block != null) {
                        int pos = mLastCachedXmlBlockIndex+1;
                        if (pos >= num) pos = 0;
                        mLastCachedXmlBlockIndex = pos;
                        XmlBlock oldBlock = mCachedXmlBlocks[pos];
                        if (oldBlock != null) {
                            oldBlock.close();
                        }
                        mCachedXmlBlockIds[pos] = id;
                        mCachedXmlBlocks[pos] = block;
                        //System.out.println("**** CACHING NEW XML BLOCK!  id="
                        //                   + id + ", index=" + pos);
                        return block.newParser();
                    }
                }
            } catch (Exception e) {
                NotFoundException rnf = new NotFoundException(
                        "File " + file + " from xml type " + type + " resource ID #0x"
                        + Integer.toHexString(id));
                rnf.initCause(e);
                throw rnf;
            }
        }

        throw new NotFoundException(
                "File " + file + " from xml type " + type + " resource ID #0x"
                + Integer.toHexString(id));
    }
    ........
}
~~~
通过上面代码我们知道，最后通过XmlBlock.newParser()生成一个xml解析器，也即是XmlResourceParser，**从以上我们知道Android中布局文件用的xml解析方法就是PULL解析方式。** 下面我们具体来看填充方法4：
~~~java
 /**
     * 填充方法4
     * Inflate a new view hierarchy from the specified XML node. Throws
     * {@link InflateException} if there is an error.
     * <p>
     */
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }
                final String name = parser.getName();
                ..........
                if (TAG_MERGE.equals(name)) {//核心代码1
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
                    rInflate(parser, root, inflaterContext, attrs, false);//核心代码2
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);//核心代码3

                    ViewGroup.LayoutParams params = null;
                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " + root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);//核心代码4

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }
            ........
            return result;
        }
    }
    ..........
}
~~~
**我们在写布局文件的时候，经常也会考虑到一些布局优化的问题，所以难免会用到include、merge、ViewStub标签，所以在XML解析的时候，需要针对此类标签作特别处理。** 

从上面代码中，我们来看核心代码1, TAG_MERGE.equals(name)，其中 TAG_MERGE就为merge，这里主要就是针对merge标签处理。我们先来看核心代码3和4，
核心代码3： createViewFromTag(root, name, inflaterContext, attrs),具体我们也来看看源码：
~~~java
   private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
        return createViewFromTag(parent, name, context, attrs, false);
    }
    /**
     * Creates a view from a tag name using the supplied attribute set.
     */
    View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
        if (name.equals("view")) {
            name = attrs.getAttributeValue(null, "class");
        }

        // Apply a theme wrapper, if allowed and one is specified.
        if (!ignoreThemeAttr) {
            final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
            final int themeResId = ta.getResourceId(0, 0);
            if (themeResId != 0) {
                context = new ContextThemeWrapper(context, themeResId);
            }
            ta.recycle();
        }

        if (name.equals(TAG_1995)) {
            // Let's party like it's 1995!
            return new BlinkLayout(context, attrs);
        }

        try {
            View view;
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs);
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, context, attrs);
            } else {
                view = null;
            }

            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }

            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {
                        view = onCreateView(parent, name, attrs);
                    } else {
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
        } catch (InflateException e) {
            throw e;

        } catch (ClassNotFoundException e) {
            final InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;

        } catch (Exception e) {
            final InflateException ie = new InflateException(attrs.getPositionDescription()
                    + ": Error inflating class " + name);
            ie.initCause(e);
            throw ie;
        }
    }
~~~
我们也来看看核心代码4：rInflateChildren(parser, temp, attrs, true)，我们具体来看看相关代码：
~~~java
    final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
            boolean finishInflate) throws XmlPullParserException, IOException {
        rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
    }
~~~
通过上面发现，最后也还是调用了核心代码2，rInflate(parser, root, inflaterContext, attrs, false)，下面我们具体来看看核心代码2：
~~~java
void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        final int depth = parser.getDepth();
        int type;
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();
            
            if (TAG_REQUEST_FOCUS.equals(name)) {
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {//代码1
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {//代码2
                throw new InflateException("<merge /> must be the root element");
            } else {
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);
                viewGroup.addView(view, params);
            }
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }
~~~
这里我们看到，主要就是通过PULL解析布局文件的方式解析出相关的View结构，具体PULL解析方式这里就不介绍了，小伙伴可以自行百度。在这里PULL解析的过程中，还对标签include,merge做了特别处理，是不是发现貌似代码中没有对标签ViewStub进行处理，其实是有的，主要是在createView()方法中，这里就不介绍了，小伙伴可以自行查一下源码。

好了，到这里，就差不多分析完了。

**注：源码采用android-6.0.1_r1版本，建议下载源码然后自己走一遍流程，这样更能加深理解。**

# 二、参考文档
[Activity布局加载流程源码分析(I)](http://blog.csdn.net/awenyini/article/details/78934390)

[Activity布局加载流程源码分析(II)](https://blog.csdn.net/awenyini/article/details/78964353)

[Android解析XML的三种方式](https://blog.csdn.net/d_shadow/article/details/55253586)
