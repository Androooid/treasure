# PinnedListView实现原理

## 1.概述

此ListView可以实现类似于ele.me的菜单页右列的效果，下面的标题可以把已经固顶的标题往上推并最后固顶的效果，具体请参照ele.me、美团外卖、百度外卖等都是类似效果。

## 2.实现方式
主要的核心原理就是以下四点：

1. 用ListView的ViewType把子View分为两类，一类为标题、一类为内容，其中一类设置为需要固顶的类型（例如标题）。
2. 在ListView的滚动位置处于任意时刻，根据ListView当前显示的第一个子View向前查找，当获取到一个标题类型的时候，把此View绘制到ListView的顶部（滚动的时候实时监听）。
3. 同时如果此时页面中显示的item中存在一个已经显示的标题类型的View，那么需要判断它是否与固顶的View重叠，如果重叠则需要把固顶的View进行偏移。

另需要注意ListView中的点击事件分发，主要是固顶区域及其下方覆盖的正常区域的点击事件。

### 2.1 onScroll方法

核心需要重写的方法是ListView的OnScrollListener中的onScroll方法，其代码为：

```java
//获取ListView的Adapter，并处理异常情况
ListAdapter adapter = getAdapter();
if (adapter == null || visibleItemCount == 0) return;

final boolean isFirstVisibleItemSection =
        isItemViewTypePinned(adapter, adapter.getItemViewType(firstVisibleItem));

//判断当前显示的第一个子View是否是标题类型
if (isFirstVisibleItemSection) {
    View sectionView = getChildAt(0);
    if (sectionView.getTop() == getPaddingTop()){//临界状态，标题item与固顶item重合         
    	destroyPinnedShadow();
    } else {        
    	ensureShadowForPosition(firstVisibleItem, firstVisibleItem, visibleItemCount);
    }
} else {
    int sectionPosition = findCurrentSectionPosition(firstVisibleItem);
    if (sectionPosition > -1) {
        ensureShadowForPosition(sectionPosition, firstVisibleItem, visibleItemCount);
    } else {
        destroyPinnedShadow();
    }
}
```

onScroll方法的逻辑为：

1. 判断第一个子View是不是需要固顶的类型。
2. 如果是，则直接获取ListView的第一个子View。如果它的顶部和ListView的顶部高度一致，如下所示图，那么说明此时的固顶View需要更换了。但此时并不知道是从上往下滑还是从下往上滑，所以交给后续的onScroll触发进行处理。故此时可以统一把固顶的View移除掉，直接显示ListView的默认内容即可。

<img src="../img/pinnedlistview/example.png" width="400" height="200"/>
3. 如果高度不一致，则说明是在上面的状态的基础上又往上滑动了一小段距离，如下图所示，此时就需要把固顶的View绘制出来了。
4. 再回到最初的判断，如果第一个子View不是需要固顶的类型，那么就需要从这个子View在Adapter中的位置向前遍历找到一个标题类型。如果找到了就把固顶的View绘制出来，否则就销毁当前固顶的View。

### 2.2 绘制固顶View

那么剩下需要解决的就是如何把固顶的View绘制出来，其中也有两个步骤：

 - 创建一个固顶的pinnedView
 - 根据下一个标题的View的位置，对此时固顶的View进行偏移处理。

代码如下：

```java
if (visibleItemCount < 2) {//边界条件，可见item数量小于2
    destroyPinnedShadow();
    return;
}

if (mPinnedSection != null && mPinnedSection.position != sectionPosition) {
    destroyPinnedShadow();  //固顶的View需要替换为新的View
}

if (mPinnedSection == null) { //当前没有固顶的View时，创建一个
    createPinnedShadow(sectionPosition);
}

//对当前固顶View需要的偏移量进行计算
int nextPosition = sectionPosition + 1;
if (nextPosition < getCount()) {
    int nextSectionPosition = findFirstVisibleSectionPosition(nextPosition,
            visibleItemCount - (nextPosition - firstVisibleItem));
    if (nextSectionPosition > -1) {
        View nextSectionView = getChildAt(nextSectionPosition - firstVisibleItem);
        final int bottom = mPinnedSection.view.getBottom() + getPaddingTop();
        mSectionsDistanceY = nextSectionView.getTop() - bottom;
        if (mSectionsDistanceY < 0) {
            mTranslateY = mSectionsDistanceY;
        } else {
            mTranslateY = 0;
        }
    } else {
        mTranslateY = 0;
        mSectionsDistanceY = Integer.MAX_VALUE;
    }
}
```
首先介绍一下计算偏移的过程：

通过对当前在屏幕上显示的View进行遍历（剔除第一个可见View，防止当前第一个View就是需要固顶的View），如果找到了，并且找到的View的顶部比固顶的View的底部位置值要小，就说明它们重叠了，则需要设置偏移值，其他情况偏移值为零。

创造当前固顶的View的代码如下：

```java
PinnedSection pinnedShadow = mRecycleSection;
mRecycleSection = null;

//使用Adapter的getView获取View
if (pinnedShadow == null) pinnedShadow = new PinnedSection();
View pinnedView = getAdapter().getView(position, pinnedShadow.view, PinnedSectionListView.this);

//测量和布局这个View
LayoutParams layoutParams = (LayoutParams) pinnedView.getLayoutParams();
if (layoutParams == null) {
    layoutParams = (LayoutParams) generateDefaultLayoutParams();
    pinnedView.setLayoutParams(layoutParams);
}

int heightMode = MeasureSpec.getMode(layoutParams.height);
int heightSize = MeasureSpec.getSize(layoutParams.height);
if (heightMode == MeasureSpec.UNSPECIFIED) heightMode = MeasureSpec.EXACTLY;
int maxHeight = getHeight() - getListPaddingTop() - getListPaddingBottom();
if (heightSize > maxHeight) heightSize = maxHeight;

int ws = MeasureSpec.makeMeasureSpec(getWidth() - getListPaddingLeft() - getListPaddingRight(), MeasureSpec.EXACTLY);
int hs = MeasureSpec.makeMeasureSpec(heightSize, heightMode);
pinnedView.measure(ws, hs);
pinnedView.layout(0, 0, pinnedView.getMeasuredWidth(), pinnedView.getMeasuredHeight());
mTranslateY = 0;

//把新的固顶的View信息存储下来
pinnedShadow.view = pinnedView;
pinnedShadow.position = position;
pinnedShadow.id = getAdapter().getItemId(position);

mPinnedSection = pinnedShadow;
```

 - 创造固顶的View首先需要使用的是getAdapter的getView方法获取View对象。
 - 随后对此View进行测量和布局，主要方法是设置LayoutParams，获取高度的mode和size，随后设置对应的MeasureSpec形式的width和height，最后measure和layout出来即可。
 - 这里使用一个Recycler的对象暂存前面销毁的固顶View信息是为了避免getView中的inflate造成的资源浪费。

ListView是一个ViewGroup，所以最终的绘制都交给dispatchDraw来完成即可：

```
if (mPinnedSection != null) {

    int pLeft = getListPaddingLeft();
    int pTop = getListPaddingTop();
    View view = mPinnedSection.view;

    canvas.save();

    int clipHeight = view.getHeight() +
            (mShadowDrawable == null ? 0 : Math.min(mShadowHeight, mSectionsDistanceY));
    canvas.clipRect(pLeft, pTop, pLeft + view.getWidth(), pTop + clipHeight);

    canvas.translate(pLeft, pTop + mTranslateY);
    drawChild(canvas, mPinnedSection.view, getDrawingTime());

    canvas.restore();
}
```
流程为：

 - 先确定需要绘制的区域
 - 把canvas偏移所需的距离
 - 绘制这个View


### 2.3 容错和优化
 - 各函数需要对传入的位置信息做处理，需要在0－getCount()-1之内。
 - 可视View数量只有1的情况下，不用显示固顶的View，因为会把内容挡住。
 - 绘制的时候需要对当前的固顶View做判断，如果和需要绘制的一致，那么就可以不重新创建这个View，创建View的次数尽可能少，避免资源消耗
 - 点击事件的分发：这里的实现效果不同app有些不同，有些是把固顶View的点击事件截取了，有些还是让点击事件渗透到ListView本身去了。

### 2.4 点击事件
这里主要介绍一下如何获取固顶View的点击事件，并把它屏蔽掉。需要重写的方法dispatchTouchEvent。对不同Action的处理如下：

 - **ACTION_DOWN**：判断当前是否有固顶的View、判断按键事件的区域是否在固顶的View的区域内，如果是则记录一下需要分发按键事件的View、当前事件的坐标及这个事件，主要是在后续的action中使用。
 - **ACTION_UP**：此时如果分发事件的目标View不为空，那么就把前一个down的事件和这个up的事件都分发给此View，并返回true。
 - **ACTION_MOVE**：MOVE事件主要针对ListView可滑动这一特性需要做判断，这里需要通过ViewConfiguration获取ListView默认的最短判定的滑动距离，然后进行判断。如果小于则不做事件分发直到UP为止，如果大于就需要把之前的down事件和此次的move事件都分发下去，系统就会自动进行滑动事件响应。


## 3.参考资料：
<http://blog.csdn.net/jdsjlzx/article/details/20697257>

<https://github.com/beworker/pinned-section-listview>
