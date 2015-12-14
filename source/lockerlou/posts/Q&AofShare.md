# ListView分享中的Q&A

## Q1. ListView中子View如何增加和删除？
ListView滚动过程中子View的添加和删除还是在trackMotionScroll方法中进行。此方法根据移动的距离计算出哪些子view应该移除，
随后使用了detachViewsFromParent方法。

此方法在ViewGroup中，主要作用是把其子View和ViewGroup本身的双向引用关系取消掉。方法如下所示：
```java
protected void detachAllViewsFromParent() {
    final int count = mChildrenCount;
    if (count <= 0) {
        return;
    }

    final View[] children = mChildren;
    mChildrenCount = 0;

    for (int i = count - 1; i >= 0; i--) {
        children[i].mParent = null;
        children[i] = null;
    }
}
```
随后可能会执行invalidate和现有的View的整体平移（offsetChildrenTopAndBottom），并执行fillGap方法。
而上次分享中已经说过，最后会调用到makeAndAddView方法，其中我们已经看过了obtainView是如何生成子View的，而另一个方法
setupChild则是如何操作这个子View的，这个方法最后会调用到attachViewToParent或addViewInLayout，这两者都会调用
addInArray来添加子View和ViewGroup之间的双向引用。

## Q2. notifyDataSetChanged原理是什么？
BaseAdapter中的notifyDataSetChanged方法如下：
```java
private final DataSetObservable mDataSetObservable = new DataSetObservable();
public void notifyDataSetChanged() {
    mDataSetObservable.notifyChanged();
}
```
而DataSetObservable中的notifyChanged方法也只是onChanged接口的调用，并没有实际的页面刷新逻辑。
那么这个观察者的绑定就肯定在ListView的方法中，而在使用任何AdapterView的过程中一定会调用的方法是setAdapter，
其中有两行代码进行了观察者的绑定：
```java
mDataSetObserver = new AdapterDataSetObserver();
mAdapter.registerDataSetObserver(mDataSetObserver);
```
而对于AdapterDataSetObserver的基本实现是在AdapterView中：
```java
public void onChanged() {
    mDataChanged = true;
    mOldItemCount = mItemCount;
    mItemCount = getAdapter().getCount();

    // Detect the case where a cursor that was previously invalidated has
    // been repopulated with new data.
    if (AdapterView.this.getAdapter().hasStableIds() && mInstanceState != null
            && mOldItemCount == 0 && mItemCount > 0) {
        AdapterView.this.onRestoreInstanceState(mInstanceState);
        mInstanceState = null;
    } else {
        rememberSyncState();
    }
    checkFocus();
    requestLayout();
}
```
其中的requestLayout调用就是notifyDataSetChanged刷新控件的根本。

## Q3. 正常的ListView工作原理是怎么样的？
对于一个普通的ListView来说，其使用核心只有一个方法，即setAdapter方法。
这里就来看一下setAdapter方法到底干了些什么事，具体请看注释：
```java
@Override
public void setAdapter(ListAdapter adapter) {
    //如果已绑定观察者则注销掉它
    if (mAdapter != null && mDataSetObserver != null) {
        mAdapter.unregisterDataSetObserver(mDataSetObserver);
    }

    //清除头部和底部信息，并清空RecyclerBin
    resetList();
    mRecycler.clear();

    if (mHeaderViewInfos.size() > 0|| mFooterViewInfos.size() > 0) {
        mAdapter = new HeaderViewListAdapter(mHeaderViewInfos, mFooterViewInfos, adapter);
    } else {
        mAdapter = adapter;
    }

    mOldSelectedPosition = INVALID_POSITION;
    mOldSelectedRowId = INVALID_ROW_ID;

    // AbsListView#setAdapter will update choice mode states.
    super.setAdapter(adapter);

    if (mAdapter != null) {
        //获取最新的Adapter信息
        mAreAllItemsSelectable = mAdapter.areAllItemsEnabled();
        mOldItemCount = mItemCount;
        mItemCount = mAdapter.getCount();
        checkFocus();

        //绑定数据观察者
        mDataSetObserver = new AdapterDataSetObserver();
        mAdapter.registerDataSetObserver(mDataSetObserver);

        mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());

        //查找可选的position并进行设置
        int position;
        if (mStackFromBottom) {
            position = lookForSelectablePosition(mItemCount - 1, false);
        } else {
            position = lookForSelectablePosition(0, true);
        }
        setSelectedPositionInt(position);
        setNextSelectedPositionInt(position);

        if (mItemCount == 0) {
            // Nothing selected
            checkSelectionChanged();
        }
    } else {
        mAreAllItemsSelectable = true;
        checkFocus();
        // Nothing selected
        checkSelectionChanged();
    }

    requestLayout();
}
```

## Q4. Adapter里各接口的作用？
系统基本的Adapter结构为抽象类BaseAdapter实现了ListAdapter和SpinnerAdapter两个接口，这两个接口又都继承了Adapter。
所以核心的方法还是在Adapter中，接下来看一下Adapter中的方法：

- registerDataSetObserver：注册一个观察者监听Adapter的数据变化
- unregisterDataSetObserver：注销一个观察者
- getCount：Adapter中的数据数量
- getItem：获得指定位置的数据
- getItemId：主要在hasStableIds为true时调用，赋予每个子View一个id
- hasStableIds：此方法对应的是AbsListView中的mAdapterHasStableIds变量，其主要在TransientState的操作中出现，
瞬时态View的保存采用的是SparseArray，普通情况下key是position，而有了id后，这个稀疏数组会按照id的顺序进行排列，
根据网上的资料这个顺序决定了ListView的刷新顺序。AbsListView中其他用到这个变量的是RecyclerBin中的retrieveFromScrap方法，
如果此变量为true，那么子View在回收重用的时候也会参考这个id，尽可能的返回id相同的View。
- getView： 根据数据和位置生成对应的子View，核心方法
- getItemViewType：子View的种类
- getViewTypeCount：子View种类数量
- isEmpty：判断Adapter中是否有数据，主要用于带Header或者footer的空态View的展示。

## Q5. base中的MeasuredListView的原理？
```java
int expandSpec = MeasureSpec.makeMeasureSpec(Integer.MAX_VALUE >> 2, MeasureSpec.AT_MOST);
```
size和mode组成起来是一个完整的int值，其中前两位是mode，后面30位代表size，所以上面这行的代码表示mode为AT_MOST，即为可达到的父控件最大值，
而值的最大值能是多少呢？也就是30位长度的整型的最大值，也就是Integer.MAX_VALUE >> 2。
