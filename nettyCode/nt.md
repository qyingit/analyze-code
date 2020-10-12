## NIOEventloopGroup

### 创建EventLoopGroup

```
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
```

#### 步骤：

1.创建provider

```java
//io.netty.channel.nio.NioEventLoopGroup#NioEventLoopGroup(int, java.util.concurrent.Executor, java.nio.channels.spi.SelectorProvider)
this(nThreads, executor, SelectorProvider.provider());
 // java.nio.channels.spi.SelectorProvider#provider
 public static SelectorProvider provider() {
        synchronized (lock) {
            if (provider != null)
                return provider;
            return AccessController.doPrivileged(
                new PrivilegedAction<SelectorProvider>() {
                    public SelectorProvider run() {
                            if (loadProviderFromProperty())
                                return provider;
                            if (loadProviderAsService())
                                return provider;
                            provider = sun.nio.ch.DefaultSelectorProvider.create();
                            return provider;
                        }
                    });
        }
    }
```

2.设置选择策略工厂

```java
this(nThreads, executor, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
```

3.设置拒绝策略

```java
super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
```

4.判断并设置线程数

```java
super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
// 如果不设置线程，创建线程数为默认cpu核心数*2
//io.netty.channel.MultithreadEventLoopGroup#DEFAULT_EVENT_LOOP_THREADS
 static {
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
        }
    }
```

5.创建executor

```java
// io.netty.util.concurrent.MultithreadEventExecutorGroup#MultithreadEventExecutorGroup(int, java.util.concurrent.Executor, io.netty.util.concurrent.EventExecutorChooserFactory, java.lang.Object...)
executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
// io.netty.channel.MultithreadEventLoopGroup#newDefaultThreadFactory
protected ThreadFactory newDefaultThreadFactory() {
        return new DefaultThreadFactory(getClass(), Thread.MAX_PRIORITY);
    }

public DefaultThreadFactory(Class<?> poolType, boolean daemon, int priority) {
        this(toPoolName(poolType), daemon, priority);
    }

//根据poolType为NioEventLoopGroup.class创建线程名nioEventLoopGroup
public static String toPoolName(Class<?> poolType) {
        ObjectUtil.checkNotNull(poolType, "poolType");

        String poolName = StringUtil.simpleClassName(poolType);
        switch (poolName.length()) {
            case 0:
                return "unknown";
            case 1:
                return poolName.toLowerCase(Locale.US);
            default:
                if (Character.isUpperCase(poolName.charAt(0)) && Character.isLowerCase(poolName.charAt(1))) {
                    return Character.toLowerCase(poolName.charAt(0)) + poolName.substring(1);
                } else {
                    return poolName;
                }
        }
    }

//设置prefix，daemon，优先级。threadGroup
  public DefaultThreadFactory(String poolName, boolean daemon, int priority, ThreadGroup threadGroup) {
        ObjectUtil.checkNotNull(poolName, "poolName");

        if (priority < Thread.MIN_PRIORITY || priority > Thread.MAX_PRIORITY) {
            throw new IllegalArgumentException(
                    "priority: " + priority + " (expected: Thread.MIN_PRIORITY <= priority <= Thread.MAX_PRIORITY)");
        }

        prefix = poolName + '-' + poolId.incrementAndGet() + '-';
        this.daemon = daemon;
        this.priority = priority;
        this.threadGroup = threadGroup;
    }

```

6.创建线程数组并赋值

```java
// 根据传递线程数创建线程数组
children = new EventExecutor[nThreads];
//循环赋值
 for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
              
//创建具体的线程信息  io.netty.channel.nio.NioEventLoopGroup#newChild    
  protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    // 判断参数是否齐全
        EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
    // 创建线程
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
    }
```

7.创建NioEventLoop

```java
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
             SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
             EventLoopTaskQueueFactory queueFactory) {
    super(parent, executor, false, newTaskQueue(queueFactory), newTaskQueue(queueFactory),
            rejectedExecutionHandler);
    this.provider = ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
    this.selectStrategy = ObjectUtil.checkNotNull(strategy, "selectStrategy");
    final SelectorTuple selectorTuple = openSelector();
    this.selector = selectorTuple.selector;
    this.unwrappedSelector = selectorTuple.unwrappedSelector;
}

// 创建SingleThreadEventLoop
 protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor,
                                    boolean addTaskWakesUp, Queue<Runnable> taskQueue, Queue<Runnable> tailTaskQueue,
                                    RejectedExecutionHandler rejectedExecutionHandler) {
        super(parent, executor, addTaskWakesUp, taskQueue, rejectedExecutionHandler);
        tailTasks = ObjectUtil.checkNotNull(tailTaskQueue, "tailTaskQueue");
    }

// 为属性赋值
 protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                        boolean addTaskWakesUp, Queue<Runnable> taskQueue,
                                        RejectedExecutionHandler rejectedHandler) {
        super(parent);
        this.addTaskWakesUp = addTaskWakesUp;
        this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
        this.executor = ThreadExecutorMap.apply(executor, this);
        this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
        this.rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
    }

//io.netty.util.internal.ThreadExecutorMap#apply(java.lang.Runnable, io.netty.util.concurrent.EventExecutor)
public static Runnable apply(final Runnable command, final EventExecutor eventExecutor) {
        ObjectUtil.checkNotNull(command, "command");
        ObjectUtil.checkNotNull(eventExecutor, "eventExecutor");
        return new Runnable() {
            @Override
            public void run() {
              // 将nioEventLoop设置到threadLocalMap中
                setCurrentEventExecutor(eventExecutor);
                try {
                    command.run();
                } finally {
                    setCurrentEventExecutor(null);
                }
            }
        };
    }

//创建Selector
private SelectorTuple openSelector() {
	// 利用操作系统创建
	unwrappedSelector = provider.openSelector();
  
   // DISABLE_KEY_SET_OPTIMIZATION是否对key优化
  
  //对key优化流程 对sun.nio.ch.SelectorImpl里面的属性进行替换
  // private Set<SelectionKey> publicKeys;
  // private Set<SelectionKey> publicSelectedKeys;
  final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

        Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
                    Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

                    if (PlatformDependent.javaVersion() >= 9 && PlatformDependent.hasUnsafe()) {
                        // Let us try to use sun.misc.Unsafe to replace the SelectionKeySet.
                        // This allows us to also do this in Java9+ without any extra flags.
                        long selectedKeysFieldOffset = PlatformDependent.objectFieldOffset(selectedKeysField);
                        long publicSelectedKeysFieldOffset =
                                PlatformDependent.objectFieldOffset(publicSelectedKeysField);

                        if (selectedKeysFieldOffset != -1 && publicSelectedKeysFieldOffset != -1) {
                            PlatformDependent.putObject(
                                    unwrappedSelector, selectedKeysFieldOffset, selectedKeySet);
                            PlatformDependent.putObject(
                                    unwrappedSelector, publicSelectedKeysFieldOffset, selectedKeySet);
                            return null;
                        }
                        // We could not retrieve the offset, lets try reflection as last-resort.
                    }

                    Throwable cause = ReflectionUtil.trySetAccessible(selectedKeysField, true);
                    if (cause != null) {
                        return cause;
                    }
                    cause = ReflectionUtil.trySetAccessible(publicSelectedKeysField, true);
                    if (cause != null) {
                        return cause;
                    }

                    selectedKeysField.set(unwrappedSelector, selectedKeySet);
                    publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
                    return null;
                } catch (NoSuchFieldException e) {
                    return e;
                } catch (IllegalAccessException e) {
                    return e;
                }
            }
        });
  
}
// 创建一个新的SelectorTuple组合，里面有没有包装的selecor与包装的selector
 return new SelectorTuple(unwrappedSelector,
                                 new SelectedSelectionKeySetSelector(unwrappedSelector, selectedKeySet));
```

8.创建线程选择器

```java
chooser = chooserFactory.newChooser(children);
//io.netty.util.concurrent.DefaultEventExecutorChooserFactory#newChooser
//是2与不是2的倍数创建两种线程选择器
 public EventExecutorChooser newChooser(EventExecutor[] executors) {
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTwoEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }

//PowerOfTwoEventExecutorChooser
  @Override
  public EventExecutor next() {
    return executors[idx.getAndIncrement() & executors.length - 1];
  }
//GenericEventExecutorChooser
  public EventExecutor next() {
    return executors[Math.abs(idx.getAndIncrement() % executors.length)];
  }
```

9.创建FutureListener并注册到EventExecutor上

```java
final FutureListener<Object> terminationListener = new FutureListener<Object>();

for (EventExecutor e: children) {
  e.terminationFuture().addListener(terminationListener);
}

```

10.生成只读的集合

```
Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
Collections.addAll(childrenSet, children);
readonlyChildren = Collections.unmodifiableSet(childrenSet);
```



## ServerBootstrap

1.创建ServerBootstrap

```java
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class)
 // 设置属性信息
 .option(ChannelOption.SO_BACKLOG, 100)
 .handler(new LoggingHandler(LogLevel.INFO))
 .childHandler(new TelnetServerInitializer(sslCtx));
 // io.netty.bootstrap.ServerBootstrap#group(io.netty.channel.EventLoopGroup, io.netty.channel.EventLoopGroup)
```

2.创建SocketChannel的channelFactory

```java
public B channel(Class<? extends C> channelClass) {
    return channelFactory(new ReflectiveChannelFactory<C>(
            ObjectUtil.checkNotNull(channelClass, "channelClass")
    ));
}
```

3.设置handler

```java
public B handler(ChannelHandler handler) {
    this.handler = ObjectUtil.checkNotNull(handler, "handler");
    return self();
}
```

4.设置childHandler,

```java
serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
	//
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline p = ch.pipeline();
        if (sslCtx != null) {
       		// 在绑定端口的时候会调用initChannel()方法添加p.addLast
            p.addLast(sslCtx.newHandler(ch.alloc()));
        }
        //p.addLast(new LoggingHandler(LogLevel.INFO));
        p.addLast(serverHandler);
    }
});

```

5.调用bind方法绑定端口号

```java
b.bind(PORT).sync().channel().closeFuture().sync();
// io.netty.bootstrap.AbstractBootstrap#bind(int)
// 创建socket
public ChannelFuture bind(int inetPort) {
  return bind(new InetSocketAddress(inetPort));
}
```

6.参数校验validate

```java
public ChannelFuture bind(SocketAddress localAddress) {
    validate();
    return doBind(ObjectUtil.checkNotNull(localAddress, "localAddress"));
}
// group与channelFactory不能为空
public B validate() {
  if (group == null) {
  throw new IllegalStateException("group not set");
  }
  if (channelFactory == null) {
  throw new IllegalStateException("channel or channelFactory not set");
  }	
  return self();
}
```

7.调用doBind方法

io.netty.bootstrap.AbstractBootstrap#doBind

8.初始化并注册channel

```java
final ChannelFuture regFuture = initAndRegister();
// 利用工厂创建channle，本次为NioServerSocketChannel
final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
          // io.netty.bootstrap.AbstractBootstrap#initAndRegister
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
                // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
        return regFuture;
    }

//init初始化channel
 void init(Channel channel) {
   		//设置bootstrap配置的options
        setChannelOptions(channel, newOptionsArray(), logger);
   		//设置bootstrap配置的attrs
        setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));
		//获取pipeline
        ChannelPipeline p = channel.pipeline();
		
   		//赋值
        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
   		// 子group的Options
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
        }
   		// 子group的Attrs
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);
		
   		// 增加handler
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }
				// 这是一个运行bossgroup的方法
                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        //pipelinde增加ServerBootstrapAcceptor
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```

9.将channel注册到bossgroup中

```java
//config().group() 获取bossgroup 
ChannelFuture regFuture = config().group().register(channel);

//io.netty.channel.SingleThreadEventLoop#register(io.netty.channel.Channel)
@Override
public ChannelFuture register(Channel channel) {
  return register(new DefaultChannelPromise(channel, this));
}

// 校验channel不能为空,同时EventLoop不能为空 
 public DefaultChannelPromise(Channel channel, EventExecutor executor) {
        super(executor);
        this.channel = checkNotNull(channel, "channel");
    }

// 注册promise与eventloopgroup
 @Override
public ChannelFuture register(final ChannelPromise promise) {
  ObjectUtil.checkNotNull(promise, "promise");
  promise.channel().unsafe().register(this, promise);
  return promise;
}
// eventLoop绑定在channel上
AbstractChannel.this.eventLoop = eventLoop;
//判断当前线程是否为创建的eventloop
eventLoop.inEventLoop()
  
//使用创建的eventloop注册promise
  try {
    eventLoop.execute(new Runnable() {
      @Override
      public void run() {
        register0(promise);
      }
    });
  }
```

10.使用创建的eventloop注册promise

```java
private void register0(ChannelPromise promise) {
    try {
        // check if the channel is still open as it could be closed in the mean time when the register
        // call was outside of the eventLoop
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        doRegister();
        neverRegistered = false;
        registered = true;

        // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
        // user may already fire events through the pipeline in the ChannelFutureListener.
      //调用handler增加的事件
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                // This channel was registered before and autoRead() is set. This means we need to begin read
                // again so that we process inbound data.
                //
                // See https://github.com/netty/netty/issues/4805
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}

// 
@Override
protected void doRegister() throws Exception {
  boolean selected = false;
  for (;;) {
    try {
      //1.返回NioChannel
      //2.返回没有包装的selector
      selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
      return;
    } catch (CancelledKeyException e) {
      if (!selected) {
        // Force the Selector to select now as the "canceled" SelectionKey may still be
        // cached and not removed because no Select.select(..) operation was called yet.
        eventLoop().selectNow();
        selected = true;
      } else {
        // We forced a select operation on the selector before but the SelectionKey is still cached
        // for whatever reason. JDK bug ?
        throw e;
      }
    }
  }
}
```

11.调用handler增加的事件

```java
 pipeline.invokeHandlerAddedIfNeeded();
//调用add事件
final void invokeHandlerAddedIfNeeded() {
        assert channel.eventLoop().inEventLoop();
        if (firstRegistration) {
            firstRegistration = false;
            // We are now registered to the EventLoop. It's time to call the callbacks for the ChannelHandlers,
            // that were added before the registration was done.
            callHandlerAddedForAllHandlers();
        }
    }


 private void callHandlerAddedForAllHandlers() {
        final PendingHandlerCallback pendingHandlerCallbackHead;
        synchronized (this) {
            assert !registered;

            // This Channel itself was registered.
            registered = true;

            pendingHandlerCallbackHead = this.pendingHandlerCallbackHead;
            // Null out so it can be GC'ed.
            this.pendingHandlerCallbackHead = null;
        }

        // This must happen outside of the synchronized(...) block as otherwise handlerAdded(...) may be called while
        // holding the lock and so produce a deadlock if handlerAdded(...) will try to add another handler from outside
        // the EventLoop.
        PendingHandlerCallback task = pendingHandlerCallbackHead;
        while (task != null) {
            task.execute();
            task = task.next;
        }
    }

//获取executor执行新增
@Override
void execute() {
  EventExecutor executor = ctx.executor();
  if (executor.inEventLoop()) {
    //调用io.netty.channel.ChannelInitializer#handlerAdded方法
    callHandlerAdded0(ctx);
  } else {
    try {
      executor.execute(this);
    } catch (RejectedExecutionException e) {
      if (logger.isWarnEnabled()) {
        logger.warn(
          "Can't invoke handlerAdded() as the EventExecutor {} rejected it, removing handler {}.",
          executor, ctx.name(), e);
      }
      atomicRemoveFromHandlerList(ctx);
      ctx.setRemoved();
    }
  }
}

//调用initChannel初始化
 @Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
  if (ctx.channel().isRegistered()) {
    // This should always be true with our current DefaultChannelPipeline implementation.
    // The good thing about calling initChannel(...) in handlerAdded(...) is that there will be no ordering
    // surprises if a ChannelInitializer will add another ChannelInitializer. This is as all handlers
    // will be added in the expected order.
    
    if (initChannel(ctx)) {

      // We are done with init the Channel, removing the initializer now.
      removeState(ctx);
    }
  }
}

// 回调io.netty.channel.ChannelInitializer#initChannel方法
 @SuppressWarnings("unchecked")
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
  if (initMap.add(ctx)) { // Guard against re-entrance.
    try {
      initChannel((C) ctx.channel());
    } catch (Throwable cause) {
      // Explicitly call exceptionCaught(...) as we removed the handler before calling initChannel(...).
      // We do so to prevent multiple calls to initChannel(...).
      exceptionCaught(ctx, cause);
    } finally {
      ChannelPipeline pipeline = ctx.pipeline();
      if (pipeline.context(this) != null) {
        pipeline.remove(this);
      }
    }
    return true;
  }
  return false;
}
```

12.回调initChannel方法

```java
//io.netty.channel.ChannelInitializer#initChannel
p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(final Channel ch) {
        final ChannelPipeline pipeline = ch.pipeline();
        //获取并增加handler
        ChannelHandler handler = config.handler();
        if (handler != null) {
            pipeline.addLast(handler);
        }

        ch.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        });
    }
});
```

13.创建ServerBootstrapAcceptor

```java
//  pipeline.addLast(new ServerBootstrapAcceptor(ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
ServerBootstrapAcceptor(
        final Channel channel, EventLoopGroup childGroup, ChannelHandler childHandler,
        Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
    //线程组
    this.childGroup = childGroup;
    //子处理器
    this.childHandler = childHandler;
    //子类配置信息
    this.childOptions = childOptions;
    this.childAttrs = childAttrs;

    // Task which is scheduled to re-enable auto-read.
    // It's important to create this Runnable before we try to submit it as otherwise the URLClassLoader may
    // not be able to load the class because of the file limit it already reached.
    //
    // See https://github.com/netty/netty/issues/1328
    //自动读任务
    enableAutoReadTask = new Runnable() {
        @Override
        public void run() {
            channel.config().setAutoRead(true);
        }
    };
}
```

14.执行execute方法

```java
ch.eventLoop().execute(new Runnable() {
   //增加连接器
});
// io.netty.util.concurrent.SingleThreadEventExecutor#execute(java.lang.Runnable)
private void execute(Runnable task, boolean immediate) {
  boolean inEventLoop = inEventLoop();
  addTask(task);
  if (!inEventLoop) {
    startThread();
    if (isShutdown()) {
      boolean reject = false;
      try {
        if (removeTask(task)) {
          reject = true;
        }
      } catch (UnsupportedOperationException e) {
        // The task queue does not support removal so the best thing we can do is to just move on and
        // hope we will be able to pick-up the task before its completely terminated.
        // In worst case we will log on termination.
      }
      if (reject) {
        reject();
      }
    }
  }

  if (!addTaskWakesUp && immediate) {
    wakeup(inEventLoop);
  }
}
```

15.启用线程

```java
private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() {
        @Override
        public void run() {
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;
            updateLastExecutionTime();
            try {
                //调用run方法
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                for (;;) {
                    int oldState = state;
                    if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                            SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                        break;
                    }
                }

                // Check if confirmShutdown() was called at the end of the loop.
                if (success && gracefulShutdownStartTime == 0) {
                    if (logger.isErrorEnabled()) {
                        logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                                "be called before run() implementation terminates.");
                    }
                }

                try {
                    // Run all remaining tasks and shutdown hooks. At this point the event loop
                    // is in ST_SHUTTING_DOWN state still accepting tasks which is needed for
                    // graceful shutdown with quietPeriod.
                    for (;;) {
                        if (confirmShutdown()) {
                            break;
                        }
                    }

                    // Now we want to make sure no more tasks can be added from this point. This is
                    // achieved by switching the state. Any new tasks beyond this point will be rejected.
                    for (;;) {
                        int oldState = state;
                        if (oldState >= ST_SHUTDOWN || STATE_UPDATER.compareAndSet(
                                SingleThreadEventExecutor.this, oldState, ST_SHUTDOWN)) {
                            break;
                        }
                    }

                    // We have the final set of tasks in the queue now, no more can be added, run all remaining.
                    // No need to loop here, this is the final pass.
                    confirmShutdown();
                } finally {
                    try {
                        cleanup();
                    } finally {
                        // Lets remove all FastThreadLocals for the Thread as we are about to terminate and notify
                        // the future. The user may block on the future and once it unblocks the JVM may terminate
                        // and start unloading classes.
                        // See https://github.com/netty/netty/issues/6596.
                        FastThreadLocal.removeAll();

                        STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                        threadLock.countDown();
                        int numUserTasks = drainTasks();
                        if (numUserTasks > 0 && logger.isWarnEnabled()) {
                            logger.warn("An event executor terminated with " +
                                    "non-empty task queue (" + numUserTasks + ')');
                        }
                        terminationFuture.setSuccess(null);
                    }
                }
            }
        }
    });
}
```

16.run方法开启循环监听端口信息

```java
//io.netty.channel.nio.NioEventLoop#run
//hasTasks()判断队列有没有任务，有任务返回SelectStrategy.SELECT
strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
//如果没有任务调用selectNow方法
private final IntSupplier selectNowSupplier = new IntSupplier() {
  @Override
  public int get() throws Exception {
    return selectNow();
  }
};
//调用jdk的selectNow方法，判断是否有读写事件
int selectNow() throws IOException {
  return selector.selectNow();
}

//如果有任务则等待一段时间再select
 if (!hasTasks()) {
   strategy = select(curDeadlineNanos);
 }

//根据ioRatio去控制处理任务时间
final int ioRatio = this.ioRatio;
boolean ranTasks;
//you'xuan
if (ioRatio == 100) {
  try {
    if (strategy > 0) {
      //处理有消息事件
      processSelectedKeys();
    }
  } finally {
    // Ensure we always run tasks.
    //运行任务
    ranTasks = runAllTasks();
  }
} else if (strategy > 0) {
  final long ioStartTime = System.nanoTime();
  try {
    //处理事件
    processSelectedKeys();
  } finally {
    // Ensure we always run tasks.
    final long ioTime = System.nanoTime() - ioStartTime;
    //在一定时间范围内运行任务
    ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
  }
} else {
  ranTasks = runAllTasks(0); // This will run the minimum number of tasks
}


 if (ranTasks || strategy > 0) {
   // 有任务运行或者有事件
   //如果计数器值大于3将计数器置为0
   if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
     logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                  selectCnt - 1, selector);
   }
   selectCnt = 0;
   //unexpectedSelectorWakeup 会对计数器进行判断，如果大于系统默认值512会重新注册selector防止空轮询
 } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
   selectCnt = 0;
 }
```

17.pipeline增加handler

```java
//pipeline在channel初始化的时候赋值
pipeline.addLast(new ServerBootstrapAcceptor(
        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));

//io.netty.channel.DefaultChannelPipeline#addLast(io.netty.util.concurrent.EventExecutorGroup, io.netty.channel.ChannelHandler...)
//判断handker不能为空，循环添加
 public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
   ObjectUtil.checkNotNull(handlers, "handlers");

   for (ChannelHandler h: handlers) {
     if (h == null) {
       break;
     }
     addLast(executor, null, h);
   }

   return this;
 }

//
 public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
   final AbstractChannelHandlerContext newCtx;
   synchronized (this) {
     //判断handler是否重复添加，Sharable注解可以重复添加
     checkMultiplicity(handler);
	//生成ctx，名字不能够重复
     newCtx = newContext(group, filterName(name, handler), handler);
	
     //使用链表增加
     addLast0(newCtx);

     // If the registered is false it means that the channel was not registered on an eventLoop yet.
     // In this case we add the context to the pipeline and add a task that will call
     // ChannelHandler.handlerAdded(...) once the channel is registered.
     if (!registered) {
       newCtx.setAddPending();
       callHandlerCallbackLater(newCtx, true);
       return this;
     }

     EventExecutor executor = newCtx.executor();
     if (!executor.inEventLoop()) {
       callHandlerAddedInEventLoop(newCtx, executor);
       return this;
     }
   }
   //ctx新增事件
   callHandlerAdded0(newCtx);
   return this;
 }

//生成 ctx
private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
        return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
    }
```

## GOODMethod

```java
/*
 * Copyright 2014 The Netty Project
 *
 * The Netty Project licenses this file to you under the Apache License, version 2.0 (the
 * "License"); you may not use this file except in compliance with the License. You may obtain a
 * copy of the License at:
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software distributed under the License
 * is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express
 * or implied. See the License for the specific language governing permissions and limitations under
 * the License.
 */
package io.netty.util.internal;

import java.util.Collection;

/**
 * A grab-bag of useful utility methods.
 */
public final class ObjectUtil {

    private ObjectUtil() {
    }

    /**
     * Checks that the given argument is not null. If it is, throws {@link NullPointerException}.
     * Otherwise, returns the argument.
     */
    public static <T> T checkNotNull(T arg, String text) {
        if (arg == null) {
            throw new NullPointerException(text);
        }
        return arg;
    }

    /**
     * Checks that the given argument is strictly positive. If it is not, throws {@link IllegalArgumentException}.
     * Otherwise, returns the argument.
     */
    public static int checkPositive(int i, String name) {
        if (i <= 0) {
            throw new IllegalArgumentException(name + ": " + i + " (expected: > 0)");
        }
        return i;
    }

    /**
     * Checks that the given argument is strictly positive. If it is not, throws {@link IllegalArgumentException}.
     * Otherwise, returns the argument.
     */
    public static long checkPositive(long i, String name) {
        if (i <= 0) {
            throw new IllegalArgumentException(name + ": " + i + " (expected: > 0)");
        }
        return i;
    }

    /**
     * Checks that the given argument is positive or zero. If it is not , throws {@link IllegalArgumentException}.
     * Otherwise, returns the argument.
     */
    public static int checkPositiveOrZero(int i, String name) {
        if (i < 0) {
            throw new IllegalArgumentException(name + ": " + i + " (expected: >= 0)");
        }
        return i;
    }

    /**
     * Checks that the given argument is positive or zero. If it is not, throws {@link IllegalArgumentException}.
     * Otherwise, returns the argument.
     */
    public static long checkPositiveOrZero(long i, String name) {
        if (i < 0) {
            throw new IllegalArgumentException(name + ": " + i + " (expected: >= 0)");
        }
        return i;
    }

    /**
     * Checks that the given argument is neither null nor empty.
     * If it is, throws {@link NullPointerException} or {@link IllegalArgumentException}.
     * Otherwise, returns the argument.
     */
    public static <T> T[] checkNonEmpty(T[] array, String name) {
        checkNotNull(array, name);
        checkPositive(array.length, name + ".length");
        return array;
    }

    /**
     * Checks that the given argument is neither null nor empty.
     * If it is, throws {@link NullPointerException} or {@link IllegalArgumentException}.
     * Otherwise, returns the argument.
     */
    public static <T extends Collection<?>> T checkNonEmpty(T collection, String name) {
        checkNotNull(collection, name);
        checkPositive(collection.size(), name + ".size");
        return collection;
    }

    /**
     * Resolves a possibly null Integer to a primitive int, using a default value.
     * @param wrapper the wrapper
     * @param defaultValue the default value
     * @return the primitive value
     */
    public static int intValue(Integer wrapper, int defaultValue) {
        return wrapper != null ? wrapper : defaultValue;
    }

    /**
     * Resolves a possibly null Long to a primitive long, using a default value.
     * @param wrapper the wrapper
     * @param defaultValue the default value
     * @return the primitive value
     */
    public static long longValue(Long wrapper, long defaultValue) {
        return wrapper != null ? wrapper : defaultValue;
    }
}

```