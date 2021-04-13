---
layout: post
title: 设计模式之工厂模式（Factory）
tags: [设计模式,工厂模式,Factory]
categories: 
- 设计模式
---

# 概述
根据`依赖倒置原则`，我们知道，我们应优先依赖抽象类而不是具体类。在应用开发过程中，有很多实体类都是非常易变的，依赖它们会带来问题，所以我们更应该依赖于抽象接口，已使我们免受大多数变化的影响。
`工厂模式（Factory）`允许我们只依赖于抽象接口就能创建出具体对象的实例，所以在开发中，如果具体类是高度易变的，那么该模式就非常有用。

接下来我们就通过代码举例说明什么是工厂模式

<!-- more -->

# 简单工厂模式

假设我们现在有个需求：把一段数据用Wi-Fi或者蓝牙发送出去。
需求很简单是吧？刷刷刷就写下了以下实现：
```java
    private String mode; //Wi-Fi|Bluetooth

    public void onClick() {
        byte[] data = {0x00, 0x01};

        if ("Wi-Fi".equals(mode)) {
            sendDataByWiFi(data);
        } else {
            sendDataByBluetooth(data);
        }
    }

    private void sendDataByWiFi(byte[] data) {
        // send data via Wi-Fi
    }

    private void sendDataByBluetooth(byte[] data) {
        // send data via Bluetooth
    }
```
但是上面的代码扩展性并不高，违反了开放封闭原则。比如现在又有了个新的需求，需要用zigbee把数据发送出去，就得再新增一个sendDataByZigbee方法了，而且还得修改onClick里面的逻辑。那么比较好的方法是怎么样的呢？

定义一个数据发送器类：
```java
/**
 * 数据发送器Sender
 *
 * @author HansChen
 */
public interface Sender {

    void sendData(byte[] data);
}
```

实现WiFi数据发送：
```java
/**
 * Sender的实现类，通过Wi-Fi发送数据
 *
 * @author HansChen
 */
public class WiFiSender implements Sender {

    @Override
    public void sendData(byte[] data) {
        System.out.println("Send data by Wi-Fi");
    }
}
```

实现蓝牙数据发送：
```java
/**
 * Sender的实现类，通过蓝牙发送数据
 *
 * @author HansChen
 */
public class BluetoothSender implements Sender {

    @Override
    public void sendData(byte[] data) {
        System.out.println("Send data by Bluetooth");
    }
}
```

这样，原来发送数据的地方就改为了：
```java
    private String mode; //Wi-Fi|Bluetooth

    public void onClick() {
        byte[] data = {0x00, 0x01};

        Sender sender;
        if ("Wi-Fi".equals(mode)) {
            sender = new WiFiSender();
        } else {
            sender = new BluetoothSender();
        }
        sender.sendData(data);
    }
```
有没有觉得代码优雅了一点？但是随着发送器Sender的实现类越来越多，每增加一个实现类，就需要在onClick里面实例化相应的实现类，能不能用一个单独的类来做这个创造实例的过程呢？这就是我们讲到的工厂。我们新增一个工厂类：
```java
/**
 * 简单工厂类
 *
 * @author HansChen
 */
public class SimpleFactory {

    public static Sender createSender(String mode) {
        switch (mode) {
            case "Wi-Fi":
                return new WiFiSender();
            case "Bluetooth":
                return new BluetoothSender();
            default:
                throw new IllegalArgumentException("illegal type: " + mode);
        }
    }
}
```

这样一来，怎么实例化数据发送器我们也不用管了，最终代码变为：
```java
    private String mode; //Wi-Fi|Bluetooth

    public void onClick() {
        byte[] data = {0x00, 0x01};

        Sender sender = SimpleFactory.createSender(mode);
        sender.sendData(data);
    }
```

好了，到这里我们就完成了简单工厂模式的应用了，下图就是简单工厂模式的结构图：
![2019-9-2-11-28-47.png](https://gitee.com/hanschencoder/Images/raw/master/2019-9-2-11-28-47.png)

# 工厂方法模式
简单工厂模式的优点在于工厂类包含了必要的判断逻辑，根据传入的参数动态实例化相关的类，对于客户端来说，去除了与具体产品的依赖。但是这里还是会有个问题，假设上面例子中新增了一个zigbee发送器，那么一定是需要修改简单工厂类的，也就是说，我们不但对扩展开放了，对修改也开放了，这是不好的。解决的方法是使用工厂方法模式，工厂方法模式是指**定义一个用于创建对象的接口，让子类决定实例化哪一个类**。下面还是通过代码来说明：

在简单工厂模式的基础上，让我们对工厂类也升级一下，首先定义一个工厂类接口：
```java
public interface SenderFactory {

    Sender createSender();
}
```

然后为每一个发送器的实现类各创建一个具体的工厂方法去实现这个接口

定义WiFiSender的工厂类：
```java
public class WiFiSenderFactory implements SenderFactory {

    @Override
    public Sender createSender() {
        return new WiFiSender();
    }
}
```

定义BluetoothSender的工厂类：
```java
public class BluetoothSenderFactory implements SenderFactory {

    @Override
    public Sender createSender() {
        return new BluetoothSender();
    }
}
```

这样，即使有新的Sender实现类加进来，我们只需要新增相应的工厂类就行了，不需要修改原有的工厂，下图就是工厂方法模式的结构图：
![2019-9-2-11-28-17.png](https://gitee.com/hanschencoder/Images/raw/master/2019-9-2-11-28-17.png)

客户端调用代码：
```java
    private String mode; //Wi-Fi|Bluetooth

    public void onClick() {
        byte[] data = {0x00, 0x01};

        SenderFactory factory;
        if ("Wi-Fi".equals(mode)) {
            factory = new WiFiSenderFactory();
        } else {
            factory = new BluetoothSenderFactory();
        }
        Sender sender = factory.createSender();
        sender.sendData(data);
    }
```
细心的读者可能已经发现了，工厂方法模式实现时，客户端需要决定实例化哪一个工厂类，相比于简单工厂模式，客户端多了一个选择判断的问题，也就是说，**工厂方法模式把简单工厂模式的内部逻辑判断移到了客户端！**你想要加功能，本来是修改简单工厂类的，现在改为修改客户端。但是这样带来的好处是整个工厂和产品体系都没有“修改”的变化，只有“扩展”的变化，完全符合了开放封闭原则。

# 总结
简单工厂模式和工厂方法模式都封装了对象的创建，它们使得高层策略模块在创建类的实例时无需依赖于这些类的具体实现。但是两种工厂模式之间又有差异：

 - 简单工厂模式：最大的优点在于工厂类包含了必要的判断逻辑，根据客户端的条件动态地实例化相关的类。但这也是它的缺点，当扩展功能的时候，需要修改工厂方法，违反了开放封闭原则
 - 工厂方法模式：符合开放封闭原则，但这带来的代价是扩展的时候要增加相应的工厂类，增加了开发量，而且需要修改客户端代码