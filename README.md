# [Android](https://developer.android.com/develop/index.html)中的[Toast](https://developer.android.com/reference/android/widget/Toast.html)源码分析和自定义[Toast](https://developer.android.com/reference/android/widget/Toast.html)

---

[我的主页](http://www.jianshu.com/users/64f479a1cef7/latest_articles)

[Demo下载地址](https://github.com/PingerWan/CustomToastDemo)
## 一、系统自带[Toast](https://developer.android.com/reference/android/widget/Toast.html)的源码分析

### 1. [Toast](https://developer.android.com/reference/android/widget/Toast.html)的调用显示
* 学过Android的人都知道，弹出一个系统API吐司只需要一行代码，调用的是[Toast](https://developer.android.com/reference/android/widget/Toast.html)对象的makeText()方法，方法里给一个上下文，显示的文字，和显示的时长，然后再调用show方法就能显示。


		Toast.makeText(this,"下载失败",Toast.LENGTH_SHORT).show();

### 2. [Toast](https://developer.android.com/reference/android/widget/Toast.html)对象的makeText()方法分析
* 跟进[Toast](https://developer.android.com/reference/android/widget/Toast.html)类的源码，找到makeText()方法，可以看到这是个静态方法，并且返回值仍然是Toast，所以在调用了makeText()方法之后，可以继续调用show()方法。
* 这个方法一进来就根据上下文创建了一个吐司对象，最后返回的也是这个吐司对象，而最后吐司的显示就是用这个对象里面的方法，所以我们要跟进这个Toast对象的构造。
* 方法里面的代码就是获取一个布局填充器，然后将系统布局文件转换为View的对象，再使用这个View对象里边的TextView控件显示设置的文本。

		public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
			// 根据传进来的上下文，创建一个吐司对象
	        Toast result = new Toast(context);
			
			// 布局填充器
	        LayoutInflater inflate = (LayoutInflater)
	                context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);

			// 填充系统布局文件，转换为View对象
	        View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);

			// 获取TextView文本对象，设置显示的文字
	        TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
	        tv.setText(text);
	        
			// 初始化显示的时长
	        result.mNextView = v;
	        result.mDuration = duration;
	
			// 返回处理过的吐司对象
	        return result;
   		 }


### 3. [Toast](https://developer.android.com/reference/android/widget/Toast.html)对象的show()方法分析
* 跟进第二步创建的[Toast](https://developer.android.com/reference/android/widget/Toast.html)构造方法，可以看到里面并没有关于show()方法的踪迹，但是我们可以很清楚的发现，它又创建了一个TN对象。其他的代码都是做一些初始化操作。




		public Toast(Context context) {
	        mContext = context;
	        mTN = new TN();
	        mTN.mY = context.getResources().getDimensionPixelSize(
	                com.android.internal.R.dimen.toast_y_offset);
	        mTN.mGravity = context.getResources().getInteger(
	                com.android.internal.R.integer.config_toastDefaultGravity);
	    }


* 继续跟进TN对象，可以发现这是个内部类，而且关于吐司的大部分代码都是在里面实现的。先来看一下构造方法，这里面只做了一件事情，就是创建了窗口管理器并且进行初始化，这个初始化的窗体其实就是谈吐司的窗体。


		
	     TN() {
            // XXX This should be changed to use a Dialog, with a Theme.Toast
            // defined that sets up the layout params appropriately.
			// 窗口管理器，布局参数对象
            final WindowManager.LayoutParams params = mParams;
			
			// 设置吐司的宽高是包裹内容
            params.height = WindowManager.LayoutParams.WRAP_CONTENT;
            params.width = WindowManager.LayoutParams.WRAP_CONTENT;
			// 透明
            params.format = PixelFormat.TRANSLUCENT;
			
			// 吐司弹出动画
            params.windowAnimations = com.android.internal.R.style.Animation_Toast;
            params.type = WindowManager.LayoutParams.TYPE_TOAST;
            params.setTitle("Toast");
			
			// 设置吐司窗体的标识
            params.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
                    | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
        }


* 我们再来看看TN类的其他方法，很快我们就找到了一个特别明显的方法就是show()方法和hide()方法，而这两个方法正是我们要的，继续找到mHandler的方法。



		// show()方法和hide()方法	
		 @Override
        public void show() {
            if (localLOGV) Log.v(TAG, "SHOW: " + this);
            mHandler.post(mShow);
        }

        /**
         * schedule handleHide into the right thread
         */
        @Override
        public void hide() {
            if (localLOGV) Log.v(TAG, "HIDE: " + this);
            mHandler.post(mHide);
        }


* 在mHandler的方法中，我们发现最后是调用了handleShow()和handlerHide()方法，这两个方法里有一句代码特别重要，因为这句代码就是让吐司显示出来的代码。


		 // 显示
		 public void handleShow() {
            	...

				// 窗口管理器
                mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
               
				...
				// 判断View对象的父控件不为空
                if (mView.getParent() != null) {
                    mWM.removeView(mView);
                }

				// 将当前的View添加到窗口管理器中，这句代码能让View显示出来
                mWM.addView(mView, mParams);
            }
        }

	
		// 隐藏
		public void handleHide() {
            if (mView != null) {
                // note: checking parent() just to make sure the view has
                // been added...  i have seen cases where we get here when
                // the view isn't yet added, so let's try not to crash.
                if (mView.getParent() != null) {
					// 将View对象从窗体管理器移除
                    mWM.removeView(mView);
                }

                mView = null;
            }
        }



---

## 二、自定义[Toast](https://developer.android.com/reference/android/widget/Toast.html)
> 通过以上的源码分析,我想大家已经对Android系统中[Toast](https://developer.android.com/reference/android/widget/Toast.html)的弹出和隐藏有了一定的理解，那么知道了原理，我们也可以自己做一个吐司，还可以通知指定显示的布局文件还做各种不同样式的吐司。

### 1. 自定义吐司效果图
* 明白了原理，我们可以专门做一个自定义[Toast](https://developer.android.com/reference/android/widget/Toast.html)的工具类，这个类负责弹出吐司和隐藏吐司。然后在MainActivity中调用，这样点击按钮就能弹出和隐藏吐司了。除此之外，Demo里还做了双击吐司居中，三击吐司隐藏的点击事件逻辑，大家可以学习一下。

![CustomToastDemo显示效果图](http://upload-images.jianshu.io/upload_images/2786991-f7ada3b539e2dd11.gif?imageMogr2/auto-orient/strip)

### 2. 分析工具类的代码实现
> 在Demo的[Toast](https://developer.android.com/reference/android/widget/Toast.html)工具类中，我们只需要像源代码中一样，在CustomToastUtil的构造方法中初始化LayoutParams，然后再写弹出[Toast](https://developer.android.com/reference/android/widget/Toast.html)的方法，隐藏吐司的方法就可以了。



* 工具类构造方法的实现，同源码一样，我们这里也是初始化窗体的一些基本参数，并且初始化吐司要显示的布局



		   /**
		     * 构造
		     * @param context
		     */
		    public CustomToastUtil(Context context) {
		        this.mContext = context;
		
		        initParams();
		    }
		
		    /**
		     * 初始化窗体属性
		     */
		    private void initParams() {
		        mWm = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
		
		        mParams = new WindowManager.LayoutParams();
		
		        mParams.height = WindowManager.LayoutParams.WRAP_CONTENT;
		        mParams.width = WindowManager.LayoutParams.WRAP_CONTENT;
		        mParams.flags = WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON
		                | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;
		        // 类型
		        mParams.type = WindowManager.LayoutParams.TYPE_PHONE;
		
		        // 透明，不透明会出现重叠效果
		        mParams.format = PixelFormat.TRANSLUCENT;
		
		        // 位置属性
		        mParams.gravity = Gravity.TOP + Gravity.LEFT;  // 左上
		
		        // 进来的时候把存储的位置读取显示出来
		        mParams.x = PreferenceUtil.getInt(mContext, "lastX");
		        mParams.y = PreferenceUtil.getInt(mContext, "lastY");
		
		        // 初始化吐司窗口布局
		        mView = View.inflate(mContext, R.layout.view_toast, null);
		    }

* 接下来就是显示吐司的逻辑，其实特别简单，核心代码只有一句，这里要先获取自定义填充布局文件中的TextView显示文本显示你要弹出的文本，然后对View的点击进行了监听，实现了双击将吐司居中显示，和三击隐藏吐司的功能。





			
	    /**
	     * 弹出自定义吐司
	     */
	    public void popToast(String text) {
	        TextView tvName = (TextView) mView.findViewById(R.id.tv_toast_name);
	        // 设置显示的文字
	        tvName.setText(text);
	
	        // 吐司窗体的背景可以在布局文件之中指定也可以在代码中设置
	
	
	        // 设置吐司的双击事件，点击之后会到中心点
	        mView.setOnClickListener(new OnClickListener() {
	
	            @Override
	            public void onClick(View v) {
	
	                // 双击事件处理逻辑
	                System.arraycopy(mHits1, 1, mHits1, 0, mHits1.length - 1);
	                mHits1[mHits1.length - 1] = SystemClock.uptimeMillis();
	                if (mHits1[0] >= (SystemClock.uptimeMillis() - 500)) {
	                    // 双击之后执行
	                    // 让吐司移动到x中心，y不需要对中
	                    // 更新窗体的坐标
	                    mParams.x = (mWm.getDefaultDisplay().getWidth() - mView.getWidth()) / 2;
	                    mWm.updateViewLayout(mView, mParams);
	
	                    // 点击完退出的时候也把位置信息存储起来
	                    PreferenceUtil.putInt(mContext, "lastX", mParams.x);
	                    PreferenceUtil.putInt(mContext, "lastY", mParams.y);
	                }
	
	
	                // 三击事件处理逻辑
	                System.arraycopy(mHits2, 1, mHits2, 0, mHits2.length - 1);
	                mHits2[mHits2.length - 1] = SystemClock.uptimeMillis();
	                if (mHits2[0] >= (SystemClock.uptimeMillis() - 600)) {
	                    // 点击之后将吐司移除掉
	                    if (mView != null) {
	                        if (mView.getParent() != null) {
	                            mWm.removeView(mView);
	                        }
	                    }
	                }
	            }
	        });
	
	
	        // 设置吐司的触摸滑动事件
	        mView.setOnTouchListener(new View.OnTouchListener() {
	
	            int startX;
	            int startY;
	
	            @Override
	            public boolean onTouch(View v, MotionEvent event) {
	
	                switch (event.getAction()) {
	                    case MotionEvent.ACTION_DOWN: // 按下
	
	                        // 手指按下时的坐标位置
	                        startX = (int) event.getRawX();
	                        startY = (int) event.getRawY();
	
	                        break;
	                    case MotionEvent.ACTION_MOVE: // 移动
	
	                        // 移动后的坐标位置
	                        int newX = (int) event.getRawX();
	                        int newY = (int) event.getRawY();
	
	                        // 偏移量
	                        int dx = newX - startX;
	                        int dy = newY - startY;
	
	                        // 给偏移量设置边距
	                        // 小于x轴
	                        if (mParams.x < 0) {
	                            mParams.x = 0;
	                        }
	                        // 小于y轴
	                        if (mParams.y < 0) {
	                            mParams.y = 0;
	                        }
	
	                        // 超出x轴
	                        if (mParams.x > mWm.getDefaultDisplay().getWidth() - mView.getWidth()) {
	                            mParams.x = mWm.getDefaultDisplay().getWidth() - mView.getWidth();
	                        }
	                        // 超出y轴
	                        if (mParams.y > mWm.getDefaultDisplay().getHeight() - mView.getHeight()) {
	                            mParams.y = mWm.getDefaultDisplay().getHeight() - mView.getHeight();
	                        }
	
	                        // 更新窗体的坐标
	                        mParams.x += dx;
	                        mParams.y += dy;
	                        mWm.updateViewLayout(mView, mParams);
	
	                        // 重新赋值起始坐标
	                        startX = (int) event.getRawX();
	                        startY = (int) event.getRawY();
	                        break;
	                    case MotionEvent.ACTION_UP: // 抬起
	                        // 抬起来的时候保存最后一次的位置，下次进来时直接显示出来
	                        PreferenceUtil.putInt(mContext, "lastX", mParams.x);
	                        PreferenceUtil.putInt(mContext, "lastX", mParams.y);
	                        break;
	                    default:
	                        break;
	                }
	
	                return false;  // 当有父控件有点击事件时，这里要返回false，不然父控件就拿不到点击事件了
	            }
	        });
	
	        if (mView != null) {
	            if (mView.getParent() != null) {
	                mWm.removeView(mView);
	            }
	        }
	        // 添加到窗体管理器中才能显示出来
	        mWm.addView(mView, mParams);
	    }


* 最后是吐司的隐藏方法，这个方法只需要将View从窗体管理器中移除掉就好了。



		    /**
		     * 从父窗体中移除吐司
		     */
		    public void hideToast() {
		        if (mView != null) {
		            if (mView.getParent() != null) {
		                mWm.removeView(mView);
		            }
		        }
		    }


### 以上代码简单的实现了自定义[Toast](https://developer.android.com/reference/android/widget/Toast.html)的显示和隐藏，当然你也可以给显示的[Toast](https://developer.android.com/reference/android/widget/Toast.html)加一个背景，这样吐司看起来就更加的漂亮了。以上就是本次分享的全部内容，谢谢大家。
---



[我的主页](http://www.jianshu.com/users/64f479a1cef7/latest_articles)

[Demo下载地址](https://github.com/PingerWan/CustomToastDemo)






























