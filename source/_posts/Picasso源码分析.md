---
title: Picasso源码分析
date: 2017-01-20 19:03:32
tags: [Android,源码分析]
categories: 安卓开发

---

## Picasso简介
Picasso是大名鼎鼎的Square公司推出的安卓图片下载，缓存的第三方库。上手简单，功能强大。在此本文就不介绍它具体是怎么使用的了，给出几个已经讲的很好的博文地址即可：  

Github地址： [http://square.github.io/picasso/](http://square.github.io/picasso/)  
Picasso使用详解： [http://www.jianshu.com/p/ae8e6ed18e23](http://www.jianshu.com/p/ae8e6ed18e23)  

Github上的源码中的示例代码是个不错的学习资源哦！

## Picasso工作原理
工作原理如图所示，图中所标记的类下文会一一进行剖析。（笔者的图确实画的不怎么样，见谅）  

![Picasso Structure](http://ok34fi9ya.bkt.clouddn.com/picasso_structure.png) 

**注：** 图中没有列出像LruCache, OkHttpDownloader等涉及到大量技术细节的类，没有什么值得讨论的，感兴趣的可以自行看源码。

**Picasso**类  
提供给用户使用的顶层类，通过一个依附在主线程（mainThread)的Handler与Dispatcher中的DispatcherThread的Handler交互，其主要类成员和方法为：  

	Map<Object, Action> targetToAction; //Picasso管理Action的工具
	Map<ImageView, DeferredRequestCreator> targetToRequestCreator; //管理ImageView
	Handler HANDLER = new Handler(Looper.getMainLooper()) { ... } //用于通知Action图片加载成功（或者失败），以及重新提交Action(resumeAction)
	Dispatcher dispatcher; //用于处理提交（submit)和取消（cancel)操作
	
	...
	
	Picasso.with(Context); //单例模式提供默认的Picasso类
	picasso.load(Uri); //向外提供一个RequestCreator
	picasso.enqueueAndSubmit(Action); //提交Action给Dispatcher处理
	picasso.resumeAction(Action); //重新提交Action给Dispatcher
	picasso.complete(BitmapHunter); //从中获取Action，通知主线程图片加载失败或者成功
	picasso.deliverAction(Action); //通知主线程图片加载成功（action.complete)或者失败(action.error)
	picasso.cancelExistingRequest(Object); //通过Tag从targetToAction中获取到Action, 并通知dispatcher取消此action， 如果Target是ImageView的话还会从targetToRequestCreator中得到DeferredRequestCreator并通知起取消。

**Dispatcher类**  
dispatcher，顾名思义，掌管“事件”的奋发，Picasso通过它来取消或者提交Action，BitmapHunter通过它来与Picasso当中的HANDLER进行通信。其内部有一个DispatcherThread和相应的DispatcherHandler，在此线程内处理各种事件。其主要的类成员和方法为： 
 
	Map<String, BitmapHunter> hunterMap; // String是key类型，从Action.getKey()获取
	Map<Object, Action> failedAction; // 失败的Action
	Map<Object, Action> pausedAction; //处在暂停状态的Action
	Handler handler; //即DispatcherThread的Handler
	Handler mainThreadHandler; //即Picasso维持的主线程的Handler
	
	...
	
	dispatchSubmit(Action); //提交Action
	dispatchCancel(Action); //取消Action
	dispatchPauseTag(Object); //根据Tag暂停相应的Action
	dispatchResumeTag(Object); //根据相应的Tag重新开始相应的Action
	dispatchComplete(BitmapHunter); //BitmapHunter成功抓取图片回调
	dispatchFailed(BitmapHunter); //BitmapHunter抓取图片失败回调
	
**Action类**  
在笔者看来，这个类代表着从开始请求到请求成功或者失败的“动作”，在这整个流程里面，包括一开始使用的Picasso, 还有所构造出来的Request, 所使用的存储策略(memoryPolicy)、网络策略(networkPolicy),失败时的默认图片，以及标志这个动作的Tag都包含在里面。代表成功或者失败则是两个抽象方法：

	abstract void complete(Bitmap result, Picasso.LoadedFrom from);

    abstract void error(Exception e);
    
这两个方法由它的子类具体来实现，典型的[策略模式](https://zh.wikipedia.org/wiki/策略模式)，其主要类成员如下所示：

	Picasso picasso;
    Request request;
    WeakReference<T> target;
    int memoryPolicy;
    int networkPolicy;
    int errorResId;
    Drawable errorDrawable;
    String key;     //一个key对应一个相应的BitmapHunter
    Object tag;		//标志此Action
    
**BitmapHunter类**  
这个嘛，顾名思义（英文好是多么重要），就是用来抓图的，它是一个Runnable，并且通过key和一个（或者一些）Action关联起来。当Dispatcher调用perform(Action)这个方法的时候，会通过Action的key找到相应的BitmapHunter， 并且通过从Picasso传来的ExcutorService来运行这个Hunter，抓取成功或者失败后则会通知其内部持有的Dispatcher。主要的成员及方法如下：

	Dispatcher dispatcher;
	String key;
	Action action; //当前处理的Action
	List<Action> actions; //key相同的actions.
	Future<?> future; //提交到ExcutorService后对应的Future，可以用来取消请求
	
	...
	
	hunt(); //抓取图片
	forRequest(Picasso picasso, Dispatcher dispatcher, Cache cache, Stats stats, Action action); //静态方法构造Hunter.
	attach(Action); //将其加入到actions当中
	detach(Action); //与上述方法对应
	cancel(); //取消此请求
hunt()方法源码为：

	Bitmap hunt() throws IOException {
        Bitmap bitmap = null;
        
		//是否能从缓存读取
        if (shouldReadFromMemoryCache(memoryPolicy)) {
            bitmap = cache.get(key);
            if (bitmap != null) {
                stats.dispatchCacheHit();
                loadedFrom = MEMORY;
                ...
                ...
                return bitmap;
            }
        }

        data.networkPolicy = retryCount == 0 ? NetworkPolicy.OFFLINE.index : networkPolicy;
        //通过requetHandler来处理请求
        RequestHandler.Result result = requestHandler.load(data, networkPolicy);
        if (result != null) {
            loadedFrom = result.getLoadedFrom();
            exifOrientation = result.getExifOrientation();
            bitmap = result.getBitmap();

            // If there was no Bitmap then we need to decode it from the stream.
            if (bitmap == null) {
                InputStream is = result.getStream();
                try {
                    bitmap = decodeStream(is, data);
                } finally {
                    Utils.closeQuietly(is);
                }
            }
        }

        if (bitmap != null) {
           ...
           ...
           ... //对其进行相应的转换操作
        }

        return bitmap;
    }
	
**Request类**  
此类包含着请求的数据，包括请求的Uri，对图片所要进行的转换（Transformation)，是否进行centerCrop，centerInside，是否进行旋转，旋转的角度……太简单不必细说。

**RequestCreator类**  
此类由Picasso.load(Uri)方法暴露给用户，提供各种方法，如resize(), centerCrop(), centerInside(), fit()来记录用户对图片的需求，并且更新内部所持有的Request，还提供了三个很重要的方法， fetch(), get(), into(Target)，这三个方法最后都是根据实际情况了构造了不同的Action，并且同Picasso通信，通知Picasso将Action发送出去（即通知Dispatcher来处理它）。

**RequestHandler类**  
注意这个可不是安卓线程通信所使用的Handler，这是用来处理请求的，其内部类RequestHandler.Result表示处理的结果，其结构为：

	Picasso.LoadedFrom loadedFrom;
    Bitmap bitmap;
    InputStream stream;
    int exifOrientation;
    
    //构造函数
    Result(
                @Nullable Bitmap bitmap,
                @Nullable InputStream stream,
                @NonNull Picasso.LoadedFrom loadedFrom,
                int exifOrientation) {
            //两个至少一个不为空
            if ((bitmap != null) == (stream != null)) {
                throw new AssertionError();
            }
            this.bitmap = bitmap;
            this.stream = stream;
            this.loadedFrom = checkNotNull(loadedFrom, "loadedFrom == null");
            this.exifOrientation = exifOrientation;
        }

可以通过这个类来实现自己加载图片的逻辑，需要实现两个抽象方法：

	abstract boolean canHandleRequest(Request data);
	
	abstract Result load(Request request, int networkPolicy)；
此外，Picasso库也给我们提供了一系列内置的RequestHandler，如NetworkRequestHandler, MediaStoreRequestHandler, FileRequestHandler,分别用来处理不同类型的请求，这些内置的RequestHandler将默认提供给Picasso类。

**Target类**  
这是一个接口，可以看作是一个简单的图片加载的监听器，内部有三个方法：

	void onBitmapLoaded(Bitmap bitmap, LoadedFrom from);
	
	void onBitmapFailed(Exception e, Drawable errorDrawable);
	
	void onPrepareLoad(Drawable placeHolderDrawable);
一看名字就知道是干什么的，不必多说。需要注意的一点就是ImageView可以看作是一种特殊的Target，但是IamgeView没有实现Target接口，因此Picasso库针对ImageView有特殊的处理。

**DeferredRequestHandler**  
这个既和线程通信没有联系，也和之前的RequestHandler没有联系，它实现了[ViewTreeObserver.OnPreDrawListener](https://developer.android.com/reference/android/view/ViewTreeObserver.OnDrawListener.html)这个接口。这个就是上文所说Picasso库针对ImageView的情况提供的特殊的类。当我们调用RequestCreator.fit()的时候，会触发RequestCreator中的defer标志，最后在调用into(ImageView)的时候会对目标ImageView的[ViewTreeObserver](https://developer.android.com/reference/android/view/ViewTreeObserver.html)注册一个DeferredRequestCreator，这样便在目标ImageView加入到所属的ViewTree的时候在preDraw过程中对其进行处理，处理的大致逻辑如下所示：

	public boolean onPreDraw() {
		...
		...
		int width = target.getWidth();
        int height = target.getHeight();

        if (width <= 0 || height <= 0 || target.isLayoutRequested()) {
            return true;
        }

        vto.removeOnPreDrawListener(this);
        this.target.clear();

        this.creator.unfit().resize(width, height).into(target, callback);
        return true;
	}
最后还是回归到RequestCreator的resize操作，来达到所谓的"fit exactly"的效果。

## Picasso工作流程源码分析
首先，我们从最基本的操作开始，一步一步加以分析。

	Picasso.with(Context)
          .load(Uri)
          ....
          .into(Target);
**注：**第三行的...对应的是RequestCreator的各种“操作符”，由于数量太多，这里不一一列举。  

**Picasso.with(context)**  
源码如下：

	public static Picasso with(@NonNull Context context) {
        if (context == null) {
            throw new IllegalArgumentException("context == null");
        }
        if (singleton == null) {
            synchronized (Picasso.class) {
                if (singleton == null) {
                    //默认使用OkHttp3Downloader和LruCache.
                    singleton = new Builder(context).build();
                }
            }
        }
        return singleton;
    }
这个不需要过多的解释，经典的DCL单例模式，其中需要注意的是Builder里面的build()方法：

	public Picasso build() {
            Context context = this.context;

            if (downloader == null) {
                downloader = new OkHttp3Downloader(context);
            }
            if (cache == null) {
                cache = new LruCache(context);
            }
            if (service == null) {
                service = new PicassoExecutorService();
            }
            if (transformer == null) {
                transformer = RequestTransformer.IDENTITY;
            }

            Stats stats = new Stats(cache);

            Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);

            return new Picasso(context, dispatcher, cache, listener, transformer, requestHandlers, stats,
                    defaultBitmapConfig, indicatorsEnabled, loggingEnabled);
        }
由此可以看出如果采用默认的Picasso类的话，会采用OkHttp3Downloader, LruCache和PicassoExcutorService，这些便是默认配置。

**picasso.load(Uri)**  
不论调用的是picasso.load(String)也好，还是picasso.load(File)也好都会调用picasso.load(Uri)方法(picasso.load(int)除外)，只不过是将String和File转成Uri而已，其源码也比较简单，就是构造一个RequestCreator返回而已，不再细谈。

**RequestCreator的各种“操作符”**  
由于操作符实在是太多，这里只粘出几个具有代表性的：

	public RequestCreator resize(int targetWidth, int targetHeight) {
        data.resize(targetWidth, targetHeight);
        return this;
    }
    
    public RequestCreator centerCrop() {
        data.centerCrop(Gravity.CENTER);
        return this;
    }
    
    public RequestCreator error(@DrawableRes int errorResId) {
        if (errorResId == 0) {
            throw new IllegalArgumentException("Error image resource invalid.");
        }
        if (errorDrawable != null) {
            throw new IllegalStateException("Error image already set.");
        }
        this.errorResId = errorResId;
        return this;
    }
    
    public RequestCreator fit() {
        deferred = true;
        return this;
    }
    ...
可以看到，有些操作是改变的RequestCreator本身具有的一些属性（error(), fit())，还有一些便是将参数传给了data，这个data类型是一个Request.Builder，RequestCreator在这个时候便充当了“中介人”的角色吧。

**RequestCreator.into(...)**  
这个方法便是根据参数的类型来构造一个Action并通知Picasso类发送出去。

	public void into(@NonNull Target target) {
        ...
        ...

        Request request = createRequest(started);
        String requestKey = createKey(request);
			
		//先判断能否从缓存当中读取
        if (shouldReadFromMemoryCache(memoryPolicy)) {
            Bitmap bitmap = picasso.quickMemoryCacheCheck(requestKey);
            if (bitmap != null) {
                picasso.cancelRequest(target);
                target.onBitmapLoaded(bitmap, MEMORY);
                return;
            }
        }

        target.onPrepareLoad(setPlaceholder ? getPlaceholderDrawable() : null);
        
		//构造Action并发送
        Action action =
                new TargetAction(picasso, target, request, memoryPolicy, networkPolicy, errorDrawable,
                        requestKey, tag, errorResId);
        picasso.enqueueAndSubmit(action);
    }
当参数为ImageView的时候有特殊情况：

	public void into(ImageView target, Callback callback) {
        ...
        ...

        if (deferred) {
            if (data.hasSize()) {
                throw new IllegalStateException("Fit cannot be used with resize.");
            }
            int width = target.getWidth();
            int height = target.getHeight();
            if (width == 0 || height == 0 || target.isLayoutRequested()) {
                if (setPlaceholder) {
                    setPlaceholder(target, getPlaceholderDrawable());
                }
                picasso.defer(target, new DeferredRequestCreator(this, target, callback));
                return;
            }
            data.resize(width, height);
        }

        ...
        ...

        Action action =
                new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,
                        errorDrawable, requestKey, tag, callback, noFade);

        picasso.enqueueAndSubmit(action);
    }
如果之前调用过fit方法的时候，deferred标志位就会开启，这个时候就会构造一个DeferredRequestCreator，并注册到目标ImageView的ViewTreeObserver当中，并且将其写入picasso所持有的targetToDeferredRequestCreator当中。

**Picasso.enqueueAndSubmit(Action)**  
正如上面看到的，RequestCreator.into(...)到最后一步都是构建对应的Action并调用picasso.enqueueAndSubmit(action)方法，下面就顺着这个方法进行分析：

	void enqueueAndSubmit(Action action) {
        Object target = action.getTarget();
        if (target != null && targetToAction.get(target) != action) {
            // This will also check we are on the main thread.
            cancelExistingRequest(target);
            targetToAction.put(target, action);
        }
        submit(action);
    }
先判断以传进来的的action的target作为键在targetToAction中得来的的action值是不是当前传进来的action，如果不是，则先取消target的请求，再将target和当前传递的action键值对传到字典当中，最后调用submit方法，submit方法如下：

	void submit(Action action) {
        dispatcher.dispatchSubmit(action);
    }
很简单粗暴地就调用了dispatcher的dispatchSubmit(action)方法来处理此action，下面来分析dispatcher.dispatchSubmit(action)方法。

**dispatcher.dispatchSubmit(action)**  
源码如下：

	void dispatchSubmit(Action action) {
    	handler.sendMessage(handler.obtainMessage(REQUEST_SUBMIT, action));
    }
一层一层往下看，最终调用的是performSubmit(Action)方法：  

	void performSubmit(Action action) {
    	performSubmit(action, true);
  	}

  	void performSubmit(Action action, boolean dismissFailed) {
   		 if (pausedTags.contains(action.getTag())) {
      		pausedActions.put(action.getTarget(), action);
      		...
      		...
      		return;
    	}

    	BitmapHunter hunter = hunterMap.get(action.getKey());
    	if (hunter != null) {
      		hunter.attach(action);
      		return;
    	}

    	if (service.isShutdown()) {
      		...
      		...
      		return;
    	}

    	hunter = forRequest(action.getPicasso(), this, cache, stats, action);
    	hunter.future = service.submit(hunter);
    	hunterMap.put(action.getKey(), hunter);
    	if (dismissFailed) {
      		failedActions.remove(action.getTarget());
    	}
    	...
    	...
    }
逻辑也不是多么的复杂，如果标记Action的Tag在pausedTag当中的话（下面会讲tag对应的操作），便直接返回不做处理。再判断hunterMap中是不是有传入的action的key对应的BitmapHunter，如果有，便调用attach方法并返回。attach方法较简单，更新BitmapHunter当前的action或者将其加入到List<Action>当中。下一步判断ExcutorService是不是已经处在关闭的状态，这是为下面一步做准备。如果没有，便构造出一个BitmapHunter，并将这个Hunter（本身便是一个Runnable)加入到service当中，最后更新hunterMap。

**dispatcher.dispatchComplete(BitmapHunter) dispatcher.dispatchFailed(BitmapHunter)**  
当此Runnable加入到ExcutorService当中后便开始运行，其run方法如下所示：

	public void run() {
        try {
            updateThreadName(data);

            ...
            ...

            result = hunt();

            if (result == null) {
                dispatcher.dispatchFailed(this);
            } else {
                dispatcher.dispatchComplete(this);
            }
        } ...
         finally {
            Thread.currentThread().setName(Utils.THREAD_IDLE_NAME);
        }
    }
每一个catch字句最后都会调用dispatcher.dispatchFailed(BitmapHunter)，逻辑比较简单，这里省去。run()方法内部还是比较直观的，先是更新当前线程的名称，再调用hunt()方法获取到一个Bitmap，如果为空就调用dispatchFailed，不为空就调用dispatchComplete。  

同样，两个dispatch方法也只是通过DispatcherThreadHandler通知DispatcherThread进行相应的操作，最后调用的是performComplete(BitmapHunter)和performError(BitmapHunter)方法，源码如下：

	void performComplete(BitmapHunter hunter) {
		//是否应该写入缓存
    	if (shouldWriteToMemoryCache(hunter.getMemoryPolicy())) {
      		cache.set(hunter.getKey(), hunter.getResult());
    	}
    	hunterMap.remove(hunter.getKey());
    	batch(hunter);
  	}
  	
  	void performError(BitmapHunter hunter, boolean willReplay) {
   		hunterMap.remove(hunter.getKey());
    	batch(hunter);
   	}
逻辑比较简单，两者共同的操作是更新hunterMap和调用batch(hunter)，区别就是performComplete方法第一步先判断是否要写入缓存。batch(BitmapHunter)方法源码如下：

	private void batch(BitmapHunter hunter) {
    	if (hunter.isCancelled()) {
      		return;
    	}
    	batch.add(hunter);
    	if (!handler.hasMessages(HUNTER_DELAY_NEXT_BATCH)) {
      		handler.sendEmptyMessageDelayed(HUNTER_DELAY_NEXT_BATCH, BATCH_DELAY);
    	}
  	}
逻辑也是比较直观的，其中batch是一个List<BitmapHunter>，这个和下面sendEmptyMessageDelayed方法结合在一块用，起到了一定的缓存的作用。这个消息发送出去之后，最终调用的是performBatchComplete()，源码如下：

	void performBatchComplete() {
    	List<BitmapHunter> copy = new ArrayList<>(batch);
    	batch.clear();
    	mainThreadHandler.sendMessage(mainThreadHandler.obtainMessage(HUNTER_BATCH_COMPLETE, copy));
  	}
此方法先将缓存复制一份，再将缓存清空，最后通知主线程开始处理复制好的缓存数据，最终调用的是Picasso当中的complete(BitmapHunter)方法。

**Picasso.complete(BitmapHunter)**  
源码如下：

	void complete(BitmapHunter hunter) {
        Action single = hunter.getAction();
        List<Action> joined = hunter.getActions();

        boolean hasMultiple = joined != null && !joined.isEmpty();
        boolean shouldDeliver = single != null || hasMultiple;

        if (!shouldDeliver) {
            return;
        }

        Uri uri = hunter.getData().uri;
        Exception exception = hunter.getException();
        Bitmap result = hunter.getResult();
        LoadedFrom from = hunter.getLoadedFrom();

        if (single != null) {
            deliverAction(result, from, single, exception);
        }

        if (hasMultiple) {
            //noinspection ForLoopReplaceableByForEach
            for (int i = 0, n = joined.size(); i < n; i++) {
                Action join = joined.get(i);
                deliverAction(result, from, join, exception);
            }
        }

        if (listener != null && exception != null) {
            listener.onImageLoadFailed(this, uri, exception);
        }
    }
如果这个Hunter只有一个那就调用一次deliverAction(...)，如果获得的joined列表不为空，那就对其中的Action依次调用deliverAction(...)， deliverAction(...)源码如下：

	private void deliverAction(Bitmap result, LoadedFrom from, Action action, Exception e) {
        ...
        ...
        if (result != null) {
            ...
            action.complete(result, from);
            ...
        } else {
            action.error(e);
            ...
        }
    }
不必多说，最后根据result是否为空调用action的complete或者error方法。就拿ImageViewAction来说吧，成功的话，就会将图片加载到对应的ImageView中去（此时ImageView是此Action的Target)，失败则加载errorImage，整个工作流程就此结束。