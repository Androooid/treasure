EventBus分享
===============
EventBus是什么
--------------------
EventBus是一种为Android设计的事件分发／订阅系统。
<img src="../img/eventbus/eventbus.png" width="500" height="187"/>

EventBus的优点
---------------------------
* 简化了组件之间的消息传递

  1.将消息发送者／接收者解耦

  2.适用于Activity、Fragment、及后台进程

  3.避免了复杂的易出错的依赖管理及生命周期问题
* 代码简洁
* 快速
* 轻量(<50k)
* 在安装量超过100,000,000的app中被验证过功能
* 具有优先级、sticky event等高级功能 

将EventBus引入项目
------------------
Gradle
```
compile 'de.greenrobot:eventbus:2.4.0'
```
如何使用EventBus的订阅／分发功能
---------------------
1.Define events:
```java
public class MessageEvent { /* Additional fields if needed */ }
```
2.Prepare subscribers:
```java
eventBus.register(this);
```
```java
public void onEvent(AnyEventType event) {/* Do something */};
```
3.Post events:
```java
eventBus.post(event);
```
Demo
--------------------------
<img src="../img/eventbus/demo.jpeg" width="600" height="400"/>

实现功能
* 两个标签页面A&B，用Fragment实现
* 页面A点击 ```Add``` ，页面B数字递增

使用接口回调方式实现
-----------------------
在 ```FragmentA``` 中声明了自己的 ```Add``` 按钮对应的回调接口。
```java
public interface PerformAddListener {
    void performAdd();
}
```
在外层Activity中实现了该接口
```java
@Override
public void performAdd() {
    if (displayFragment == null) {
        FragmentTransaction transaction = fragmentManager.beginTransaction();
        displayFragment = new DisplayFragment();
        transaction.add(containerId, displayFragment);
        transaction.hide(displayFragment);
        transaction.commit();
        fragmentManager.executePendingTransactions();
    }
    displayFragment.add();
}
```
上述方法有如下缺点：

* Fragment之间无法直接通信，需要容器Activity充当中转器

* Activity需要实现消息发送者提供的接口，以支持回调

* Activity需要显示调用消息接受者提供的方法，以完成消息传递

* 以上各点造成了强烈的耦合，可变的东西与不可变的东西没有分离开。后续在增加／删除消息发送者／接收者时，成本成倍增加
使用EventBus实现
--------------------------
事件发送者Fragment
```java
private void postAddEvent() {
    EventBus.getDefault().post(new Event.AddEvent());
}
```
事件接收者Fragment
```java
@Override
public void onStart() {
    super.onStart();
    EventBus.getDefault().register(this);
}
```
```java
@Override
public void onStop() {
    EventBus.getDefault().unregister(this);
    super.onStop();
}
```
```java
public void onEvent(Event.AddEvent addEvent) {
    add();
}
```

从上述EventBus实现中，可以看出：

* EventBus使用的是单例模式
* 发布者需要注册，订阅者无需注册
* 通过Event类型来区分不同的事件（支持事件继承）


背景知识：观察者模式
-----------------------
<img src="../img/eventbus/observer1" width="600" height="400">
<img src="../img/eventbus/observer2" width="600" height="400">

**定义：**观察者模式定义了对象之间的一对多依赖，这样一来，当一个对象改变状态时，它的所有依赖者都会收到通知并自动更新。

**设计原则：**解耦合。

**主要步骤：**订阅、发放通知、取消订阅


