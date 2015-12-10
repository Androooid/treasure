# Reactive Programming


---

##Rx概念介绍
Reactive是一种编程思想
Reactive编程就是异步数据流的编程，基于事件的编程

**一切皆流**

一个流是将要发生的有序序列事件的一部分。  
你可以创建任何事物的数据流。任何事物都可以是流：变量，用户输入，属性 ，缓存，数据结构等等

它可以发出三种不同的事件：`value`，`error`或者`completed`  

流（Observable）是被观察的对象。对于流的监听被称作订阅。我们定义的函数被称作观察者。  

Rx扩展(Rxjava,RxJs,RxScala)提供了一个用于创建，变换，连接，过滤任何流的函数库。不仅某个流可以用于另一个流的输入，多个流同样可以作为其它流的输入。你也可以合并两个流。如果你对某些事件感兴趣，也可以通过对一个流的过滤获得另一个目标流。也可以将一个流中的数据映射到一个新的数据流。

##优点
- 简洁
- 抽象层次高，你可以聚焦于定义业务逻辑的事件依赖，而不是大量的实现细节
- 有效避免callback hell，更少的中间状态变量

##缺点
- 代码抽象层次高，真正使用Rx思想解决问题需要一个过程
- 对于android来说，包比较大，方法数也不少

#适用场景
异步 ？
线程切换 ？
事件组合？（多个请求和UI操作组合） 

**EveryWhere**  
Rx是一种思想，这里的一切都是流，你可以定义任何事物的流，可以是事件，可以是数据结构，任意发挥你的想象，通过Rx的方式来解决问题。（Twitter suggestion的实现）  

###Callback Hell
比如有一个链式请求调用，你首先需要根据第一个请求的结果去判断下一步的操作。那么就要处理多个请求的回调。不管是正确还是错误，你总需要通过callback处理。无形中多了不少代码量，创建了变量，浪费了内存，同时增加了错误的可能性。

##二 通过RxJava创建一条完整的事件链

**被观察者**
Observable，对应我们上面所说的流，任何事物：数据，事件

**订阅者**
Observer，Subcreiber

###2.1 简单的创建
####2.1.1  创建 Observer
Observer 即观察者，它决定事件触发的时候将有怎样的行为。 RxJava 中的 Observer 接口的实现方式：
```java
//new one
Observer<String> observer = new Observer<String>() {
    @Override
    public void onNext(String s) {}

    @Override
    public void onCompleted() {}

    @Override
    public void onError(Throwable e) {}
};
```
除了 Observer 接口之外，RxJava 还内置了一个实现了 Observer 的抽象类：Subscriber。   
Subscriber是对Observer接口的扩展，但它们的基本使用方式是完全一样的：

```java
Subscriber subscriber = new Subscriber() {
    @Override
    public void onCompleted() {}

    @Override
    public void onError(Throwable e) {}

    @Override
    public void onNext(Object o) {}
};
```

Observer和Subscriber不仅基本使用方式一样，实质上，在 RxJava 的 subscribe 过程中，Observer 也总是会先被转换成一个 Subscriber 再使用。所以如果你只想使用基本功能，选择 Observer 和 Subscriber 是完全一样的。对于使用者来说它们的区别主要有两点：

- onStart()  
这是 Subscriber 增加的方法。它会在 subscribe 刚开始，而事件还未发送之前被调用，可以用于做一些准备工作，例如数据的清零或重置。这是一个可选方法，默认情况下它的实现为空。需要注意的是，如果对准备工作的线程有要求（例如弹出一个显示进度的对话框，这必须在主线程执行）， onStart() 就不适用了，因为它总是在 subscribe 所发生的线程被调用，而不能指定线程。要在指定的线程来做准备工作，可以使用 doOnSubscribe() 方法，具体可以在后面的文中看到。

- unsubscribe()  
这是 Subscriber 所实现的另一个接口 Subscription 的方法，用于取消订阅。在这个方法被调用后，Subscriber 将不再接收事件。一般在这个方法调用前，可以使用 isUnsubscribed() 先判断一下状态。 unsubscribe() 这个方法很重要，因为在 subscribe() 之后， Observable 会持有 Subscriber 的引用，这个引用如果不能及时被释放，将有内存泄露的风险。所以最好保持一个原则：在不再使用的时候尽快在合适的地方（例如 onPause() onStop() 等方法中）调用unsubscribe() 来解除引用关系，以避免内存泄露的发生。

####2.1.2 创建 Observable
Observable 即被观察者，我更倾向于叫它“流”。因为任何事物都可以是流，而不限于数据。你可以对流做任何想做的处理，转换，过滤，合并等等。 RxJava 使用 create() 方法来创建一个 Observable ，并为它定义事件触发规则：
```java
Observable<String> observable = Observable.create(new Observable.OnSubscribe<String>() {

    @Override
    public void call(Subscriber<? super String> subscriber){
        subscriber.onNext("Hello");
    }
});
```
这里传入了一个 OnSubscribe 对象作为参数。OnSubscribe 会被存储在返回的 Observable 对象中，它的作用相当于一个计划表，定义要执行的事件，当 Observable 被订阅的时候，OnSubscribe 的 call() 方法会自动被调用，事件序列就会依照设定依次触发。

**创建Rx队列**  
create() 方法是 RxJava 最基本的创造事件序列的方法。基于这个方法， RxJava 还提供了一些方法用来快捷创建事件队列，例如：

- just(T...): 将传入的参数依次发送出来。
```java
Observable observableSequece = Observable.just("Hello", "Hi", "Aloha");
// 将会依次调用：
// onNext("Hello");
// onNext("Hi");
// onNext("Aloha");
// onCompleted();
```

- from(T[]) / from(Iterable<? extends T>) : 将传入的数组或 Iterable 拆分成具体对象后，依次发送出来。
```java
String[] words = {"Hello", "Hi", "Aloha"};
Observable observableFromSequece = Observable.from(words);
// 将会依次调用：
// onNext("Hello");
// onNext("Hi");
// onNext("Aloha");
// onCompleted();
上面 just(T...) 的例子和 from(T[]) 的例子，都和之前的 create(OnSubscribe) 的例子是等价的。
```
####2.1.3 Subscribe (订阅)

通过subscribe() 将 Observable 和 Observer 联结起来，形成了一个完整的事件监听和回调：
```java
observable.subscribe(observer);
// 或者：
observable.subscribe(subscriber);
```
当subscribe，onSubscribe就开始执行了。

###2.2 自定义回调 —— Action
有时你不需要关心所有的回调onNext，onComplete或onError，那么可以针对感兴趣的事件进行监听：RxJava 会自动根据定义创建出Subscriber 。形式如下：
```java
Action1<String> onNextAction = new Action1<String>() {
            // onNext()
            @Override
            public void call(String s) {
                Log.d(tag, s);
            }
        };
        Action1<Throwable> onErrorAction = new Action1<Throwable>() {
            // onError()
            @Override
            public void call(Throwable throwable) {
                // Error handling
            }
        };
        Action0 onCompletedAction = new Action0() {
            // onCompleted()
            @Override
            public void call() {
                Log.d(tag, "completed");
            }
        };

// 自动创建 Subscriber ，并使用 onNextAction 来定义onNext()
    observable.subscribe(onNextAction);
    
// 自动创建 Subscriber ，并使用 onNextAction 和 onErrorAction 来定义 onNext() 和 onError()
    observable.subscribe(onNextAction, onErrorAction);
    
// 自动创建 Subscriber ，并使用 onNextAction、 onErrorAction 和 onCompletedAction 来定义 onNext()、 onError() 和 onCompleted()
    observable.subscribe(onNextAction, onErrorAction, onCompletedAction);
```

**Action0**   
RxJava 的一个接口，它只有一个方法 `call()`，这个方法是无参无返回值的；由于 onCompleted() 方法也是无参无返回值的，因此 `Action0` 可以被当成一个包装对象，将 `onCompleted()` 的内容打包起来将自己作为一个参数传入 `subscribe()` 以实现不完整定义的回调。这样其实也可以看做将 `onCompleted()` 方法作为参数传进了`subscribe()`，相当于其他某些语言中的『闭包』。 

**Action1**   
也是一个接口，它同样只有一个方法 call(T param)，这个方法也无返回值，但有一个参数；与 Action0 同理，由于 onNext(T obj) 和 onError(Throwable error) 也是单参数无返回值的，因此 Action1可以将 onNext(obj) 和 onError(error) 打包起来传入 subscribe() 以实现不完整定义的回调。事实上，虽然 Action0 和 Action1在 API 中使用最广泛，但 RxJava 是提供了多个 ActionX 形式的接口 (例如 Action2, Action3) 的，它们可以被用以包装不同的无返回值的方法。

##三 线程控制 —— Scheduler

在不指定线程的情况下， RxJava 遵循的是线程不变的原则，即：在哪个线程调用 subscribe()，就在哪个线程生产事件；在哪个线程生产事件，就在哪个线程消费事件。如果需要切换线程，就需要用到 Scheduler （调度器）。

###3.1 Scheduler API 

###3.1.1 Scheduler
在RxJava 中，Scheduler —— 调度器，相当于线程控制器，通过它可以指定每一段代码应该运行在什么样的线程。RxJava 已经内置了几种Scheduler ，它们已经适合大多数的使用场景：

- Schedulers.immediate()  
相当于不指定线程，直接在当前线程运行，这是默认的 Scheduler。

- Schedulers.newThread():
总是启用新线程，并在新线程执行操作。

- Schedulers.io()
I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 Scheduler。行为模式和 newThread() 差不多，区别在于 io() 的内部实现是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 io() 比 newThread() 更有效率。不要把计算工作放在 io() 中，可以避免创建不必要的线程。

- Schedulers.computation()  
计算所使用的 Scheduler。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。

- AndroidSchedulers.mainThread()  
Android 专用的scheduler ，它指定操作在 Android 主线程运行。


###3.1.2 线程切换
可以通过 subscribeOn() 和 observeOn() 两个方法来对线程进行控制了。

- subscribeOn()  
指定 subscribe() 所发生的线程，即 Observable.OnSubscribe 被激活时所处的线程。或者叫做事件产生的线程。

- observeOn()  
指定 Subscriber 所运行在的线程。看名字，observeOn：观察者所在的线程。或者叫做事件消费的线程。

```java
Observable.just(1, 2, 3, 4)
        .subscribeOn(Schedulers.io()) // 指定 subscribe() 发生在 IO 线程
        .observeOn(AndroidSchedulers.mainThread()) // 指定 Subscriber 的回调发生在主线程
        .subscribe(new Action1<Integer>() {
            @Override
            public void call(Integer number) {
                Log.d(tag, "number:" + number);
            }
        });
```
上面这段代码中，由于 subscribeOn(Schedulers.io()) 的指定，被创建的事件的内容 1、2、3、4 将会在 IO 线程发出；而由于observeOn(AndroidScheculers.mainThread()) 的指定，因此 subscriber 数字的打印将发生在主线程 。事实上，这种在subscribe() 之前写上subscribeOn(Scheduler.io()) 和 observeOn(AndroidSchedulers.mainThread()) 的使用方式非常常见，它适用于多数的 『后台线程取数据，主线程显示』的程序策略。

那么，加载图片将会发生在 IO 线程，而设置图片则被设定在了主线程。这就意味着，即使加载图片耗费了几十甚至几百毫秒的时间，也不会造成丝毫界面的卡顿。

##四 操作符

你可以对流进行各种处理：过滤，链接，合并，转换等等。
###4.1 map变换  
事件对象的直接变换，具体功能上面已经介绍过。它是 RxJava 最常用的变换。

```java
Observable.just("images/logo.png") // 输入类型 String
        .map(new Func1<String, Bitmap>() {
            @Override
            public Bitmap call(String filePath) { // 参数类型 String
                return getBitmapFromPath(filePath); // 返回类型 Bitmap
            }
        })
        .subscribe(new Action1<Bitmap>() {
            @Override
            public void call(Bitmap bitmap) { // 参数类型 Bitmap
                showBitmap(bitmap);
            }
        });
```
上面出现的Func1 和 Action1 非常相似，也是 RxJava 的一个接口，用于包装含有一个参数的方法。  

**FuncX 和ActionX 的区别**  
FuncX 包装的是有返回值的方法。  

可以看到，map() 方法将参数中的 String 对象转换成一个 Bitmap 对象后返回，而在经过 map() 方法后，事件的参数类型也由 String转为了 Bitmap。可以看到在subscribe处理的Action的参数已经就是bitmap了。  

>Rx扩展(Rxjava,RxJs,RxScala)提供了一个用于创建，变换，连接，过滤任何流的函数库。不仅某个流可以用于另一个流的输入，多个流同样可以作为其它流的输入。你也可以合并两个流。如果你对某些事件感兴趣，也可以通过对一个流的过滤获得另一个目标流。也可以将一个流中的数据映射到一个新的数据流。

###4.2 flatMap()  
Observable.flatMap()接收一个Observable的输出作为输入，同时输出另外一个Observable。可以用于实现一对多的变换。  

这是一个很有用但非常难理解的变换，首先假设这么一种需求：假设有一个数据结构『学生』，现在需要打印出一组学生的名字。实现方式很简单：

**打印所有学生的name**  
```java
Student[] students = ...;
Subscriber<String> subscriber = new Subscriber<String>() {
    @Override
    public void onNext(String name) {
        Log.d(tag, name);
    }
    ...
};
Observable.from(students)
    .map(new Func1<Student, String>() {
        @Override
        public String call(Student student) {
            return student.getName();
        }
    })
    .subscribe(subscriber);
```
  
**打印所有学生的所有courseName**  
```java
Student[] students = ...;
Subscriber<Student> subscriber = new Subscriber<Student>() {
    @Override
    public void onNext(Student student) {
        List<Course> courses = student.getCourses();
        for (int i = 0; i < courses.size(); i++) {
            Course course = courses.get(i);
            Log.d(tag, course.getName());
        }
    }
    ...
};
Observable.from(students)
    .subscribe(subscriber);
```

看上去实现了我们需要的功能，但是subscriber不该去做数据处理的工作，真正的工作应该只做响应。而数据处理应该放在Observable中，这时候就要引入flatMap：

```java
Student[] students = ...;
Subscriber<Course> subscriber = new Subscriber<Course>() {
    @Override
    public void onNext(Course course) {
        Log.d(tag, course.getName());
    }
    ...
};
Observable.from(students)
        .flatMap(new Func1<Student, Observable<Course>>() {
            @Override
            public Observable<Course> call(Student student) {
                return Observable.from(student.getCourses());
            }
        })
        .subscribe(subscriber);
```
 flatMap() 和 map() 有一个相同点：它也是把传入的参数转化之后返回另一个对象。但需要注意，和 map() 不同的是， flatMap() 中返回的是个 Observable 对象，并且这个 Observable 对象并不是被直接发送到了 Subscriber 的回调方法中。   

**flatMap() 的原理**  
1. 使用传入的事件对象创建一个 Observable 对象  
2. 并不发送这个 Observable，而是将它激活，于是它开始发送事件
3. 每一个创建出来的 Observable 发送的事件，都被汇入同一个 Observable ，而这个 Observable 负责将这些事件统一交给 Subscriber 的回调方法。  

这三个步骤，把事件拆成了两级，通过一组新创建的 Observable 将初始的对象『铺平』之后通过统一路径分发了下去。而这个『铺平』就是 flatMap() 所谓的 flat。

扩展：由于可以在嵌套的 Observable 中添加异步代码， flatMap() 也常用于嵌套的异步操作，例如嵌套的网络请求。示例代码（Retrofit + RxJava）：
```java
networkClient.token() // 返回 Observable<String>，在订阅时请求 token，并在响应后发送 token
    .flatMap(new Func1<String, Observable<Messages>>() {
        @Override
        public Observable<Messages> call(String token) {
            // 返回 Observable<Messages>，在订阅时请求消息列表，并在响应后发送请求到的消息列表
            return networkClient.messages();
        }
    })
    .subscribe(new Action1<Messages>() {
        @Override
        public void call(Messages messages) {
            // 处理显示消息列表
            showMessages(messages);
        }
    });
```

传统的嵌套请求需要使用嵌套的 Callback 来实现。而通过 flatMap() ，可以把嵌套的请求写在一条链中，从而保持程序逻辑的清晰。

###4.3 其它操作符

- filter  
过滤
```java
query("Hello, world!")
    .flatMap(urls -> Observable.from(urls))
    .flatMap(url -> getTitle(url))
    .filter(title -> title != null)
    .subscribe(title -> System.out.println(title));
```
- take()  
输出最多指定数量的结果
```java
query("Hello, world!")
    .flatMap(urls -> Observable.from(urls))
    .flatMap(url -> getTitle(url))
    .filter(title -> title != null)
    .take(5)
    .subscribe(title -> System.out.println(title));
```

- doOnNext()  
允许我们在每次输出一个元素之前做一些额外的事情，比如这里的保存标题。
```java
query("Hello, world!")
    .flatMap(urls -> Observable.from(urls))
    .flatMap(url -> getTitle(url))
    .filter(title -> title != null)
    .take(5)
    .doOnNext(title -> saveTitle(title))
    .subscribe(title -> System.out.println(title));
```

- merge   
合并两个流  

- combineLatest 
关联两个流

更多API可以参考：[Rx官方文档中文翻译](https://mcxiaoke.gitbooks.io/rxdocs/content/Intro.html)    

##五 RxAndroid
RxAndroid是RxJava的一个针对Android平台的扩展。它包含了一些能够简化Android开发的工具。

**AndroidSchedulers**  
提供了针对Android的线程系统的调度器。

```retrofitService.getImage(url)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(bitmap -> myImageView.setImageBitmap(bitmap));
```

**AndroidObservable**  
配合Android的生命周期  
bindActivity()和bindFragment()方法默认使用AndroidSchedulers.mainThread()来执行观察者代码，这两个方法会在Activity或者Fragment结束的时候通知被观察者停止发出新的消息。
```
AndroidObservable.bindActivity(this, retrofitService.getImage(url))
    .subscribeOn(Schedulers.io())
    .subscribe(bitmap -> myImageView.setImageBitmap(bitmap);
```

**AndroidObservable.fromBroadcast()**  
允许你创建一个类似BroadcastReceiver的Observable对象。下面的例子展示了如何在网络变化的时候被通知到：
```
IntentFilter filter = new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION);
AndroidObservable.fromBroadcast(context, filter)
    .subscribe(intent -> handleConnectivityChange(intent));
```

**ViewObservable**  
使用它可以给View添加了一些绑定。如果你想在每次点击view的时候都收到一个事件，可以使用ViewObservable.clicks()，或者你想监听TextView的内容变化，可以使用ViewObservable.text()。

```
ViewObservable.clicks(mCardNameEditText, false)
    .subscribe(view -> handleClick(view));
```

##六 RxBinding

RxBinding 是 Jake Wharton 的一个开源库，它提供了一套在 Android 平台上的基于 RxJava 的 Binding API。所谓 Binding，就是类似设置 OnClickListener 、设置 TextWatcher 这样的注册绑定对象的 API。

举个设置点击监听的例子。使用 RxBinding ，可以把事件监听用这样的方法来设置：  
```java
Button button = ...;
RxView.clickEvents(button) // 以 Observable 形式来反馈点击事件
    .subscribe(new Action1<ViewClickEvent>() {
        @Override
        public void call(ViewClickEvent event) {
            // Click handling
        }
    });
```

通过 RxBinding 把点击监听转换成 Observable之后，就有了对它进行扩展的可能。扩展的方式有很多，根据需求而定。一个例子是前面提到过的 throttleFirst() ，用于去抖动，也就是消除手抖导致的快速连环点击：

```java
RxView.clickEvents(button)
    .throttleFirst(500, TimeUnit.MILLISECONDS)
    .subscribe(clickAction);
如果想对 RxBinding 有更多了解，可以去它的 GitHub 项目 下面看看。
```

##七 这些你应该了解
**扩展Rx**   
前面举的 Retrofit 和 RxBinding 的例子，是两个可以提供现成的 Observable 的库。而如果你有某些异步操作无法用这些库来自动生成 Observable，也完全可以自己写。例如数据库的读写、大图片的载入、文件压缩/解压等各种需要放在后台工作的耗时操作，都可以用 RxJava 来实现，有了之前几章的例子，这里应该不用再举例了。

**内存泄漏**   
Observable持有Context导致的内存泄露  
这个问题是因为创建subscription的时候，以某种方式持有了context的引用，尤其是当你和view交互的时候，这太容易发生！如果Observable没有及时结束，内存占用就会越来越大。 

**使用缓存，减少Observable的创建**  
RxJava内置有缓存机制，这样你就可以对同一个Observable对象执行unsubscribe/resubscribe，却不用重复运行得到Observable的代码。cache() (或者 replay())会继续执行网络请求（甚至你调用了unsubscribe也不会停止）。这就是说你可以在Activity重新创建的时候从cache()的返回值中创建一个新的Observable对象。 

```java
Observable<Photo> request = service.getUserPhoto(id).cache();
Subscription sub = request.subscribe(photo -> handleUserPhoto(photo));

// ...When the Activity is being recreated...
sub.unsubscribe();

// ...Once the Activity is recreated...
request.subscribe(photo -> handleUserPhoto(photo));
```

注意，两次sub是使用的同一个缓存的请求。在哪里去存储请求的结果还是要你自己来做，和所有其他的生命周期相关的解决方案一样，必须在生命周期外的某个地方存储。（retained fragment或者单例等等）。

**及时取消订阅**
在生命周期的某个时刻及时取消订阅，释放对context的引用。一个常见的情景就是使用CompositeSubscription来持有所有的Subscriptions，然后在onDestroy()或者onDestroyView()里取消所有的订阅。

```java
private CompositeSubscription mCompositeSubscription
    = new CompositeSubscription();

private void doSomething() {
    mCompositeSubscription.add(
        AndroidObservable.bindActivity(this, Observable.just("Hello, World!"))
        .subscribe(s -> System.out.println(s)));
}

@Override
protected void onDestroy() {
    super.onDestroy();

    mCompositeSubscription.unsubscribe();
}
```
你可以在Activity/Fragment的基类里创建一个CompositeSubscription对象，在子类中使用它。

注意! 一旦你调用了 CompositeSubscription.unsubscribe()，这个CompositeSubscription对象就不可用了, 如果你还想使用CompositeSubscription，就必须在创建一个新的对象了。  

本文参考了[@扔物线](http://weibo.com/rengwuxian?from=feed&loc=at&nick=%E6%89%94%E7%89%A9%E7%BA%BF)的[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)  和[@hi大头鬼](http://weibo.com/brucefromsdu?from=feed&loc=at&nick=hi%E5%A4%A7%E5%A4%B4%E9%AC%BChi)翻译的[深入浅出Rxjava](http://blog.csdn.net/lzyzsd/article/details/41833541)系列文章，一些内容直接copy，有些按照自己的方式梳理了一下。更详细的内容建议参看原文。

##资料
[Rx官方网站](http://reactivex.io/)  
[tutorials](http://reactivex.io/tutorials.html)  
[RxJava Wiki](https://github.com/ReactiveX/RxJava/wiki)  
[How-To-Use-RxJava](https://github.com/ReactiveX/RxJava/wiki/How-To-Use-RxJava)    
[Rx Github组织](https://github.com/ReactiveX)  
[RxJava Github](https://github.com/ReactiveX/RxJava)  
[Twitter](https://twitter.com/ReactiveX)  

[Awesome－RxJava](https://github.com/lzyzsd/Awesome-RxJava) 
[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)  
[Rx官方文档中文翻译](https://mcxiaoke.gitbooks.io/rxdocs/content/Intro.html)    

[Reactive Programming in the Netflix API with RxJava](http://techblog.netflix.com/2013/02/rxjava-netflix-api.html)  
[Advanced RxJava](http://akarnokd.blogspot.hu/)  
