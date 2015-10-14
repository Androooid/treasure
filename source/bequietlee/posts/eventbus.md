EventBus分享
===============
EventBus是什么
--------------------
EventBus是一种为Android设计的时间分发／订阅系统。

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
[此处应有截图]

* 两个标签页面，用Fragment实现
* 页面A点击 ```+``` ，页面B数字递增
Fragment hide/show
对比两种实现Interface/EventBus
[show code]


背景知识：观察者模式
[此处应有图片]
观察者模式定义了对象之间的一对多以来，这样一来，当一个对象改变状态时，它的所有依赖者都会收到通知并自动更新。
设计原则：解耦合。
主要步骤：订阅、发放通知、取消订阅

