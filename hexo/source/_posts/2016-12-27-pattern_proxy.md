---
layout: post
title: 设计模式之代理模式
tags: [设计模式,代理模式]
categories: 
- 设计模式
---

# 概述
我们执行一个功能的函数时，经常需要在其中写入与功能不是直接相关但很有必要的代码，如日志记录、信息发送、安全和事务支持等，这些`枝节性代码`虽然是必要的，但它会带来以下麻烦：

 - 枝节性代码游离在功能性代码之外，它下是函数的目的
 - 枝节性代码会造成功能性代码对其它类的依赖，加深类之间的耦合
 - 枝节性代码带来的耦合度会造成功能性代码移植困难，可重用性降低

毫无疑问，枝节性代码和功能性代码需要分开来才能降低耦合程度，我们可以使用**代理模式(委托模式)**完成这个要求。代理模式的作用是：为其它对象提供一种代理以控制对这个对象的访问。在某些情况下，一 个客户不想直接引用另一个对象，而代理对象可以在客户端和目标对象之间起到中介作用。

代理模式一般涉及到三个角色：

 - **抽象角色**：声明真实对象和代理对象的共同接口
 - **代理角色**：代理对象内部包含有真实角色的引用，从而可以操作真实角色，同时代理对象 与真实对象有相同的接口，能在任何时候代替真实对象，代理对象可以在执行真实对 象前后加入特定的逻辑以实现功能的扩展。
 - **真实角色**：代理角色所代表的真实对象，是我们最终要引用的对象

常见的代理应用场景有：

 - **远程代理**：对一个位于不同的地址空间对象提供一个局域代表对象，如RMI中的stub
 - **虚拟代理**：根据需要将一个资源消耗很大或者比较复杂的对象，延迟加载，在真正需要的时候才创建
 - **保护代理**：控制对一个对象的访问权限
 - **智能引用**：提供比目标对象额外的服务和功能


接下来，我们用代码来说明什么是代理模式

<!-- more -->

# 代理模式
## UML图
先看看代理模式的结构图：
![2019-9-2-12-29-26.png](http://image.hanschen.site/master/2019-9-2-12-29-26.png)

## 代码
下面给出一个小栗子说明代理模式，先定义一个抽象角色，也就是一个公共接口，声明一些需要代理的方法,本文定义一个`Subject`接口，为了简单说明，只是在里面定义一个request方法：
```java
public interface Subject {

    void request();
}
```
定义Subject的实现类`RealSubject`，它是一个真实角色：
```java
public class RealSubject implements Subject {

    @Override
    public void request() {
        System.out.print("do real request");
    }
}
```
定义一个代理角色`ProxySubject`，跟RealSubject一样，它也继承了Subject接口：
```java
public class ProxySubject implements Subject {

    private RealSubject mSubject;

    public ProxySubject() {
        mSubject = new RealSubject();
    }

    @Override
    public void request() {
        System.out.print("before");
        mSubject.request();
        System.out.print("after");
    }
}
```
客户端调用代码
```java
public class Client {

    public static void main(String[] args) {

        Subject subject = new ProxySubject();
        subject.request();
    }
}
```

这样，一个简易的代理模式模型就建立了，客户端在使用过程中，无需关注RealSubject，只需要关注ProxySubject就行了，并且可以在ProxySubject中插入一些非功能信的代码，比如输出Log，统计执行时间等等

## 远程代理
远程代理，对一个位于不同的地址空间对象提供一个局域代表对象。这样说大家可能比较抽象，不太能理解，但其实童鞋们可能在就接触过了，在Android中，Binder的使用就是典型的远程代理。比如ActivityManager：
![2019-9-2-12-29-57.png](http://image.hanschen.site/master/2019-9-2-12-29-57.png)

在启动Activity的时，会调用`ActivityManager`的startActivity方法，我们看看Activity是怎么获取的：
```java
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        // 返回代理类
        return new ActivityManagerProxy(obj);
    }
```
可以看到，最终是返回了一个ActivityManager的代理类，因为真正的ActivityManager是运行在内核空间的，Android应用无法直接访问得到，那么就可以借助这个ActivityManagerProxy，通过Binder与真正的ActivityManager，也就是`ActivityManagerService`交互。其中ActivityManagerService和ActivityManagerProxy都实现了同一个接口：`IActivityManager`。这个就是Android中典型的代理模式的栗子了。至于ActivityManagerService和ActivityManagerProxy是如何通过Binder实现远程调用，这个就是另一个话题Binder的内容了，这里不再做阐述

## 延迟加载
根据需要将一个资源消耗很大或者比较复杂的对象，延迟加载，在真正需要的时候才创建。假设我们创建RealSubject需要耗费一定的资源，那么，我们可以把创建它延迟到实际调用的时候，优化Client初始化速度，比如，这样修改ProxySubject以达到延迟加载:
```java
public class ProxySubject implements Subject {

    private RealSubject mSubject;

    public ProxySubject() {
    }

    @Override
    public void request() {
        // 延时加载
        if (mSubject == null) {
            mSubject = new RealSubject();
        }
        mSubject.request();
    }
}
```
Client在实例化ProxySubject的时候，不需消耗资源，而是等到真正调用request的时候，才会加载RealSubject，达到延时加载的效果

## 保护代理
可以在Proxy类中加入进行权限，验证是否具有执行真实代码的权限，只有权限验证通过了才进行真实对象的调用
```java
public class ProxySubject implements Subject {

    private RealSubject mSubject;
    private User        mUser;

    public ProxySubject(User user) {
        this.mUser = user;
    }

    @Override
    public void request() {
        // 验证权限
        if (mUser.isLogin()) {
            mSubject.request();
        }
    }
}
```

## 额外功能
通过引入代理类，可以方便地在功能性代码前后插入扩展，如Log输出，调用统计等，实现对原代码的**无侵入**式代码扩展，如：
```java
public class ProxySubject implements Subject {

    private RealSubject mSubject;

    public ProxySubject() {
        mSubject = new RealSubject();
    }

    @Override
    public void request() {
        System.out.print("Log: before");
        mSubject.request();
        System.out.print("Log: after");
    }
}
```

# 静态代理和动态代理
静态代理和动态代理的概念和使用可以参考我另一篇文章：[Java动态代理：http://blog.csdn.net/shensky711/article/details/52872249][1]



  [1]: http://blog.csdn.net/shensky711/article/details/52872249