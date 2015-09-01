
#前言#
尽管Java虚拟机可以帮我们对内存进行回收，但是其回收的是Java虚拟机不再引用的对象。很多时候我们使用系统的IO流，Cursor，Receiver如果不及时释放，就会导致内存泄漏，这些场景是常见的，一般开发人员也都能够避免。但是，很多时候内存泄漏的现象不是很明显，比如内部类，Handler相关的使用导致的内存泄漏，或者你使用了第三方library的一些引用，比较消耗资源，但又不是像系统资源那样会引起你足够的注意去手动释放它们。当代码越来越多，如果结构不是很清晰，即使是常见的资源也有可能略掉，从而导致内存泄漏。内存泄漏很有可能会导致内存溢出，就是常说的OOM，从而导致应用crash，给用户一种糟糕的体验。该篇文章就是介绍内存分析工具MAT以及实战来帮你更好的分析内存问题。前面是相关概念介绍，最后通过内存泄漏分析，集合使用率，Hash性能分析，OQL快读定位空集合实战演示如何在实际应用中使用MAT。（通过一些静态检测也可以在开发期发现一些内存泄漏的问题，后面会有一些静态检测的文章）


##
#一  相关概念
Java虚拟机如何判定内存泄漏的呢？下面介绍一些相关概念

##1.1 GC Root ##
JAVA虚拟机通过可达性（Reachability)来判断对象是否存活，基本思想：以"GC Roots"的对象作为起始点向下搜索，搜索形成的路径称为引用链，当一个对象到GC Roots没有任何引用链相连（即不可达的），则该对象被判定为可以被回收的对象，反之不能被回收。

GC Roots可以是以下任意对象
- 一个在current thread(当前线程)的call stack(调用栈)上的对象（例如方法参数和局部变量）
- 线程自身或者system class loader(系统类加载器)加载的类
- native code(本地代码)保留的活动对象


##1.2 内存泄漏
对象无用了，但仍然可达（未释放），垃圾回收器无法回收。

##1.3 强（strong）、软（soft）、弱（weak）、虚（phantom）引用 ##

## Strong references ##
普通的java引用，我们通常new的对象就是：
`StringBuffer buffer = new StringBuffer();`
如果一个对象通过一串强引用链可达，那么它就不会被垃圾回收。你肯定不希望自己正在使用的引用被垃圾回收器回收吧。但对于集合中的对象，应在不使用的时候移除掉，否则会占用更多的内存，导致内存泄漏。

##Soft reference
当对象是Soft reference可达时，gc会向系统申请更多内存，而不是直接回收它，当内存不足的时候才回收它。因此Soft reference适合用于构建一些缓存系统，比如图片缓存。

## WeakReference ##
WeakReference不会强制对象保存在内存中。它拥有比较短暂的生命周期，允许你使用垃圾回收器的能力去权衡一个对象的可达性。在垃圾回收器扫描它所管辖的内存区域过程中，一旦gc发现对象是weakReference可达，就会把它放到ReferenceQueue中，等下次gc时回收它。
`WeakReference<Widget> weakWidget = new WeakReference<Widget>(widget);`
系统为我们提供了WeakHashMap，和HashMap类似，只是其key使用了weak reference。如果WeakHashMap的某个key被垃圾回收器回收，那么entity也会自动被remove。

由于WeakReference被GC回收的可能性较大，因此，在使用它之前，你需要通过weakObj.get()去判断目的对象引用是否已经被回收.

## Reference queque ##
一旦WeakReference.get()返回null，它指向的对象就会被垃圾回收，那么WeakReference对象就没有用了，意味着你应该进行一些清理。比如在WeakHashMap中要把回收过的key从Map中删除掉，避免无用的的weakReference不断增长。
ReferenceQueue可以让你很容易地跟踪dead references。WeakReference类的构造函数有一个ReferenceQueue参数，当指向的对象被垃圾回收时，会把WeakReference对象放到ReferenceQueue中。这样，遍历ReferenceQueue可以得到所有回收过的WeakReference。


## Phantom reference
和soft，weak Reference区别较大，它的get()方法总是返回null。这意味着你只能用PhantomReference本身，而得不到它指向的对象。当WeakReference指向的对象变得弱可达(weakly reachable）时会立即被放到ReferenceQueue中，这在finalization、garbage collection之前发生。理论上，你可以在finalize()方法中使对象“复活”（使一个强引用指向它就行了，gc不会回收它）。但没法复活PhantomReference指向的对象。而PhantomReference是在garbage collection之后被放到ReferenceQueue中的，没法复活。

关于Phantom reference的更多讨论，请参考：[understanding-weak-references](https://weblogs.java.net/blog/2006/05/04/understanding-weak-references)

##


#二  MAT相关视图和概念

## 2.1 Shallow Heap ##
Shallow size就是对象本身占用内存的大小，不包含其引用的对象内存，实际分析中作用不大。
常规对象（非数组）的ShallowSize由其成员变量的数量和类型决定
数组的shallow size有数组元素的类型（对象类型、基本类型）和数组长度决定


Shallow Size of a String object
```java
class String{
    public final class String {8 Bytes header
    private char value[]; 4 Bytes
    private int offset; 4 Bytes
    private int count; 4 Bytes
    private int hash = 0; 4 Bytes
…}
"Shallow size“ of a String ==24 Bytes
```

java的对象成员都是些引用。真正的内存都在堆上，看起来是一堆原生的byte[], char[], int[]，对象本身的内存都很小。所以我们可以看到以Shallow Heap进行排序的Histogram图中，排在第一位第二位的是byte，char

## 2.2 Retained Heap ##
retained heap值的计算方式是将retained set中的所有对象大小叠加。或者说，由于X被释放，导致其它所有被释放对象（包括被递归释放的）所占的heap大小。

**Retained Set**
当X被回收时那些将被GC回收的对象集合。

比如:
一个ArrayList持有100,000个对象，每一个占用16 bytes，移除这些ArrayList可以释放16 x 100,000 + X，X代表ArrayList的shallow大小。相对于shallow heap，RetainedHeap可以更精确的反映一个对象实际占用的大小（因为如果该对象释放，retained heap都可以被释放）。


## 2.3 Histogram ##
可列出每一个类的实例数。支持正则表达式查找，也可以计算出该类所有对象的retained size

<img src="../img/mat/histogram.png" width="700" height="470"/>

## 2.4 Dominator Tree ##
Dominator Tree：对象之间dominator关系树。如果从GC Root到达Y的的所有path都经过X，那么我们称X dominates Y，或者X是Y的Dominator Dominator Tree由系统中复杂的对象图计算而来。从MAT的dominator tree中可以看到占用内存最大的对象以及每个对象的dominator。
我们也可以右键选择**Immediate Dominator”**来查看某个对象的dominator。

<img src="../img/mat/dominator_tree.png" width="700" height="470"/>

## 2.5 Path to GC Roots ##
查看一个对象到RC  Roots的引用链
通常在排查内存泄漏的时候，我们会选择exclude all phantom/weak/soft etc.references,
意思是查看排除虚引用/弱引用/软引用等的引用链，因为被虚引用/弱引用/软引用的对象可以直接被GC给回收，我们要看的就是某个对象否还存在Strong 引用链（在导出HeapDump之前要手动出发GC来保证），如果有，则说明存在内存泄漏，然后再去排查具体引用。

<img src="../img/mat/path_to_gc_roots.png" width="700" height="470"/>


###查看当前Object所有引用,被引用的对象：
####List objects with （以Dominator Tree的方式查看）
incoming references   引用到该对象的对象
outcoming references  被该对象引用的对象

####Show objects by class  （以class的方式查看）
incoming references   引用到该对象的对象
outcoming references  被该对象引用的对象

## 2.6 OQL(Object Query Language)
类似SQL查询语言
Classes：Table
Objects：Rows
Fileds： Cols

`select * from com.example.mat.Listener`

**查找size＝0并且未使用过的ArrayList**
`select * from java.util.ArrayList where size=0 and modCount=0`

**查找所有的Activity**
select * from instanceof android.app.Activity

## 2.7 内存快照对比

方式一：Compare To Another Heap Dump

直接进行比较
<img src="../img/mat/mat_baseket_compare_1.png" width="700" height="470"/>

<img src="../img/mat/mat_baseket_compare_2.png" width="700" height="470"/>

<img src="../img/mat/mat_baseket_compare_3.png" width="700" height="470"/>




方式二：Compare Baseket

<img src="../img/mat/mat_baseket_0.png" width="700" height="330"/>

<img src="../img/mat/mat_baseket_1.png" width="700" height="470"/>

<img src="../img/mat/mat_baseket_2.png" width="700" height="260"/>

<img src="../img/mat/mat_baseket_3.png" width="700" height="470"/>


方式二比较根全面，可以直接给出百分比，而且还有更多比较选项

引出一个同事开发过程中的一个真实的例子，通过AS的Memory监测，他发现在微信支付完成后内存有突然大内存飙升的情况，后来通过Compare Baseket进行对比，发现内存增大了8M，并通过工具查看了bitmap的原图（如何查看Bitmap原图，可以参考高建武的文章：[打开MAT中的Bitmap原图](http://androidperformance.com/2015/04/11/AndroidMemory-Open-Bitmap-Object-In-MAT/)）。发现是微信回调页面一张背景图片占用了很大内存。



##

#三 MAT内存分析实战
##
##实战一 内存泄漏分析
关于如何安装和导出HeapDump文件 [@高建武](http://weibo.com/u/1315612820) 已经写了，这里就不啰嗦了，请移步：
[Android内存优化MAT使用入门](http://androidperformance.com/2015/04/11/AndroidMemory-Usage-Of-MAT/)
[Android内存优化MAT使用进阶](http://androidperformance.com/2015/04/11/AndroidMemory-Usage-Of-MAT-Pro/)
这里强调一点就是**在导出prof文件前，先手动出发一次GC，这样可以确保只保存那些无法回收的对象内存快照**，另外Android Studio提供自动转换。

### 查找导致内存泄漏的类 ##

既然环境已经搭好，heap dump也成功倒入，接下来就去分析问题

方式一：
- 1.查找目标类
如果在开发过程中，你的目标很明确，比如就是查找自己负责的Activity，那么通过包名或者Class筛选，OQL搜索都可以快速定位到
**OQL：**
点击OQL图标,在窗口输入`select * from instanceof android.app.Activity` 并按Ctrl + F5或者!按钮执行


- 2.Paths to GC Roots：exclude all phantom/weak/soft etc.references
查看一个对象到RC  Roots是否存在引用链。要将虚引用/弱引用/软引用等排除，因为被虚引用/弱引用/软引用的对象可以直接被GC给回收.

- 3.分析具体的引用为何没有被释放，并进行修复


**小技巧：**
- 当目的不明确时，可以直接定位到RetainedHeap最大的Object，Select incoming references ，查看引用链，定位到可疑的对象然后Path to GC Roots进行引用链分析

- 如果大对象筛选看不出区别，可以试试按照class分组，再寻找可疑对象进行GC引用链分析

- 直接按照包名直接查看GC引用链，可以一次性筛选多个类，但是如下图所示，选项是 `Merge Shortest Path to GCRoots`，这个选项具体不是很明白，不过也能筛选出存在GC引用链的类，这种方式的准确性还待验证。

<img src="../img/mat/mat_gc_roots_package.png" width="700" height="470"/>

所以有时候进行MAT分析还是需要一些经验，能够帮你更快更准确的定位。


##

##实战二  集合使用率分析
集合在开发中会经常使用到，如何选择合适的数据结构的集合，初始容量是多少（太小，可能导致频繁扩容），太大，又会开销跟多内存。当这些问题不是很明确时或者想查看集合的使用情况时，可以通过MAT来进行分析。

###1.筛选目标对象
<img src="../img/mat/collection_usage_1.png" width="700" height="470"/>
##
###2.Show Retained Set（查找当X被回收时那些将被GC回收的对象集合）
<img src="../img/mat/collection_usage_2.png" width="700" height="470"/>
##
###3.筛选指定的Object（Hash Map，ArrayList）并按照大小进行分组
<img src="../img/mat/collection_usage_3.png" width="700" height="470"/>
##
###4.查看指定类的Immediate dominators
<img src="../img/mat/collection_usage_4.png" width="700" height="470"/>


### Collections fill ratio
这种方式只能查看那些具有预分配内存能力的集合，比如HashMap，ArrayList。计算方式："size / capacity"

<img src="../img/mat/collection_usage_fill_radio_1.png" width="700" height="470"/>
##
<img src="../img/mat/collection_usage_fill_radio_2.png" width="700" height="260"/>

我们可以与方式一中最后一张图所得的结果对比，一个是具体数，另一个有比例，是对应的。

##实战三  Hash相关性能分析
当Hash集合中过多的对象返回相同Hash值的时候，会严重影响性能（Hash算法原理自行搜索），这里来查找导致Hash集合的碰撞率较高的罪魁祸首。

### 1. Map Collision Ratio
检测每一个HashMap或者HashTable实例并按照碰撞率排序
碰撞率 = 碰撞的实体/Hash表中所有实体

<img src="../img/mat/map_collision_1.png" width="700" height="470"/>
##

### 2. 查看Immediate dominators
<img src="../img/mat/map_collision_2.png" width="700" height="470"/>
<img src="../img/mat/map_collision_4.png" width="700" height="260"/>
##
### 通过HashEntries查看key value
<img src="../img/mat/map_collision_3.png" width="700" height="470"/>

##Array等其它集合分析方法类似

##
##实战四 通过OQL快速定位未使用的集合 ##

###1. 通过OQL查询empty并且未修改过的集合：
`select * from java.util.ArrayList where size=0 and modCount=0`
类似的
`select * from java.util.HashMap where size=0 and modCount=0`
`select * from java.util.Hashtable where count=0 and modCount=0`
##
<img src="../img/mat/empty_list_1.png" width="700" height="470"/>

##
###2. Immediate dominators(查看引用者)
<img src="../img/mat/empty_list_3.png" width="700" height="330"/>

### 计算空集合的Retained Size值，查看浪费了多少内存
<img src="../img/mat/empty_list_2.png" width="700" height="320"/>

##
#四 LeakCanary － 强大的内存泄漏分析工具
[LeakCanary](https://github.com/square/leakcanary)是square开源的内存泄漏排查项目，很强大，内部会帮你手动触发GC然后分析强引用的GC引用链。如果存在GC引用链，说明有内存泄漏，会在你的手机上弹出个提示框，并且会自动在你手机上创建一个App。记录了每一次内存泄漏的GC引用链，通过它可以直接定位到内存泄漏的未释放的对象。原理和通过MAT分析内存泄漏是一样的，只是它完全自动化，省去了很大一部分的工作量。强烈建议集成LeakCanary。LeakCanary肯定是无法取代强大的MAT，因为它只是只分析内存泄漏，从上面的实战中，我们可以看到，MAT的强大之处是可以对内存中的任何信息进行分析。所以掌握MAT也是非常有必要的。另外在之前用的LeakCanary中发现在解析Heap Dump内存快照的时候会出现问题，存在小bug。


##参考文档
[MemoryAnalyzer Wiki](http://wiki.eclipse.org/index.php/MemoryAnalyzer)  
[Android内存优化MAT使用入门](http://androidperformance.com/2015/04/11/AndroidMemory-Usage-Of-MAT/)  
[Android内存优化MAT使用进阶](http://androidperformance.com/2015/04/11/AndroidMemory-Usage-Of-MAT-Pro/)  
[10-tips-for-using-the-eclipse-memory-analyzer](http://eclipsesource.com/blogs/2013/01/21/10-tips-for-using-the-eclipse-memory-analyzer/)
[analyzing-java-collections-usage-with-memory-analyzer](http://scn.sap.com/people/krum.tsvetkov/blog/2007/11/05/analyzing-java-collections-usage-with-memory-analyzer)
[How-to-Find-Memory-Leaks](https://sites.google.com/site/eclipsebiz/How-to-Find-Memory-Leaks)
[How_to_analyze_heap_dumps](http://docwiki.cisco.com/wiki/How_to_analyze_heap_dumps)
[memory-for-nothing](http://scn.sap.com/people/krum.tsvetkov/blog/2007/08/02/memory-for-nothing)
