---
layout: post
title: 设计模式之桥接模式
tags: [设计模式,桥接模式]
categories: 
- 设计模式
---

# 场景问题

## 发送消息

现在我们要实现这样一个功能：发送消息。从业务上看，消息又分成普通消息、加急消息和特急消息多种，不同的消息类型，业务功能处理是不一样的，比如加急消息是在消息上添加“加急”字样，而特急消息除了添加特急外，还会做一条催促的记录，多久不完成会继续催促。从发送消息的手段上看，又有系统内短消息、手机短消息、邮件等等。现在要实现这样的发送提示消息的功能，该如何实现呢？

<!-- more -->
## 不用模式的解决方案

### 实现简化版本

先实现一个简单点的版本：消息只是实现发送普通消息，发送的方式先实现系统内短消息和邮件。其它的功能，等这个版本完成过后，再继续添加，这样先把问题简单化，实现起来会容易一点。由于发送普通消息会有两种不同的实现方式，为了让外部能统一操作，因此，把消息设计成接口，然后由两个不同的实现类，分别实现系统内短消息方式和邮件发送消息的方式。此时系统结构如下：
![2019-9-2-12-30-41.png](http://image.hanschen.site/master/2019-9-2-12-30-41.png)

先来看看消息的统一接口，示例代码如下：
```java
public interface Message {

    /**
     * 发送消息
     *
     * @param message 要发送的消息内容
     * @param toUser  消息发送的目的人员
     */
    void send(String message, String toUser);
}
```

再来分别看看两种实现方式，这里只是为了示意，并不会真的去发送Email和站内短消息，先看站内短消息的方式，示例代码如下：
```java
public class CommonMessageSMS implements Message {

    @Override
    public void send(String message, String toUser) {
        System.out.println("使用站内短消息的方式，发送消息'" + message + "'给" + toUser);
    }
}
```

同样的，实现以Email的方式发送普通消息，示例代码如下：
```java
public class CommonMessageEmail implements Message {

    @Override
    public void send(String message, String toUser) {
        System.out.println("使用Email的方式，发送消息'" + message + "'给" + toUser);
    }
}
```

### 实现发送加急消息
上面的实现，看起来很简单，对不对。接下来，添加发送加急消息的功能，也有两种发送的方式，同样是站内短消息和Email的方式。
加急消息的实现跟普通消息不同，加急消息会自动在消息上添加加急，然后再发送消息；另外加急消息会提供监控的方法，让客户端可以随时通过这个方法来了解对于加急消息处理的进度，比如：相应的人员是否接收到这个信息，相应的工作是否已经开展等等。因此加急消息需要扩展出一个新的接口，除了基本的发送消息的功能，还需要添加监控的功能，这个时候，系统的结构如图所示：
![2019-9-2-12-31-39.png](http://image.hanschen.site/master/2019-9-2-12-31-39.png)

先看看扩展出来的加急消息的接口，示例代码如下：
```java
public interface UrgencyMessage extends Message {

    /**
     * 监控某消息的处理过程
     *
     * @param messageId 被监控的消息的编号
     * @return 包含监控到的数据对象，这里示意一下，所以用了Object
     */
    Object watch(String messageId);
}
```

相应的实现方式还是发送站内短消息和Email两种，同样需要两个实现类来分别实现这两种方式，先看站内短消息的方式，示例代码如下：
```java
public class UrgencyMessageSMS implements UrgencyMessage {

    @Override
    public void send(String message, String toUser) {
        message = "加急：" + message;
        System.out.println("使用站内短消息的方式，发送消息'" + message + "'给" + toUser);
    }

    @Override
    public Object watch(String messageId) {
        //获取相应的数据，组织成监控的数据对象，然后返回
        return null;
    }
}
```

再看看Emai的方式，示例代码如下：
```java
public class UrgencyMessageEmail implements UrgencyMessage {

    @Override
    public void send(String message, String toUser) {
        message = "加急：" + message;
        System.out.println("使用Email的方式，发送消息'" + message + "'给" + toUser);
    }

    @Override
    public Object watch(String messageId) {
        //获取相应的数据，组织成监控的数据对象，然后返回
        return null;
    }
}
```

事实上，在实现加急消息发送的功能上，可能会使用前面发送不同消息的功能，也就是让实现加急消息处理的对象继承普通消息的相应实现，这里为了让结构简单一点，清晰一点，所以没有这样做。

## 有何问题
上面这样实现，好像也能满足基本的功能要求，可是这么实现好不好呢？有没有什么问题呢？
我们继续向下来添加功能实现，为了简洁，就不再去进行代码示意了，通过实现的结构示意图就可以看出实现上的问题。

### 继续添加特急消息的处理
特急消息不需要查看处理进程，只要没有完成，就直接催促，也就是说，对于特急消息，在普通消息的处理基础上，需要添加催促的功能。而特急消息、还有催促的发送方式，相应的实现方式还是发送站内短消息和Email两种，此时系统的结构如图所示：
![2019-9-2-12-32-9.png](http://image.hanschen.site/master/2019-9-2-12-32-9.png)

仔细观察上面的系统结构示意图，会发现一个很明显的问题，那就是：通过这种继承的方式来扩展消息处理，会非常不方便。
你看，实现加急消息处理的时候，必须实现站内短消息和Email两种处理方式，因为业务处理可能不同；在实现特急消息处理的时候，又必须实现站内短消息和Email这两种处理方式。
这意味着，以后每次扩展一下消息处理，都必须要实现这两种处理方式，是不是很痛苦，这还不算完，如果要添加新的实现方式呢？继续向下看吧。

### 继续添加发送手机消息的处理方式

如果看到上面的实现，你还感觉问题不是很大的话，继续完成功能，添加发送手机消息的处理方式
仔细观察现在的实现，如果要添加一种新的发送消息的方式，是需要在每一种抽象的具体实现里面，都要添加发送手机消息的处理的。也就是说：发送普通消息、加急消息和特急消息的处理，都可以通过手机来发送。这就意味着，需要添加三个实现。此时系统结构如图所示：
![2019-9-2-12-32-31.png](http://image.hanschen.site/master/2019-9-2-12-32-31.png)
 
这下能体会到这种实现方式的大问题了吧。

### 小结一下出现的问题

采用通过继承来扩展的实现方式，有个明显的缺点：扩展消息的种类不太容易，不同种类的消息具有不同的业务，也就是有不同的实现，在这种情况下，每个种类的消息，需要实现所有不同的消息发送方式。
更可怕的是，如果要新加入一种消息的发送方式，那么会要求所有的消息种类，都要加入这种新的发送方式的实现。
要是考虑业务功能上再扩展一下呢？比如：要求实现群发消息，也就是一次可以发送多条消息，这就意味着很多地方都得修改，太恐怖了。
那么究竟该如何实现才能既实现功能，又能灵活的扩展呢？

# 解决方案
## 桥接模式来解决

用来解决上述问题的一个合理的解决方案，就是使用桥接模式。那么什么是桥接模式呢？
桥接模式定义：
> 将抽象部分和实现部分分离，使它们都可以独立地变化

应用桥接模式来解决的思路

仔细分析上面的示例，根据示例的功能要求，示例的变化具有两个维度，一个维度是抽象的消息这边，包括普通消息、加急消息和特急消息，这几个抽象的消息本身就具有一定的关系，加急消息和特急消息会扩展普通消息；另一个维度在具体的消息发送方式上，包括站内短消息、Email和手机短信息，这几个方式是平等的，可被切换的方式。这两个维度一共可以组合出9种不同的可能性来。
现在出现问题的根本原因，就在于消息的抽象和实现是混杂在一起的，这就导致了，一个维度的变化，会引起另一个维度进行相应的变化，从而使得程序扩展起来非常困难。
要想解决这个问题，就必须把这两个维度分开，也就是将抽象部分和实现部分分开，让它们相互独立，这样就可以实现独立的变化，使扩展变得简单。
桥接模式通过引入实现的接口，把实现部分从系统中分离出去；那么，抽象这边如何使用具体的实现呢？肯定是面向实现的接口来编程了，为了让抽象这边能够很方便的与实现结合起来，把顶层的抽象接口改成抽象类，在里面持有一个具体的实现部分的实例。
这样一来，对于需要发送消息的客户端而言，就只需要创建相应的消息对象，然后调用这个消息对象的方法就可以了，这个消息对象会调用持有的真正的消息发送方式来把消息发送出去。也就是说客户端只是想要发送消息而已，并不想关心具体如何发送。

## 模式结构和说明
桥接模式的结构图：
![2019-9-2-12-32-50.png](http://image.hanschen.site/master/2019-9-2-12-32-50.png)

 - Abstraction：抽象部分的接口。通常在这个对象里面，要维护一个实现部分的对象引用，在抽象对象里面的方法，需要调用实现部分的对象来完成。这个对象里面的方法，通常都是跟具体的业务相关的方法。
 - RefinedAbstraction：扩展抽象部分的接口，通常在这些对象里面，定义跟实际业务相关的方法，这些方法的实现通常会使用Abstraction中定义的方法，也可能需要调用实现部分的对象来完成。
 - Implementor：定义实现部分的接口，这个接口不用和Abstraction里面的方法一致，通常是由Implementor接口提供基本的操作，而Abstraction里面定义的是基于这些基本操作的业务方法，也就是说Abstraction定义了基于这些基本操作的较高层次的操作。
 - ConcreteImplementor：真正实现Implementor接口的对象。

## 桥接模式示例代码
先看看Implementor接口的定义，示例代码如下：
```java
public interface Implementor {

    void operationImpl();
}
```

再看看Abstraction接口的定义，注意一点，虽然说是接口定义，但其实是实现成为抽象类。示例代码如下：
```java
public abstract class Abstraction {

    /**
     * 持有一个实现部分的对象
     */
    protected Implementor impl;

    /**
     * 构造方法，传入实现部分的对象
     *
     * @param impl 实现部分的对象
     */
    public Abstraction(Implementor impl) {
        this.impl = impl;
    }

    public void operation() {
        impl.operationImpl();
    }
}
```

该来看看具体的实现了，示例代码如下：
```java
public class ConcreteImplementorA implements Implementor {

    public void operationImpl() {
        //真正的实现
    }
}
```

另外一个实现，示例代码如下：
```java
public class ConcreteImplementorB implements Implementor {

    public void operationImpl() {
        //真正的实现
    }
}
```

最后来看看扩展Abstraction接口的对象实现，示例代码如下：
```java
public class RefinedAbstraction extends Abstraction {

    public RefinedAbstraction(Implementor impl) {
        super(impl);
    }

    /**
     * 示例操作，实现一定的功能
     */
    public void otherOperation() {

        //实现一定的功能，可能会使用具体实现部分的实现方法，
        //但是本方法更大的可能是使用Abstraction中定义的方法，
        //通过组合使用Abstraction中定义的方法来完成更多的功能
    }
}
```

## 使用桥接模式重写示例

学习了桥接模式的基础知识过后，该来使用桥接模式重写前面的示例了。通过示例，来看看使用桥接模式来实现同样的功能，是否能解决“既能方便的实现功能，又能有很好的扩展性”的问题。
要使用桥接模式来重新实现前面的示例，首要任务就是要把抽象部分和实现部分分离出来，分析要实现的功能，抽象部分就是各个消息的类型所对应的功能，而实现部分就是各种发送消息的方式。
其次要按照桥接模式的结构，给抽象部分和实现部分分别定义接口，然后分别实现它们就可以了。

### 从简单功能开始
从相对简单的功能开始，先实现普通消息和加急消息的功能，发送方式先实现站内短消息和Email这两种。使用桥接模式来实现这些功能的程序结构如图所示
![2019-9-2-12-33-19.png](http://image.hanschen.site/master/2019-9-2-12-33-19.png)

还是看看代码实现，会更清楚一些。先看看消息发送器接口，示例代码如下：
```java
/**
 * 消息发送器
 *
 * @author HansChen
 */
public interface MessageSender {

    /**
     * 发送消息
     *
     * @param message 要发送的消息内容
     * @param toUser  消息发送的目的人员
     */
    void send(String message, String toUser);
}
```

再看看抽象部分定义的接口，示例代码如下：
```java
/**
 * 抽象的消息对象
 *
 * @author HansChen
 */
public class AbstractMessageController {

    /**
     * 持有一个实现部分的对象
     */
    MessageSender impl;

    /**
     * 构造方法，传入实现部分的对象
     *
     * @param impl 实现部分的对象
     */
    AbstractMessageController(MessageSender impl) {
        this.impl = impl;
    }

    /**
     * 发送消息，转调实现部分的方法
     *
     * @param message 要发送的消息内容
     * @param toUser  消息发送的目的人员
     */
    protected void sendMessage(String message, String toUser) {
        impl.send(message, toUser);
    }
}
```

看看如何具体的实现发送消息，先看站内短消息的实现吧，示例代码如下：
```java
/**
 * 以站内短消息的方式发送消息
 *
 * @author HansChen
 */
public class MessageSenderSMS implements MessageSender {

    @Override
    public void send(String message, String toUser) {
        System.out.println("使用站内短消息的方式，发送消息'" + message + "'给" + toUser);
    }
}

```

再看看Email方式的实现，示例代码如下：
```java
/**
 * 以Email的方式发送消息
 *
 * @author HansChen
 */
public class MessageSenderEmail implements MessageSender {

    @Override
    public void send(String message, String toUser) {
        System.out.println("使用Email的方式，发送消息'" + message + "'给" + toUser);
    }
}

```

接下来该看看如何扩展抽象的消息接口了，先看普通消息的实现，示例代码如下：
```java
public class CommonMessageController extends AbstractMessageController {

    public CommonMessageController(MessageSender impl) {
        super(impl);
    }

    @Override
    public void sendMessage(String message, String toUser) {
        //对于普通消息，什么都不干，直接调父类的方法，把消息发送出去就可以了
        super.sendMessage(message, toUser);
    }
}
```

再看看加急消息的实现，示例代码如下：
```java
public class UrgencyMessageController extends AbstractMessageController {

    public UrgencyMessageController(MessageSender impl) {
        super(impl);
    }

    @Override
    protected void sendMessage(String message, String toUser) {
        message = "加急：" + message;
        super.sendMessage(message, toUser);
    }

    /**
     * 扩展自己的新功能：监控某消息的处理过程
     *
     * @param messageId 被监控的消息的编号
     * @return 包含监控到的数据对象，这里示意一下，所以用了Object
     */
    public Object watch(String messageId) {
        //获取相应的数据，组织成监控的数据对象，然后返回
        return null;
    }
}
```

### 添加功能

看了上面的实现，发现使用桥接模式来实现也不是很困难啊，关键得看是否能解决前面提出的问题，那就来添加还未实现的功能看看，添加对特急消息的处理，同时添加一个使用手机发送消息的方式。该怎么实现呢？
很简单，只需要在抽象部分再添加一个特急消息的类，扩展抽象消息就可以把特急消息的处理功能加入到系统中了；对于添加手机发送消息的方式也很简单，在实现部分新增加一个实现类，实现用手机发送消息的方式，也就可以了。
这么简单？好像看起来完全没有了前面所提到的问题。的确如此，采用桥接模式来实现过后，抽象部分和实现部分分离开了，可以相互独立的变化，而不会相互影响。因此在抽象部分添加新的消息处理，对发送消息的实现部分是没有影响的；反过来增加发送消息的方式，对消息处理部分也是没有影响的。

接着看看代码实现，先看看新的特急消息的处理类，示例代码如下：
```java
public class SpecialUrgencyMessageController extends AbstractMessageController {

    public SpecialUrgencyMessageController(MessageSender impl) {
        super(impl);
    }

    @Override
    protected void sendMessage(String message, String toUser) {
        message = "特急：" + message;
        super.sendMessage(message, toUser);
    }

    public void hurry(String messageId) {
        //执行催促的业务，发出催促的信息
    }
}
```

再看看使用手机短消息的方式发送消息的实现，示例代码如下：
```java
public class MessageSenderMobile implements MessageSender {

    @Override
    public void send(String message, String toUser) {
        System.out.println("使用手机的方式，发送消息'" + message + "'给" + toUser);
    }
}
```

### 测试一下功能
看了上面的实现，可能会感觉得到，使用桥接模式来实现前面的示例过后，添加新的消息处理，或者是新的消息发送方式是如此简单，可是这样实现，好用吗？写个客户端来测试和体会一下，示例代码如下：
```java
public class Client {

    public static void main(String[] args) {
        //创建具体的实现对象
        MessageSender impl = new MessageSenderSMS();

        //创建一个普通消息对象
        AbstractMessageController controller = new CommonMessageController(impl);
        controller.sendMessage("请喝一杯茶", "小李");

        //创建一个紧急消息对象
        controller = new UrgencyMessageController(impl);
        controller.sendMessage("请喝一杯茶", "小李");

        //创建一个特急消息对象
        controller = new SpecialUrgencyMessageController(impl);
        controller.sendMessage("请喝一杯茶", "小李");


        //把实现方式切换成手机短消息，然后再实现一遍
        impl = new MessageSenderMobile();
        controller = new CommonMessageController(impl);
        controller.sendMessage("请喝一杯茶", "小李");

        controller = new UrgencyMessageController(impl);
        controller.sendMessage("请喝一杯茶", "小李");

        controller = new SpecialUrgencyMessageController(impl);
        controller.sendMessage("请喝一杯茶", "小李");
    }
}
```

运行结果如下：
```
使用站内短消息的方式，发送消息'请喝一杯茶'给小李
使用站内短消息的方式，发送消息'加急：请喝一杯茶'给小李
使用站内短消息的方式，发送消息'特急：请喝一杯茶'给小李
使用手机的方式，发送消息'请喝一杯茶'给小李
使用手机的方式，发送消息'加急：请喝一杯茶'给小李
使用手机的方式，发送消息'特急：请喝一杯茶'给小李
```

前面三条是使用的站内短消息，后面三条是使用的手机短消息，正确的实现了预期的功能。看来前面的实现应该是正确的，能够完成功能，且能灵活扩展。

# 广义桥接-Java中无处不桥接
使用Java编写程序，一个很重要的原则就是“面向接口编程”，说得准确点应该是“面向抽象编程”，由于在Java开发中，更多的使用接口而非抽象类，因此通常就说成“面向接口编程”了。接口把具体的实现和使用接口的客户程序分离开来，从而使得具体的实现和使用接口的客户程序可以分别扩展，而不会相互影响。

桥接模式中的抽象部分持有具体实现部分的接口，最终目的是什么，还不是需要通过调用具体实现部分的接口中的方法，来完成一定的功能，这跟直接使用接口没有什么不同，只是表现形式有点不一样。再说，前面那个使用接口的客户程序也可以持有相应的接口对象，这样从形式上就一样了。

也就是说，从某个角度来讲，桥接模式不过就是对“面向抽象编程”这个设计原则的扩展。正是通过具体实现的接口，把抽象部分和具体的实现分离开来，抽象部分相当于是使用实现部分接口的客户程序，这样抽象部分和实现部分就松散耦合了，从而可以实现相互独立的变化。

这样一来，几乎可以把所有面向抽象编写的程序，都视作是桥接模式的体现，至少算是简化的桥接模式，就算是广义的桥接吧。而Java编程很强调“面向抽象编程”，因此，广义的桥接，在Java中可以说是无处不在。

# 桥接模式在Android中的应用
如果各位童鞋看到这里仍然对桥接模式还是不太清楚，在这里给大家举个在Android中非常常用的桥接模式栗子：`AbsListView`与`ListAdapter`之间的桥接模式。童鞋们可以根据这个栗子体会一下桥接模式的好处。