---
layout: post
title: "Butter Knife框架源码解析"
date: 11/17/2017 11:22:25 AM  
comments: true
tags: 
	- 技术 
	- Android
	- 开源框架源码解析
	- Butterknife框架源码解析
---
---
最初的开始，findViewById()获取View控件，setOnClickListener设置View的监听事件，然后UI界面开始有响应。当初完成这个操作，有点兴奋，而这也成为我Android开发的起点。随着时间的推移，android也越来越熟悉，findViewById和setOnClickListener不知写了多少遍，偶发现有好大一部分时间，就是在写findViewById获取变量。针对这问题，在网络上发现了Jake Wharton大神的Butterknife开源框架，后用之，节约了很多时间。本篇博文将会对Butterknife源码进行解析。

# 一、什么是Butterknife？
Butterknife，是专门为Android View设计的绑定注解框架，专业解放各种findViewById和setOnClickListener。

Butterknife地址：[https://github.com/JakeWharton/butterknife](https://github.com/JakeWharton/butterknife)

# 二、Butterknife源码解析
**1.Butterknife绑定分析**
```java
public class FeedbackActivity extends Activity{

    @BindView(R.id.mContentEdit)
    EditText mContentEdit;

    @BindView(R.id.mSubmitBtn)//1.View注解
    TextView mSubmitBtn;

    @BindView(R.id.mContentCountTv)
    TextView mContentCountTv;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_feedback);
        ButterKnife.bind(this);//3.与Activity绑定
    }

    @OnClick(R.id.mSubmitBtn)//2.View监听事件注解
    public void onClick() {
    }
}
```
<!-- more -->
由1知，@BindView注解FeedbackActivity的3个View变量，由2知，@OnClick注解View的监听事件。然而这三个变量和监听事件，是怎么和FeedbackActivity关联起来的呢？这里就需要我们注意3与Activity的绑定，即ButterKnife.bind(this)方法。进入ButterKnife源码看看。
```java
  /**
   * BindView annotated fields and methods in the specified {@link Activity}. The current content
   * view is used as the view root.
   *
   * @param target Target activity for view binding.
   */
  @NonNull @UiThread
  public static Unbinder bind(@NonNull Activity target) {
    View sourceView = target.getWindow().getDecorView();
    return createBinding(target, sourceView);
  }
```
ButterKnife中bind有多个重载，针对View，Dialog，Activtiy等，具体可以自行查看ButterKnife源码，这里主要针对Activity。我们继续看createBinding方法,
```java
 private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    Class<?> targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());

    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);//查找辅助绑定类的构造方法

    if (constructor == null) {
      return Unbinder.EMPTY;
    }

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
      return constructor.newInstance(target, source);//初始化了构造方法
    } catch (IllegalAccessException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InstantiationException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InvocationTargetException e) {
      Throwable cause = e.getCause();
      if (cause instanceof RuntimeException) {
        throw (RuntimeException) cause;
      }
      if (cause instanceof Error) {
        throw (Error) cause;
      }
      throw new RuntimeException("Unable to create binding instance.", cause);
    }
  }
```
findBindingConstructorForClass()通过传入目标类即Activity，查找辅助绑定类的构造方法。继续看findBindingConstructorForClass()方法
```java
  @Nullable @CheckResult @UiThread
  private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
    String clsName = cls.getName();//获取绑定类的name
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }
    try {
      Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");//核心方法，通过反射加载类"FeedbackActivity_ViewBinding"
      //noinspection unchecked
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);//获取"FeedbackActivity_ViewBinding"构造函数，
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;//返回绑定类的构造函数，初始化
  }
```
通过反射加载FeedbackActivity_ViewBinding类，而FeedbackActivity_ViewBinding类到底像啥样呢？这里我们可以先看一下此类,后面将会分析此类怎么生成。
```java
public class FeedbackActivity_ViewBinding<T extends FeedbackActivity> implements Unbinder {
  protected T target;

  private View view2131624065;

  @UiThread
  public FeedbackActivity_ViewBinding(final T target, View source) {
    this.target = target;
    View view;
    target.mContentEdit = Utils.findRequiredViewAsType(source, R.id.mContentEdit, "field 'mContentEdit'", EditText.class);
    view = Utils.findRequiredView(source, R.id.mSubmitBtn, "field 'mSubmitBtn' and method 'onClick'");
    target.mSubmitBtn = Utils.castView(view, R.id.mSubmitBtn, "field 'mSubmitBtn'", TextView.class);
    view2131624065 = view;
    view.setOnClickListener(new DebouncingOnClickListener() {
      @Override
      public void doClick(View p0) {
        target.onClick();
      }
    });
    target.mContentCountTv = Utils.findRequiredViewAsType(source, R.id.mContentCountTv, "field 'mContentCountTv'", TextView.class);
  }
...省略...
}
```
其中findRequiredViewAsType和findRequiredView方法皆是Utils的方法，我们查看Utils方法可以知道，两个方法最后都调了findViewById方法,从而就绑定了注解的View变量，绑定View监听事件。
```java
  public static <T> T findRequiredViewAsType(View source, @IdRes int id, String who,
      Class<T> cls) {
    View view = findRequiredView(source, id, who);
    return castView(view, id, who, cls);
  }


  public static View findRequiredView(View source, @IdRes int id, String who) {
    View view = source.findViewById(id);
    if (view != null) {
      return view;
    }
    String name = getResourceEntryName(source, id);
    throw new IllegalStateException("Required view '"
        + name
        + "' with ID "
        + id
        + " for "
        + who
        + " was not found. If this view is optional add '@Nullable' (fields) or '@Optional'"
        + " (methods) annotation.");
  }
```
到这里，ButterKnife的绑定流程就介绍完了。**ButterKnife主要就是通过注解，然后生成一个辅助类动态绑定View,解放开发者**。下面我们将会分析辅助类FeedbackActivity_ViewBinding的生成。

**2.Butterknife辅助类的生成**

由上，我们知Butterknife框架是用注解的方式实现的，注解的实现方式，很容易让我们想到通过Java反射机制实现，但是我们知道如果通过反射，是在运行时（Runtime）来处理View的绑定等一些列事件，这样比较耗费资源，会影响应用的性能。**Butterknife框架没有用java反射机制，而是使用APT(Annotation Processing Tool)编译时解析技术。**提到APT，我们就必须要了解AbstractProcessor类，通过继承此类，编译器在编译的时候，会扫描所有带有你注解的类，然后再调用AbstractProcessor的process方法，对注解进行处理，然后生成相关的辅助类，如上FeedbackActivity_ViewBinding。

在Butterknife框架中ButterKnifeProcessor继承了AbstractProcessor，所以我们重点来关注一下ButterKnifeProcessor的process方法
```java
  @Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
    Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);//1.核心方法

    for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
      TypeElement typeElement = entry.getKey();
      BindingSet binding = entry.getValue();

      JavaFile javaFile = binding.brewJava(sdk, debuggable);//2.核心方法
      try {
        javaFile.writeTo(filer);//3.核心方法
      } catch (IOException e) {
        error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
      }
    }

    return false;
  }
```
通过遍历bindingMap，再利用JavaFile，就可以生成辅助类了。bindingMap的类型为Map<TypeElement, BindingSet>，其中TypeElement为key,BindingSet为值，TypeElement和BindingSet两个类在Butterknife框架中都是辅助生成辅助类的方法。**BindingSet对辅助类的生成起到非常重要的作用，类名，变量，方法名各种信息都是储存在BindingSet类中。**这里不重讲了。让我们继续分析findAndParseTargets方法。
```java
private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
    Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
    Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();

    scanForRClasses(env);//1.扫描所有注解类

    .........//省略的内容雷同BindView注解

    // Process each @BindView element.
    for (Element element : env.getElementsAnnotatedWith(BindView.class)) {//2.遍历有@BindView注解的类
      // we don't SuperficialValidation.validateElement(element)
      // so that an unresolved View type can be generated by later processing rounds
      try {
        parseBindView(element, builderMap, erasedTargetNames);//3.解析@BindView注解类
      } catch (Exception e) {
        logParsingError(element, BindView.class, e);
      }
    }
    .........//省略的内容雷同BindView注解

    // Process each annotation that corresponds to a listener.
    for (Class<? extends Annotation> listener : LISTENERS) {//4.遍历监听事件注解类
      findAndParseListener(env, listener, builderMap, erasedTargetNames);
    }

    // Associate superclass binders with their subclass binders. This is a queue-based tree walk
    // which starts at the roots (superclasses) and walks to the leafs (subclasses).
    Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries =
        new ArrayDeque<>(builderMap.entrySet());
    Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
    while (!entries.isEmpty()) {
      Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();

      TypeElement type = entry.getKey();
      BindingSet.Builder builder = entry.getValue();

      TypeElement parentType = findParentType(type, erasedTargetNames);
      if (parentType == null) {
        bindingMap.put(type, builder.build());
      } else {
        BindingSet parentBinding = bindingMap.get(parentType);
        if (parentBinding != null) {
          builder.setParent(parentBinding);
          bindingMap.put(type, builder.build());
        } else {
          // Has a superclass binding but we haven't built it yet. Re-enqueue for later.
          entries.addLast(entry);
        }
      }
    }
    return bindingMap;
  }

```
方法scanForRClasses，主要是扫描所有类，区分注解类。这里不重点讲了。拿到@BindView的注解类后，遍历，然后通过parseBindView方法解析，我们继续看parseBindView方法
```java
 private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap,
      Set<TypeElement> erasedTargetNames) {
    TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

    .......//省略代码，主要检测注解的合法性，这里不介绍

    if (hasError) {
      return;
    }

    // Assemble information on the field.
    int id = element.getAnnotation(BindView.class).value();//1.获取注解值，相当于获取view ID（R.id.mSubmitBtn）

    BindingSet.Builder builder = builderMap.get(enclosingElement);//2.builderMap中获取BindingSet.Builder
    QualifiedId qualifiedId = elementToQualifiedId(element, id);
    if (builder != null) {//3.builder是否为空
      String existingBindingName = builder.findExistingBindingName(getId(qualifiedId));
      if (existingBindingName != null) {//4.检测是否重复绑定
        error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
            BindView.class.getSimpleName(), id, existingBindingName,
            enclosingElement.getQualifiedName(), element.getSimpleName());
        return;
      }
    } else {
      builder = getOrCreateBindingBuilder(builderMap, enclosingElement);//5.为空，需要重新get或生成一个新的BindingSet.Builder
    }

    String name = simpleName.toString();
    TypeName type = TypeName.get(elementType);
    boolean required = isFieldRequired(element);

    builder.addField(getId(qualifiedId), new FieldViewBinding(name, type, required));//6.往Builder中添加新值

    // Add the type-erased version to the valid binding targets set.
    erasedTargetNames.add(enclosingElement);
  }
```
第一次，一般都为空值，所以会执行getOrCreateBindingBuilder(builderMap, enclosingElement)方法，其中传入了BuilderMap和enclosingElement,我们继续来看方法getOrCreateBindingBuilder
```java
  private BindingSet.Builder getOrCreateBindingBuilder(
      Map<TypeElement, BindingSet.Builder> builderMap, TypeElement enclosingElement) {
    BindingSet.Builder builder = builderMap.get(enclosingElement);
    if (builder == null) {
      builder = BindingSet.newBuilder(enclosingElement);
      builderMap.put(enclosingElement, builder);
    }
    return builder;
  }
```
其中，再次判断了builderMap中是否已存在BindingSet.Builder，不存在就新建，通过BindingSet.newBuilder(enclosingElement)方法，让我继续看BindingSet中newBuilder方法
```java
  static Builder newBuilder(TypeElement enclosingElement) {
    TypeMirror typeMirror = enclosingElement.asType();

    boolean isView = isSubtypeOfType(typeMirror, VIEW_TYPE);
    boolean isActivity = isSubtypeOfType(typeMirror, ACTIVITY_TYPE);
    boolean isDialog = isSubtypeOfType(typeMirror, DIALOG_TYPE);

    TypeName targetType = TypeName.get(typeMirror);
    if (targetType instanceof ParameterizedTypeName) {
      targetType = ((ParameterizedTypeName) targetType).rawType;
    }

    String packageName = getPackage(enclosingElement).getQualifiedName().toString();
    String className = enclosingElement.getQualifiedName().toString().substring(
        packageName.length() + 1).replace('.', '$');
    ClassName bindingClassName = ClassName.get(packageName, className + "_ViewBinding");//辅助类的类名

    boolean isFinal = enclosingElement.getModifiers().contains(Modifier.FINAL);
    return new Builder(targetType, bindingClassName, isFinal, isView, isActivity, isDialog);
  }
```
上面说到BindingSet对辅助类的生成起到非常重要的作用，这里生成了辅助类的类名，并保存在bindingSet.builder中，最后put入了builderMap中。在解析的过程中，BindingSet的都伴随左右，可以看出她在ButterKnifeProcessor类中的重要性。到这里我们在回溯一下。

**1.BindingSet.newBuilder(enclosingElement) 生成类名及相关类型定义 -> 2.BuilderMap.put(enclosingElement, builder) put入Map<TypeElement, BindingSet.Builder>中 ->3.builderMap.get(enclosingElement)获取BindingSet.builder -> 4.builder.addField(getId(qualifiedId), new FieldViewBinding(name, type, required))往Builder中添加注解值 ->Map<TypeElement, BindingSet.Builder>转换成Map<TypeElement, BindingSet> -> 遍历BindingMap -> bindingSet.brewJava(sdk, debuggable).write(filter)生成辅助类。**

到这里，Butterknife框架生成辅助类，就讲解的差不多了。其中核心原理就是利用BindingSet类强大的类构建能力，生成相关类名，方法和核心代码。对于BindingSet具体是怎么生成辅助类的，想深入了解的朋友也可以下载ButterKnife源码对BindingSet进行深度了解，这里主要介绍ButterKnife框架原理。

# 三、总结
通过了解Butterknife框架，知其思想真是值得称赞。注解绑定一下，就告别重复的findViewById等方法，节约了开发时间，提高了效率。

# 四、相关及参考文档

[Butterknife官方指引](http://jakewharton.github.io/butterknife/)

[ButterKnife源码剖析](http://blog.csdn.net/chenkai19920410/article/details/51020151)

[Annontation注解的应用及介绍](http://blog.csdn.net/awenyini/article/details/77478230)

[Android编译时注解APT实战（AbstractProcessor）](http://www.jianshu.com/p/07ef8ba80562)

