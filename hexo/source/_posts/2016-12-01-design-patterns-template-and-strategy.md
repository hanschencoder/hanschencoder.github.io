---
layout: post
title: 设计模式之模板方法模式和策略模式
tags: [设计模式,模板方法模式,策略模式]
categories: 
- 设计模式
---

# 概述
我们知道，OOP三个基本特征是:封装、继承、多态。通过继承，我们可以基于差异编程，也就是说，对于一个满足我们大部分需求的类，可以创建它的一个子类并只改变我们不期望的那部分。但是在实际使用中，继承很容易被过度使用，并且过度使用的代价是比较高的，所以我们减少了继承的使用，使用组合或委托代替

> 优先使用对象组合而不是类继承

在本文中，我们会分别介绍`模板方法模式`和`策略模式`，这两个模式分别使用了继承和委托两种方式。这两种模式解决的问题是类似的，经常可以互换使用，它们都可以分离通用的算法和具体的上下文。比如我们有一个通用的算法，算法有不同的实现方式，为了遵循依赖倒置原则，我们希望算法不依赖于具体实现。

<!-- more -->

本文冒泡排序法来进行举例说明：
```java
/**
 * @author HansChen
 */
public class Sorter {

    /**
     * 冒泡排序
     */
    public int sort(int[] array) {
        int operations = 0;
        if (array.length <= 1) {
            return operations;
        }

        for (int i = 0; i < array.length - 1; i++) {
            for (int j = 0; j < array.length - i - 1; j++) {
                operations++;
                if (needSwap(array, j)) {
                    swap(array, j);
                }
            }
        }

        return operations;
    }

    /**
     * @return 是否需要交换数组中 index 和 index+1 元素
     */
    private boolean needSwap(int[] array, int index) {
        return array[index] > array[index + 1];
    }

    /**
     * 交换array数组中的 index 和 index+1 元素
     */
    private void swap(int[] array, int index) {
        int temp = array[index];
        array[index] = array[index + 1];
        array[index + 1] = temp;
    }
}
```
这是我们实现的冒泡排序算法，这个sort方法可以对int数组进行排序。但我们发现，这种写法的扩展性是不强的，如果我们要实现double数组排序呢？如果我们需要排序的是一个对象数组？难道需要各自定义一个方法吗？如果它们都使用冒泡排序算法，那么sort的算法逻辑肯定是相似的，有没有一种方法能让这个算法逻辑复用呢？下面用模板方法模式和策略模式对它进行改造

# 模板方法模式
> 模板方法模式:定义一个算法的骨架，将骨架中的特定步骤延迟到子类中。模板方法模式使得子类可以不改变算法的结构即可重新定义该算法的某些特定步骤

下图是用模板方法模式对冒泡排序重构后的结构图：
![2019-9-2-11-29-33.png](https://gitee.com/hanschencoder/Images/raw/master/2019-9-2-11-29-33.png)

首先，我们在BubbleSorter的sort方法中定义算法骨架，再定义一些延迟到子类中的抽象方法：
```java
/**
 * @author HansChen
 */
public abstract class BubbleSorter<T> {

    /**
     * 冒泡排序
     */
    public int sort(T array) {

        setArray(array);
        int length = getLength();

        int operations = 0;
        if (length <= 1) {
            return operations;
        }

        for (int i = 0; i < length - 1; i++) {
            for (int j = 0; j < length - i - 1; j++) {
                operations++;
                if (needSwap(j)) {
                    swap(j);
                }
            }
        }

        return operations;
    }

    /**
     * 初始化排序数组
     */
    protected abstract void setArray(T array);

    /**
     * @return 返回数组长度
     */
    protected abstract int getLength();

    /**
     * @return 是否需要交换数组中 index 和 index+1 元素
     */
    protected abstract boolean needSwap(int index);

    /**
     * 交换array数组中的 index 和 index+1 元素
     */
    protected abstract void swap(int index);
}
```

有了`BubbleSorter`类，我们就可以创建任意不同类型的对象排序的简单派生类，比如创建`IntBubbleSorter`去排序整型数组：
```java
public class IntBubbleSorter extends BubbleSorter<int[]> {

    private int[] array;

    @Override
    protected void setArray(int[] array) {
        this.array = array;
    }

    @Override
    protected int getLength() {
        return array == null ? 0 : array.length;
    }

    @Override
    protected boolean needSwap(int index) {
        return array != null && (array[index] > array[index + 1]);
    }

    @Override
    protected void swap(int index) {
        int temp = array[index];
        array[index] = array[index + 1];
        array[index + 1] = temp;
    }
}
```

再比如创建`DoubleBubbleSorter`去排序双精度型数组：
```java
public class DoubleBubbleSorter extends BubbleSorter<double[]> {

    private double[] array;

    @Override
    protected void setArray(double[] array) {
        this.array = array;
    }

    @Override
    protected int getLength() {
        return array == null ? 0 : array.length;
    }

    @Override
    protected boolean needSwap(int index) {
        return array != null && (array[index] > array[index + 1]);
    }

    @Override
    protected void swap(int index) {
        double temp = array[index];
        array[index] = array[index + 1];
        array[index + 1] = temp;
    }
}
```

甚至我们**不仅限于对数组**排序，还可以对List集合排序，比如创建`IntegerListBubbleSorter`对List集合进行冒泡排序：
```java
public class IntegerListBubbleSorter extends BubbleSorter<List<Integer>> {

    private List<Integer> list;

    @Override
    protected void setArray(List<Integer> list) {
        this.list = list;
    }

    @Override
    protected int getLength() {
        return list == null ? 0 : list.size();
    }

    @Override
    protected boolean needSwap(int index) {
        return list != null && (list.get(index) > list.get(index + 1));
    }

    @Override
    protected void swap(int index) {
        int temp = list.get(index);
        list.set(index, list.get(index + 1));
        list.set(index + 1, temp);
    }
}
```

定义上述类之后，我们看下怎么使用上面的类：
```java
public class Test {

    public static void main(String[] args) {
    
        //对整型数组排序
        int[] intArray = {9, 8, 7, 6, 5, 4, 3, 2, 1, 0};
        int operations = new IntBubbleSorter().sort(intArray);
        System.out.println("[Template Method] operations:" + operations + ", array:" + Arrays.toString(intArray));

        //对double数组排序
        double[] doubleArray = {9.9, 8.8, 7.7, 6.6, 5.5, 4.4, 3.3, 2.2, 1.1, 0.0};
        operations = new DoubleBubbleSorter().sort(doubleArray);
        System.out.println("[Template Method] operations:" + operations + ", array:" + Arrays.toString(doubleArray));

        //对List集合排序
        List<Integer> list = Arrays.asList(9, 8, 7, 6, 5, 4, 3, 2, 1, 0);
        operations = new IntegerListBubbleSorter().sort(list);
        System.out.println("[Template Method] operations:" + operations + ", list:" + list.toString());
    }
}
```

模板方法模式展示了经典重用的一种形式，通用算法被放在基类中，通过继承在不同的子类中实现该通用算法。我们通过定义通用类BubbleSorter，把冒泡排序的算法骨架放在基类，然后实现不同的子类分别对int数组、double数组、List集合进行排序。但这样是有代价的，因为继承是非常强的关系，派生类不可避免地与基类绑定在一起了。但如果我现在需要用快速排序而不是冒泡排序来进行排序，但快速排序却没有办法重用`setArray`、`getLength`、`needSwap`和`swap`方法了。不过，策略模式提供了另一种可选的方案

# 策略模式

策略模式属于对象的行为模式。其用意是针对一组算法，将每一个算法封装到具有共同接口的独立的类中，从而使得它们可以相互替换，下面用策略模式对冒泡排序进行重构

下图是用策略模式对冒泡排序重构后的结构图：
![2019-9-2-11-35-32.png](https://gitee.com/hanschencoder/Images/raw/master/2019-9-2-11-35-32.png)

首先定义一个BubbleSorter类，它持有一个抽象策略接口：
```java
public class BubbleSorter<T> {

    /**
     * 抽象策略接口，可以有不同的实现
     */
    private SortHandler<T> sortHandler;

    public BubbleSorter(SortHandler<T> sortHandler) {
        this.sortHandler = sortHandler;
    }

    /**
     * 冒泡排序
     */
    public int sort(T array) {

        sortHandler.setArray(array);
        int length = sortHandler.getLength();

        int operations = 0;
        if (length <= 1) {
            return operations;
        }

        for (int i = 0; i < length - 1; i++) {
            for (int j = 0; j < length - i - 1; j++) {
                operations++;
                if (sortHandler.needSwap(j)) {
                    sortHandler.swap(j);
                }
            }
        }

        return operations;
    }
}
```

定义抽象策略接口：
```java
public interface SortHandler<T> {

    /**
     * 初始化排序数组
     */
    void setArray(T array);

    /**
     * @return 返回数组长度
     */
    int getLength();

    /**
     * @return 是否需要交换数组中 index 和 index+1 元素
     */
    boolean needSwap(int index);

    /**
     * 交换array数组中的 index 和 index+1 元素
     */
    void swap(int index);
}
```

创建具体的策略类`IntSortHandler`对整型数组进行操作：
```java
public class IntSortHandler implements SortHandler<int[]> {

    private int[] array;

    @Override
    public void setArray(int[] array) {
        this.array = array;
    }

    @Override
    public int getLength() {
        return array == null ? 0 : array.length;
    }

    @Override
    public boolean needSwap(int index) {
        return array != null && (array[index] > array[index + 1]);
    }

    @Override
    public void swap(int index) {
        int temp = array[index];
        array[index] = array[index + 1];
        array[index + 1] = temp;
    }
}
```

创建具体的策略类`DoubleSortHandler`对双精度型数组进行操作：
```java
public class DoubleSortHandler implements SortHandler<double[]> {

    private double[] array;

    @Override
    public void setArray(double[] array) {
        this.array = array;
    }

    @Override
    public int getLength() {
        return array == null ? 0 : array.length;
    }

    @Override
    public boolean needSwap(int index) {
        return array != null && (array[index] > array[index + 1]);
    }

    @Override
    public void swap(int index) {
        double temp = array[index];
        array[index] = array[index + 1];
        array[index + 1] = temp;
    }
}
```

创建具体的策略类`IntegerListSortHandler`对List集合进行操作：
```java
public class IntegerListSortHandler implements SortHandler<List<Integer>> {

    private List<Integer> list;

    @Override
    public void setArray(List<Integer> list) {
        this.list = list;
    }

    @Override
    public int getLength() {
        return list == null ? 0 : list.size();
    }

    @Override
    public boolean needSwap(int index) {
        return list != null && (list.get(index) > list.get(index + 1));
    }

    @Override
    public void swap(int index) {
        int temp = list.get(index);
        list.set(index, list.get(index + 1));
        list.set(index + 1, temp);
    }
}
```

定义上述类之后，我们看下怎么使用策略模式
```java
public class Test {

    public static void main(String[] args) {
        //对整型数组排序
        int[] intArray = {9, 8, 7, 6, 5, 4, 3, 2, 1, 0};
        BubbleSorter<int[]> intBubbleSorter = new BubbleSorter<>(new IntSortHandler());
        int operations = intBubbleSorter.sort(intArray);
        System.out.println("[Strategy] operations:" + operations + ", array:" + Arrays.toString(intArray));
        
        //对double数组排序
        double[] doubleArray = {9.9, 8.8, 7.7, 6.6, 5.5, 4.4, 3.3, 2.2, 1.1, 0.0};
        BubbleSorter<double[]> doubleBubbleSorter = new BubbleSorter<>(new DoubleSortHandler());
        operations = doubleBubbleSorter.sort(doubleArray);
        System.out.println("[Strategy] operations:" + operations + ", array:" + Arrays.toString(doubleArray));

        //对List集合排序
        List<Integer> list = Arrays.asList(9, 8, 7, 6, 5, 4, 3, 2, 1, 0);
        BubbleSorter<List<Integer>> integerListBubbleSorter = new BubbleSorter<>(new IntegerListSortHandler());
        operations = integerListBubbleSorter.sort(list);
        System.out.println("[Strategy] operations:" + operations + ", list:" + list);
    }
}
```

策略模式不是将通用方法放到基类中，而是把它放进`BubbleSorter`的sort方法中，把排序算法中必须调用的抽象方法定义在`SortHandler`接口中，从这个接口中派生出不同的子类。把派生出的子类传给BubbleSorter后，sort方法就可以把具体工作委托给接口去完成。注意：SortHandler对BubbleSorter是一无所知的，它不依赖于冒泡排序的具体实现，这个和模板方法模式是不同的。如果其他排序算法也需要用到SortHandler，完全也可以在相关的排序算法中使用SortHandler


# 总结
模板方法模式和策略模式都可以用来分离高层的算法和低层的具体实现细节，都允许高层的算法独立于它的具体实现细节重用。但策略模式还有一个额外的好处就是允许具体实现细节独立于高层的算法重用，但这也以一些额外的复杂性、内存以及运行事件开销作为代价


[文中示例代码下载：https://github.com/shensky711/awesome-demo/tree/master/Patterns](https://github.com/shensky711/awesome-demo/tree/master/Patterns)
