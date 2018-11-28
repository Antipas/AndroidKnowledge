# View

## 基本概念

* 每个Activity都有一个Window（接口），通常是由PhoneWindow类来实现的。 PhoneWindow将DecorView作为整个应用窗口的根View，DecorView将屏幕分成两部分：TitleView和ContentView。所有view的操作、绘制都由ViewRootImpl完成。
* setContentView() [做了什么？](https://www.jianshu.com/p/687010ccad66)
* ViewGroup extends View；
* view的位置参数：top、left、right、bottom，分别对应View的左上角和右下角相对于父容器的横纵坐标值。
* 从Android 3.0开始，view增加了x、y、translationX、translationY四个参数，这几个参数也是相对于父容器的坐标。x和y是左上角的坐标，而translationX和translationY是view左上角相对于父容器的偏移量，默认值都是0。x = left + translationX  ；y = top + translationY
* MotionEvent主要有`ACTION_UP`、`ACTION_DOWN`、`ACTION_MOVE`、`ACTION_CANCEL`等。
	1. 点击一次，事件序列是`ACTION_DOWN` -> `ACTION_UP`；
	2. 点击后滑动一会再离开，事件序列是`ACTION_DOWN -> ACTION_MOVE -> ACTION_MOVE -> … -> ACTION_UP`；
	3. `ACTION_CANCEL`触发时机：子view接受到了`ACTION_DOWN`事件，父容器拦截了`ACTION_MOVE、ACTION_UP`之后，子view会收到`ACTION_CANCEL`
	4. 如果view不处理`ACTION_DOWN`则后面的事件都不会再给它
	5. `ACTION_DOWN`会让ViewGroup重置滑动标记(requestDisallowInterceptTouchEvent)
* getRawX和getRawY是相对于手机屏幕左上角的x和y坐标。 


## 绘制流程
ViewRootImpl.performTravserals()负责对所有view**递归绘制**

1. ### onMeasure()
	* 测量模式 MeasureSpec 是一个32位的int值，其中高2位是测量的模式，低30位是测量的大小 （使用位运算是为了提高效率和节省空间）
	* `EXACTLY`：精确值模式，属性设置为精确数值或者match_parent时
	* `AT_MOST`：最大值模式，属性设置为wrap_content时
	* `UNSPECIFIED`：不指定大小测量模式，父容器不对大小限制，通常情况下用于系统内部多次Measure或在绘制自定义View时才会用到，表示一种测量的状态
	* View类默认的onMeasure()方法只支持EXACTLY模式，所以如果在自定义View的时候不重写onMeasure方法的话，就只能使用EXACTLY模式。自定义View可以响应你指定的具体的宽高值或者是`match_parent`属性，**但是如果要让自定义View支持`wrap_content`属性的话，那么就必须要重写onMeasure方法来指定`wrap_content`时view的大小。**
	* 对于DecorView，它的MeasureSpec由window尺寸和自己的LayoutParams决定
	* 对于普通View，它的MeasureSpec由父容器的MeasureSpec和自己的LayoutParams决定


2. ### onLayout()
	* 它的作用是ViewGroup确定子元素的位置，onLayout() --> layout()
	* 通过serFrame()确定view四个顶点的位置(mLeft、mRight、mTop、mBittom)
	
	#### getWidth 和 getMeasureWidth 区别
	* 在view默认实现中二者值相等
	* getWidth/getHeight 形成在layout过程中，也就是二者的赋值时机不同
	* 改写layout方法可以让 二者不同

	#### LinearLayout 和 RelativeLayout 有何不同？
	* RelativeLayout需要对其子View进行两次measure过程。而LinearLayout则只需一次measure过程，所以显然会快于RelativeLayout，但是如果LinearLayout中有weight属性，则也需要进行两次measure
	* RelativeLayout的子View如果高度和RelativeLayout不同，会引发效率问题，可以使用padding代替margin以优化此问题
	* [参考](https://www.jianshu.com/p/8a7d059da746)

3. ### onDraw()
	1. 绘制背景
	2. 绘制内容
	3. 绘制子元素
	4. 绘制滚动条



## 自定义view
* 继承view重写onDraw方法需要自己支持wrap_content，并且padding也要自己处理(onMeasure、onLayout)。继承特定的View例如TextView不需要考虑。
* 尽量不要在View中使用Handler，因为view内部本身已经提供了post系列的方法，完全可以替代Handler的作用。
* view中如果有线程或者动画，需要在onDetachedFromWindow方法中及时停止。
* 处理好view的滑动冲突情况。

## 事件分发
这是段伪代码，能说明整个流程

```
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean consume = false;
    if (onInterceptTouchEvent(ev)) {
        consume = onTouchEvent(ev);
    } else {
        consume = child.dispatchTouchEvent(ev);
    }
    return consume;
}
```

* `public boolean dispatchTouchEvent(MotionEvent ev)`
属于View的方法，**父容器、子view都有**，用来进行事件的分发。如果事件能够传递给当前view，那么此方法一定会被调用，返回结果受当前view的onTouchEvent和下级view的dispatchTouchEvent方法的影响，表示是否消耗当前事件。
* `public boolean onInterceptTouchEvent(MotionEvent event)`
在dispatchTouchEvent方法内部调用，用来判断是否拦截某个事件，**只有ViewGroup才有。**如果当前view拦截了某个事件，那么在同一个事件序列当中，此方法不会再被调用，返回结果表示是否拦截当前事件。
若返回值为True事件会传递到自己的onTouchEvent()；
若返回值为False传递到子view的dispatchTouchEvent()。
* `public boolean onTouchEvent(MotionEvent event)`
在dispatchTouchEvent方法内部调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前view无法再次接收到事件。
若返回值为True，事件由自己处理，后续事件序列让其处理；
若返回值为False，自己不消耗事件，**向上返回让父容器的onTouchEvent接受处理，一直到Activity的onTouchEvent**
* 优先级：OnTouch > onTouchEvent > OnClick > OnLongClick
* 某个view一旦开始处理事件，如果它不消耗`ACTION_DOWN`事件，那么同一事件序列的其他事件都不会再交给它来处理，并且**后续事件将重新交给它的父容器去处理(父容器的onTouchEvent方法)；**如果它消耗`ACTION_DOWN`事件，但是不消耗其他类型事件，那么这个点击事件会消失，父容器的onTouchEvent方法不会被调用，当前view依然可以收到后续的事件，但是这些事件最后都会传递给Activity处理。
* ViewGroup默认不拦截任何事件，因为它的onInterceptTouchEvent方法默认返回false
* View的enable属性不影响onTouchEvent的默认返回值。哪怕一个view是disable状态，只要它的clickable或者longClickable有一个是true，那么它的onTouchEvent就会返回true。
* ViewGroup的dispatchTouchEvent方法中有一个标志位`FLAG_DISALLOW_INTERCEPT`，这个标志位就是通过子view调用requestDisallowInterceptTouchEvent方法来设置的，一旦设置为true，那么ViewGroup不会拦截该事件。


> **由此可见，整个事件分发是由Acitivity开始的向下递归过程**


## 滑动
1. 通过view本身提供的scrollTo和scrollBy方法：操作简单，适合对view内容的滑动；
2. 通过动画给view施加平移效果来实现滑动：操作简单，适用于没有交互的view和实现复杂的动画效果；
3. 通过改变view的LayoutParams使得view重新布局从而实现滑动：操作稍微复杂，适用于有交互的view。

* scrollTo和scrollBy方法只能改变view内容的位置而不能改变view在布局中的位置。 scrollBy是基于当前位置的相对滑动，而scrollTo是基于所传参数的绝对滑动。通过View的getScrollX和getScrollY方法可以得到滑动的距离。

### Scroller
Scroller的工作原理：Scroller本身并不能实现view的滑动，它需要配合view的computeScroll方法才能完成弹性滑动的效果，它不断地让view重绘，而每一次重绘距滑动起始时间会有一个时间间隔，通过这个时间间隔Scroller就可以得出view的当前的滑动位置，知道了滑动位置就可以通过scrollTo方法来完成view的滑动。就这样，view的每一次重绘都会导致view进行小幅度的滑动，而多次的小幅度滑动就组成了弹性滑动。

使用：

 1. 初始化Scroller对象，一般在view初始化的时候同时初始化scroller； 
 2. 重写view的computeScroll方法，computeScroll方法是不会自动调用的，只能通过invalidate->draw->computeScroll来间接调用，实现循环获取scrollX和scrollY的目的，当移动过程结束之后，Scroller.computeScrollOffset方法会返回false，从而中断循环； 
 3. 调用Scroller.startScroll方法，将起始位置、偏移量以及移动时间(可选)作为参数传递给startScroll方法。

```
private void ininView(Context context) {
    setBackgroundColor(Color.BLUE);
    // 初始化Scroller
    mScroller = new Scroller(context);
}

@Override
public void computeScroll() {
    super.computeScroll();
    // 判断Scroller是否执行完毕
    if (mScroller.computeScrollOffset()) {
        ((View) getParent()).scrollTo( mScroller.getCurrX(), mScroller.getCurrY());
        // 通过重绘来不断调用computeScroll
        invalidate();//很重要
    }
}

@Override
public boolean onTouchEvent(MotionEvent event) {
    int x = (int) event.getX();
    int y = (int) event.getY();
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            lastX = (int) event.getX();
            lastY = (int) event.getY();
            break;
        case MotionEvent.ACTION_MOVE:
            int offsetX = x - lastX;
            int offsetY = y - lastY;
            ((View) getParent()).scrollBy(-offsetX, -offsetY);
            break;
        case MotionEvent.ACTION_UP:
            // 手指离开时，执行滑动过程
            View viewGroup = ((View) getParent());
            mScroller.startScroll( viewGroup.getScrollX(), viewGroup.getScrollY(),
                    -viewGroup.getScrollX(), -viewGroup.getScrollY());
            invalidate();//很重要
            break;
    }
    return true;
}

```

## 滑动冲突
#### 分两类：

* 外部滑动方向与内部滑动方向一致
* 外部滑动方向与内部滑动方向不一致

1. 外部拦截法
	* 重写父容器的onInterceptTouchEvent方法，根据适当条件来决定`return true/false`，其他均不需要做修改。 

	```
	public boolean onInterceptTouchEvent(MotionEvent event) {
	    boolean intercepted = false;
	    int x = (int) event.getX();
	    int y = (int) event.getY();
	
	    switch (event.getAction()) {
	    case MotionEvent.ACTION_DOWN: {
	        intercepted = false;
	        break;
	    }
	    case MotionEvent.ACTION_MOVE: {
	        int deltaX = x - mLastXIntercept;
	        int deltaY = y - mLastYIntercept;
	        if (父容器需要拦截当前点击事件的条件，例如：Math.abs(deltaX) > Math.abs(deltaY)) {
	            intercepted = true;
	        } else {
	            intercepted = false;
	        }
	        break;
	    }
	    case MotionEvent.ACTION_UP: {
	        intercepted = false;
	        break;
	    }
	    default:
	        break;
	    }
	
	    mLastXIntercept = x;
	    mLastYIntercept = y;
	
	    return intercepted;
	}
	```

2. 内部拦截法
	* 父容器不拦截任何事件，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗掉，否则就交给父容器来处理。**需要配合requestDisallowInterceptTouchEvent方法才能正常工作**
	
	```
	public boolean dispatchTouchEvent(MotionEvent event) {
	    int x = (int) event.getX();
	    int y = (int) event.getY();
	
	    switch (event.getAction()) {
		    case MotionEvent.ACTION_DOWN: {
		        getParent().requestDisallowInterceptTouchEvent(true);
		        break;
		    }
		    case MotionEvent.ACTION_MOVE: {
		        int deltaX = x - mLastX;
		        int deltaY = y - mLastY;
		        if (当前view需要拦截当前点击事件的条件，例如：Math.abs(deltaX) > Math.abs(deltaY)) {
		            getParent().requestDisallowInterceptTouchEvent(false);
		        }
		        break;
		    }
		    case MotionEvent.ACTION_UP: {
		        break;
		    }
		    default:
		        break;
	    }
	
	    mLastX = x;
	    mLastY = y;
	    return super.dispatchTouchEvent(event);
	}
	```
	* [外部拦截法使用示例](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_3/src/com/ryg/chapter_3/ui/HorizontalScrollViewEx.java)
	* [内部拦截法使用示例](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_3/src/com/ryg/chapter_3/ui/ListViewEx.java)



## 参考
* [参考1](https://hujiaweibujidao.github.io/blog/2015/12/01/art-of-android-development-reading-notes-3/)
* [参考2](https://hujiaweibujidao.github.io/blog/2015/12/01/art-of-android-development-reading-notes-4/)
* [这个很全面](https://www.jianshu.com/p/38015afcdb58)