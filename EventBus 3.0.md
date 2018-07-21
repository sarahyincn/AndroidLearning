# EventBus 3.0使用

## 背景

EventBus是一个Android端优化的publish/subscribe消息总线，简化了应用程序内各组件间、组件与后台线程间的通信。比如请求网络，等网络返回时通过Handler或Broadcast通知UI，两个Fragment之间需要通过Listener通信，这些需求都可以通过EventBus实现。

大家谈到EventBus，总会想到greenrobot的EventBus，但是实际上EventBus是一个通用的叫法。例如Google出品的Guava，Guava是一个庞大的库，EventBus只是它附带的一个小功能，因此实际项目中使用并不多。用的最多的是greenrobot/EventBus，这个库的优点是接口简洁，集成方便。另一个库square/otto修改自 Guava ，用的人也不少。

目前本文只讨论greenrobot的EventBus库。

EventBus 3.0是greenrobot在2016发布的一个大版本迭代，加入了注解。

## 集成方法

### 添加EventBus库

1. Gradle

```java
compile 'org.greenrobot:eventbus:3.1.1'
```

截止到笔者写作的时候，EventBus最新的版本号是3.1.1，所以此处用该版本号举例。具体的最新版本号，可以去EventBus的Github上查看。

2. Maven

```java
<dependency>
    <groupId>org.greenrobot</groupId>
    <artifactId>eventbus</artifactId>
    <version>3.1.1</version>
</dependency>
```

### 简单使用

1. 定义Event

   ```java
   public static class MessageEvent { /* Additional fields if needed */ }
   ```

2. 注册EventBus并订阅Event

   ```java
   @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }
   
    @Override
    public void onStop() {
        super.onStop();
        EventBus.getDefault().unregister(this);
    }
   ```

   ```java
   @Subscribe(threadMode = ThreadMode.MAIN)  
   public void onMessageEvent(MessageEvent event) {/* Do something */};
   ```

3. 发布Event

   ```java
   EventBus.getDefault().post(new MessageEvent());
   ```

## 技术细节

### Delivery Threads

Delivery Thread的定义是在EventBus2.0中的ThreadMode基础上完成的。EventBus具有跨线程功能，即Event的发布和订阅可以不在同一个线程。大家都知道，在Android中，UI操作必须在主线程中完成，而网络请求等耗时操作必须在子线程中完成，所以，Android跨线程的场景非常多。

#### POSTING

1. 默认设置，订阅者和发布者会在同一个线程调用。
2. 避免了线程切换，开销最小
3. 可能在主线程被调用，所以必须快速返回，防止阻塞主线程
4. 适合于没有UI操作的不耗时操作

```java
@Subscribe(threadMode = ThreadMode.POSTING)
public void onMessage(MessageEvent event) {
    log(event.message);
}
```

#### MAIN

1. 订阅方法会在主线程被调用
2. 必须快速返回，防止阻塞主线程

```java
@Subscribe(threadMode = ThreadMode.MAIN)
public void onMessage(MessageEvent event) {
    textField.setText(event.message);
}
```

#### MAIN_ORDERED

1. 订阅方法会在主线程被调用
2. 订阅方法会按照发布顺序有序调用
3. 必须快速返回，防止阻塞主线程

```java
@Subscribe(threadMode = ThreadMode.MAIN_ORDERED)
public void onMessage(MessageEvent event) {
    textField.setText(event.message);
}
```

#### BACKGROUND

1. 订阅者会在后台线程中调用
2. 如果发布线程是后台线程，则直接在发布线程中调用，类似于POSTING
3. 如果发布线程是主线程，则会在一个后台线程中顺序调用
4. 必须快速返回，防止阻塞后台线程

```java
@Subscribe(threadMode = ThreadMode.BACKGROUND)
public void onMessage(MessageEvent event){
    saveToDisk(event.message);
}
```

#### ASYNC

1. 独立于主线程和发布线程
2. 底层使用线程池管理
3. 可以有耗时操作

```java
@Subscribe(threadMode = ThreadMode.ASYNC)
public void onMessage(MessageEvent event){
    backend.send(event.message);
}
```

### Configuration

#### 单独配置EventBus

我们可以使用EventBusBuilder^[3]^来配置自定义的EventBus。例如，配置EventBus当发布的时间没有订阅时只打印Log而不是真的发布。

```java
EventBus eventBus = EventBus.builder()
    .logNoSubscriberMessages(false)
    .sendNoSubscriberEvent(false)
    .build();
```

#### 默认配置EventBus

我们可以使用`EventBus.getDefault()`在任何地方获取EventBus的实例。也可以使用`installDefaultEventBus()`来配置这个默认实例。

```java
EventBus.builder().throwSubscriberException(BuildConfig.DEBUG).installDefaultEventBus();
```

使用要求：

1. 在第一次使用getDefault之前调用
2. 只能调用一次，之后的调用会引发异常
3. 最好在Application类中调用

### Sticky Event

我们可能需要缓存一些Event，即使在Event发布之后也能够触发订阅者的订阅事件，例如传感器和位置信息等。和Broadcast中的Sticky Broadcast类似。

#### 发布Sticky Event

1. 发布

   ```java
   EventBus.getDefault().postSticky(new MessageEvent("Hello everyone!"));
   ```

2. 注册

   ```java
   @Subscribe(sticky = true, threadMode = ThreadMode.MAIN)
   public void onEvent(MessageEvent event) {   
       textField.setText(event.message);
   }
   ```

#### 移除Sticky Event

```java
MessageEvent stickyEvent = EventBus.getDefault().getStickyEvent(MessageEvent.class);
// Better check that an event was actually posted before
if(stickyEvent != null) {
    // "Consume" the sticky event
    EventBus.getDefault().removeStickyEvent(stickyEvent);
    // Now do something with it
}

MessageEvent stickyEvent = EventBus.getDefault().removeStickyEvent(MessageEvent.class);
```

### 优先级

订阅者之间通过优先级来确定收到订阅消息的先后顺序。默认优先级都是0，优先级大的订阅者会比其他订阅者更早收到消息，并且可以通过`cancelEventDelivery()`来取消订阅消息的下一步传递。

```java
@Subscribe(priority = 1, threadMode = ThreadMode.POSTING)
public void onEvent(MessageEvent event) {
    // 只能在发布线程中调用
    EventBus.getDefault().cancelEventDelivery(event) ;
}
```

## 原理解析



## 参考文献

[1] https://github.com/greenrobot/EventBus

[2] http://greenrobot.org/eventbus

[3] http://greenrobot.org/files/eventbus/javadoc/3.0/org/greenrobot/eventbus/EventBusBuilder.html

