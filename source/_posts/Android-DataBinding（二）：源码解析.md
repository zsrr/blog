---
title: Android DataBinding（二）：源码解析
date: 2017-02-15 15:27:19
tags: [Android,Java,源码分析]
categories: 安卓开发

---
# 准备工作
看源码有两种方式可供选择，第一种是直接将整个AOSP下载下来，然后自行编译，这种方法的好处看源码不会遇到爆红，整个代码结构一目了然，缺点就是占用空间大，下载时间长，编译坑比较多……第二种方法是直接用sdk目录之下的。在这里笔者采用第一种方法，下载并编译AOSP请见：[Mac OS Sierra下编译Android 7.1.1](http://zsrr.coding.me/2017/02/16/Mac-OS-Sierra%E4%B8%8B%E7%BC%96%E8%AF%91Android-7-1-1/)。

用Android Studio导入安卓源码之后，在Android视图，java包下会看到这几个包：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-17%20%E4%B8%8B%E5%8D%889.24.48.png)
这就是我们要分析的源码了。

如果要采用第二种方法，platform请升到最新，不然可能出现当前platform找不到databinding的源码的问题，具体步骤如下：    

- 随便新建一个Android项目
- 找到sdk文件夹，依次进入sources，android-xx（最新的），android文件夹，将里面的databinding文件夹复制到刚才新建的安卓项目当中。
- 更改databinding文件夹中所有java文件的包名。

另外，如果你是Windows用户，那么请用[Source Insight](https://www.sourceinsight.com/download/)查看安卓源码。
# 源码解析
## DataBindingUtil
这是数据绑定的入口类，一般用它来创建相应的Binding类，我们由此开始分析。

先看最常见的用法：

	public static <T extends ViewDataBinding> T inflate(LayoutInflater inflater, int layoutId, ViewGroup parent, boolean attachToParent);
	public static <T extends ViewDataBinding> T setContentView(Activity activity, int layoutId);
	public static <T extends ViewDataBinding> T bind(View root);
第一个方法最终调用的是**bind(DataBindingComponent bindingComponent, View root, int layoutId)**方法（如果attachToParent设置为false的话），这个方法的定义为：

	static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View root,
            int layoutId) {
        return (T) sMapper.getDataBinder(bindingComponent, root, layoutId);
    }
直接交给了sMapper来处理，sMapper是一个类型为DataBinderMapper的静态成员，其getDataBinder方法会根据xml文件生成的ViewDataBinding子类动态改变，这是笔者的getDataBinder源码（源自另一个已经写好的安卓项目）：

	public android.databinding.ViewDataBinding getDataBinder(android.databinding.DataBindingComponent bindingComponent, android.view.View view, int layoutId) {
        switch(layoutId) {
                case com.stephen.learning.R.layout.item_view:
                    return com.stephen.learning.databinding.ViewHolderBinding.bind(view, bindingComponent);
                case com.stephen.learning.R.layout.activity_data_binding:
                    return com.stephen.learning.databinding.ImageViewBinding.bind(view, bindingComponent);
        }
        return null;
    }
可以看到，DataBinderMapper类将行为委托给了实际产生的ViewDataBinding的子类上，其具体的bind方法的实现，我们稍后再谈。现在再看**setContentView**方法，此方法的实现：

	public static <T extends ViewDataBinding> T setContentView(Activity activity, int layoutId) {
        return setContentView(activity, layoutId, sDefaultComponent);
    }
    
    public static <T extends ViewDataBinding> T setContentView(Activity activity, int layoutId,
            DataBindingComponent bindingComponent) {
        activity.setContentView(layoutId);
        View decorView = activity.getWindow().getDecorView();
        ViewGroup contentView = (ViewGroup) decorView.findViewById(android.R.id.content);
        return bindToAddedViews(bindingComponent, contentView, 0, layoutId);
    }
bindToAddedViews方法的实现：
	
	private static <T extends ViewDataBinding> T bindToAddedViews(DataBindingComponent component,
            ViewGroup parent, int startChildren, int layoutId) {
        final int endChildren = parent.getChildCount();
        final int childrenAdded = endChildren - startChildren;
        if (childrenAdded == 1) {
            final View childView = parent.getChildAt(endChildren - 1);
            return bind(component, childView, layoutId);
        } else {
            final View[] children = new View[childrenAdded];
            for (int i = 0; i < childrenAdded; i++) {
                children[i] = parent.getChildAt(i + startChildren);
            }
            return bind(component, children, layoutId);
        }
    }
由上述代码可以发现，setContentView先获得activity中DecorView中的FrameLayout，再将其作为参数传入**bindToAddedViews**方法，此方法先判断parent当中子视图的数目，一般情况下我们activity的xml文件只有一个RelativeLayout或者LinearLayout的根元素，所以子视图一般只有一个，于是又开始调用bind方法，重新回到上面的流程。

下面分析**bind(View root)**方法：

	public static <T extends ViewDataBinding> T bind(View root) {
        return bind(root, sDefaultComponent);
    }
    
    public static <T extends ViewDataBinding> T bind(View root,
            DataBindingComponent bindingComponent) {
        T binding = getBinding(root);
        if (binding != null) {
            return binding;
        }
        Object tagObj = root.getTag();
        if (!(tagObj instanceof String)) {
            throw new IllegalArgumentException("View is not a binding layout");
        } else {
            String tag = (String) tagObj;
            int layoutId = sMapper.getLayoutId(tag);
            if (layoutId == 0) {
                throw new IllegalArgumentException("View is not a binding layout");
            }
            return (T) sMapper.getDataBinder(bindingComponent, root, layoutId);
        }
    }
bind方法先调用**getBinding**方法，**getBinding**方法直接调用了ViewDataBinding的静态**getBinding**方法，这个方法之后分析ViewDataBinding的时候再讲。如果得到的binding是空的，那就创建一个binding并且返回，创建过程是先得到view的tag，这个tag是个String类型。例如ListView的item的xml文件名为item_ view，里面的根元素是layout标签，那么我们用LayoutInflater类inflate出来的view便有一个String类型的Tag，并且值为："layout/item_ view_ 0"。好，找到tag之后，便调用DataBinderMapper类的**getLayoutId**方法，这个方法也是根据xml文件的定义自动生成的，笔者的方法定义如下所示：

	int getLayoutId(String tag) {
        if (tag == null) {
            return 0;
        }
        final int code = tag.hashCode();
        switch(code) {
            case -1223782307: {
                if(tag.equals("layout/item_view_0")) {
                    return com.stephen.learning.R.layout.item_view;
                }
                break;
            }
            case 1514339820: {
                if(tag.equals("layout/activity_data_binding_0")) {
                    return com.stephen.learning.R.layout.activity_data_binding;
                }
                break;
            }
        }
        return 0;
    }
嗯……这么拿hashCode做判断再拿tag做判断的写法比较智障啊……  
拿到layoutId之后，最后调用sMapper的**getDataBinder**，结束。
## DataBinderMapper
这个类随着我们对xml的定义改变而改变，它的职责其实在上文对DataBindingUtil类的分析中已经展现：  

- 作为DataBindingUtil和ViewDataBinding类的中介类
- 根据tag得到对应的id

此外，它的内部还维护一个InnerBrLookUp的静态类，此静态类又维护一个String类型的数组，用来将BR类的id转成对应的String，通常用作打log，这里不再细谈。
## BaseObservable
把此类放在这个位置是为了之后介绍**ViewDataBinding**类。

此类的实现相当简单：

	public class BaseObservable implements Observable {
	    private transient PropertyChangeRegistry mCallbacks;
	
	    public BaseObservable() {
	    }
	
	    @Override
	    public synchronized void addOnPropertyChangedCallback(OnPropertyChangedCallback callback) {
	        if (mCallbacks == null) {
	            mCallbacks = new PropertyChangeRegistry();
	        }
	        mCallbacks.add(callback);
	    }
	
	    @Override
	    public synchronized void removeOnPropertyChangedCallback(OnPropertyChangedCallback callback) {
	        if (mCallbacks != null) {
	            mCallbacks.remove(callback);
	        }
	    }
	
	    
	    public synchronized void notifyChange() {
	        if (mCallbacks != null) {
	            mCallbacks.notifyCallbacks(this, 0, null);
	        }
	    }
	    
	    public void notifyPropertyChanged(int fieldId) {
	        if (mCallbacks != null) {
	            mCallbacks.notifyCallbacks(this, fieldId, null);
	        }
	    }
	}
将所有的职责都托付给了内部成员**mCallbacks**身上。

### Observable
**BaseObservable**实现了**Observable**接口。先看一下官方文档对此接口的介绍：

>Observable classes provide a way in which data bound UI can be notified of changes.

简而言之，这个接口就是为了保证UI元素和数据之间的双向绑定，其内部有两个方法：

	void addOnPropertyChangedCallback(OnPropertyChangedCallback callback);
	void removeOnPropertyChangedCallback(OnPropertyChangedCallback callback);
OnPropertyChangedCallback是一个抽象类，其内部只有一个抽象方法:

	public abstract void onPropertyChanged(Observable sender, int propertyId);
sender就是持有此类的**Observable**类，propertyId是绑定的数据id。当我们用**Bindable**注解标注一个getter时，自动生成的**BR**类会生成此id。

### CallbackRegistry
上面说到**BaseObservable**将职责托付给了**mCallbacks**，而mCallbacks是一个**PropertyChangeRegistry**类，这个类的超类**CallbackRegistry**。此类是管理callback注册，删除，通知的核心类，通过内部**NotifierCallback**类来通知callbacks，此类的代码虽然不多，但是对于我这种算法弱鸡来说看上去的感觉是这样的：
![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1487508874434&di=188c6fb1e6fe4e69bf1cab8c82844595&imgtype=0&src=http%3A%2F%2Fwanzao2.b0.upaiyun.com%2Fsystem%2Fpictures%2F31274020%2Foriginal%2F1449639129_500x500.png)

这个牛逼的算法就不由我这个弱鸡去分析了，它实现了通知Callback的可重入性，就是比方说有两个线程，一个线程执行到通知某个Callback时cpu切到了另一个线程，这个线程把刚才的要通知的Callback给删了，这种情况下切换到之前的线程进行通知是没有任何影响的。如果各位实在想知道这个算法是怎么一回事，请看这篇大牛的文章吧：[回调通知管理器 CallbackRegistry 解析](https://gold.xitu.io/post/5840e2e3ac502e006cc0ef26)
### PropertyChangeRegistry
此类比较简单：

	public class PropertyChangeRegistry extends
        CallbackRegistry<Observable.OnPropertyChangedCallback, Observable, Void> {

	    private static final CallbackRegistry.NotifierCallback<Observable.OnPropertyChangedCallback, Observable, Void> NOTIFIER_CALLBACK = new CallbackRegistry.NotifierCallback<Observable.OnPropertyChangedCallback, Observable, Void>() {
	        @Override
	        public void onNotifyCallback(Observable.OnPropertyChangedCallback callback, Observable sender,
	                int arg, Void notUsed) {
	            callback.onPropertyChanged(sender, arg);
	        }
	    };
	
	    public PropertyChangeRegistry() {
	        super(NOTIFIER_CALLBACK);
	    }
	
	    public void notifyChange(Observable observable, int propertyId) {
	        notifyCallbacks(observable, propertyId, null);
	    }
	}
Callback类型是**Observable**的内部类**OnPropertyChangedCallback**，Sender类型是**Observable**。NotifierCallback内部的实现仅仅是调用callback的**onPropertyChanged**方法，arg便是**BR**类里面的资源id。**notifyChange**方法是将外层传来的sender以及资源id传给**notifyCallbacks**方法，以此达到了最终调用每一个callback的**onPropertyChanged**的目的。
## ViewDataBinding
系统会根据xml的描述生成此类的子类，我们先看一下自动生成的类的代码，一步一步加以分析。

xml文件：
	
	<layout xmlns:android="http://schemas.android.com/apk/res/android">
	    <data class="BindingForTest">
	        <variable
	            name="testData"
	            type="com.stephen.learning.databinding.Model"/>
	    </data>
	
	    <LinearLayout
	        xmlns:tools="http://schemas.android.com/tools"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        tools:context="com.stephen.learning.databinding.TestActivity"
	        android:orientation="vertical">
	
	        <TextView
	            android:id="@+id/main_text_view"
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content" />
	        <TextView
	            android:id="@+id/second_text_view"
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:text="@{testData.testData}"/>
	        <Button
            	android:layout_width="wrap_content"
            	android:layout_height="wrap_content"
            	android:text="@{testData.testData}" />
	    </LinearLayout>
	</layout>
生成的ViewBinding子类：
![](http://ok34fi9ya.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-02-19%20%E4%B8%8B%E5%8D%888.36.08.png)
先看构造函数：

    public BindingForTest(android.databinding.DataBindingComponent bindingComponent, View root) {
        super(bindingComponent, root, 0);
        final Object[] bindings = mapBindings(bindingComponent, root, 4, sIncludes, sViewsWithIds);
        this.mainTextView = (android.widget.TextView) bindings[3];
        this.mboundView0 = (android.widget.LinearLayout) bindings[0];
        this.mboundView0.setTag(null);
        this.mboundView2 = (android.widget.Button) bindings[2];
        this.mboundView2.setTag(null);
        this.secondTextView = (android.widget.TextView) bindings[1];
        this.secondTextView.setTag(null);
        setRootTag(root);
        // listeners
        invalidateAll();
    }
先调用超类的静态**mapBindings**方法得到一个**View**数组，再从数组中初始化View成员，View成员有三种：

- 在xml中有id的定义。
- 在xml中存在数据的绑定。
- 是根View。

满足以上三个条件任意一个便会生成相应的类成员，下面是笔者的类里面的类成员：

	//存在id，不存在数据绑定
	public final android.widget.TextView mainTextView;
	//根View
    private final android.widget.LinearLayout mboundView0;
    //存在数据绑定，不存在id
    private final android.widget.Button mboundView2;
    //既存在数据绑定，也存在id
    public final android.widget.TextView secondTextView;
再来看一下类的静态初始化：

    static {
        sIncludes = null;
        sViewsWithIds = new android.util.SparseIntArray();
        sViewsWithIds.put(R.id.main_text_view, 3);
    }
**sIncludes**只有在xml中存在**&lt;include&gt;**标签且存在数据绑定的时候才不为空，在这里笔者暂时没有用到使用数据绑定的**&lt;include&gt;**的需求（觉得也不可能有），而且源码的实现相对简单，这里不太值得分析，有兴趣自己去看。**sViewsWithIds**是一个**SparseIntArray**，这个成员的作用是存放不存在数据绑定但存在id定义的View成员。键是id，值是对应在上面所提在View数组中的索引值。

下面来看**mapBindings**方法：

	protected static Object[] mapBindings(DataBindingComponent bindingComponent, View root,
            int numBindings, IncludedLayouts includes, SparseIntArray viewsWithIds) {
        Object[] bindings = new Object[numBindings];
        mapBindings(bindingComponent, root, bindings, includes, viewsWithIds, true);
        return bindings;
    }
    
    private static void mapBindings(DataBindingComponent bindingComponent, View view,
            Object[] bindings, IncludedLayouts includes, SparseIntArray viewsWithIds,
            boolean isRoot) {
        final int indexInIncludes;
        final ViewDataBinding existingBinding = getBinding(view);
        //已经绑定过，返回
        if (existingBinding != null) {
            return;
        }
        
        Object objTag = view.getTag();
        final String tag = (objTag instanceof String) ? (String) objTag : null;
        boolean isBound = false;
        
        //如果是根View
        if (isRoot && tag != null && tag.startsWith("layout")) {
            final int underscoreIndex = tag.lastIndexOf('_');
            if (underscoreIndex > 0 && isNumeric(tag, underscoreIndex + 1)) {
                final int index = parseTagInt(tag, underscoreIndex + 1);
                if (bindings[index] == null) {
                    bindings[index] = view;
                }
                indexInIncludes = includes == null ? -1 : index;
                isBound = true;
            } else {
                indexInIncludes = -1;
            }
         //如果不是根View
        } else if (tag != null && tag.startsWith(BINDING_TAG_PREFIX)) {
            int tagIndex = parseTagInt(tag, BINDING_NUMBER_START);
            if (bindings[tagIndex] == null) {
                bindings[tagIndex] = view;
            }
            isBound = true;
            indexInIncludes = includes == null ? -1 : tagIndex;
        } else {
            // Not a bound view
            indexInIncludes = -1;
        }
        //处理存在id但是不存在数据绑定的view成员
        if (!isBound) {
            final int id = view.getId();
            if (id > 0) {
                int index;
                if (viewsWithIds != null && (index = viewsWithIds.get(id, -1)) >= 0 &&
                        bindings[index] == null) {
                    bindings[index] = view;
                }
            }
        }

        if (view instanceof  ViewGroup) {
            final ViewGroup viewGroup = (ViewGroup) view;
            final int count = viewGroup.getChildCount();
            int minInclude = 0;
            for (int i = 0; i < count; i++) {
                final View child = viewGroup.getChildAt(i);
                boolean isInclude = false;
                if (indexInIncludes >= 0 && child.getTag() instanceof String) {
                   ...
                }
                //对每一个子View进行递归处理
                if (!isInclude) {
                    mapBindings(bindingComponent, child, bindings, includes, viewsWithIds, false);
                }
            }
        }
    }
此方法的思想相对简单，先判断是否绑定过，如果已经绑定过，则返回。接下来分成三种情况：

- 根视图。
- 存在数据绑定的视图。
- 不存在数据绑定，但是在xml中有id定义。

最后如果传入的是一个ViewGroup，则对它的子视图进行递归处理。下面我们来看几个细节：

**视图数组索引的判断**  

上文中我们说过用**&lt;layout&gt;**做标记的xml文件在用**LayoutInflater**类inflate出来的根View会有一个**tag**，这个**tag**是一个**String**类型，且设置的规则如下：

- 根视图设置为**layout/[xml文件名]_[在View数组中的索引]**
- 不是根视图，但是存在数据的绑定，则设置为**binding_[在View数组中的索引]**

根据tag获得数组索引的代码如下：

	private static int parseTagInt(String str, int startIndex) {
        final int end = str.length();
        int val = 0;
        for (int i = startIndex; i < end; i++) {
            val *= 10;
            char c = str.charAt(i);
            val += (c - '0');
        }
        return val;
    }
这样得出的数组索引便是最后一个"_"后面的数字值。

还有一种情况是不存在数据绑定，但是存在id定义的view成员。上面提到过如果存在这种情况，**ViewDataBinding**在静态初始化的时候会将对应的id和数组索引值放在一个**SparseIntArray**当中，这时候处理过程是这样的：

	if (!isBound) {
        final int id = view.getId();
        if (id > 0) {
            int index;
            if (viewsWithIds != null && (index = viewsWithIds.get(id, -1)) >= 0 &&
                    bindings[index] == null) {
                bindings[index] = view;
            }
        }
好，现在继续往下看**invalidateAll**方法：

	public void invalidateAll() {
        synchronized(this) {
                mDirtyFlags = 0x2L;
        }
        requestRebind();
    }
**requestRebind**:

	protected void requestRebind() {
        synchronized (this) {
            if (mPendingRebind) {
                return;
            }
            mPendingRebind = true;
        }
        if (USE_CHOREOGRAPHER) {
            mChoreographer.postFrameCallback(mFrameCallback);
        } else {
            mUIThreadHandler.post(mRebindRunnable);
        }

    }
其中**USE_CHOREOGRAPHER**是判断当前的SDK是不是16以上版本来决定是否能用**Choreographer**，如果能则直接设置一个**FrameCallback**，不能则在主线程运行一个**Runnable**，我们来看**mFrameCallback**的定义：

	mFrameCallback = new Choreographer.FrameCallback() {
                @Override
                public void doFrame(long frameTimeNanos) {
                    mRebindRunnable.run();
                }
            };
**mRebindRunnable**:

	private final Runnable mRebindRunnable = new Runnable() {
        @Override
        public void run() {
            synchronized (this) {
                mPendingRebind = false;
            }
            if (VERSION.SDK_INT >= VERSION_CODES.KITKAT) {
                // Nested so that we don't get a lint warning in IntelliJ
                if (!mRoot.isAttachedToWindow()) {
                    // Don't execute the pending bindings until the View
                    // is attached again.
                    mRoot.removeOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                    mRoot.addOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                    return;
                }
            }
            executePendingBindings();
        }
    };
**ROOT_ REATTACHED_LISTENER**是一个**OnAttachStateChangeListener**，其定义如下：

	ROOT_REATTACHED_LISTENER = new OnAttachStateChangeListener() {
                @TargetApi(VERSION_CODES.KITKAT)
                @Override
                public void onViewAttachedToWindow(View v) {
                    // execute the pending bindings.
                    final ViewDataBinding binding = getBinding(v);
                    binding.mRebindRunnable.run();
                    v.removeOnAttachStateChangeListener(this);
                }

                @Override
                public void onViewDetachedFromWindow(View v) {
                }
            };
很简单，当根视图还没有attach到window上的时候，**mRebindRunnable**直接设置一个**OnAttachStateChangeListener**并返回，当attach到window上的时候重新运行**mRebindRunnable**，此时将运行**executePendingBindings**，定义如下：

	public void executePendingBindings() {
        if (mIsExecutingPendingBindings) {
            requestRebind();
            return;
        }
        if (!hasPendingBindings()) {
            return;
        }
        mIsExecutingPendingBindings = true;
        mRebindHalted = false;
        if (mRebindCallbacks != null) {
            mRebindCallbacks.notifyCallbacks(this, REBIND, null);

            // The onRebindListeners will change mPendingHalted
            if (mRebindHalted) {
                mRebindCallbacks.notifyCallbacks(this, HALTED, null);
            }
        }
        if (!mRebindHalted) {
            executeBindings();
            if (mRebindCallbacks != null) {
                mRebindCallbacks.notifyCallbacks(this, REBOUND, null);
            }
        }
        mIsExecutingPendingBindings = false;
    }
其中**hasPendingBindings**方法是一个抽象方法，一般情况下子类通过其**mDirtyFlags**判断是否有未处理的绑定情况，返回相应的布尔值：

	public boolean hasPendingBindings() {
        synchronized(this) {
            if (mDirtyFlags != 0) {
                return true;
            }
        }
        return false;
    }
接下来便要通知相应的mRebindCallbacks，我们来看它的定义：

	private CallbackRegistry<OnRebindCallback, ViewDataBinding, Void> mRebindCallbacks;
	
	public void addOnRebindCallback(OnRebindCallback listener) {
        if (mRebindCallbacks == null) {
            mRebindCallbacks = new CallbackRegistry<OnRebindCallback, ViewDataBinding, Void>(REBIND_NOTIFIER);
        }
        mRebindCallbacks.add(listener);
    }
它是一个Callback类型是**OnRebindCallback**，Sender类型是**ViewDataBinding**的**CallbackRegistry**类型，下面来看**OnRebindCallback**的定义，它是一个抽象类，里面有三个方法：

	public abstract class OnRebindCallback<T extends ViewDataBinding> {
	    public boolean onPreBind(T binding) {
	        return true;
	    }
	
	    public void onCanceled(T binding) {
	    }
	
	    public void onBound(T binding) {
	    }
	}
泛型的参数T就是上面的Sender，第一个方法**onPreBind**的返回值决定着rebind是否会继续进行。接着再来看**REBIND_NOTIFIER**:

	private static final CallbackRegistry.NotifierCallback<OnRebindCallback, ViewDataBinding, Void>
        REBIND_NOTIFIER = new NotifierCallback<OnRebindCallback, ViewDataBinding, Void>() {
        @Override
        public void onNotifyCallback(OnRebindCallback callback, ViewDataBinding sender, int mode,
                Void arg2) {
            switch (mode) {
                case REBIND:
                    if (!callback.onPreBind(sender)) {
                        sender.mRebindHalted = true;
                    }
                    break;
                case HALTED:
                    callback.onCanceled(sender);
                    break;
                case REBOUND:
                    callback.onBound(sender);
                    break;
            }
        }
    };
现在再来看**executePendingBindings**方法，先是向**mRebindCallbacks**传入**REBIND**参数，进而调用**OnRebindCallback**的**onPreBind**方法，根据其返回的布尔值来决定**mRebindHalted**的值（是否停止）。接下来判断**mRebindHalted**的值，如果是真值的话，则通过向**mRebindCallbacks**传入**HALTED**参数进而调用**OnRebindCallback**的**onCanceled**方法；如果是假值，则先调用**executeBindings**方法，此方法是抽象方法，再通过向**mRebindCallbacks**传入**REBOUND**参数，进而调用**OnRebindCallback**的**onRebound**方法。

下面来看**executeBindings**方法的具体实现：

	@Override
    protected void executeBindings() {
        long dirtyFlags = 0;
        synchronized(this) {
        	//读取mDirtyFlags
            dirtyFlags = mDirtyFlags;
            //重置mDirtyFlags
            mDirtyFlags = 0;
        }
        java.lang.String testDataTestData = null;
        //读取testData
        com.stephen.learning.databinding.Model testData = mTestData;

        if ((dirtyFlags & 0x3L) != 0) {



                if (testData != null) {
                    // read testData.testData
                    testDataTestData = testData.testData;
                }
        }
        // batch finished
        if ((dirtyFlags & 0x3L) != 0) {
            // api target 1

            android.databinding.adapters.TextViewBindingAdapter.setText(this.mboundView2, testDataTestData);
            android.databinding.adapters.TextViewBindingAdapter.setText(this.secondTextView, testDataTestData);
        }
    }
方法的最后通过**TextViewBindingAdapter**来实际设置**TextView**的值（**Button**继承于**TextView**），除了**TextViewBindingAdapter**之外，官方还提供了四十多种Adapter用来使数据绑定更加便捷，在这里我就不一一分析了，只说一下大体的形式。一般来说一个Adapter由两部分组成：

- 顶部**BindingMethods**注解，用此注解来说明当发生数据绑定时调用哪个类的哪个方法。
- 一系列用**BindingAdapter**注释的静态方法，用来供**ViewDataBinding**的子类调用。

值得注意的是，xml中的数据绑定并不都是通过Adapter来实现的，下面举一个例子：

将xml文件更改成：

	<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:bind="http://schemas.android.com/apk/res-auto">
	    
	    ...
	
	    <LinearLayout
	        xmlns:tools="http://schemas.android.com/tools"
	        android:layout_width="match_parent"
	        android:layout_height="match_parent"
	        tools:context="com.stephen.learning.databinding.TestActivity"
	        android:orientation="vertical">
	
	        ...
	        
	        <Button
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:text="@{testData.testData}"
	            android:onClick="@{(v) -> testData.onButtonClick()}"/>
	    </LinearLayout>
	</layout>
这个时候rebuild project，**BindingForTest**类变成了这样：

	public class BindingForTest extends android.databinding.ViewDataBinding implements android.databinding.generated.callback.OnClickListener.Listener {

	    ...
	
	    public BindingForTest(android.databinding.DataBindingComponent bindingComponent, View root) {
	        ...
	        // listeners
	        mCallback1 = new android.databinding.generated.callback.OnClickListener(this, 1);
	        invalidateAll();
	    }
	
	    @Override
	    protected void executeBindings() {
	        ...
	        if ((dirtyFlags & 0x2L) != 0) {
	            // api target 1
	
	            this.mboundView2.setOnClickListener(mCallback1);
	        }
	    }
	    // Listener Stub Implementations
	    // callback impls
	    public final void _internalCallbackOnClick(int sourceId , android.view.View callbackArg_0) {
	        // localize variables for thread safety
	        // testData != null
	        boolean testDataObjectnull = false;
	        // testData
	        com.stephen.learning.databinding.Model testData = mTestData;
	
	
	
	        testDataObjectnull = (testData) != (null);
	        if (testDataObjectnull) {
	
	
	            testData.onButtonClick();
	        }
	    }
	}
在构造函数中初始化了databinding包中的**OnClickListener**，并且实现了其中的**Listener**接口，**OnClickListener**类的定义如下：

	public final class OnClickListener implements android.view.View.OnClickListener {
	    final Listener mListener;
	    final int mSourceId;
	    public OnClickListener(Listener listener, int sourceId) {
	        mListener = listener;
	        mSourceId = sourceId;
	    }
	    @Override
	    public void onClick(android.view.View callbackArg_0) {
	        mListener._internalCallbackOnClick(mSourceId , callbackArg_0);
	    }
	    public interface Listener {
	        void _internalCallbackOnClick(int sourceId , android.view.View callbackArg_0);
	    }
	}
它实现了**View**当中的**OnClickListener**接口，其**onClick**方法则调用**mListener**当中的**_internalCallbackOnClick**方法（这个命名怎么这么像native方法）。**BindingForTest**的对此方法的定义如下：

	public final void _internalCallbackOnClick(int sourceId , android.view.View callbackArg_0) {
	        // localize variables for thread safety
	        // testData != null
	        boolean testDataObjectnull = false;
	        // testData
	        com.stephen.learning.databinding.Model testData = mTestData;
	        testDataObjectnull = (testData) != (null);
	        if (testDataObjectnull) {
	            testData.onButtonClick();
	        }
	    }
这边开始调用testData的**onButtonClick**，与xml中的定义对应起来了。