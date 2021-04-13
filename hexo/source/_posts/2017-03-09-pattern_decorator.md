---
layout: post
title: 设计模式之装饰模式
tags: [设计模式,装饰模式,包装器模式]
categories: 
- 设计模式
---

# 概述
装饰模式（Decorator）也叫包装器模式（Wrapper），是指动态地给一个对象添加一些额外的职责，就增加功能来说装饰模式比生成子类更为灵活。它通过创建一个包装对象，也就是装饰来包裹真实的对象

# 情景举例
我们先来分析这样一个画图形的需求：

 1. 它能绘制各种背景，如红色、蓝色、绿色
 2. 它能绘制形状，如三角形，正方形，圆形
 3. 它能给形状加上阴影
 
就先列这三个简单的需求吧，下面让我们比较下各种实现的优缺点

# 丑陋的实现
来看看我们用继承是如何实现的，首先，抽象出一个`Shape`接口我想大家都不会有意见的是不是？
```java
/**
 * @author HansChen
 */
public interface Shape {

    /**
     * 绘制图形
     */
    void draw();
}
```

然后我们定义各种情况下的子类，结构如下，看到这么多的子类，是不是有点要爆炸的感觉？真是想想都可怕
![2019-9-2-12-35-9.png](https://gitee.com/hanschencoder/Images/raw/master/2019-9-2-12-35-9.png)

而且如果再新增一种需求，比如现在要画椭圆，那么维护的人员估计就要爆粗了吧？

<!-- more -->

为了避免写出上面的代码，聪明的童鞋们可能会提出第二种方案：
```java
/**
 * @author HansChen
 */
public class ShapeImpl implements Shape {

    enum Type {
        Circle,
        Square,
        Trilatera
    }

    enum Color {
        Red,
        Green,
        Blue
    }

    private Type    type;
    private Color   color;
    private boolean shadow;

    public ShapeImpl() {
    }

    public Type getType() {
        return type;
    }

    public void setType(Type type) {
        this.type = type;
    }

    public Color getColor() {
        return color;
    }

    public void setColor(Color color) {
        this.color = color;
    }

    public boolean isShadow() {
        return shadow;
    }

    public void setShadow(boolean shadow) {
        this.shadow = shadow;
    }

    @Override
    public void draw() {
        // TODO: 2017/3/9 根据属性情况画出不同的图
    }
}
```
这样，根据不同的画图需求，只需要设置不同的属性就可以了，这样确实避免了类爆炸增长的问题，但这种方式违反了开放封闭原则，比如画正方形的方式变了，需要对`ShapeImpl`进行修改，或者如果新增需求，如画椭圆，也需要对`ShapeImpl`进行修改。而且这个类不方便扩展，子类将继承一些对自身并不合适的方法。

# 装饰模式
## 概念介绍
> 装饰模式（Decorator）也叫包装器模式（Wrapper），是指动态地给一个对象添加一些额外的职责

以下情况使用Decorator模式：

 - 需要扩展一个类的功能，或给一个类添加附加职责。
 - 需要动态的给一个对象添加功能，这些功能可以再动态的撤销。
 - 需要增加由一些基本功能的排列组合而产生的非常大量的功能，从而使继承关系变的不现实。
 - 当不能采用生成子类的方法进行扩充时。一种情况是，可能有大量独立的扩展，为支持每一种组合将产生大量的子类，使得子类数目呈爆炸性增长。另一种情况可能是因为类定义被隐藏，或类定义不能用于生成子类

但这种灵活也会带来一些缺点，这种比继承更加灵活机动的特性，也同时意味着更加多的复杂性。装饰模式会导致设计中出现许多小类，如果过度使用，会使程序变得很复杂

下面来看看装饰模式的结构：
![2019-9-2-12-35-32.png](https://gitee.com/hanschencoder/Images/raw/master/2019-9-2-12-35-32.png)
 
 1. Component抽象组件，是一个接口或者是抽象类，就是定义我们最核心的对象，也就是最原始的对象。（注：在装饰模式中，必然有一个最基本、最核心、最原始的接口或者抽象类充当Component抽象组件）
 2. ConcreteComponent具体组件，是最核心、最原始、最基本的接口或抽象类的实现，我们需要装饰的就是它
 3. Decorator装饰角色， 一般是一个抽象类，实现接口或者抽象方法，它的属性里必然有一个private变量指向Component抽象组件。
 4. 具体装饰角色，如上图中的ConcreteDecoratorA和ConcreteDecoratorB，我们要把我们最核心的、最原始的、最基本的东西装饰成其它东西。

 
代码示例如下：
```java
 /**
 * @author HansChen
 */
public interface Component {

    void operation();
}
```
```java
public class ConcreteComponent implements Component {

    @Override
    public void operation() {
        System.out.print("do something");
    }
}
```
```java
public class Decorator implements Component {

    private Component component;

    public Decorator(Component component) {
        this.component = component;
    }

    @Override
    public void operation() {
        component.operation();
    }
}
```
```java
public class ConcreteDecoratorA extends Decorator {

    public ConcreteDecoratorA(Component component) {
        super(component);
    }

    @Override
    public void operation() {
        super.operation();
        System.out.println("do something");
    }
}
```
```java
public class ConcreteDecoratorB extends Decorator {

    public ConcreteDecoratorB(Component component) {
        super(component);
    }

    @Override
    public void operation() {
        super.operation();
        System.out.println("do something");
    }
}
```

上面说了一堆结构和示例代码，但大家可能还是不太好理解，下面用装饰模式来重新实现画图的功能

## 用装饰模式实现需求
先上结构图
![2019-9-2-12-35-51.png](https://gitee.com/hanschencoder/Images/raw/master/2019-9-2-12-35-51.png)

首先定义可动态扩展对象的抽象
```java
public interface Shape {

    /**
     * 绘制图形
     */
    void draw();
}
```

定义具体的组件，每一个组件代表一个形状
```java
public class Square implements Shape {

    @Override
    public void draw() {
        System.out.print("正方形");
    }
}
```
```java
public class Trilateral implements Shape {

    @Override
    public void draw() {
        System.out.print("三角形");
    }
}
```
```java
public class Circle implements Shape {

    @Override
    public void draw() {
        System.out.print("圆形");
    }
}
```

定义可装饰者的抽象类
```java
public class ShapeDecorator implements Shape {

    private Shape shape;

    public ShapeDecorator(Shape shape) {
        this.shape = shape;
    }

    @Override
    public void draw() {
        shape.draw();
    }
}
```

定义具体的装饰者
```java
public class Blue extends ShapeDecorator {

    public Blue(Shape shape) {
        super(shape);
    }

    @Override
    public void draw() {
        super.draw();
        System.out.print(" 蓝色");
    }
}
```
```java
public class Green extends ShapeDecorator {

    public Green(Shape shape) {
        super(shape);
    }

    @Override
    public void draw() {
        super.draw();
        System.out.print(" 绿色");
    }
}
```
```java
public class Red extends ShapeDecorator {

    public Red(Shape shape) {
        super(shape);
    }

    @Override
    public void draw() {
        super.draw();
        System.out.print(" 红色");
    }
}
```
```java
public class Shadow extends ShapeDecorator {

    public Shadow(Shape shape) {
        super(shape);
    }

    @Override
    public void draw() {
        super.draw();
        System.out.print(" 有阴影");
    }
}
```

好了，现在让我们看看具体怎么使用：
```java
public class Test {

    public static void main(String[] args) {
        //正方形 红色 有阴影
        Shape shape = new Square();
        shape = new Red(shape);
        shape = new Shadow(shape);
        shape.draw();

        //圆形 绿色
        shape = new Circle();
        shape = new Green(shape);
        shape.draw();

        //三角形 蓝色 有阴影
        shape = new Trilateral();
        shape = new Blue(shape);
        shape = new Shadow(shape);
        shape.draw();
    }
}
```
可以看到，装饰模式是非常灵活的，通过不同的装饰，实现不同的效果


# 装饰模式的应用举例
这里再列举一些用到了装饰模式的情景，童鞋们可以根据这些场景加深对装饰模式的理解

 - Java中`IO`设计
 - Android中`Context`和`ContextWrapper`的设计

# 总结
装饰模式是为已有功能动态地添加功能的一种方式，它把每个要装饰的功能放在单独的类中，并让这个类包括要装饰的对象，有效地把核心职能和装饰功能区分开了。但它带来灵活的同时，也容易导致别人不了解自己的设计方式，不知如何使用。就像Java中I/O库，人们第一次接触的时候，往往无法轻易理解它。这其中的平衡取舍，就看自己咯