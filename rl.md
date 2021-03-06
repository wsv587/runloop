[toc]



# 前言

对iOS开发者而言，runloop是一个老生常谈的话题，但凡是iOS开发者，在工作中必然直接或间接的接触过runloop。而对于面试者而言，runloop又几乎是必考点。在4年前，笔者写过一篇文章[NSRunLoop](https://www.cnblogs.com/wsnb/p/4753685.html)，对runloop原理以及应用场景做了基本介绍。但是对于不了解源码的同学，读起来也只能理解皮毛，不看源码的话，读了n多篇也是似懂非懂，对于原理也只能是死记硬背。所以，本文将从源码的角度剖析runloop的组成，强化自己对runloop的认识，验证我们脑海中一直以来似懂非懂的原理，真心希望这篇文章能够帮助到大家。
# 为什么是runLoop
runloop顾名思义就是”跑圈“，所谓跑圈就给人一种循环的感觉。runloop运行的核心代码就是一个有状态的do...while循环。每循环一次就相当于跑了一圈，线程就会对当前这一圈里面产生的事件进行处理。那么为什么线程要有runloop呢？其实我们的APP可以理解为是靠event驱动的（包括iOS和Android应用）。我们触摸屏幕、网络回调等都是一个个的event，也就是事件。这些事件产生之后会分发给我们的APP，APP接收到事件之后分发给对应的线程。通常情况下，如果线程没有runloop，那么一个线程一次只能执行一个任务，执行完成后线程就会退出。要想APP的线程一直能够处理事件或者等待事件（比如异步事件），就要保活线程，也就是不能让线程早早的退出，此时runloop就派上用场了。我们已经说了，runloop本质上就是一个有状态的do...while循环，所以只要不是超时或者故意退出状态，那么runLoop就会一直执行do...while，所以可以保证线程不退出。其实也不是必须要给线程指定一个runloop，如果需要我们线程能够持续的处理事件，那么就需要给线程绑定一个runloop。也就是说，runloop能够保证线程一直可以一直处理事件。所以runloop的作用可以理解为：

- 使程序一直运行并处理各种事件。这些事件包括但不限于用户操作、定时器任务、内核消息
- 有顺序的处理各种Event。因为runLoop有状态，可以决定线程在什么时候处理什么事件
- 节省CPU资源。通常情况下，事件并不是永无休止的产生，所以也就没必要让线程永无休止的运行。runloop可以在无事件处理时进入休眠状态，避免无休止的do...while跑空圈。

# 无处不在的runLoop

<img src="/Users/wangsong/Library/Application Support/typora-user-images/image-20191026150209717.png" alt="image-20191026150209717" style="zoom:50%;" />

所有线程的几乎所有的函数都是从以下6个函数调起的:

```c
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__	// 回调observer的callback指针，通知runloop的activity状态变化
__CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__												// 处理添加到runloop的block
__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__						// 处理分发给主线程的事件
__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__			// 处理timer回调
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__		// 处理source0回调
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__		// 处理source1回调
```

以上这6个函数名字之所以如此之长，也只是为了实现自解释。通过查看名字我们可以看出这几个函数都是calling_out的，也就是都是**向外**回调的函数。所谓的**向外**是相对于runloop的，其实就是runLoop向上层回调，通过回调函数runloop可以通知上层runloop当前处于什么状态或正在处理什么事件。具体的每个函数的作用会在下文详细解释，不必在此纠结。

# runLoop 结构
runLoop的结构如下图所示：

为了便于说明，下文将对线程（thread）简称T，RunLoop简称RL，CFRunLoopMode简称RLM，CFRunLoopSource简称RLS，CFRunLoopObserver简称RLO。下图说明了以下信息：

- 一个thread对应一个runloop
- Cocoa层的NSRunLoop是对CF层的CFRunLoop的封装
- 一个runloop对应多个runLoopMode
- 一个runloop一次只能执行一个runLoopMode，runloop在同一个时间只能且必须在一种特定mode下run。要想切换RLM需要停止并退出当前RLM重新进入新的runLoopMode。
  - 一次执行一个mode的好处在于，底层设计相对简单，避免不同的mode耦合在一起，代码相互影响
  - 另一个好处是这样可以在不同的mode下执行不同的代码，避免上层业务代码相互影响。
  - 多个mode以及mode的切换是iOS app滑动顺畅的关键。
  - 主线程中不同的代码指定在不同的mode下运行可以提高app的流畅度。
- 每个runLoopMode包括若干个runLoopSource、若干个runLoopTimer、若干个runLoopObserver。统称为item。



![RunLoop结构](/Users/wangsong/Library/Application Support/typora-user-images/image-20191023161055638.png)



## RunLoop结构体定义

```c
// RunLoop的结构体定义
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// 手动唤醒runloop的端口。初始化runloop时设置，仅用于CFRunLoopWakeUp，CFRunLoopWakeUp函数会向_wakeUpPort发送一条消息
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



## RunLoop操作item

```c
// source相关操作
CF_EXPORT Boolean CFRunLoopContainsSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef mode);
CF_EXPORT void CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef mode);
CF_EXPORT void CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef mode);

// observer相关操作
CF_EXPORT Boolean CFRunLoopContainsObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef mode);
CF_EXPORT void CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef mode);
CF_EXPORT void CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef mode);

// timer相关操作
CF_EXPORT Boolean CFRunLoopContainsTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
CF_EXPORT void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
CF_EXPORT void CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
```



## 手动唤醒runloop的方式

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

**除手动滑动runloop外，内核通过向port发送消息也可以自动唤醒runloop。**

### 手动唤醒runloop的代码

```c
void CFRunLoopWakeUp(CFRunLoopRef rl) {
    // ...
    // __CFSendTrivialMachMessage内部调用mach_msg函数向runloop的wakeUpPort发送消息以唤醒runloop
    kern_return_t ret = __CFSendTrivialMachMessage(rl->_wakeUpPort, 0, MACH_SEND_TIMEOUT, 0);
  	// ..
}

// 手动调用 mach_msg 向 rl->_wakeUpPort sendMsg 以唤醒runloop
static uint32_t __CFSendTrivialMachMessage(mach_port_t port, uint32_t msg_id, CFOptionFlags options, uint32_t timeout) {
    kern_return_t result;
   // 配置header...
    mach_msg_header_t header;
    header.msgh_remote_port = port;
    header.msgh_id = msg_id; 
    // 向内核发送消息唤醒runloop
    result = mach_msg(&header, MACH_SEND_MSG|options, header.msgh_size, 0, MACH_PORT_NULL, timeout, MACH_PORT_NULL);
		// ... 
    return result;
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
    CFMutableDictionaryRef _portToV1SourceMap;	 //字典。key是mach_port_t，value是CFRunLoopSourceRef
    __CFPortSet _portSet;	// 端口的集合。保存所有需要监听的port，比如_wakeUpPort，_timerPort都保存在这个数组中
    CFIndex _observerMask; // 添加obsever时设置_observerMask为observer的_activities（CFRunLoopActivity状态）
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
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```


在Core Foundation中，针对Mode的操作，苹果只开放了以下3个API（cocoa中也有功能一样的函数，不再列出）：

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

```c
typedef struct __CFRunLoopSource * CFRunLoopSourceRef;	// 定义在.h文件中
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;	// 位域，标记source状态
    pthread_mutex_t _lock;
    CFIndex _order;			// souce的顺序（不可变）
    CFMutableBagRef _runLoops;  // 集合（允许元素重复）说明一个source可以对应多个runloop
    union {
				CFRunLoopSourceContext version0; // source0的结构体（不可变）
        CFRunLoopSourceContext1 version1; // source1的结构体（不可变）
    } _context;
};
```

source对应的runloop是一个集合，说明source可以被添加到多个runloop中。

## Source0和Source1区别

Source0：source0是App内部事件，由App自己管理的，像UIEvent、CFSocket都是source0。source0并不能主动触发事件，当一个source0事件准备处理时，要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理。然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。这些框架已经帮我们做好了，像网络请求的回调、滑动触摸的回调，我们不需要自己处理。 

Source1：由RunLoop和内核管理，Mach port驱动，如CFMachPort、CFMessagePort。source1包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 的线程。

source相关的函数

```c
Boolean CFRunLoopContainsSource(CFRunLoopRef rl, CFRunLoopSourceRef rls, CFStringRef modeName);
void CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef rls, CFStringRef modeName);
void CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef rls, CFStringRef modeName);
```



## 添加Source的源码

作用：把source添加到对应mode的source0或source1集合中。只是这里区分了下source被指定的mode是否为commonMode，如果source被指定的mode是commonMode，还需要把source添加到runloop的commonModeItems集合中。

```c
void CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef rls, CFStringRef modeName) {
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
    }
    // 如果是source0，则向外层（上层）调用source0的schedule回调函数
    if (doVer0Callout) {
	rls->_context.version0.schedule(rls->_context.version0.info, rl, modeName);	/* CALLOUT */
    }
}
```



##  CFRunLoopSourceSignal源码

上面已经说过，source0并不能主动触发事件，当一个source0事件准备处理时，要先手动调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理。然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。在runloop被唤醒后，如果source0被标记为待处理状态，那么runloop在执行source0（doSource0）时就会处理source0事件。接下来我们看下CFRunLoopSourceSignal的源码：

```c
// 把source0标记为待处理状态
void CFRunLoopSourceSignal(CFRunLoopSourceRef rls) {
    if (__CFIsValid(rls)) {
	__CFRunLoopSourceSetSignaled(rls);
    }
}
// rls->_bits位域标记为待处理状态
CF_INLINE void __CFRunLoopSourceSetSignaled(CFRunLoopSourceRef rls) {
    __CFBitfieldSetValue(rls->_bits, 1, 1, 1);
}
```

获取source0状态的代码此处省略。

上面CFRunLoopSourceSignal调用了\__CFRunLoopSourceSetSignaled函数，\_\_CFRunLoopSourceSetSignaled函数内部设置了rls->bits的bitfiled（位域）。对于位域不了解的同学可以自行百度，[这里](https://baike.baidu.com/item/bitfield/2007712?fr=aladdin)是百度百科的解释。

同理timer也有一个同样的位域标记位来标记timer是否需要被处理。如果timer被标记为待触发状态，那么runloop在执行timer（doTimer）是会触发timer事件。



# runLoop timer 

**CFRunLoopTimerRef** 是基于时间的触发器，它和 NSTimer 是toll-free bridged 的，可以混用。其包含一个时间长度和一个回调（函数指针）。当其加入到 RunLoop 时，RunLoop会注册对应的时间点，当达到对应的时间点时，RunLoop会被唤醒以执行那个回调。

CFRunLoopTimer结构体实现：

```c
struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;					// 位域，标记timer是否为待处理状态
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;			// timer所处的runloop
    CFMutableSetRef _rlModes;		// 集合。存放所有包含该timer的mode的modeName，意味着一个timer可能会在多个mode中存在
    CFAbsoluteTime _nextFireDate;	// 下次触发时间
    CFTimeInterval _interval;		// 理想时间间隔(不可变)
    CFTimeInterval _tolerance;      // 允许的误差(可变)
    uint64_t _fireTSR;			/* TSR units */
    CFIndex _order;			/* immutable */
    CFRunLoopTimerCallBack _callout;// timer回调	/* immutable */
    CFRunLoopTimerContext _context;	/* immutable, except invalidation */
};
```

和source不同，timer对应的runloop是一个runloop指针，而非数组，所以此处说明一个timer只能添加到一个runloop。

## 添加timer源码

作用：添加timer到rl->commonModeItems中，添加timer到runloopMode中，根据触发时间调整timer在runloopMode->timers数组中的位置。

```c
// 添加timer到runloopMode中，添加timer到rl->commonModeItems中
void CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef rlt, CFStringRef modeName) {    

    if (!__CFIsValid(rlt) || (NULL != rlt->_runLoop && rlt->_runLoop != rl)) return;
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
		return;
	    }
        // 更新rlt的rlModes集合。将rlm->name添加到name中
  	    CFSetAddValue(rlt->_rlModes, rlm->_name);
            // Reposition释义复位。所以顾名思义该函数用于复位timer
            // 此处调用该函数本质上是按照timer下次触发时间长短，计算timer需要插入到runloopMode->timers数组中的位置，然后把timer插入到runloopMode->timers数组中
            __CFRepositionTimerInMode(rlm, rlt, false);
            __CFRunLoopTimerFireTSRUnlock();
            // 为了向后兼容，如果系统版本低于CFSystemVersionLion且timer执行的rl不是当前runloop，则唤醒rl
            if (!_CFExecutableLinkedOnOrAfter(CFSystemVersionLion)) {
                if (rl != CFRunLoopGetCurrent()) CFRunLoopWakeUp(rl);
            }
	}
    }
}
```



## 执行timer的源码

```c
static Boolean __CFRunLoopDoTimers(CFRunLoopRef rl, CFRunLoopModeRef rlm, uint64_t limitTSR) {	/* DOES CALLOUT */
    Boolean timerHandled = false;
    CFMutableArrayRef timers = NULL;
    // 获取timer数组
    for (CFIndex idx = 0, cnt = rlm->_timers ? CFArrayGetCount(rlm->_timers) : 0; idx < cnt; idx++) {
        CFRunLoopTimerRef rlt = (CFRunLoopTimerRef)CFArrayGetValueAtIndex(rlm->_timers, idx);
        
        if (__CFIsValid(rlt) && !__CFRunLoopTimerIsFiring(rlt)) {
            if (rlt->_fireTSR <= limitTSR) {
                if (!timers) timers = CFArrayCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeArrayCallBacks);
                CFArrayAppendValue(timers, rlt);
            }
        }
    }
    // 执行timer
    for (CFIndex idx = 0, cnt = timers ? CFArrayGetCount(timers) : 0; idx < cnt; idx++) {
        CFRunLoopTimerRef rlt = (CFRunLoopTimerRef)CFArrayGetValueAtIndex(timers, idx);
        Boolean did = __CFRunLoopDoTimer(rl, rlm, rlt);
        timerHandled = timerHandled || did;
    }
    if (timers) CFRelease(timers);
    return timerHandled;
}
```





##  设置timer下次触发时间的源码

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

Observer顾名思义，观察者，和我们设计模式中的观察者模式如出一辙。每个 Observer 都包含了一个回调（函数指针），observer主要观察runloop的状态变化，然后执行回调函数。runloop可观察的状态主要有6种状态，如下：

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
    CFOptionFlags _activities; // CFOptionFlags是UInt类型的别名，_activities用来说明要观察runloop的哪些状态。一旦指定了就不可变。
    CFIndex _order;
    CFRunLoopObserverCallBack _callout;	// 观察到runloop状态变化后的回调(不可变)
    CFRunLoopObserverContext _context;
};
```

和source不同，observer对应的runloop是一个runloop指针，而非数组，此处说明一个observer只能观察一个runloop，所以observer只能添加到一个runloop的一个或者多个mode中。

## 添加Observer源码

```objective-c
void CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef rlo, CFStringRef modeName) {
    CHECK_FOR_FORK();
    CFRunLoopModeRef rlm;
    if (__CFRunLoopIsDeallocating(rl)) return;
    if (!__CFIsValid(rlo) || (NULL != rlo->_runLoop && rlo->_runLoop != rl)) return;
    __CFRunLoopLock(rl);
    if (modeName == kCFRunLoopCommonModes) {
            // 导出runloop的commonModes
	CFSetRef set = rl->_commonModes ? CFSetCreateCopy(kCFAllocatorSystemDefault, rl->_commonModes) : NULL;
        // 创建commonModeItems
	if (NULL == rl->_commonModeItems) {
	    rl->_commonModeItems = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
	}
        // 添加observer到commonModeItems
	CFSetAddValue(rl->_commonModeItems, rlo);
	if (NULL != set) {
	    CFTypeRef context[2] = {rl, rlo};
        // 添加observer到所有被标记为commonMode的mode中
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
    	// 设置runloopMode的_observerMask为观察者的_activities（CFRunLoopActivity状态）
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

CFRunLoopRunSpecific主要由CFRunLoopRun和CFRunLoopRunInMode调用。

<img src="/Users/wangsong/Desktop/study/runloop/img/CFRunLoopRun调用链.png" alt="CFRunLoopRun调用链" style="zoom:50%;" />

CFRunLoopRun源码：

```c
// 一个do...while循环 如果不是stop或finish就不断的循环 还可以重新启动runloop
void CFRunLoopRun(void) {	/* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}
```

CFRunLoopRunInMode源码：

```c
SInt32 CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
```

CFRunLoopRunSpecific源码：

```c
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
		// 如果runloop正在销毁则直接返回finish
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    // 根据指定的modeName获取指定的mode，也就是将要运行的mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    // 出现以下情况就不会return finish：
    // 1>.将要运行的mode不为空
    // 以下这几条是在__CFRunLoopModeIsEmpty函数中判断的:
    // 2>.将要运行的currentMode是source0、source1、timer任一个不为空
    // 3>.待执行的block的mode和将要运行的mode相同
    // 4>.待执行的block的mode是commonMode且待运行的mode包含在commonMode中
    // 5>.待执行的block的mode包含待运行的mode
    // 6>.待执行的block的mode包含commonMode且待运行的mode包含在commonMode中
    // 所谓待执行的block是外部(开发者)通过调用CFRunLoopPerformBlock函数添加到runloop中的
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
	return kCFRunLoopRunFinished;
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;
	// 1.通知observer即将进入runloop
  // 这里使用currentMode->_observerMask 和 kCFRunLoopEntry按位与操作
  // 如果按位与的结果不是0则说明即将进入runloop
  // 而currentMode->_observerMask是个什么东西呢？
  // currentMode->_observerMask本质上是Int类型的变量，标识当前mode的CFRunLoopActivity状态
  // 那么currentMode->_observerMask是在哪里赋值的呢？
  // 调用CFRunLoopAddObserver函数向runloop添加observer的时候会把observer的activities按位或赋值给mode->_observerMask
	if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
  // RunLoop的运行的最核心函数
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
	// 10.通知observer即将退出runloop
	if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

        __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopPopPerRunData(rl, previousPerRun);
	rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}
```



```c
/**RunLoop的运行的最核心函数（进入和退出时runloop和runloopMode都会被加锁）
 * rl: 运行的runloop
 * rlm: runloop Mode
 * seconds: runloop超时时间
 * stopAfterHandle: 处理完时间后runloop是否stop，默认为false
 * previousMode: runloop上次运行的mode
 */
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    // 获取基于系统启动后的时钟"嘀嗒"数，其单位是纳秒
    uint64_t startTSR = mach_absolute_time();
    
    if (__CFRunLoopIsStopped(rl)) {
        __CFRunLoopUnsetStopped(rl);
        return kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
        rlm->_stopped = false;
        return kCFRunLoopRunStopped;
    }
    // 获取主线程接收消息的port备用。如果runLoop是mainRunLoop且后续内核唤醒的port等于主线程接收消息的port，主线程就处理这个消息
    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF();
    
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    // 初始化获取timer的port(source1)
    // 如果这个port和mach_msg发消息的livePort相等则说明timer时间到了，处理timer
    mach_port_name_t modeQueuePort = MACH_PORT_NULL;
    if (rlm->_queue) {
        modeQueuePort = _dispatch_runloop_root_queue_get_port_4CF(rlm->_queue);
    }
#endif
    // 使用GCD实现runloop超时功能
    dispatch_source_t timeout_timer = NULL;
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    // seconds是设置的runloop超时时间，一般为1.0e10，11.574万年，所以不会超时
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        timeout_context->termTSR = 0ULL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) {
        dispatch_queue_t queue = pthread_main_np() ? __CFDispatchQueueGetGenericMatchingMain() : __CFDispatchQueueGetGenericBackground();
        timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_retain(timeout_timer);
        timeout_context->ds = timeout_timer;
        timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
        // 设置超时的时间点（从现在开始 + 允许运行的时长）
        timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
        dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
        dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        dispatch_resume(timeout_timer);
    } else { // infinite timeout
        seconds = 9999999999.0;
        timeout_context->termTSR = UINT64_MAX;
    }
    
    Boolean didDispatchPortLastTime = true;
    // returnValue 标识runloop状态，如果returnValue不为0就不退出。
    // returnValue可能的值：
    // enum {
    //     kCFRunLoopRunFinished = 1,
    //     kCFRunLoopRunStopped = 2,
    //     kCFRunLoopRunTimedOut = 3,
    //     kCFRunLoopRunHandledSource = 4
    // };
    int32_t retVal = 0;
    do {
        voucher_mach_msg_state_t voucherState = VOUCHER_MACH_MSG_STATE_UNCHANGED;
        voucher_t voucherCopy = NULL;
        // 消息缓冲区，用户缓存内核发的消息
        uint8_t msg_buffer[3 * 1024];
        // 消息缓冲区指针，用于指向msg_buffer
        mach_msg_header_t *msg = NULL;
        // 用于保存被内核唤醒的端口（调用mach_msg函数时会把livePort地址传进去供内核写数据）
        mach_port_t livePort = MACH_PORT_NULL;
        __CFPortSet waitSet = rlm->_portSet;
        
        __CFRunLoopUnsetIgnoreWakeUps(rl);
        // 2. 通知 Observers: RunLoop 即将触发 Timer 回调。
        // __CFRunLoopDoObservers内部会调用__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__这个函数，这个函数的参数包括observer的回调函数、observer、runloop状态
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        // 3. 通知 Observers: RunLoop 即将触发 Source0 (非port) 回调。
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        //  执行被加入的block
        // 外部通过调用CFRunLoopPerformBlock函数向当前runloop增加block。新增加的block保存咋runloop.blocks_head链表里。
        // __CFRunLoopDoBlocks会遍历链表取出每一个block，如果block被指定执行的mode和当前的mode一致，则调用__CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__执行之
        __CFRunLoopDoBlocks(rl, rlm);
        // 4. RunLoop 触发 Source0 (非port) 回调
        // __CFRunLoopDoSources0函数内部会调用__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__函数
        // __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__函数会调用source0的perform回调函数，即rls->context.version0.perform
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        // 如果rl处理了source0事件，那再处理source0之后的block
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
        }
        // 标记是否需要轮询，如果处理了source0则轮询，否则休眠
        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
        
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
            msg = (mach_msg_header_t *)msg_buffer;
            // 5. 如果有 Source1 (基于port的source) 处于 ready 状态，直接处理这个 Source1 然后跳转到第9步去处理消息。
            // __CFRunLoopServiceMachPort函数内部调用了mach_msg，mach_msg函数会监听内核给端口发送的消息
            // 如果mach_msg监听到消息就会执行goto跳转去处理这个消息
            // 第五个参数为0代表不休眠立即返回
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;
            }
        }
        
        didDispatchPortLastTime = false;
        // 6. 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
        // 根据上面第4步是否处理过source0，来判断如果也没有source1消息的时候是否让线程进入睡眠
        if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        // runloop置为休眠状态
        __CFRunLoopSetSleeping(rl);
        // 通知进入休眠状态后，不要做任何用户级回调
        __CFPortSetInsert(dispatchPort, waitSet);
        // 标记休眠开始时间
        CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent();
        
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        do {
            msg = (mach_msg_header_t *)msg_buffer;
            // 7. __CFRunLoopServiceMachPort内部调用mach_msg函数等待接受mach_port的消息。随即线程将进入休眠，等待被唤醒。 以下事件会会唤醒runloop:
            // mach_msg接收到来自内核的消息。本质上是内核向我们的port发送了一条消息。即收到一个基于port的Source事件（source1）。
            // 一个timer的时间到了（处理timer）
            // RunLoop自身的超时时间到了（几乎不可能）
            // 被其他调用者手动唤醒（source0）
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
            
            if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
                // Drain the internal queue. If one of the callout blocks sets the timerFired flag, break out and service the timer.
                while (_dispatch_runloop_root_queue_perform_4CF(rlm->_queue));
                if (rlm->_timerFired) {
                    // Leave livePort as the queue port, and service timers below
                    rlm->_timerFired = false;
                    break;
                } else {
                    if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
                }
            } else {
                // Go ahead and leave the inner loop.
                break;
            }
        } while (1);
#else
        msg = (mach_msg_header_t *)msg_buffer;
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
#endif
        // 计算线程沉睡的时长
        rl->_sleepTime += (poll ? 0.0 : (CFAbsoluteTimeGetCurrent() - sleepStart));
        
        __CFPortSetRemove(dispatchPort, waitSet);
        
        __CFRunLoopSetIgnoreWakeUps(rl);
        // runloop置为唤醒状态
        __CFRunLoopUnsetSleeping(rl);
        // 8. 通知 Observers: RunLoop对应的线程刚被唤醒。
        if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);
        // 9. 收到&处理source1消息（第5步的goto会到达这里开始处理source1）
    handle_msg:;
        // 忽略端口唤醒runloop，避免在处理source1时通过其他线程或进程唤醒runloop(保证线程安全)
        __CFRunLoopSetIgnoreWakeUps(rl);
        
        if (MACH_PORT_NULL == livePort) {
            // livePort为null则什么也不做
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            // livePort为wakeUpPort则只需要简单的唤醒runloop（rl->_wakeUpPort是专门用来唤醒runloop的）
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
        }
#if USE_DISPATCH_SOURCE_FOR_TIMERS
        else if (modeQueuePort != MACH_PORT_NULL && livePort == modeQueuePort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            // 9.1 如果一个 Timer 到时间了，触发这个Timer的回调
            // __CFRunLoopDoTimers返回值代表是否处理了这个timer
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer, because we apparently fired early
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
#if USE_MK_TIMER_TOO
        else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) {
            CFRUNLOOP_WAKEUP_FOR_TIMER();
            if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) {
                // Re-arm the next timer
                __CFArmNextTimerInMode(rlm, rl);
            }
        }
#endif
        else if (livePort == dispatchPort) {
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            /// 9.2 如果有dispatch到main_queue的block，执行block（也就是处理GCD通过port提交到主线程的事件）。
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
        } else {
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            /// 9.3 如果一个 Source1 (基于port) 发出事件了，处理这个事件
            // 根据livePort获取source（不需要name，从mode->_portToV1SourceMap字典中以port作为key即可取到source）
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
                mach_msg_header_t *reply = NULL;
                // 处理source1事件（触发source1的回调）
                // runloop 触发source1的回调，__CFRunLoopDoSource1内部会调用__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__
                sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
                // 如果__CFRunLoopDoSource1响应的数据reply不为空则通过mach_msg 再send给内核
                if (NULL != reply) {
                    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
                    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
                }
            }
        }
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
        /// 执行加入到Loop的block
        __CFRunLoopDoBlocks(rl, rlm);
        
        if (sourceHandledThisLoop && stopAfterHandle) {
            /// 进入loop时参数说处理完事件就返回。
            retVal = kCFRunLoopRunHandledSource; // 4
        } else if (timeout_context->termTSR < mach_absolute_time()) {
            /// 超出传入参数标记的超时时间了
            retVal = kCFRunLoopRunTimedOut; // 3
        } else if (__CFRunLoopIsStopped(rl)) {
            /// 被外部调用者强制停止了
            __CFRunLoopUnsetStopped(rl); // 2
            retVal = kCFRunLoopRunStopped;
        } else if (rlm->_stopped) {
            // 调用了_CFRunLoopStopMode将mode停止了
            rlm->_stopped = false;
            retVal = kCFRunLoopRunStopped; // 2
        } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
            // source/timer/observer一个都没有了
            retVal = kCFRunLoopRunFinished; // 1
        }
        // 如果retVal不是0，即未超时，mode不是空，loop也没被停止，那继续loop
    } while (0 == retVal);
    
    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }
    
    return retVal;
}
```



# 主线程RunLoop和GCD的关系

当调用 dispatch_async(dispatch_get_main_queue(), block) 时，libDispatch  会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调  __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__() 里执行这个  block。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

# 总结

一句话概括runLoop：一个有状态的事件驱动的do...while循环。

# 参考文章

[CFBag](https://developer.apple.com/documentation/corefoundation/cfbag-s1l)

[RunLoop](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW23)

[ CFOptionFlags](https://developer.apple.com/documentation/corefoundation/cfoptionflags)

[mach_absolute_time 使用](https://www.cnblogs.com/zpsoe/p/6994811.html)

[CFRunLoop掘金](https://juejin.im/post/5b6817aee51d4519610e44f7)





