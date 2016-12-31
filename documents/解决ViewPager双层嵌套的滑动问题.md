#解决ViewPager双层嵌套的滑动问题

这是我第一次写博客，一直是在别人博客上寻找自己想要的，积累了不少，感觉是时候展现或者释放出来了。

写技术的博客有很多因素，我最大的因素可能是自己记不住这些已经用过的方法、经验，写下来，大家可以一起探讨学习，也可以在日后需要的时候回来找到它。

今天我分享一下ViewPager的双层嵌套时影响内部ViewPager的触摸滑动问题。

之前在做自己的一个项目的时候，遇到广告栏图片动态切换，我第一时间想到的就是ViewPager，整个软件只有广告这一部分ViewPager还好说，但是软件越复杂出现的问题越多，尤其是遇到ViewPager双层嵌套问题，找了很多资料 

----------

**解决方法一：自定义ViewPager做为父ViewPager控件** 

```
public class ParentViewPager extends ViewPager {
	private int childVPHeight = 0;

	public ParentViewPager(Context context) {
		super(context);
		init(context);
	}

	public ParentViewPager(Context context, AttributeSet attrs) {
		super(context, attrs); 
		init(context);
	}

	private void init(Context context) {
		// 获取屏幕宽高
		WindowManager windowManager = (WindowManager) context.getSystemService(context.WINDOW_SERVICE);
		int disWidth = windowManager.getDefaultDisplay().getWidth();
		// 根据屏幕的密度来过去dp值相应的px值
		childVPHeight = (int) (context.getResources().getDisplayMetrics().density * disWidth + 0.5f);
	}

	@Override
	public boolean onInterceptTouchEvent(MotionEvent arg0) {
		// 触摸在子ViewPager所在的页面和子ViewPager控件高度之内时
		// 返回false，此时将会将触摸的动作传给子ViewPager
		if (getCurrentItem() == 1 && arg0.getY() < childVPHeight) {
			return false;
		}
		return super.onInterceptTouchEvent(arg0);
	}
}
```

此方法虽然简单可行，但是会出现，子ViewPager如果为ScrollView的时候，子ViewPager虽然已经滑动到看不到的地方，但是设定的高度内还是不能让父ViewPager左右滑动，onTouch的动作透过了父Viewpager传递到了子控件 

----------
**解决方法二：自定义Viewpager做为子控件** 

```
public class ChildViewPager extends ViewPager {
	/** 触摸时按下的点 **/
	PointF downP = new PointF();
	/** 触摸时当前的点 **/
	PointF curP = new PointF();
	
	OnSingleTouchListener onSingleTouchListener;
	
	public ChildViewPager(Context context, AttributeSet attrs) {
		super(context, attrs);
	}

	public ChildViewPager(Context context) {
		super(context);
	}

	@Override
	public boolean onTouchEvent(MotionEvent arg0) {
		// 每次进行onTouch事件都记录当前的按下的坐标
		curP.x = arg0.getX();
		curP.y = arg0.getY();
		if (arg0.getAction() == MotionEvent.ACTION_DOWN) {
			// 记录按下时候的坐标
			// 切记不可用 downP = curP ，这样在改变curP的时候，downP也会改变
			downP.x = arg0.getX();
			downP.y = arg0.getY();
			// 此句代码是为了通知他的父ViewPager现在进行的是本控件的操作，不要对我的操作进行干扰
			getParent().requestDisallowInterceptTouchEvent(true);
		}
		int curIndex = getCurrentItem();
		if (curIndex == 0) {
			if (downP.x <= curP.x) {
				getParent().requestDisallowInterceptTouchEvent(false);
			} else {
				getParent().requestDisallowInterceptTouchEvent(true);
			}
		} else if (curIndex == getAdapter().getCount() - 1) {
			if (downP.x >= curP.x) {
				getParent().requestDisallowInterceptTouchEvent(false);
			} else {
				getParent().requestDisallowInterceptTouchEvent(true);
			}
		} else {
			getParent().requestDisallowInterceptTouchEvent(true);
		}

		if (arg0.getAction() == MotionEvent.ACTION_UP) {
			// 在up时判断是否按下和松手的坐标为一个点
			// 如果是一个点，将执行点击事件，这是我自己写的点击事件，而不是onclick
			if (downP.x == curP.x && downP.y == curP.y) {
				onSingleTouch();
				return true;
			}
		}
		return super.onTouchEvent(arg0);
	}

	/** * 单击 */
	public void onSingleTouch() {
		if (onSingleTouchListener != null) {
			onSingleTouchListener.onSingleTouch();
		}
	}

	/** * 创建点击事件接口 */
	public interface OnSingleTouchListener {
		public void onSingleTouch();
	}

	public void setOnSingleTouchListener(
			OnSingleTouchListener onSingleTouchListener) {
		this.onSingleTouchListener = onSingleTouchListener;
	}
} 
```

为什么要自己定义onSingleTouch呢？
因为在ViewPager的onTouchEvent中我对onDown进行了操作，进行了操作后就无法将touch事件继续往下传给onClick和其内部控件的任何事件，所以自己做了判断，做了个singleTouch来实现点击的事件 
方法二可以完美解决双层ViewPager嵌套后子ViewPager的触摸滑动问题，并实现了类似网易新闻的那样，可以在子view滑动到边缘时，滑动父层