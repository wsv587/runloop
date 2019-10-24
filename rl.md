[toc]



# 前言

对iOS开发者而言，runloop是一个老生常谈的话题，但凡是iOS开发者，在工作中必然直接或间接的接触过runloop。而对于面试者而言，runloop又几乎是必考点。在4年前，笔者写过一篇文章[NSRunLoop](https://www.cnblogs.com/wsnb/p/4753685.html)，对runloop原理以及应用场景做了基本介绍。但是对于不了解源码的同学，读起来也只能理解皮毛，不看源码的话，读了n多篇也是似懂非懂，对于原理也只能是死记硬背。所以，本文将从源码的角度剖析runloop的组成，强化自己对runloop的认识，验证我们脑海中一直以来似懂非懂的原理，真心希望这篇文章能够帮助到大家。
# 为什么是runLoop
runloop顾名思义就是”跑圈“，所谓跑圈就给人一种循环的感觉。runloop核心代码就是一个有状态的do...while循环。每循环一次就相当于跑了一圈，线程就会对当前这一圈里面产生的事件进行处理。那么为什么线程要有runloop呢？其实我们的APP可以理解为是靠event驱动的。我们触摸屏幕、网络回调等都是一个个的event，也就是事件。这些事件产生之后会分发给我们的APP，APP接收到事件之后分发给对应的线程。要想APP的线程一直能够处理事件，就要保活线程，也就是不能让线程早早的退出，此时runloop就派上用场了。我们已经说了，runloop本质上就是一个有状态的do...while循环，所以只要不是超时或者退出状态，那么runLoop就会一直执行do...while，所以可以保证线程不退出。其实也不是必须要给线程指定一个runloop，如果需要我们线程能够持续的处理事件，那么就需要给线程绑定一个runloop。也就是说，runloop能够保证线程一直可以一直处理事件。
一句话概括runLoop：一个有状态的事件驱动的do...while循环。

# runLoop 结构
runLoop的结构如下图所示：

为了便于说明，下文将对线程（thread）简称T，RunLoop简称RL，CFRunLoopMode简称RLM，CFRunLoopSource简称RLS，CFRunLoopObserver简称RLO。下图说明了以下信息：

- 一个T对应一个RL
- Cocoa层的NSRunLoop是对CF层的CFRunLoop的封装
- 一个RL对应多个RLM
- 一个RL一次只能执行一个RLM，要想切换RLM需要退出当前RLM重新进入新的RLM。
  - 一次执行一个mode的好处在于，底层设计相对简单，避免不同的mode耦合在一起，代码相互影响
  - 另一个好处是这样可以在不同的mode下执行不同的代码，避免业务代码相互影响
- 每个RLM包括若干个RLS、若干个RLT、若干个RLO

![RunLoop结构](/Users/wangsong/Library/Application Support/typora-user-images/image-20191023161055638.png)



```objective-c
// RunLoop的结构体定义
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// 唤醒runloop的端口。内核向该端口发送消息可以唤醒runloop，used for CFRunLoopWakeUp
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;					// 对应的线程
    uint32_t _winthread;
    CFMutableSetRef _commonModes;		// 集合，存储的是字符串，记录所有标记为common的modeName
    CFMutableSetRef _commonModeItems;   // 存储所有commonMode的sources、timers、observers
    CFRunLoopModeRef _currentMode;		// 当前modeName
    CFMutableSetRef _modes;				// 集合，存储的是CFRunLoopModeRef
    struct _block_item *_blocks_head;   // 链表头指针，该链表保存了所有需要被runloop执行的block。外部通过调用CFRunLoopPerformBlock函数来向链表中添加一个block节点。runloop会在CFRunLoopDoBlock时遍历该链表，逐一执行block
    struct _block_item *_blocks_tail;   // 链表尾指针，之所以有尾指针，是为了降低增加block时的时间复杂度
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};
```



唤醒runloop的方式：

- static void __CFRunLoopTimeout(void *arg) {}
  - The interval is DISPATCH_TIME_FOREVER, so this won't fire again。因为runloop的执行时长是forever，所有runloop永远不会超时，也就说函数__CFRunLoopTimeout永远不会执行到。
- CFRunLoopStop(CFRunLoopRef rl) {}
  - 调用了CFRunLoopStop代表runloop被强制终止了。即便调用了CFRunLoopWakeUp，当前的runloop也永远不会被唤醒了**。因为CFRunLoopStop函数内部调用了\_ \_CFRunLoopSetStopped函数。而```__CFRunLoopSetStopped```的实现是``` rl->_perRunData->stopped = 0x53544F50;	// 'STOP'```。加之CFRunLoopWakeUp函数中通过调用```__CFRunLoopIsIgnoringWakeUps(rl)```检查了rl->_perRunData->stopped的值是否为true，如果值为true则CFRunLoopWakeUp函数直接返回，不再执行唤醒操作。详细代码如下：
- CF_EXPORT void _CFRunLoopStopMode(CFRunLoopRef rl, CFStringRef modeName) {}
  - \_CFRunLoopStopMode函数只是通过modeName查找对应的mode，然后把mode的stopped置为true ```rlm->_stopped = true;```。不会操作runloop->perRunData->stopped。
- void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef rlt, CFStringRef modeName) {}
  - CFRunLoopAddTimer函数调用CFRunLoopWakeUp函数纯粹是为了向后兼容，如果系统版本低于CFSystemVersionLion且timer执行的rl不是当前runloop，则唤醒rl。
  - 通常情况下，在主流机型上，CFRunLoopAddTimer函数不会调用到CFRunLoopWakeUp函数，但因为timer handler发生了变化，所以需要兼容旧的实现。在旧版本系统上调用CFRunLoopWakeUp函数。
- static void __CFRunLoopSourceWakeUpLoop(const void *value, void *context) {}
  - 直接调用```CFRunLoopWakeUp((CFRunLoopRef)value);```
- void CFRunLoopTimerSetNextFireDate(CFRunLoopTimerRef rlt, CFAbsoluteTime fireDate) {}
  - 如果timer执行的rl不是当前的runloop，则调用```CFRunLoopWakeUp```手动唤醒rl

唤醒runloop的代码:

```objective-c
void CFRunLoopWakeUp(CFRunLoopRef rl) {
    CHECK_FOR_FORK();
    // This lock is crucial to ignorable wakeups, do not remove it.
    __CFRunLoopLock(rl);
    if (__CFRunLoopIsIgnoringWakeUps(rl)) {
        __CFRunLoopUnlock(rl);
        return;
    }
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
    kern_return_t ret;
    /* We unconditionally try to send the message, since we don't want
     * to lose a wakeup, but the send may fail if there is already a
     * wakeup pending, since the queue length is 1. */
    ret = __CFSendTrivialMachMessage(rl->_wakeUpPort, 0, MACH_SEND_TIMEOUT, 0);
    if (ret != MACH_MSG_SUCCESS && ret != MACH_SEND_TIMED_OUT) CRASH("*** Unable to send message to wake up port. (%d) ***", ret);
#elif DEPLOYMENT_TARGET_WINDOWS
    SetEvent(rl->_wakeUpPort);
#endif
    __CFRunLoopUnlock(rl);
}
```



# CFRunLoopMode

下面是CFRunLoopMode的结构体定义，从RLM的定义不难看出以下信息：

- 定义了一个结构体指针CFRunLoopModeRef指向__CFRunLoopMode *，从此之后通篇只是用CFRunLoopModeRef。相当于：
  - ```typedef NSString * StringRef;  StringRef name = @"VV木公子";```

- RLM结构体包含一个名字，用于标识该RLM

- RLM包含两个集合，分别存放sources0和source1。之所以用集合而非数组，原因不言而喻
- RLM包含两个数组，分别存放observer和timer

```objective-c
typedef struct __CFRunLoopMode *CFRunLoopModeRef;

struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
    CFStringRef _name;	// mode名字
    Boolean _stopped;	  // mode的状态，标识mode是否停止
    char _padding[3];
    CFMutableSetRef _sources0;	// sources0事件集合
    CFMutableSetRef _sources1;	// sources1事件集合
    CFMutableArrayRef _observers;	// 观察者数组
    CFMutableArrayRef _timers;	// 定时器数组
    CFMutableDictionaryRef _portToV1SourceMap;	 //字典  key是mach_port_t，value是CFRunLoopSourceRef
    __CFPortSet _portSet;	// 端口的集合。保存所有需要监听的port，比如_wakeUpPort，_timerPort都保存在这个数组中
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```


在Core Foundation中，针对Mode的操作，苹果只开放了以下3个API（ocoa中也有功能一样的函数，不再列出）：

```c
CF_EXPORT CFStringRef CFRunLoopCopyCurrentMode(CFRunLoopRef rl); // 返回当前运行的mode的name
CF_EXPORT CFArrayRef CFRunLoopCopyAllModes(CFRunLoopRef rl); // 返回当前RunLoop的所有mode
CF_EXPORT void CFRunLoopAddCommonMode(CFRunLoopRef rl, CFStringRef mode); // 向当前RunLoop的common modes中添加一个mode
```

我们没有办法直接创建一个CFRunLoopMode对象，但是我们可以调用CFRunLoopAddCommonMode传入一个字符串向RunLoop中添加Mode，传入的字符串即为Mode的名字，Mode对象应该是此时在RunLoop内部创建的。



# CFRunLoopSource

主要函数：

```c
CF_EXPORT Boolean CFRunLoopContainsSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFRunLoopMode mode);
CF_EXPORT void CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFRunLoopMode mode);
CF_EXPORT void CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFRunLoopMode mode);
```



如下图所示，RLS结构体定义中包括两个版本的source，分别是version0和version1。version0和version1分别对用source0和source1。

![image-20191023174620729](/Users/wangsong/Library/Application Support/typora-user-images/image-20191023174620729.png)

- ```typedef struct __CFRunLoopSource * CFRunLoopSourceRef;```之所以定义在.h中，是为了给开发者提供创建并使用source的能力。
- 一个source对应多个runloop。之所以使用CFMutableBagRef这种集合结构保存runloop而非array或set。主要原因是bag是无序的且允许重复。更多信息详见：https://developer.apple.com/documentation/corefoundation/cfbag-s1l

Source0：source0是App内部事件，由App自己管理的，像UIEvent、CFSocket都是source0。当一个source0事件准备处理时，要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。这些框架已经帮我们做好了，像网络请求的回调、滑动触摸的回调，我们不需要自己处理。 

Source0 ：自定义事件源，Source0 只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(Source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(RunLoop) 来唤醒 RunLoop，让其处理这个事件。 

```objective-c
typedef struct __CFRunLoopSource * CFRunLoopSourceRef;	// 定义在.h文件中
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;			/* immutable */
    CFMutableBagRef _runLoops;  // 允许重复的集合。一个source对应多个runloop
    union {
				CFRunLoopSourceContext version0;	/* immutable, except invalidation */ // source0的结构体
        CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */ // source1的结构体
    } _context;
};
```

source对应的runloop是一个集合，说明source可以被添加到多个runloop中。

## Source0

```objective-c
typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
    void	(*schedule)(void *info, CFRunLoopRef rl, CFStringRef mode);
    void	(*cancel)(void *info, CFRunLoopRef rl, CFStringRef mode);
    void	(*perform)(void *info);
} CFRunLoopSourceContext;	// source0的结构体
```



## Source1

```objective-c
typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
#if (TARGET_OS_MAC && !(TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)) || (TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)
    mach_port_t	(*getPort)(void *info);
    void *	(*perform)(void *msg, CFIndex size, CFAllocatorRef allocator, void *info);
#else
    void *	(*getPort)(void *info);
    void	(*perform)(void *info);
#endif
} CFRunLoopSourceContext1;	// source1的结构体
```



source相关的函数：

```objective-c
Boolean CFRunLoopContainsSource(CFRunLoopRef rl, CFRunLoopSourceRef rls, CFStringRef modeName);
void CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef rls, CFStringRef modeName);
void CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef rls, CFStringRef modeName);
```



添加Source的源码：

作用：把source添加到对应mode的source0或source1集合中。只是这里区分了下source被指定的mode是否为commonMode，如果source被指定的mode是commonMode，还需要把source添加到runloop的commonModeItems集合中。

```objective-c
void CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef rls, CFStringRef modeName) {	/* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return;
    if (!__CFIsValid(rls)) return;
    Boolean doVer0Callout = false;
    __CFRunLoopLock(rl);
    // 如果modeName是commonMode，获取runloop的commonMode副本
    if (modeName == kCFRunLoopCommonModes) {
	CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
        // runloop的commonModeItems集合为空则初始化一个集合
        if (NULL == rl->_commonModeItems) {
	    rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
	}
	// 把source添加到commonModeItems集合中
	CFSetAddValue(rl->_commonModeItems, rls);
	if (NULL != set) {
        // 创建一个长度为2的数组，分别存储runloop和runloopSource
	    CFTypeRef context[2] = {rl, rls};
	    /* add new item to all common-modes */
        // 添加新的item也就是runloopSource到所有的commonMode中
        // set是commonMode集合，CFSetApplyFunction遍历set，添加runloopSource到所有被标记为commonMode的mode->source0(source1)中
	    CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
	    CFRelease(set);
	}
    } else {
    // 根据modeName和runloop获取runloop的mode
	CFRunLoopModeRef rlm = __CFRunLoopFindMode(rl, modeName, true);
	// runloopMode的source0这个集合为空就赋值一个空的集合
	if (NULL != rlm && NULL == rlm->_sources0) {
	    rlm->_sources0 = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
	    rlm->_sources1 = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
	    rlm->_portToV1SourceMap = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, NULL);
	}
	// 如果runloopMode的sources0集合和sources1都不包含将要添加的runloopSource则把runloopSource添加到对应的集合中
	if (NULL != rlm && !CFSetContainsValue(rlm->_sources0, rls) && !CFSetContainsValue(rlm->_sources1, rls)) {
	    if (0 == rls->_context.version0.version) {
	    	// rls是source0
	        CFSetAddValue(rlm->_sources0, rls);
	    } else if (1 == rls->_context.version0.version) {
	    	// rls是source1
	        CFSetAddValue(rlm->_sources1, rls);
		__CFPort src_port = rls->_context.version1.getPort(rls->_context.version1.info);
		if (CFPORT_NULL != src_port) {
			// key是src_port，value是rls，存储到runloopMode的_portToV1SourceMap字典中
		    CFDictionarySetValue(rlm->_portToV1SourceMap, (const void *)(uintptr_t)src_port, rls);
		    __CFPortSetInsert(src_port, rlm->_portSet);
	        }
	    }
	    __CFRunLoopSourceLock(rls);
	    if (NULL == rls->_runLoops) {
            // source有一个集合成员变量runLoops。source每被添加进一个runloop，都会把runloop添加到他的这个集合中
            // 如官方注释所言：sources retain run loops!（source会持有runloop！）
	        rls->_runLoops = CFBagCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeBagCallBacks); // sources retain run loops!
	    }
	    // 更新runloopSource的runLoops集合，将rl添加到rls->_runloops中
	    CFBagAddValue(rls->_runLoops, rl);
	    __CFRunLoopSourceUnlock(rls);
        // 如果rls是source0则doVer0Callout标记置为true，即需要向外调用回调
	    if (0 == rls->_context.version0.version) {
	        if (NULL != rls->_context.version0.schedule) {
	            doVer0Callout = true;
	        }
	    }
	}
        if (NULL != rlm) {
	    __CFRunLoopModeUnlock(rlm);
	}
    }
    __CFRunLoopUnlock(rl);
    // 如果是source0，则向外层（上层）调用source0的schedule回调函数
    if (doVer0Callout) {
        // although it looses some protection for the source, we have no choice but
        // to do this after unlocking the run loop and mode locks, to avoid deadlocks
        // where the source wants to take a lock which is already held in another
        // thread which is itself waiting for a run loop/mode lock
	rls->_context.version0.schedule(rls->_context.version0.info, rl, modeName);	/* CALLOUT */
    }
}
```





# runLoop timer 

CFRunLoopTimer结构体实现：

```c
// CFRunLoopTimerRef 是基于时间的触发器，它和 NSTimer 是toll-free bridged 的，可以混用。
// 其包含一个时间长度和一个函数回调。当其加入到 RunLoop 时，RunLoop会注册对应的时间点，当时间点到时，RunLoop会被唤醒以执行那个回调
struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;					// 标记fire状态
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;			// timer所处的runloop
    CFMutableSetRef _rlModes;		// 存放所有 包含该timer的 mode的 modeName，意味着一个timer可能会在多个mode中存在
    CFAbsoluteTime _nextFireDate;	// 下次触发时间
    CFTimeInterval _interval;		// 理想时间间隔 /* immutable */
    CFTimeInterval _tolerance;      // 允许的误差    /* mutable */
    uint64_t _fireTSR;			/* TSR units */
    CFIndex _order;			/* immutable */
    CFRunLoopTimerCallBack _callout;// timer回调	/* immutable */
    CFRunLoopTimerContext _context;	/* immutable, except invalidation */
};
```

和source不同，timer对应的runloop是一个runloop指针，而非数组，所以此处说明一个timer只能添加到一个runloop。



主要函数：

```c
CF_EXPORT Boolean CFRunLoopContainsTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFRunLoopMode mode);
CF_EXPORT void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFRunLoopMode mode);
CF_EXPORT void CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFRunLoopMode mode);
```



添加timer源码：

作用：添加timer到rl->commonModeItems中，添加timer到runloopMode中，根据触发时间调整timer在runloopMode->timers数组中的位置。

```c
// 添加timer到runloopMode中，添加timer到rl->commonModeItems中
void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef rlt, CFStringRef modeName) {    
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return;
    if (!__CFIsValid(rlt) || (NULL != rlt->_runLoop && rlt->_runLoop != rl)) return;
    __CFRunLoopLock(rl);
    // 如果timer要添加的mode是commonMode
    if (modeName == kCFRunLoopCommonModes) {
        // 获取rl->commonModes集合
	CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
        // 如果rl->_commonModeItems为空就初始化rl->commonModeItems
	if (NULL == rl->_commonModeItems) {
	    rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
	}
	CFSetAddValue(rl->_commonModeItems, rlt);
	if (NULL != set) {
        // 长度为2的数组，分别存放rl和rlt
	    CFTypeRef context[2] = {rl, rlt};
	    /* add new item to all common-modes */
        // 添加新的item也就是timer到所有的commonMode中
        // set是commonMode集合，CFSetApplyFunction遍历set，添加context[1]存放的rlt添加到所有被标记为commonMode的mode中
	    CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
	    CFRelease(set);
	}
    } else {
	CFRunLoopModeRef rlm = __CFRunLoopFindMode(rl, modeName, true);
	if (NULL != rlm) {
            if (NULL == rlm->_timers) {
                CFArrayCallBacks cb = kCFTypeArrayCallBacks;
                cb.equal = NULL;
                rlm->_timers = CFArrayCreateMutable(kCFAllocatorSystemDefault, 0, &cb);
            }
	}
	if (NULL != rlm && !CFSetContainsValue(rlt->_rlModes, rlm->_name)) {
            __CFRunLoopTimerLock(rlt);
            if (NULL == rlt->_runLoop) {
		rlt->_runLoop = rl;
  	    } else if (rl != rlt->_runLoop) {
                __CFRunLoopTimerUnlock(rlt);
	        __CFRunLoopModeUnlock(rlm);
                __CFRunLoopUnlock(rl);
		return;
	    }
        // 更新rlt的rlModes集合。将rlm->name添加到name中
  	    CFSetAddValue(rlt->_rlModes, rlm->_name);
            __CFRunLoopTimerUnlock(rlt);
            __CFRunLoopTimerFireTSRLock();
            // Reposition释义复位。所以顾名思义该函数用于复位timer
            // 此处调用该函数本质上是按照timer下次触发时间长短，计算timer需要插入到runloopMode->timers数组中的位置，然后把timer插入到runloopMode->timers数组中
            __CFRepositionTimerInMode(rlm, rlt, false);
            __CFRunLoopTimerFireTSRUnlock();
            // 为了向后兼容，如果系统版本低于CFSystemVersionLion且timer执行的rl不是当前runloop，则唤醒rl
            if (!_CFExecutableLinkedOnOrAfter(CFSystemVersionLion)) {
                // Normally we don't do this on behalf of clients, but for
                // backwards compatibility due to the change in timer handling...
                if (rl != CFRunLoopGetCurrent()) CFRunLoopWakeUp(rl);
            }
	}
        if (NULL != rlm) {
	    __CFRunLoopModeUnlock(rlm);
	}
    }
    __CFRunLoopUnlock(rl);
}
```



设置timer下次触发时间的源码：

CFRunLoopTimerSetNextFireDate > CFRunLoopWakeUp

```c
void CFRunLoopTimerSetNextFireDate(CFRunLoopTimerRef rlt, CFAbsoluteTime fireDate) {
    CHECK_FOR_FORK();
    if (!__CFIsValid(rlt)) return;
    // 触发日期大于最大限制时间，则把触发日期调整为最大触发时间
    if (TIMER_DATE_LIMIT < fireDate) fireDate = TIMER_DATE_LIMIT;
    uint64_t nextFireTSR = 0ULL;
    uint64_t now2 = mach_absolute_time();
    CFAbsoluteTime now1 = CFAbsoluteTimeGetCurrent();
    // 下次触发时间小于现在则立即触发
    if (fireDate < now1) {
	nextFireTSR = now2;
    // 下次触发时间间隔大于允许的最大间隔TIMER_INTERVAL_LIMIT，则将下次触发时间调整为now + TIMER_INTERVAL_LIMIT
    } else if (TIMER_INTERVAL_LIMIT < fireDate - now1) {
	nextFireTSR = now2 + __CFTimeIntervalToTSR(TIMER_INTERVAL_LIMIT);
    } else {
	nextFireTSR = now2 + __CFTimeIntervalToTSR(fireDate - now1);
    }
    __CFRunLoopTimerLock(rlt);
    if (NULL != rlt->_runLoop) {
        // 获取runloopMode个数
        CFIndex cnt = CFSetGetCount(rlt->_rlModes);
        // 声明名为modes的栈结构
        STACK_BUFFER_DECL(CFTypeRef, modes, cnt);
        // rlt->rlModes赋值给modes栈结构
        CFSetGetValues(rlt->_rlModes, (const void **)modes);
        // To avoid A->B, B->A lock ordering issues when coming up
        // towards the run loop from a source, the timer has to be
        // unlocked, which means we have to protect from object
        // invalidation, although that's somewhat expensive.
        for (CFIndex idx = 0; idx < cnt; idx++) {
            // 先retain
            CFRetain(modes[idx]);
        }
        CFRunLoopRef rl = (CFRunLoopRef)CFRetain(rlt->_runLoop);
        __CFRunLoopTimerUnlock(rlt);
        __CFRunLoopLock(rl);
        // 把modes集合中存储的modeName转换为mode结构体实例，然后再存入modes集合
        for (CFIndex idx = 0; idx < cnt; idx++) {
	    CFStringRef name = (CFStringRef)modes[idx];
            modes[idx] = __CFRunLoopFindMode(rl, name, false);
            // 后release
	    CFRelease(name);
        }
        __CFRunLoopTimerFireTSRLock();
        // 把上面计算好的下次触发时间设置给rlt
	rlt->_fireTSR = nextFireTSR;
        rlt->_nextFireDate = fireDate;
        for (CFIndex idx = 0; idx < cnt; idx++) {
	    CFRunLoopModeRef rlm = (CFRunLoopModeRef)modes[idx];
            if (rlm) {
                // Reposition释义复位。所以顾名思义该函数用于复位timer，所谓复位，就是调整timer在runloopMode->timers数组中的位置
                // 此处调用该函数本质上是先移除timer，然后按照timer下次触发时间长短计算timer需要插入到runloopMode->timers数组中的位置，最后把timer插入到runloopMode->timers数组中
                __CFRepositionTimerInMode(rlm, rlt, true);
            }
        }
        __CFRunLoopTimerFireTSRUnlock();
        for (CFIndex idx = 0; idx < cnt; idx++) {
            __CFRunLoopModeUnlock((CFRunLoopModeRef)modes[idx]);
        }
        __CFRunLoopUnlock(rl);
        // This is setting the date of a timer, not a direct
        // interaction with a run loop, so we'll do a wakeup
        // (which may be costly) for the caller, just in case.
        // (And useful for binary compatibility with older
        // code used to the older timer implementation.)
        // 以上注释的意思是：这行代码的是为了给timer设置date，但不直接作用于runloop
        // 以防万一，我们手动唤醒runloop，尽管有可能这个代价是高昂的
        // 另一方面，这么做的目的也是为了兼容timer的之前的实现方式
        // 如果timer执行的rl不是当前的runloop，则手动唤醒
        if (rl != CFRunLoopGetCurrent()) CFRunLoopWakeUp(rl);
        CFRelease(rl);
     } else {
        __CFRunLoopTimerFireTSRLock();
         // 走到这里说明timer的rl还是空，所以只是简单的设置timer的下次触发时间
	rlt->_fireTSR = nextFireTSR;
        rlt->_nextFireDate = fireDate;
        __CFRunLoopTimerFireTSRUnlock();
         __CFRunLoopTimerUnlock(rlt);
     }
}
```



# runLoop observer

Observer顾名思义，观察者，和我们设计模式中的观察者模式如出一辙。observer主要观察runloop的状态变化，然后执行一些回调函数。runloop主要有6种状态，如下：

```c
// runloop的6种状态，用于通知observer runloop的状态变化
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),					// 即将进入Loop
    kCFRunLoopBeforeTimers = (1UL << 1),	// 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2),	// 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5),	// 即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),	// 刚从休眠中唤醒 但是还没开始处理事件
    kCFRunLoopExit = (1UL << 7),					// 即将退出Loop
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```



CFRunLoopObserver结构体：

```c
// CFRunLoopObserver是观察者，可以观察RunLoop的各种状态，并执行回调
struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;  // observer所观察的runloop
    CFIndex _rlCount;
    CFOptionFlags _activities;		/* immutable */
    CFIndex _order;			/* immutable */
    CFRunLoopObserverCallBack _callout;	/* immutable */	// 观察到runloop状态变化后的回调
    CFRunLoopObserverContext _context;	/* immutable, except invalidation */
};
```

和source不同，observer对应的runloop是一个runloop指针，而非数组，此处说明一个observer只能观察一个runloop，所以observer只能添加到一个runloop的一个或者多个mode中。

主要函数：

```c
CF_EXPORT Boolean CFRunLoopContainsObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFRunLoopMode mode);
CF_EXPORT void CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFRunLoopMode mode);
CF_EXPORT void CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFRunLoopMode mode);
```



添加Observer源码：

```objective-c
void CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef rlo, CFStringRef modeName) {
    CHECK_FOR_FORK();
    CFRunLoopModeRef rlm;
    if (__CFRunLoopIsDeallocating(rl)) return;
    if (!__CFIsValid(rlo) || (NULL != rlo->_runLoop && rlo->_runLoop != rl)) return;
    __CFRunLoopLock(rl);
    if (modeName == kCFRunLoopCommonModes) {
	CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
        // 创建commonModeItems
	if (NULL == rl->_commonModeItems) {
	    rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
	}
        // 添加observer到commonModeItems
	CFSetAddValue(rl->_commonModeItems, rlo);
	if (NULL != set) {
	    CFTypeRef context[2] = {rl, rlo};
	    /* add new item to all common-modes */
        // 添加item到所有被标记为commonMode的mode中
	    CFSetApplyFunction(set, (__CFRunLoopAddItemToCommonModes), (void *)context);
	    CFRelease(set);
	}
    } else {
	rlm = __CFRunLoopFindMode(rl, modeName, true);
	if (NULL != rlm && NULL == rlm->_observers) {
	    rlm->_observers = CFArrayCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeArrayCallBacks);
	}
	if (NULL != rlm && !CFArrayContainsValue(rlm->_observers, CFRangeMake(0, CFArrayGetCount(rlm->_observers)), rlo)) {
            Boolean inserted = false;
            for (CFIndex idx = CFArrayGetCount(rlm->_observers); idx--; ) {
                CFRunLoopObserverRef obs = (CFRunLoopObserverRef)CFArrayGetValueAtIndex(rlm->_observers, idx);
                if (obs->_order <= rlo->_order) {
                    CFArrayInsertValueAtIndex(rlm->_observers, idx + 1, rlo);
                    inserted = true;
                    break;
                }
            }
            if (!inserted) {
	        CFArrayInsertValueAtIndex(rlm->_observers, 0, rlo);
            }
	    rlm->_observerMask |= rlo->_activities;
	    __CFRunLoopObserverSchedule(rlo, rl, rlm);
	}
        if (NULL != rlm) {
	    __CFRunLoopModeUnlock(rlm);
	}
    }
    __CFRunLoopUnlock(rl);
}
```



# RunLoop运行相关源码

CFRunLoopRun > CFRunLoopRunSpecific > __CFRunLoopRun > 

```objective-c
// 一个do...while循环 如果不是stop或finish就不断的循环 还可以重新启动runloop
void CFRunLoopRun(void) {	/* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}
```



```objective-c
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    // 根据modeName获取mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    // 如果currentMode是空则退出
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
	Boolean did = false;
	if (currentMode) __CFRunLoopModeUnlock(currentMode);
	__CFRunLoopUnlock(rl);
	return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;
	// 通知observer即将进入runloop
	if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
	// 通知observer即将退出runloop
	if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

        __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopPopPerRunData(rl, previousPerRun);
	rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}
```



参考文章

[CFBag](https://developer.apple.com/documentation/corefoundation/cfbag-s1l)

[RunLoop](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)





