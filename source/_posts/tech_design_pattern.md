---
layout: post
title: "设计模式-六大原则"
date: 12/13/2017 4:14:35 PM 
comments: true
tags: 
	- 技术 
	- 设计模式
---
---
当初作为小白，提到设计模式，就会觉得很高大上，很牛叉。其实，在我们身边，在我们的项目中，设计模式的身影无处不在。然而，什么是设计模式呢？百度解释为：**设计模式（Design Pattern）是一套被反复使用、多数人知晓的、经过分类的、代码设计经验的总结。** 经验总是值得学习的，特别是对我们编程有极大帮助的设计模式经验。在Java中常见的设计模式有23种，而这23种设计模式都遵循了设计模式的六大原则，这六大原则分别是：
 
 - 单一职责原则
 - 里氏替换原则
 - 依赖倒转原则
 - 接口隔离原则
 - 迪米特法则
 - 开放封闭原则

# 一、单一职责原则
1.定义：不要存在多于一个导致类变更的原因。

2.通俗的说：**一个类只负责一项职责**。

3.优点

- 可以降低类的复杂度，一个类只负责一项职责，其逻辑肯定要比负责多项职责简单的多；
- 提高类的可读性，提高系统的可维护性；
- 变更引起的风险降低，变更是必然的，如果单一职责原则遵守的好，当修改一个功能时，可以显著降低对其他功能的影响。

<!-- more -->
# 二、里氏替换原则
1.定义：所有引用基类的地方必须能透明地使用其子类的对象。

2.通俗的说：**子类可以扩展父类的功能，但不能改变父类原有的功能**。

具体含义：

* 子类可以实现父类的抽象方法，但不能覆盖父类的非抽象方法。
* 子类中可以增加自己特有的方法。
* 当子类的方法重载父类的方法时，方法的前置条件（即方法的形参）要比父类方法的输入参数更宽松。
* 当子类的方法实现父类的抽象方法时，方法的后置条件（即方法的返回值）要比父类更严格。

# 三、依赖倒转原则
1.定义:高层模块不应该依赖低层模块，二者都应该依赖其抽象；抽象不应该依赖细节；细节应该依赖抽象。

2.通俗的说：**面向接口编程**。

实际操作应注意：

* 低层模块尽量都要有抽象类或接口，或者两者都有。
* **变量的声明类型尽量是抽象类或接口。**
* 使用继承时遵循里氏替换原则。


# 四、接口隔离原则
1.定义：客户端不应该依赖它不需要的接口；一个类对另一个类的依赖应该建立在最小的接口上。

2.通俗的说：**建立单一接口，不要建立庞大臃肿的接口。**

注意要点：

* 接口尽量小，但是要有限度。对接口进行细化可以提高程序设计灵活性是不挣的事实，但是如果过小，则会造成接口数量过多，使设计复杂化。所以一定要适度。
* 为依赖接口的类定制服务，只暴露给调用的类它需要的方法，它不需要的方法则隐藏起来。只有专注地为一个模块提供定制服务，才能建立最小的依赖关系。
* 提高内聚，减少对外交互。使接口用最少的方法去完成最多的事情。

# 五、迪米特法则
1.定义：**一个对象应该对其他对象保持最少的了解。**

2.通俗的说：**只与直接的朋友通信。**

什么是直接的朋友？只要两个对象之间有耦合关系，我们就说这两个对象之间是朋友关系。耦合的方式很多，依赖、关联、组合、聚合等。其中，我们称出现成员变量、方法参数、方法返回值中的类为直接的朋友，而出现在局部变量中的类则不是直接的朋友。

# 六、开放封闭原则
1.定义：一个软件实体如类、模块和函数应该**对扩展开放，对修改关闭**。

2.通俗的说：**用抽象构建框架，用实现扩展细节**。

---

**依赖倒转原则**与**里氏替换原则**实例：

```java

public interface ICar {
    void controlDirection();//控制方向
    void addGas();//加油
    void brakeCar();//刹车
}


public class CayenneCar implements ICar {
    @Override
    public void controlDirection() {
    }

    @Override
    public void addGas() {
    }

    @Override
    public void brakeCar() {
    }
}

public class HavardCar implements ICar {
    @Override
    public void controlDirection() {
    }

    @Override
    public void addGas() {
    }

    @Override
    public void brakeCar() {
    }
}

public class Person {

    private ICar car;

    public Person(ICar car) {
        this.car = car;
    }

    public void drive(){
        car.addGas();
        car.brakeCar();
        car.controlDirection();
    }
}

public class DesignPattern {

    public DesignPattern() {

        HavardCar havardCar = new HavardCar();
        Person boy = new Person(havardCar);
        boy.drive();

        CayenneCar bydCar = new CayenneCar();
        Person girl = new Person(bydCar);
        girl.drive();

    }
}
```


## 参考文献

[设计模式六大原则（1）：单一职责原则](http://blog.csdn.net/zhengzhb/article/details/7278174)

[设计模式六大原则（2）：里氏替换原则](http://blog.csdn.net/zhengzhb/article/details/7281833)

[设计模式六大原则（3）：依赖倒转原则](http://blog.csdn.net/zhengzhb/article/details/7289269)

[设计模式六大原则（4）：接口隔离原则](http://blog.csdn.net/zhengzhb/article/details/7296921)

[设计模式六大原则（5）：迪米特法则](http://blog.csdn.net/zhengzhb/article/details/7296930)

[设计模式六大原则（6）：开放封闭原则](http://blog.csdn.net/zhengzhb/article/details/7296944)




