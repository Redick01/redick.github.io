# 观察者模式和发布-订阅模式 <!-- {docsify-ignore-all} -->


## 前言

&nbsp; &nbsp; 观察者模式是23中设计模式中的一种，而发布订阅模式则不是，是一种程序设计，相关理解参考博客[理解【观察者模式】和【发布订阅】的区别](https://juejin.cn/post/6978728619782701087)，下面针对两种模式使用java简单实现


## 观察者模式

&nbsp; &nbsp; 观察者模式主要有两个角色，一个是观察者，一个是被观察者，被观察者是核心，提供注册观察者，移除观察者能力，观察者主要提供一个观察方法。实现如下：

#### 观察者代码

- **观察者接口**

```java
public interface IWatcher {

    /**
     * 观者者通知
     * @param msg
     */
    void notify(String msg);
}
```

- **观察者实现**

```java
public class DefaultWatcher implements IWatcher {

    private final String name;

    public DefaultWatcher(String name) {
        this.name = name;
    }

    @Override
    public void notify(String msg) {
        System.out.println("观察者" + name + "收到消息：" + msg);
    }
}
```

#### 被观察者代码

- **被观察者接口**

```java
public interface ISubject {

    /**
     * 注册观察者
     * @param iWatcher
     */
    void register(IWatcher iWatcher);

    /**
     * 移除观察者
     * @param iWatcher
     */
    void remove(IWatcher iWatcher);

    /**
     * 通知观察者
     * @param msg
     */
    void notify(String msg);
}
```

- **被观察者实现**

```java
public class DefaultSubject implements ISubject {

    /**
     * 观察者列表
     */
    private final List<IWatcher> watcherList = new ArrayList<>(16);

    @Override
    public void register(IWatcher iWatcher) {
        watcherList.add(iWatcher);
    }

    @Override
    public void remove(IWatcher iWatcher) {
        watcherList.remove(iWatcher);
    }

    @Override
    public void notify(String msg) {
        watcherList.forEach(e -> e.notify(msg));
    }
}
```

#### 测试

```java
public class Main {

    public static void main(String[] args) {

        ISubject iSubject = new DefaultSubject();
        IWatcher iWatcher1 = new DefaultWatcher("观察者1");
        IWatcher iWatcher2 = new DefaultWatcher("观察者2");

        iSubject.register(iWatcher1);
        iSubject.register(iWatcher2);
        iSubject.notify("元宵节不加班");
    }
}
```

- 测试结果

```
观察者观察者1收到消息：元宵节不加班
观察者观察者2收到消息：元宵节不加班
```

## 发布-订阅模式模式

&nbsp; &nbsp; 发布-订阅模式相比于观察者模式，多了一个“发布订阅中心”角色，在观察者模式中，核心是被观察者，而在发布-订阅模式中，核心是发布订阅中心，下面是一个简单的实现。

#### 事件发布者

- **事件发布者接口**

```java
public interface IPublish<T> {

    /**
     * 事件发布
     * @param msg msg
     */
    void publish(SubscribePublish subscribePublish, String topic, T msg);
}
```

- **事件发布者实现**

```java
public class DefaultPublisher<T> implements IPublish<T> {

    private final String name;

    public DefaultPublisher(String name) {
        this.name = name;
    }

    @Override
    public void publish(SubscribePublish subscribePublish, String topic, T msg) {
        System.out.println(name + "发布消息到" + topic);
        subscribePublish.publish(this.name, topic, msg);
    }
}
```

#### 事件订阅者

- **订阅者接口**

```java
public interface ISubscriber<T> {

    /**
     * 订阅
     * @param subscribePublish
     */
    void subscribe(SubscribePublish subscribePublish, String topic);

    /**
     * 取消订阅
     * @param subscribePublish
     */
    void unSubscribe(SubscribePublish subscribePublish, String topic);

    /**
     * @param publisher
     * @param m
     */
    void notify(String publisher, T m);
}
```

- **订阅者实现**

```java
public class DefaultSubscriber<T> implements ISubscriber<T> {

    public String name;

    public DefaultSubscriber(String name) {
        this.name = name;
    }

    @Override
    public void subscribe(SubscribePublish subscribePublish, String topic) {
        System.out.println(name + "订阅" + topic);
        subscribePublish.subscribe(this, topic);
    }

    @Override
    public void unSubscribe(SubscribePublish subscribePublish, String topic) {
        System.out.println(name + "取消订阅" + topic);
        subscribePublish.unSubscribe(this, topic);
    }

    @Override
    public void notify(String publisher, T m) {
        System.out.println(this.name + "消费" + publisher + "发布的消息:" + m.toString());
    }
}
```

#### 发布订阅中心

发布订阅中心是发布订阅模式实现的核心，提供一个根据根据主题管理订阅者的容器，实现了根据主题发布订阅能力

```java
public class SubscribePublish<T> {

    /**
     * 根据主题订阅
     */
    private Map<String, List<ISubscriber>> map = new ConcurrentHashMap<>(1024);

    /**
     * 发布者发布事件
     * @param publisher
     * @param message
     */
    public void publish(String publisher, String topic, T message) {
        notify(publisher, topic, message);
    }

    /**
     * 订阅者订阅，加入订阅者列表
     */
    public void subscribe(ISubscriber subscriber, String topic) {
        map.computeIfAbsent(topic, key -> new ArrayList<>()).add(subscriber);
    }

    /**
     * 取消订阅
     */
    public void unSubscribe(ISubscriber subscriber, String topic) {
        map.get(topic).remove(subscriber);
    }


    public void notify(String publisher, String topic, T Msg) {
        List<ISubscriber> subscriberList = map.get(topic);
        for (ISubscriber subscriber : subscriberList) {
            subscriber.notify(publisher, Msg);
        }
    }
}
```

#### 测试

```java
public class Main {

    public static void main(String[] args) {
        // 发布订阅器
        SubscribePublish<Msg<String>> subscribePublish = new SubscribePublish<>();
        // 订阅器
        ISubscriber<Msg<String>> iSubscriber1 = new DefaultSubscriber<>("订阅者1");
        ISubscriber<Msg<String>> iSubscriber2 = new DefaultSubscriber<>("订阅者2");
        iSubscriber1.subscribe(subscribePublish, "tpc_1");
        iSubscriber2.subscribe(subscribePublish, "tpc_2");
        // 事件发布
        IPublish<Msg<String>> iPublish1 = new DefaultPublisher<>("发布者1");
        IPublish<Msg<String>> iPublish2 = new DefaultPublisher<>("发布者2");
        // 事件
        Msg<String> msg1 = new Msg<>("发布者1", "发布消息1");
        iPublish1.publish(subscribePublish, "tpc_1", msg1);
        Msg<String> msg2 = new Msg<>("发布者2", "发布消息2");
        iPublish2.publish(subscribePublish, "tpc_2", msg2);

        iSubscriber1.unSubscribe(subscribePublish, "tpc_1");
        Msg<String> msg3 = new Msg<>("发布者1", "发布消息3");
        iPublish1.publish(subscribePublish, "tpc_1", msg3);
    }
}
```

测试结果

```
订阅者1订阅tpc_1
订阅者2订阅tpc_2
发布者1发布消息到tpc_1
订阅者1消费发布者1发布的消息:Msg(publisher=发布者1, msg=发布消息1)
发布者2发布消息到tpc_2
订阅者2消费发布者2发布的消息:Msg(publisher=发布者2, msg=发布消息2)
订阅者1取消订阅tpc_1
发布者1发布消息到tpc_1
```