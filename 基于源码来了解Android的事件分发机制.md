---
title: 基于源码来了解Android的事件分发机制
date: 2016-08-31 14:32:25
tags: Android
---

网上已经有太多的关于Android事件分发机制的文章，而今天我写这遍文章，主要是想对这段时间的学习做个总结。Android是基于事件驱动的GUI系统，所以Android的事件分发机制跟其它的GUI系统是差不多的，所以了解了Android的事件分发机制基本上也就了解了其它的GUI系统的事件分发机制。由于我们只关注分发的机制和流程，所以我选择了Android 2.1 的源码，这个版本跟后面的版本的事件分发机制的代码还是有很大区别的，但是主要的分发流程是不会变的。而且这个版本的代码关于事件分发流程相对于后面的版本少了很多噪音代码所以非常容易阅读的。还有网上的很多关于该类型的文章都是分2篇来讲解，一篇是介绍ViewGroup的分发一篇是View的分发。而我为了介绍分发的机制，需要来回在ViewGroup和View中做跳转，所以为了方便大家的阅读我决定写成一篇来介绍。

这里我们只做一个很简单的布局，一个父控件ViewGroup和一个子控件View。其实分析事件的分发机制这种简单的布局已经足够了，很复杂的布局相对于简单的布局也就多了几次递归调用而已。通过网上的其它文章我们知道事件的分发机制都是先分发到父容器，然后父容器再分发到它的子控件，如果该子控件依然是一个容器控件的话，就会继续的分发下去，直到子控件不是一个容器控件为止。

## ViewGroup的事件分发机制

先贴上ViewGroup全部的dispatchTouchEvent方法的代码


	public boolean dispatchTouchEvent(MotionEvent ev) {
        final int action = ev.getAction();
        final float xf = ev.getX();
        final float yf = ev.getY();
        final float scrolledXFloat = xf + mScrollX;
        final float scrolledYFloat = yf + mScrollY;
        final Rect frame = mTempRect;

        boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;

        if (action == MotionEvent.ACTION_DOWN) {
            if (mMotionTarget != null) {
                // this is weird, we got a pen down, but we thought it was
                // already down!
                // XXX: We should probably send an ACTION_UP to the current
                // target.
                mMotionTarget = null;
            }
            // If we're disallowing intercept or if we're allowing and we didn't
            // intercept
            if (disallowIntercept || !onInterceptTouchEvent(ev)) {
                // reset this event's action (just to protect ourselves)
                ev.setAction(MotionEvent.ACTION_DOWN);
                // We know we want to dispatch the event down, find a child
                // who can handle it, start with the front-most child.
                final int scrolledXInt = (int) scrolledXFloat;
                final int scrolledYInt = (int) scrolledYFloat;
                final View[] children = mChildren;
                final int count = mChildrenCount;
                for (int i = count - 1; i >= 0; i--) {
                    final View child = children[i];
                    if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE
                            || child.getAnimation() != null) {
                        child.getHitRect(frame);
                        if (frame.contains(scrolledXInt, scrolledYInt)) {
                            // offset the event to the view's coordinate system
                            final float xc = scrolledXFloat - child.mLeft;
                            final float yc = scrolledYFloat - child.mTop;
                            ev.setLocation(xc, yc);
                            if (child.dispatchTouchEvent(ev))  {
                                // Event handled, we have a target now.
                                mMotionTarget = child;
                                return true;
                            }
                            // The event didn't get handled, try the next view.
                            // Don't reset the event's location, it's not
                            // necessary here.
                        }
                    }
                }
            }
        }

        boolean isUpOrCancel = (action == MotionEvent.ACTION_UP) ||
                (action == MotionEvent.ACTION_CANCEL);

        if (isUpOrCancel) {
            // Note, we've already copied the previous state to our local
            // variable, so this takes effect on the next event
            mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
        }

        // The event wasn't an ACTION_DOWN, dispatch it to our target if
        // we have one.
        final View target = mMotionTarget;
        if (target == null) {
            // We don't have a target, this means we're handling the
            // event as a regular view.
            ev.setLocation(xf, yf);
            return super.dispatchTouchEvent(ev);
        }

        // if have a target, see if we're allowed to and want to intercept its
        // events
        if (!disallowIntercept && onInterceptTouchEvent(ev)) {
            final float xc = scrolledXFloat - (float) target.mLeft;
            final float yc = scrolledYFloat - (float) target.mTop;
            ev.setAction(MotionEvent.ACTION_CANCEL);
            ev.setLocation(xc, yc);
            if (!target.dispatchTouchEvent(ev)) {
                // target didn't handle ACTION_CANCEL. not much we can do
                // but they should have.
            }
            // clear the target
            mMotionTarget = null;
            // Don't dispatch this event to our own view, because we already
            // saw it when intercepting; we just want to give the following
            // event to the normal onTouchEvent().
            return true;
        }

        if (isUpOrCancel) {
            mMotionTarget = null;
        }

        // finally offset the event to the target's coordinate system and
        // dispatch the event.
        final float xc = scrolledXFloat - (float) target.mLeft;
        final float yc = scrolledYFloat - (float) target.mTop;
        ev.setLocation(xc, yc);

        return target.dispatchTouchEvent(ev);
    }

我们来依次分析源码，我们先看看参数ev，这是一个MotionEvent对象，MotionEvent对象是对当前事件的封装。通过这个对象我们可以得到当前点击相对于父容器的位置（ getX()，getY() ）和相对屏幕左上角的位置（ getRawX()，getRawY() ）,还可以通过getAction可以得到当前事件是一次ACTIONDOWN事件，还是ACTIONUP事件，或者是ACTIONMOVE。 当我们的手指点击屏幕的时候，会先触发ACTIONDOWN事件，如果移动的话会触发1到多次ACTIONMOVE事件，当手指离开屏幕的时候会触发ACTIONUP事件，如果被其它事件干扰的话会触发ACTIONCANCEL事件。一次事件序列是以ACTION_DOWN事件开始，以ACTIONUP事件或者 ACTIONCANCEL事件来结束。

		final int action = ev.getAction();
        final float xf = ev.getX();
        final float yf = ev.getY();
        final float scrolledXFloat = xf + mScrollX;
        final float scrolledYFloat = yf + mScrollY;

xf，yf是点击的位置相对于它的父容器的左上角的坐标。scrolledXFloat，scrolledYFloat 是点击的位置相对于父容器左上角的坐标加上父容器内部滚动的偏移量。mScrollX和mScrollY默认是0。

disallowIntercept 这个变量是判断当前控件是否支持拦截事件，只对父容器有用，默认是支持拦截的。如果当前是ACTIONDOWN事件且没有被拦截的情况下，我们进入到if内部，先清空 mMotionTarget这个View对象。mMotionTarget对象表示了该父容器的哪个子控件处理了ACTIONDOWN事件。然后在后续的ACTION_MOVE，ACTION_UP事件直接调用mMotionTarget对象的dispatchTouchEvent方法。

继续向下分析代码，这里会判断当前容器控件是否支持事件拦截或者当前当前控件的onInterceptTouchEvent方法是否返回flase。当没有被拦截的情况下。然后for循环mChildren这个数组对象，这个数据包含了当前父容器包含的子控件对象

	frame.contains(scrolledXInt, scrolledYInt)

这行代码来判断当前点击是位置是否在该子控件的Rect区域里面，如果在的话，该行代码会返回true。

	final float xc = scrolledXFloat - child.mLeft;
    final float yc = scrolledYFloat - child.mTop;
    ev.setLocation(xc, yc);

xc，yc表示当前点击的位置相对于当前遍历的子控件的左上角的坐标。然后把当前位置赋值给ev变量，然后把ev变量作为参数调用子控件的dispatchTouchEvent对象。由于这个子控件是个View对象，所以我们继续看View的dispatchTouchEvent方法。

## View的事件分发机制

首先我们贴上View的dispatchTouchEvent方法的代码

	public boolean dispatchTouchEvent(MotionEvent event) {
        if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&
                mOnTouchListener.onTouch(this, event)) {
            return true;
        }
        return onTouchEvent(event);
    }


View的dispatchTouchEvent方法相对于ViewGroup的dispatchTouchEvent方法还是很简单的。从上面分析的ViewGroup的dispatchTouchEvent方法我们知道，该方法参数是通过父容器传过来的。方法首先通过判断mOnTouchListener是否为null，这个mOnTouchListener对象是通过View对象的setOnTouchListener方法传递进来的，也就是判断当前View对象是否注册了Touch事件。如果注册了该事件，表示mOnTouchListener!=null。然后判断该View控件的是否设置enabled这个属性，如果前面2个条件都为true的话，然后会调用mOnTouchListener这个事件监听器的onTouch方法。如果该方法返回true就返回dispatchTouchEvent方法。如果返回false就接着调用View的onTouchEvent的方法。先看看View的onTouchEvent方法。

	public boolean onTouchEvent(MotionEvent event) {
        final int viewFlags = mViewFlags;

        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE ||
                    (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
        }

        ...

        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_UP:
                    ...

                case MotionEvent.ACTION_DOWN:
                    mPrivateFlags |= PRESSED;
                    refreshDrawableState();
                    if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) {
                        postCheckForLongClick();
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    mPrivateFlags &= ~PRESSED;
                    refreshDrawableState();
                    break;

                case MotionEvent.ACTION_MOVE:
                    ...
            }
            return true;
        }

        return false;
    }

方法的开始处判断当前控件的enabled状态，如果值是DISABLED的话，再判断当前的CLICKABLE和LONG_CLICKABLE 的值。我们选重点看 

	if ((viewFlags & ENABLED_MASK) == DISABLED) {
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE ||
                    (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
        }


	if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
			
			...

			return true;
	}


通过上面这几句代码我们可以看出只要当前控件的CLICKABLE或者LONG_CLICKABLE为true，该onTouchEvent方法就会返回true。继续分析View的onTouchEvent方法，里面有个switch语句，来判断当前事件是ACTIONUP还是ACTIONDOWN事件或者其它的。因为事件一开始从ACTIONDOWN开始的，所以我们先看看 case 是 MotionEvent.ACTIONDOWN的分支。一开始先设置当前的状态为PRESSED。然后调用refreshDrawableState()方法设置当前View的DrawableState， 然后判断当前View的是否支持长按事件，如果支持的话，调用postCheckForLongClick方法。

	private void postCheckForLongClick() {
        mHasPerformedLongPress = false;

        if (mPendingCheckForLongPress == null) {
            mPendingCheckForLongPress = new CheckForLongPress();
        }
        mPendingCheckForLongPress.rememberWindowAttachCount();
        postDelayed(mPendingCheckForLongPress, ViewConfiguration.getLongPressTimeout());
    }

通过postDelayed这个方法我们可以看出是将mPendingCheckForLongPress这个消息post到消息队列中。ViewConfiguration.getLongPressTimeout()的值是500，也就是说500ms之后，执行mPendingCheckForLongPress这个消息。我们继续看mPendingCheckForLongPress这个消息。mPendingCheckForLongPress是个Runnable对象，我们接着看这个Runnable对象的run方法

	    

        public void run() {
            if (isPressed() && (mParent != null)
                    && mOriginalWindowAttachCount == mWindowAttachCount) {
                if (performLongClick()) {
                    mHasPerformedLongPress = true;
                }
            }
        }
       
方法内部会调用performLongClick方法，我们进入到performLongClick方法
	
	public boolean performLongClick() {
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_LONG_CLICKED);

        boolean handled = false;
        if (mOnLongClickListener != null) {
            handled = mOnLongClickListener.onLongClick(View.this);
        }
        if (!handled) {
            handled = showContextMenu();
        }
        if (handled) {
            performHapticFeedback(HapticFeedbackConstants.LONG_PRESS);
        }
        return handled;
    }

该方法里面的mOnLongClickListener就是外部给View注册的长按事件监听器。通过这几段代码我们知道了给View注册的onLongClick事件是如何执行的。也就是长按View控件500ms之后，会触发onLongClick事件，前提是该View的LONG_CLICKABLE为true。 

分析完了View的dispatchTouchEvent的ACTION_DOWN事件，我们返回到ViewGroup的dispatchTouchEvent方法中，

	if (child.dispatchTouchEvent(ev))  {
                // Event handled, we have a target now.
                mMotionTarget = child;
                return true;
    }

通过分析View的dispatchTouchEvent方法我们知道，**一个控件注册了OnTouchListener事件，且该事件中的OnTouch方法返回true或者该child控件的CLICKABLE或者LONG_CLICKABLE为true，
child.dispatchTouchEvent(ev) 这个方法调用就会返回true**。然后将该child控件赋值给mMotionTarget对象，方便后续的处理。


通过ViewGroup和View 这2个类的dispatchTouchEvent方法我们分析完了ACTIONDOWN事件，接着再分析后续的事件序列：ACTIONMOVE，ACTIONUP，ACTIONCANCEL。我们接着看ViewGroup的dispatchTouchEvent方法的代码。

 	final View target = mMotionTarget;
    if (target == null) {
        // We don't have a target, this means we're handling the
        // event as a regular view.
        ev.setLocation(xf, yf);
        return super.dispatchTouchEvent(ev);
    }

先把mMotionTarget这个对象赋值给target这个局部变量，然后再判断是否为null。如果为null的话，表示没有哪个子控件会处理ACTION_DOWN事件，出现这种情况的原因是

1.该点击到的子控件的clickable和longClickable都为false，且该控件没有注册OnTouchLister事件或者注册了但是OnTouch方法返回false。

2.父容器拦截了该事件，也就是 disallowIntercept返回false且容器控件的onInterceptTouchEvent方法返回true。

只有满足了1或者2这个条件，才会出现父容器的mMotionTarget对象为null。通过 return super.dispatchTouchEvent(ev);  我们可以看出，**如果子控件没有处理该事件，父容器会自己处理该事件**，处理流程参考上面的View的dispatchTouchEvent方法中的流程。接着往下看

	if (!disallowIntercept && onInterceptTouchEvent(ev)) {

		final float xc = scrolledXFloat - (float) target.mLeft;
        final float yc = scrolledYFloat - (float) target.mTop;
        ev.setAction(MotionEvent.ACTION_CANCEL);
        ev.setLocation(xc, yc);

		if (!target.dispatchTouchEvent(ev)) {
            // target didn't handle ACTION_CANCEL. not much we can do
            // but they should have.
        }

		mMotionTarget = null;
 		return true;
		

	}

这个if语句表示该父容器控件拦截了当前事件。然后把当前的事件的类型改为ACTIONCANCEL事件，然后调用子控件dispatchTouchEvent的方法，通知子控件该事件序列已经被取消了。再将mMotionTarget置为null，这表明在该次事件序列中这是最后一次把事件分发到子控件，后面产生的MOVE或者UP事件已经跟子控件没有关系了。最后 return true。

通过上面的这个 if 语句我们可以得出：

>方法既然能运行到这里表示该拦截不是发生在ACTION_DOWN事件中。如果发生在ACTION_DOWN事件中，那么mMotionTarget就为null，如果mMotionTarget为null，在前面的if (target == null)  就会被拦截掉了。既然能运行到这个地方就表示mMotionTarget不为null，也就说明了ACTION_DOWN事件没有被拦截，而是后续ACTION_MOVE或者ACTION_CANCEL被拦截了。
 
 
通过方法的最后面的一句 return target.dispatchTouchEvent(ev);  我们看出后续的事件序列会被直接分发到在ACTIONDOWN事件中的设置的mMotionTarget子控件中做继续处理。我们再接着分析ACTIONMOVE事件，ACTIONMOVE事件的分发在ViewGroup中没什么好看的 ，我们直接进入到View的onTouchEvent方法来分析ACTION_MOVE事件，我们直接看ACTION_MOVE分支的代码

	case MotionEvent.ACTION_MOVE:
                    final int x = (int) event.getX();
                    final int y = (int) event.getY();

                    // Be lenient about moving outside of buttons
                    int slop = ViewConfiguration.get(mContext).getScaledTouchSlop();
                    if ((x < 0 - slop) || (x >= getWidth() + slop) ||
                            (y < 0 - slop) || (y >= getHeight() + slop)) {
                        // Outside button
                        if ((mPrivateFlags & PRESSED) != 0) {
                            // Remove any future long press checks
                            if (mPendingCheckForLongPress != null) {
                                removeCallbacks(mPendingCheckForLongPress);
                            }

                            // Need to switch from pressed to not pressed
                            mPrivateFlags &= ~PRESSED;
                            refreshDrawableState();
                        }
                    } else {
                        // Inside button
                        if ((mPrivateFlags & PRESSED) == 0) {
                            // Need to switch from not pressed to pressed
                            mPrivateFlags |= PRESSED;
                            refreshDrawableState();
                        }
                    }
                    break;

通过x,y记录当前的触摸的位置，ViewConfiguration.get(mContext).getScaledTouchSlop() 这条语句返回的是16，就是说slop的值是16。

	if ((x < 0 - slop) || (x >= getWidth() + slop) ||
                            (y < 0 - slop) || (y >= getHeight() + slop)) {


这条if语句是判断当前触摸的位置有没有超过当前View的边境16个像素，在16像素以内说明你当前手指还没有离开当前view，否则说明你手指已经离开了当前View的边境。如果move出了控件的范围，会移除消息队列中的长按事件的这条消息。然后移除PRESSED状态。

接着再分析ActionUp事件，跟ACTION_MOVE事件一样 ，我们直接进入到View的onTouchEvent方法中来分析ACTION_UP事件

	case MotionEvent.ACTION_UP:
                    if ((mPrivateFlags & PRESSED) != 0) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (!mHasPerformedLongPress) {
                            // This is a tap, so remove the longpress check
                            if (mPendingCheckForLongPress != null) {
                                removeCallbacks(mPendingCheckForLongPress);
                            }

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
                                performClick();
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }
                    }
                    break;

一开始会判断当前是控件的状态是否是PRESSED状态，从ActionDown和ActionMove知道，当前的状态是PRESSED状态的。然后在判断当前控件是否获得支持获得焦点，如果支持的话且没有焦点的话，使当前的控件获得焦点。然后判断是否已经执行了长按事件，如果没有执行的话且已经长按的消息不为空，会将消息队列中长按消息移除。然后执行performClick方法，先看看performClick的方法

	public boolean performClick() {
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);

        if (mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            mOnClickListener.onClick(this);
            return true;
        }

        return false;
    }

mOnClickListener变量就是为当前View对象注册的ClickListener这个Click事件监听器。如果不为null的话，就执行mOnClickListener对象中onClick方法。

## 问题解答

### 同时为View注册了OnTouchListener，OnLongClickListener，OnClickListener这三个事件监听器，这三个监听器被触发执行的先后顺序

顺序是

**OnTouchListener>OnLongClickListener>OnClickListener**

前2个事件监听器的回调方法的返回类型都是boolean，如果前面的方法返回true的话，后来的就不会去执行了。我来解释一下为什么，前面已经说过在View的dispatchTouchEvent方法中，会先执行注册的OnTouchListener的OnTouch回调方法，如果返回true的话，dispatchTouchEvent方法就会直接返回true，后面的onTouchEvent方法时不会执行的，而OnLongClickListener和
OnClickListener这2个事件监听器的触发是在onTouchEvent方法中触发的。 所以OnTouch方法返回的true的话，后面的2个事件都不会被触发。那OnLongClickListener的回调方法的返回值时如何影响到OnClickListener的执行的呢，我们再看看长按消息的代码

	class CheckForLongPress implements Runnable {

        public void run() {
            ...
            if (performLongClick()) {
                mHasPerformedLongPress = true;
            }
        
   			...
    }

performLongClick方法就会调用onLongClick事件中的回调方法，如果返回true的话mHasPerformedLongPress就为true。我们再看View的onTouchEvent方法中的MotionEvent.ACTION_UP这个分支的代码

	case MotionEvent.ACTION_UP:
                
			 ...
            if (!mHasPerformedLongPress) {
                // This is a tap, so remove the longpress check
               ...
                // Only perform take click actions if we were in the pressed state
                if (!focusTaken) {
                    performClick();
                }
            }
			...
        }
        break;

可以看出mHasPerformedLongPress为true的话，就不会执行performClick这个方法，所以也就不会执行OnClick事件中的回调方法了。

### 父容器和子控件都注册了OnTouchListener事件监听器的问题
还有一个问题，如果同时给父子容器注册了OnTouch事件的监听器，当触摸子控件的时候，为什么父容器的OnTouch事件没有被触发？

我们来继续回到ViewGroup的dispatchTouchEvent方法的代码中，我们看看

	
	if (action == MotionEvent.ACTION_DOWN) {
        ...    
            if (child.dispatchTouchEvent(ev))  {
                // Event handled, we have a target now.
                mMotionTarget = child;
                return true;
            }
    	...           
    }

	
	...

	final View target = mMotionTarget;
        if (target == null) {
            // We don't have a target, this means we're handling the
            // event as a regular view.
            ev.setLocation(xf, yf);
            return super.dispatchTouchEvent(ev);
        }


	...
	return target.dispatchTouchEvent(ev);

	

通过这几处关键代码我们知道只有当target == null的时候，才会调用当前容器控件自己的dispatchTouchEvent，不然会调用target.dispatchTouchEvent(ev)。

文章前面我们说过，在child.dispatchTouchEvent(ev)方法中，当child控件的clickable或者longClickable为true 或者注册的OnTouch事件的回调方法返回true的情况下，child.dispatchTouchEvent(ev)的方法就会返回true，如果返回true的话，mMotionTarget就不会null，所以就不会调用super.dispatchTouchEvent(ev)这个方法来触发自己的事件监听器。


**如何父子控件注册的OnTouchListener事件监听器都会被执行**

我们可以给子控件的CLICKABLE和LONG_CLICKABLE都设置为false的情况下，且子控件的OnTouchListener的OnTouch方法返回false，这样父子控件注册的事件监听器都会被触发执行。

**如何只触发父容器的注册的OnTouchListener事件监听器，而不触发子控件注册的OnTouchListener事件监听器**

我们可以重写父容器的onInterceptTouchEvent(MotionEvent ev)方法，让它返回true，表示该父容器已经拦截了事件的分发使之不会被传递到子控件中。

