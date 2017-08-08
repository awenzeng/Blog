---
layout: post
title: "【Android注解】Annontation注解的应用及介绍"
date: 8/1/2017 7:38:14 PM 
comments: true
tags: 
	- 技术 
	- Android注解
	- Java反射机制
	- Java动态代理
---
---
# 一、什么是注解？
Annontation是Java5开始引入的新特征，中文名称叫注解。它提供了一种安全的类似注释的机制，用来将任何的信息或元数据（metadata）与程序元素（类、方法、成员变量等）进行关联。为程序的元素（类、方法、成员变量）加上更直观更明了的说明，这些说明信息是与程序的业务逻辑无关，并且供指定的工具或框架使用。Annontation像一种修饰符一样，应用于包、类型、构造方法、方法、成员变量、参数及本地变量的声明语句中。

# 二、注解的用处
  - 生成文档。这是最常见的，也是java 最早提供的注解。常用的有@param @return 等
  - 跟踪代码依赖性，实现替代配置文件功能。
  - 在编译时进行格式检查。如@override 放在方法前，如果你这个方法并不是覆盖了超类方法，则编译时就能检查出。

# 三、注解介绍

**元注解**

java.lang.annotation提供了四种元注解，专门注解其他的注解：
  - @Documented  –注解是否将包含在JavaDoc中
  - @Retention   –什么时候使用该注解
  - @Target      –注解用于什么地方
  - @Inherited   – 是否允许子类继承该注解

**1）@Retention– 定义该注解的生命周期**

  - RetentionPolicy.SOURCE : 在编译阶段丢弃。这些注解在编译结束之后就不再有任何意义，所以它们不会写入字节码。@Override, @SuppressWarnings都属于这类注解。
  - RetentionPolicy.CLASS : 在类加载的时候丢弃。在字节码文件的处理中有用。注解默认使用这种方式
  - RetentionPolicy.RUNTIME : 始终不会丢弃，运行期也保留该注解，因此可以使用反射机制读取该注解的信息。我们自定义的注解通常使用这种方式。

**2）@Target – 表示该注解用于什么地方。默认值为任何元素，表示该注解用于什么地方。可用的ElementType参数包括**

- ElementType.CONSTRUCTOR:用于描述构造器
- ElementType.FIELD:成员变量、对象、属性（包括enum实例）
- ElementType.LOCAL_VARIABLE:用于描述局部变量
- ElementType.METHOD:用于描述方法
- ElementType.PACKAGE:用于描述包
- ElementType.PARAMETER:用于描述参数
- ElementType.TYPE:用于描述类、接口(包括注解类型) 或enum声明

<!-- more -->

**3)@Documented–一个简单的Annotations标记注解，表示是否将注解信息添加在java文档中。**

**4)@Inherited – 定义该注释和子类的关系**

   @Inherited 元注解是一个标记注解，@Inherited阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。

# 四、注解使用

**1）方法注解：**

```java
@Target(METHOD)
@Retention(RUNTIME)
public @interface UserMethod {
    String title() default "";
}
```

**2）参数注解：**

```java
@Target(PARAMETER)
@Retention(RUNTIME)
public @interface UserParam {
    String name() default "";
    String phone() default "";
}
```

**3）注解使用：**
```java
public interface UserInterface {
    @UserMethod(title = "AwenZeng")
    String getUser(@UserParam(name = "刘峰",phone = "110") String a);
}
```

**4）获取注解：**

通过[反射机制](http://blog.csdn.net/liujiahan629629/article/details/18013523)获取函数注解信息：

```java
Method[] declaredMethods = UserInterface.class.getDeclaredMethods();
        for (Method method : declaredMethods) {
            Annotation[]  methodAnnotations = method.getAnnotations();
            Annotation[][] parameterAnnotationsArray = method.getParameterAnnotations();
        }
```

也可以获取指定的注解:

```java
UserMethod userMethod = method.getAnnotation(UserMethod.class);
```

**5) 具体实现注解接口调用**

采用[Java动态代理机制](http://www.cnblogs.com/flyoung2008/archive/2013/08/11/3251148.html)来实现:

```java

    public <T> T create(final Class<T> service) {
        return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[]{service},
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object... args)
                            throws Throwable {
                        // Annotation[]  methodAnnotations = method.getAnnotations();//拿到函数注解数组
                        UserMethod userMethod = method.getAnnotation(UserMethod.class);
                        Log.e("good", "UserParam---getValue->" + userMethod.title());
                        Type[] parameterTypes = method.getGenericParameterTypes();
                        Annotation[][] parameterAnnotationsArray = method.getParameterAnnotations();//拿到参数注解
                        for (int i = 0; i < parameterAnnotationsArray.length; i++) {
                            Annotation[] annotations = parameterAnnotationsArray[i];
                            if (annotations != null) {
                                UserParam reqParam = (UserParam) annotations[0];
                                Log.e("good", "reqParam---reqParam->" + reqParam.name()+ ","+reqParam.phone()+ "," + args[i]);
                            }
                        }
                        //下面就可以执行相应的网络请求获取结果 返回结果
                        String result = "";//这里模拟一个结果

                        return result;
                    }
              });
    }
```

**6) 具体代码调用**

```java
        UserInterface userInterface = AnnotionProxy.create(UserInterface.class);
        userInterface.getUser("我之存在，因为有你。");
```

**7) 结果**

```java
08-01 12:11:14.688 11729-11729/com.awen.annotationdemo E/good: UserMethod---title->AwenZeng
08-01 12:11:14.688 11729-11729/com.awen.annotationdemo E/good: UserParam---userParam->刘峰,110,我之存在，因为有你。
```

# 五、参考资料、

[Java学习之注解Annotation实现原理](http://www.cnblogs.com/whoislcj/p/5671622.html)

[JAVA中的反射机制](http://blog.csdn.net/liujiahan629629/article/details/18013523)

[彻底理解JAVA动态代理](http://www.cnblogs.com/flyoung2008/archive/2013/08/11/3251148.html)
