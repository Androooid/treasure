# ListView原理分析

## 1. 简介
**ListView**是android原生控件中比较复杂的，它通过使用Adapter的方法去绑定数据源，并能够根据数据源来刷新显示。另一方面**ListView**可以加载非常多的数据而不会内存溢出，甚至在滑动加载新的item的过程中内存占用都不会有多少变化。

Adapter的作用和使用在这里就不多说了，其主要是为了使ListView具有良好的扩展性，能够适配不同的数据源，并且不用很多的改动View层的展示逻辑。另外使用Adapter需要重写一个比较重要的方法getView，这个方法与**ListView**的机制有着非常紧密的关系。

## 2. 核心机制－RecycleBin

**RecycleBin**是实现**ListView**无限数据的核心部件。它是AbsListView的一个内部类，而继承于AbsListView的除了**ListView**外还有GridView。所以这两个控件的源码级View回收重用实现机制是一致的。

下面将分析**RecycleBin**中的所有主要函数：

```java
public void setViewTypeCount(int viewTypeCount) {
	if (viewTypeCount < 1) {
    	throw new IllegalArgumentException("Can't have a viewTypeCount < 1");
    }
    //noinspection unchecked
    ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];
    for (int i = 0; i < viewTypeCount; i++) {
        scrapViews[i] = new ArrayList<View>();
    }
    mViewTypeCount = viewTypeCount;
    mCurrentScrap = scrapViews[0];
    mScrapViews = scrapViews;
}
```

setViewTypeCount函数的作用是为ListView中的每种类型的数据项都建立一个RecycleBin机制，一般情况下ListView都用同一个类型的数据，所以此函数的输入一般为1。因为BaseAdapter默认返回的就是1。
通过这个函数还是能看出一些参数的作用：

- **mViewTypeCount**：RecycleBin中的数据种类
- **mCurrentScrap**：当前的废弃View的List
- **mScrapViews**：所有类型的废弃View的List的数组，相当于一个二维数组

对于一个RecycleBin来说所有数据都在mScrapViews中，而大多数的时候主要操作的都将是mCurrentScrap和mActiveViews。

```java
public void markChildrenDirty() {
	if (mViewTypeCount == 1) {
    	final ArrayList<View> scrap = mCurrentScrap;
       	final int scrapCount = scrap.size();
        for (int i = 0; i < scrapCount; i++) {
        	scrap.get(i).forceLayout();
        }
    } else {
    	final int typeCount = mViewTypeCount;
        for (int i = 0; i < typeCount; i++) {
        	final ArrayList<View> scrap = mScrapViews[i];
            final int scrapCount = scrap.size();
            for (int j = 0; j < scrapCount; j++) {
            	scrap.get(j).forceLayout();
            }
        }
    }
	if (mTransientStateViews != null) {
    	final int count = mTransientStateViews.size();
        for (int i = 0; i < count; i++) {
        	mTransientStateViews.valueAt(i).forceLayout();
        }
    }
    if (mTransientStateViewsById != null) {
    	final int count = mTransientStateViewsById.size();
    	for (int i = 0; i < count; i++) {
   			mTransientStateViewsById.valueAt(i).forceLayout();
    	}
    }
}
```
此函数对mScrapViews、mCurrentScrap、mTransientStateViews以及mTransientStateViewsById中的每个View都进行了forceLayout的操作，此操作并不会重绘界面，但它会使这些View在下次laout的过程中进行布局。

clear函数的内容就不贴了，他会清空mCurrentScrap、mScrapViews以及两个TransientStateViews中的数据。

```java
void fillActiveViews(int childCount, int firstActivePosition) {
	if (mActiveViews.length < childCount) {
    	mActiveViews = new View[childCount];
    }
    mFirstActivePosition = firstActivePosition;

    //noinspection MismatchedReadAndWriteOfArray
    final View[] activeViews = mActiveViews;
	    for (int i = 0; i < childCount; i++) {
        	View child = getChildAt(i);
            AbsListView.LayoutParams lp = (AbsListView.LayoutParams) child.getLayoutParams();
            // Don't put header or footer views into the scrap heap
            if (lp != null && lp.viewType != ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
            	// Note:  We do place AdapterView.ITEM_VIEW_TYPE_IGNORE in active views.
                //        However, we will NOT place them into scrap views.
            	activeViews[i] = child;
            	}
            }
}
```
此函数根据传入的childCount，用当前View的子View填充mActiveViews，

```java
View getActiveView(int position) {
    int index = position - mFirstActivePosition;
    final View[] activeViews = mActiveViews;
    if (index >=0 && index < activeViews.length) {
        final View match = activeViews[index];
        activeViews[index] = null;
        return match;
    }
    return null;
}
```
通过转换把position转成了mActiveViews的下标，并把此下标对应的View返回，并把它置为null。
getTransientStateView和clearTransientStateViews两个函数从字面就可以看出是对mTransientStateViews和mTransientStateViewsById进行的操作。

```java
View getScrapView(int position) {
	if (mViewTypeCount == 1) {
		return retrieveFromScrap(mCurrentScrap, position);
	} else {
		final int whichScrap = mAdapter.getItemViewType(position);
		if (whichScrap >= 0 && whichScrap < mScrapViews.length) {
			return retrieveFromScrap(mScrapViews[whichScrap], position);
		}
	}
	return null;
}
```
这里仅看mViewTypeCount为1的情况，所以基本上此函数就做了一次调用retrieveFromScrap()的工作。
```java
private View retrieveFromScrap(ArrayList<View> scrapViews, int position) {
    final int size = scrapViews.size();
    if (size > 0) {
        // See if we still have a view for this position or ID.
        for (int i = 0; i < size; i++) {
            final View view = scrapViews.get(i);
            final AbsListView.LayoutParams params =
                    (AbsListView.LayoutParams) view.getLayoutParams();

            if (mAdapterHasStableIds) {
                final long id = mAdapter.getItemId(position);
                if (id == params.itemId) {
                    return scrapViews.remove(i);
                }
            } else if (params.scrappedFromPosition == position) {
                final View scrap = scrapViews.remove(i);
                clearAccessibilityFromScrap(scrap);
                return scrap;
            }
        }
        final View scrap = scrapViews.remove(size - 1);
        clearAccessibilityFromScrap(scrap);
        return scrap;
    } else {
        return null;
    }
}
```
retrieveFromScrap函数主要是对mCurrentScrap进行操作。首先它会遍历一遍，如果没有满足它的条件就会直接把最后一个元素移除并返回出来。
遍历过程中判断条件一共有两种：itemId或者position，其中mAdapterHasStableIds决定了使用itemid还是position，BaseAdapter默认此值为false，所以可以主要关注position的部分。
```java
void addScrapView(View scrap, int position) {
    final AbsListView.LayoutParams lp = (AbsListView.LayoutParams) scrap.getLayoutParams();
    if (lp == null) {
        return;
    }
    lp.scrappedFromPosition = position;
    final int viewType = lp.viewType;
    if (!shouldRecycleViewType(viewType)) {
        return;
    }
    scrap.dispatchStartTemporaryDetach();
    notifyViewAccessibilityStateChangedIfNeeded(
            AccessibilityEvent.CONTENT_CHANGE_TYPE_SUBTREE);
    final boolean scrapHasTransientState = scrap.hasTransientState();
    if (scrapHasTransientState) {
        ....
    } else {
        if (mViewTypeCount == 1) {
            mCurrentScrap.add(scrap);
        } else {
            mScrapViews[viewType].add(scrap);
        }

        if (mRecyclerListener != null) {
            mRecyclerListener.onMovedToScrapHeap(scrap);
        }
    }
}
```
addScrapView函数主要功能是把view放入到缓存中去。
另有一个函数名为scrapActiveViews的功能为主要是把mActiveViews的值移动到mScrapViews中去。
所以总的来说RecycleBin中主要有两个存储变量：mActiveViews以及mCurrentScrap

## 3. Layout

对于一个View来说，onMeasure、onLayout、onDraw还是必不可少的步骤。**ListView**中的onDraw部分交由了Adapter去完成，所以本身的onDraw是没有什么实际意义的。而**ListView**的onMeasure和onLayout都是在其父类AbsListView中实现的。
这里主要关注onLayout。

```java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    super.onLayout(changed, l, t, r, b);

    mInLayout = true;

    final int childCount = getChildCount();
    if (changed) {
        for (int i = 0; i < childCount; i++) {
            getChildAt(i).forceLayout();
        }
        mRecycler.markChildrenDirty();
    }

    layoutChildren();
    mInLayout = false;

    mOverscrollMax = (b - t) / OVERSCROLL_LIMIT_DIVISOR;

    // TODO: Move somewhere sane. This doesn't belong in onLayout().
    if (mFastScroll != null) {
        mFastScroll.onItemCountChanged(getChildCount(), mItemCount);
    }
}
```
其主要关键就是这个changed变量，当其为true时，会让其子元素强制重绘，并会调用RecycleBin的markChildrenDirty函数（上文已经提及）同样强制重绘。最后都会调用layoutChildren这个函数进行布局。这个函数则是在**ListView**中进行了重写，所以最关键的部分还是由各个子类自己完成。

```java
final RecycleBin recycleBin = mRecycler;
if (dataChanged) {
    for (int i = 0; i < childCount; i++) {
        recycleBin.addScrapView(getChildAt(i), firstPosition+i);
    }
} else {
    recycleBin.fillActiveViews(childCount, firstPosition);
}
```
这里是此函数中首先出现RecycleBin的位置，那么第一次执行的时候由于数据没有产生变化，所以会进入else流程。而在此时由于还没有到Adapter的draw流程，所以childCount是0，此行没有实际作用。接着继续执行会到一个switch语句，这个语句的default就是默认执行的部分。

```java
if (childCount == 0) {
    if (!mStackFromBottom) {
        final int position = lookForSelectablePosition(0, true);
        setSelectedPositionInt(position);
        sel = fillFromTop(childrenTop);
    } else {
        final int position = lookForSelectablePosition(mItemCount - 1, false);
        setSelectedPositionInt(position);
        sel = fillUp(mItemCount - 1, childrenBottom);
    }
} else {
    if (mSelectedPosition >= 0 && mSelectedPosition < mItemCount) {
        sel = fillSpecific(mSelectedPosition,
                oldSel == null ? childrenTop : oldSel.getTop());
    } else if (mFirstPosition < mItemCount) {
        sel = fillSpecific(mFirstPosition,
                oldFirst == null ? childrenTop : oldFirst.getTop());
    } else {
        sel = fillSpecific(0, childrenTop);
    }
}
```
这部分主要就是一堆fill函数了，默认应该会进fillFromTop

```java
private View fillFromTop(int nextTop) {
    mFirstPosition = Math.min(mFirstPosition, mSelectedPosition);
    mFirstPosition = Math.min(mFirstPosition, mItemCount - 1);
    if (mFirstPosition < 0) {
        mFirstPosition = 0;
    }
    return fillDown(mFirstPosition, nextTop);
}
```
这个函数主要判断了一下mFirstPosition的合法性，随后就调用了fillDown函数。

```java
private View fillDown(int pos, int nextTop) {
View selectedView = null;

int end = (mBottom - mTop);
if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
    end -= mListPadding.bottom;
}

while (nextTop < end && pos < mItemCount) {
    // is this the selected item?
    boolean selected = pos == mSelectedPosition;
    View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);

    nextTop = child.getBottom() + mDividerHeight;
    if (selected) {
        selectedView = child;
    }
    pos++;
}

setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
return selectedView;
}
```
这里面end可以近乎认为是屏幕的高度，所以while的判断条件有以下两个

- 当前子元素没有超出屏幕高度
- 子元素在Adapter中有效

其核心是调用了makeAndAddView函数

```java
private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
        boolean selected) {
    View child;


    if (!mDataChanged) {
        // Try to use an existing view for this position
        child = mRecycler.getActiveView(position);
        if (child != null) {
            // Found it -- we're using an existing child
            // This just needs to be positioned
            setupChild(child, position, y, flow, childrenLeft, selected, true);

            return child;
        }
    }

    // Make a new view for this position, or convert an unused view if possible
    child = obtainView(position, mIsScrap);

    // This needs to be positioned and measured
    setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

    return child;
}
```
然后因为是第一次执行，正如前面所说的mActiveViews是空的，所以child是null，那么会运行到if外部的obtainView和setupChild。

```java
View obtainView(int position, boolean[] isScrap) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "obtainView");

    isScrap[0] = false;

    // Check whether we have a transient state view. Attempt to re-bind the
    // data and discard the view if we fail.
    final View transientView = mRecycler.getTransientStateView(position);
    if (transientView != null) {
        final LayoutParams params = (LayoutParams) transientView.getLayoutParams();

        // If the view type hasn't changed, attempt to re-bind the data.
        if (params.viewType == mAdapter.getItemViewType(position)) {
            final View updatedView = mAdapter.getView(position, transientView, this);

            // If we failed to re-bind the data, scrap the obtained view.
            if (updatedView != transientView) {
                setItemViewLayoutParams(updatedView, position);
                mRecycler.addScrapView(updatedView, position);
            }
        }

        // Scrap view implies temporary detachment.
        isScrap[0] = true;
        return transientView;
    }

    final View scrapView = mRecycler.getScrapView(position);
    final View child = mAdapter.getView(position, scrapView, this);
    if (scrapView != null) {
        if (child != scrapView) {
            // Failed to re-bind the data, return scrap to the heap.
            mRecycler.addScrapView(scrapView, position);
        } else {
            isScrap[0] = true;

            child.dispatchFinishTemporaryDetach();
        }
    }

    if (mCacheColorHint != 0) {
        child.setDrawingCacheBackgroundColor(mCacheColorHint);
    }

    if (child.getImportantForAccessibility() == IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
        child.setImportantForAccessibility(IMPORTANT_FOR_ACCESSIBILITY_YES);
    }

    setItemViewLayoutParams(child, position);

    if (AccessibilityManager.getInstance(mContext).isEnabled()) {
        if (mAccessibilityDelegate == null) {
            mAccessibilityDelegate = new ListItemAccessibilityDelegate();
        }
        if (child.getAccessibilityDelegate() == null) {
            child.setAccessibilityDelegate(mAccessibilityDelegate);
        }
    }

    Trace.traceEnd(Trace.TRACE_TAG_VIEW);

    return child;
}

```
obtainView从其返回类型就知道，这是**ListView**的核心部分。首先调用了getTransientStateView，默认返回为null。随后就调用了getScrapView以及Adapter的getView方法。此时获取的scrapView一定还是为null的，所以跳过了if语句。接着对这个View进行了一系列的设置后返回了这个View。

至此obtainView运行结束，接着回到makeAndAddView中去运行setupChild，这个函数东西比较多，但核心是吧获得的View加入到整个ListView中去。而在调用makeAndAddView的fillDown中，前文也提到了它只会添加一屏幕的子View，所以即使再多的数据，**ListView**也不会OOM。

在Android中许多View的调用都会有两次的onMeasure和onLayout，那么**ListView**在第二次的时候会怎么样呢？它仍旧会进入layoutChildren函数，而这次childCount就不会为0，而是一屏中显示的item的个数，接着运行到fillActiveViews时，就会把子View存入到mActiveViews中去。接着会运行detachAllViewsFromParent()函数，这个函数在第一次运行时不会有任何效果，而在这次运行过程中就会把所有的子View从**ListView**中移除。随后到switch语句中的default部分，并进入else逻辑。其中else中又有3个分支，此时会进入第二个，其中运行了fillSpecific函数。

```java
private View fillSpecific(int position, int top) {
    boolean tempIsSelected = position == mSelectedPosition;
    View temp = makeAndAddView(position, top, true, mListPadding.left, tempIsSelected);
    // Possibly changed again in fillUp if we add rows above this one.
    mFirstPosition = position;

    View above;
    View below;

    final int dividerHeight = mDividerHeight;
    if (!mStackFromBottom) {
        above = fillUp(position - 1, temp.getTop() - dividerHeight);
        // This will correct for the top of the first view not touching the top of the list
        adjustViewsUpOrDown();
        below = fillDown(position + 1, temp.getBottom() + dividerHeight);
        int childCount = getChildCount();
        if (childCount > 0) {
            correctTooHigh(childCount);
        }
    } else {
        below = fillDown(position + 1, temp.getBottom() + dividerHeight);
        // This will correct for the bottom of the last view not touching the bottom of the list
        adjustViewsUpOrDown();
        above = fillUp(position - 1, temp.getTop() - dividerHeight);
        int childCount = getChildCount();
        if (childCount > 0) {
             correctTooLow(childCount);
        }
    }

    if (tempIsSelected) {
        return temp;
    } else if (above != null) {
        return above;
    } else {
        return below;
    }
}

```
这个函数和其他fill的函数也是类似的，是把一个指定位置的View添加到屏幕上去，其核心还是makeAndAddView函数。这时其中的child就会有值，那么就不会运行obtainView函数，也就不会运行Adapter中的getView了，运行效率会大大提高。接着运行setupChild函数，这时最后一个参数传入的是true，那么就不会运行addViewInLayout而是attachViewToParent，至此第二次Layout结束。

## 4. 滑动数据加载

既然是滑动，那么肯定是在onTouchEvent中处理事件的，MotionEvent.ACTION_MOVE中由onTouchMove(ev, vtev)进行处理，其中TOUCH_MODE_SCROLL状态下执行的是scrollIfNeeded函数，最终会执行到trackMotionScroll。

这个函数首先对传入的incrementalDeltaY和deltaY进行了计算和判断从而得出了down的值，假设down为true，那么将执行如下代码：

```java
int top = -incrementalDeltaY;
if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
    top += listPadding.top;
}
for (int i = 0; i < childCount; i++) {
    final View child = getChildAt(i);
    if (child.getBottom() >= top) {
        break;
    } else {
        count++;
        int position = firstPosition + i;
        if (position >= headerViewsCount && position < footerViewsStart) {
            // The view will be rebound to new data, clear any
            // system-managed transient state.
            if (child.isAccessibilityFocused()) {
                child.clearAccessibilityFocus();
            }
            mRecycler.addScrapView(child, position);
        }
    }
}
```
这里要明确一点：android的XY轴坐标原点是在左上角的，所以incrementalDeltaY为负的情况下，手指是向上的，而对于整个**ListView**来说看到的部分是在向下移动的，所以这里的代码含义为：对于那些当前显示的View的getBottom()值如果小于top，则意味着此子View将要被移出屏幕，那么就调用了addScrapView把这些View加入到mCurrentScrap中去。随后调用了detachViewsFromParent(start, count)把这些View从屏幕上detach掉。接着使用offsetChildrenTopAndBottom(incrementalDeltaY)来让所有子View偏移一段距离来显示。至此所有已经存在的子View的显示已经到了一个合适的位置。那对于那些新加入的子View则是通过fillGap函数来显示的。

```java
void fillGap(boolean down) {
    final int count = getChildCount();
    if (down) {
        int paddingTop = 0;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            paddingTop = getListPaddingTop();
        }
        final int startOffset = count > 0 ? getChildAt(count - 1).getBottom() + mDividerHeight :
                paddingTop;
        fillDown(mFirstPosition + count, startOffset);
        correctTooHigh(getChildCount());
    } else {
        int paddingBottom = 0;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            paddingBottom = getListPaddingBottom();
        }
        final int startOffset = count > 0 ? getChildAt(0).getTop() - mDividerHeight :
                getHeight() - paddingBottom;
        fillUp(mFirstPosition - 1, startOffset);
        correctTooLow(getChildCount());
    }
}
```
这里面主要根据方向来调用fillDown还是fillUp函数，这两个函数最后还是会调用makeAndAddView函数。由于在第二次Layout的时候mActiveViews已经被使用了，所以这个变量已经被清空了，那么child会变成null，程序会继续运行从而调用了obtainView函数。

```java
final View scrapView = mRecycler.getScrapView(position);
final View child = mAdapter.getView(position, scrapView, this);
if (scrapView != null) {
    if (child != scrapView) {
        // Failed to re-bind the data, return scrap to the heap.
        mRecycler.addScrapView(scrapView, position);
    } else {
        isScrap[0] = true;

        child.dispatchFinishTemporaryDetach();
    }
}
```
与第一次调用时的区别主要在这里，scrapView此时不会为null，这是因为刚才已经调用过了addScrapView，把移出屏幕的View加入到了mCurrentScrap中去.随后还会把scrapView传给Adapter的getView方法进行重用，如果无法就行重用，那么child将会不等于scrapView，这个scrapView会重新放回到mCurrentScrap中去。

从这里的逻辑就可以看出我们平时在重写Adapter的getView应该干什么，首先应该要判断convertView是否为null，这样就可以省下了inflate布局的时间，而使用viewholder方法则会省略findviewbyid的时间。故在写adapterView的时候（特别是数据数量较多的时候）强烈推荐使用viewholder，而且一定要判断covertview是否为null，不然底层的ListView的回收机制就不会起到作用，太浪费了！！！

## 5. ListView总结

```
onLayout(Abs)
      |
layoutChildren --- fillActiveViews (无用，因为childCount为0)
               --- detachAllViewsFromParent (无用，因为ListView中还没有子View)
               --- fillFromTop
 						|              
                   fillDown(控制只添加一屏的View)
                   		|
                   makeAndAddView --- getActiveView
                                  --- obtainView    --- getScrapyView(空)
                                  --- setupChild    --- getView
                                      		|
                                      addViewinLayout
```
```
onLayout(Abs)
	  |
layoutChildren --- fillActiveViews (填充了mActiveViews)
               --- detachAllViewsFromParent (全部detach)
               --- fillSpecific
 						|              
                   fillUp/fillDown(控制只添加一屏的View)
                   		|
                   makeAndAddView --- getActiveView
                                  --- setupChild
  											|
                                      attachViewToParent
```
```
onTouchEvent(Abs)
		|
onTouchMove(Abs)
		|
scrollIfNeeded(Abs)
		|
trackMotionScroll(Abs) --- addScrapView(循环)
                       --- detachViewsFromParent
                       --- offsetChildrenTopAndBottom (偏移)
                       --- fillGap
                              |
                           fillUp/fillDown
                              |
                           makeAndAddView --- getActiveView (null)
                                          --- obtainView   --- getScrapView
                                          --- setupChild   --- getView		
```
## 6.参考资料
<http://blog.csdn.net/guolin_blog/article/details/44996879>
