---
layout: post_layout
author: 宿于松下
title:  BlocksKit源码分析（一）
date:   2015-05-04 00:00:00
categories: iOS源码分析
tags: BlocksKit
abstract: 本文介绍了BlocksKit源码的组织结构：Core、DynamicDelegate、MessageUI、UIKit。并简要分析了Core的源码。该部分代码由5个小部分组成：容器、关联对象、逻辑执行、KVO、定时器。由源码可以看出BlocksKit扩展更加符合自然语义，如容器相关代码；BlocksKit一般通过中间层来承载回调函数，并在中间层回调的时候调用相应block，如KVO的BlocksKit实现。总体来说，Core代码是这些代码的最基础部分，具有很强的通用性。
---

## 1引言
众所周知Block已被广泛用于iOS编程。它们通常被用作可并发执行的逻辑单元的封装，或者作为事件触发的回调。Block比传统回调函数有2点优势：
1. **允许在调用点上下文书写执行逻辑，不用分离函数**
2. **Block可以使用local variables.**

基于以上种种优点Cocoa Touch越发支持Block式编程，这点从UIView的各种动画效果可用Block实现就可见一斑。而BlocksKit是对Cocoa Touch Block编程更进一步的支持，它简化了Block编程，发挥Block的相关优势，让更多UIKit类支持Block式编程。

## 2BlocksKit目录结构
BlocksKit代码存放在4个目录中分别是Core、DynamicDelegate、MessageUI、UIKit。其中：

- **Core** 存放Foundation Kit相关的Block category
- **DynamicDelegate** 动态代理（一种事件转发机制）相关代码
- **MessageUI** 存放MessageUI相关的Block category
- **UIKit** 存放UIKit相关的Block category

本文尝试去梳理BlocksKit的实现方式并列举BlocksKit使用的例子。

## 3Core相关代码分析和举例
Core文件夹下面的代码可以分为如下几个部分：

1. **容器相关**（NSArray、NSDictionary、NSSet、NSIndexSet、NSMutableArray、NSMutableDictionary、NSMutableSet、NSMutableIndexSet）
2. **关联对象相关**
3. **逻辑执行相关**
4. **KVO相关**
5. **定时器相关**

### 3.1容器相关的BlocksKit
不管是可变容器还是不可变容器，容器相关的BlocksKit代码总体上说是对容器原生block相关函数的封装。容器相关的BlocksKit函数更加接近自然语义，有一种函数式编程和语义编程的感觉。

以下给出部分不可变容器的BlocksKit声明：

```objc
//串行遍历容器中所有元素
- (void)bk_each:(void (^)(id obj))block; 
//并发遍历容器中所有元素（不要求容器中元素顺次遍历的时候可以使用此种遍历方式来提高遍历速度）
- (void)bk_apply:(void (^)(id obj))block;
//返回第一个符合block条件（让block返回YES）的对象
- (id)bk_match:(BOOL (^)(id obj))block;
//返回所有符合block条件（让block返回YES）的对象
- (NSArray *)bk_select:(BOOL (^)(id obj))block;
//返回所有！！！不符合block条件（让block返回YES）的对象
- (NSArray *)bk_reject:(BOOL (^)(id obj))block;
//返回对象的block映射数组
- (NSArray *)bk_map:(id (^)(id obj))block;

//查看容器是否有符合block条件的对象
//判断是否容器中至少有一个元素符合block条件
- (BOOL)bk_any:(BOOL (^)(id obj))block; 
//判断是否容器中所有元素都！！！不符合block条件
- (BOOL)bk_none:(BOOL (^)(id obj))block;
//判断是否容器中所有元素都符合block条件
- (BOOL)bk_all:(BOOL (^)(id obj))block;
```

其中bk_all函数的实现如下：

```objc
- (BOOL)bk_all:(BOOL (^)(id obj))block
{
	NSParameterAssert(block != nil);

	__block BOOL result = YES;

	[self enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
		if (!block(obj)) {
			result = NO;
			*stop = YES;
		}
	}];

	return result;
}
```

从以上各种函数声明以及bk_all函数的定义我们就可以看出，容器相关的BlocksKit函数确实更加接近自然语义，简化编程。

再给出部分不可变容器的BlocksKit声明：

```objc
/** Filters a mutable array to the objects matching the block.

 @param block A single-argument, BOOL-returning code block.
 @see <NSArray(BlocksKit)>bk_reject:
 */
//删除容器中!!!不符合block条件的对象，即只保留符合block条件的对象
- (void)bk_performSelect:(BOOL (^)(id obj))block;

//删除容器中符合block条件的对象
- (void)bk_performReject:(BOOL (^)(id obj))block;

//容器中的对象变换为自己的block映射对象
- (void)bk_performMap:(id (^)(id obj))block;
```

### 3.2关联对象相关的BlocksKit
关联对象的作用如下：
在类的定义之外为类增加额外的存储空间。使用关联，我们可以不用修改类的定义而为其对象增加存储空间。这在我们无法访问到类的源码的时候或者是考虑到二进制兼容性的时候是非常有用。关联是基于关键字的，因此，我们可以为任何对象增加任意多的关联，每个都使用不同的关键字即可。关联是可以保证被关联的对象在关联对象的整个生命周期都是可用的（ARC下也不会导致资源不可回收）。[关联对象的例子](http://blog.csdn.net/onlyou930/article/details/9299169) 

在我们的实际项目中的常见用法一般有category中用关联对象定义property，或者使用关联对象绑定一个block。

关联对象相关的BlocksKit是对objc_setAssociatedObject、objc_getAssociatedObject、objc_removeAssociatedObjects这几个原生关联对象函数的封装。主要是封装其其内存管理语义。

部分函数声明如下：

```objc
//@interface NSObject (BKAssociatedObjects)

//以OBJC_ASSOCIATION_RETAIN_NONATOMIC方式绑定关联对象
- (void)bk_associateValue:(id)value withKey:(const void *)key;
//以OBJC_ASSOCIATION_COPY_NONATOMIC方式绑定关联对象
- (void)bk_associateCopyOfValue:(id)value withKey:(const void *)key;
//以OBJC_ASSOCIATION_RETAIN方式绑定关联对象
- (void)bk_atomicallyAssociateValue:(id)value withKey:(const void *)key;
//以OBJC_ASSOCIATION_COPY方式绑定关联对象
- (void)bk_atomicallyAssociateCopyOfValue:(id)value withKey:(const void *)key;
//弱绑定
- (void)bk_weaklyAssociateValue:(__autoreleasing id)value withKey:(const void *)key;
//删除所有绑定的关联对象
- (void)bk_removeAllAssociatedObjects;
```

除了*弱绑定*以外，其它BlocksKit函数都是简单封装。只有弱绑定看起来挺牛B，有点扩展关联对象原生语义的感觉。

弱绑定实现如下：

```objc
@interface _BKWeakAssociatedObject : NSObject

@property (nonatomic, weak) id value;

@end

- (void)bk_weaklyAssociateValue:(__autoreleasing id)value withKey:(const void *)key
{
    _BKWeakAssociatedObject *assoc = objc_getAssociatedObject(self, key);
    if (!assoc) {
        assoc = [_BKWeakAssociatedObject new];
        //做一个_BKWeakAssociatedObject对象中间层
        //以非原子持有的方式绑定一个_BKWeakAssociatedObject对象
        //_BKWeakAssociatedObject该对象又有真实对象的弱引用
        [self bk_associateValue:assoc withKey:key];
    }
    //_BKWeakAssociatedObject的weak property设置为真正应该关联的对象
    assoc.value = value;
}
```

从以上实现代码可以看出，所谓弱绑定实际上是在关联对象之间做了一个中间层，让本对象以OBJC_ASSOCIATION_RETAIN_NONATOMIC的形式去关联中间层（_BKWeakAssociatedObject），而中间层又以weak property的形式去存储真正关联对象的指针。

### 3.3逻辑执行相关的BlocksKit
所谓逻辑执行，就是Block块执行。逻辑执行相关的BlocksKit是对dispatch_after函数的封装。使其更加符合语义。

主要函数如下：

```objc
//@interface NSObject (BKBlockExecution)

//主线程执行block方法，延迟时间可选
- (id)bk_performBlock:(void (^)(id obj))block afterDelay:(NSTimeInterval)delay
//后台线程执行block方法，延迟时间可选
- (id)bk_performBlockInBackground:(void (^)(id obj))block afterDelay:(NSTimeInterval)delay

//所有执行block相关的方法都是此方法的简化版，该函数在指定的block队列上以指定的时间延迟执行block
- (id)bk_performBlock:(void (^)(id obj))block onQueue:(dispatch_queue_t)queue afterDelay:(NSTimeInterval)delay;

//取消block，非常牛逼！！！一般来说一个block加到block queue上是没法取消的，此方法实现了block的取消操作（必须是用BlocksKit投放的block）
+ (void)bk_cancelBlock:(id)block;
```

```objc
- (id)bk_performBlock:(void (^)(id obj))block onQueue:(dispatch_queue_t)queue afterDelay:(NSTimeInterval)delay
{
	NSParameterAssert(block != nil);
	//cancelled是个__block变量，使得该block在加入queue后能够逻辑上取消。注意，仅仅是逻辑上取消，不能把block从queue中剔除。
	__block BOOL cancelled = NO;
	
	//在外部block之上加一层能够逻辑取消的代码，使其变为一个wrapper block
	//当调用wrapper(YES)的时候就让__block BOOL cancelled = YES,使得以后每次block主体都被跳过。
	void (^wrapper)(BOOL) = ^(BOOL cancel) {
	//cancel参数是为了在外部能够控制cancelled _block变量
		if (cancel) {
			cancelled = YES;
			return;
		}
		if (!cancelled) block(self);
	};
	
	//每个投入queue中的block实际上是wraper版的block
	dispatch_after(BKTimeDelay(delay), queue, ^{
	    //把cancel设置为NO,block能够逻辑执行
		wrapper(NO);
	});
	
	//返回wraper block，以便bk_cancelBlock的时候使用
	return [wrapper copy];
}

+ (void)bk_cancelBlock:(id)block
{
	NSParameterAssert(block != nil);
	void (^wrapper)(BOOL) = block;
	//把cancel设置为YES,修改block中_block cancelled变量，如果此时block未执行则，block在执行的时候其逻辑主体会被跳过
	wrapper(YES);
}
```

以上代码给出了bk_performBlock和bk_cancelBlock的实现。其中bk_performBlock对参数block进行一层封装后再用dispatch_after投入相应的queue。注意该函数会返回的wrapper block，以供cancel。

### 3.4KVO相关BlocksKit

KVO主要涉及两类对象，即“被观察对象“和“观察者“。

与“被观察对象”相关的函数主要有如下两个：

```objc
//添加观察者
- (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(void *)context;
//删除观察者
- (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath context:(void *)context;
```

与“观察者“相关的函数如下：

```objc
//观察到对象发生变化后的回调函数（观察回调）
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context;
```

通常的KVO做法是先对“被观察对象”添加“观察者”，同时在“观察者”中实现观察回调。这样每当“被观察对象”的指定property改变时，“观察者”就会调用观察回调。

KVO相关BlocksKit弱化了“观察者”这种对象，使得每当“被观察对象”的指定property改变时，就会调起一个block。具体实现方式是定义一个_BKObserver类，让该类实现观察回调、对被观察对象添加观察者和删除观察者。

_BKObserver类定义如下：

```objc
@interface _BKObserver : NSObject {
	BOOL _isObserving;
}

//存储“被观察的对象”
@property (nonatomic, readonly, unsafe_unretained) id observee;
@property (nonatomic, readonly) NSMutableArray *keyPaths;
//存储回调block
@property (nonatomic, readonly) id task;
@property (nonatomic, readonly) BKObserverContext context;

- (id)initWithObservee:(id)observee keyPaths:(NSArray *)keyPaths context:(BKObserverContext)context task:(id)task;

@end

static void *BKObserverBlocksKey = &BKObserverBlocksKey;
static void *BKBlockObservationContext = &BKBlockObservationContext;

@implementation _BKObserver

- (id)initWithObservee:(id)observee keyPaths:(NSArray *)keyPaths context:(BKObserverContext)context task:(id)task
{
	if ((self = [super init])) {
		_observee = observee;
		_keyPaths = [keyPaths mutableCopy];
		_context = context;
		_task = [task copy];
	}
	return self;
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    //观察者回调，在KV改变的时候调用相关block
	if (context != BKBlockObservationContext) return;

	@synchronized(self) {
		switch (self.context) {
			case BKObserverContextKey: {
				void (^task)(id) = self.task;
				task(object);
				break;
			}
			case BKObserverContextKeyWithChange: {
				void (^task)(id, NSDictionary *) = self.task;
				task(object, change);
				break;
			}
			case BKObserverContextManyKeys: {
				void (^task)(id, NSString *) = self.task;
				task(object, keyPath);
				break;
			}
			case BKObserverContextManyKeysWithChange: {
				void (^task)(id, NSString *, NSDictionary *) = self.task;
				task(object, keyPath, change);
				break;
			}
		}
	}
}

//开启KV观察
- (void)startObservingWithOptions:(NSKeyValueObservingOptions)options
{
	@synchronized(self) {
		if (_isObserving) return;

		[self.keyPaths bk_each:^(NSString *keyPath) {
		//observee的被观察对象，observer是自己，
			[self.observee addObserver:self forKeyPath:keyPath options:options context:BKBlockObservationContext];
		}];

		_isObserving = YES;
	}
}

//停止KV观察
- (void)stopObservingKeyPath:(NSString *)keyPath
{
	NSParameterAssert(keyPath);

	@synchronized (self) {
		if (!_isObserving) return;
		if (![self.keyPaths containsObject:keyPath]) return;

		NSObject *observee = self.observee;
		if (!observee) return;

		[self.keyPaths removeObject: keyPath];
		keyPath = [keyPath copy];

		if (!self.keyPaths.count) {
			_task = nil;
			_observee = nil;
			_keyPaths = nil;
		}

		[observee removeObserver:self forKeyPath:keyPath context:BKBlockObservationContext];
	}
}
```

具体添加删除KVO block的函数实现如下：

```objc
//添加观察block
- (void)bk_addObserverForKeyPaths:(NSArray *)keyPaths identifier:(NSString *)identifier options:(NSKeyValueObservingOptions)options context:(BKObserverContext)context task:(id)task
{
	NSParameterAssert(keyPaths.count);
	NSParameterAssert(identifier.length);
	NSParameterAssert(task);

    Class classToSwizzle = self.class;
    NSMutableSet *classes = self.class.bk_observedClassesHash;
    @synchronized (classes) {
        NSString *className = NSStringFromClass(classToSwizzle);
        if (![classes containsObject:className]) {
            SEL deallocSelector = sel_registerName("dealloc");
            
			__block void (*originalDealloc)(__unsafe_unretained id, SEL) = NULL;
            
            //勾住类的dealloc方法，并且在其中添加bk_removeAllBlockObservers方法
			id newDealloc = ^(__unsafe_unretained id objSelf) {
                [objSelf bk_removeAllBlockObservers];
                
                if (originalDealloc == NULL) {
                    struct objc_super superInfo = {
                        .receiver = objSelf,
                        .super_class = class_getSuperclass(classToSwizzle)
                    };
                    
                    void (*msgSend)(struct objc_super *, SEL) = (__typeof__(msgSend))objc_msgSendSuper;
                    msgSend(&superInfo, deallocSelector);
                } else {
                    originalDealloc(objSelf, deallocSelector);
                }
            };
            
            IMP newDeallocIMP = imp_implementationWithBlock(newDealloc);
            
            if (!class_addMethod(classToSwizzle, deallocSelector, newDeallocIMP, "v@:")) {
                // The class already contains a method implementation.
                Method deallocMethod = class_getInstanceMethod(classToSwizzle, deallocSelector);
                
                // We need to store original implementation before setting new implementation
                // in case method is called at the time of setting.
                originalDealloc = (void(*)(__unsafe_unretained id, SEL))method_getImplementation(deallocMethod);
                
                // We need to store original implementation again, in case it just changed.
                originalDealloc = (void(*)(__unsafe_unretained id, SEL))method_setImplementation(deallocMethod, newDeallocIMP);
            }
            
            [classes addObject:className];
        }
    }

   //生成_BKObserver对象（真正的KV Observer）
	NSMutableDictionary *dict;
	_BKObserver *observer = [[_BKObserver alloc] initWithObservee:self keyPaths:keyPaths context:context task:task];
	//启动观察
	[observer startObservingWithOptions:options];

   //将新生成的_BKObserver对象添加到dic中
	@synchronized (self) {
		dict = [self bk_observerBlocks];

		if (dict == nil) {
			dict = [NSMutableDictionary dictionary];
			[self bk_setObserverBlocks:dict];
		}
	}

	dict[identifier] = observer;
}

//删除观察block
- (void)bk_removeAllBlockObservers
{
	NSDictionary *dict;

	@synchronized (self) {
		dict = [[self bk_observerBlocks] copy];
		[self bk_setObserverBlocks:nil];
	}

	[dict.allValues bk_each:^(_BKObserver *trampoline) {
		[trampoline stopObserving];
	}];
}
```

### 3.5定时器相关BlocksKit
NSTimer有个比较恶心的特性，它会持有它的target。比如在一个controller中使用了timer，并且timer的target设置为该controller本身，那么想在controller的dealloc中fire掉timer是做不到的，必须要在其他的地方fire。这会让编码很难受。具体参考《Effective Objective C》的最后一条。 BlocksKit解除这种恶心，其方式是把timer的target设置为timer 的class对象。把要执行的block保存在timer的userInfo中执行。因为timer 的class对象一直存在，所以是否被持有其实无所谓。

实现代码如下：

```objc
+ (id)bk_scheduledTimerWithTimeInterval:(NSTimeInterval)inTimeInterval block:(void (^)(NSTimer *timer))block repeats:(BOOL)inRepeats
{
    NSParameterAssert(block != nil);
    return [self scheduledTimerWithTimeInterval:inTimeInterval target:self selector:@selector(bk_executeBlockFromTimer:) userInfo:[block copy] repeats:inRepeats];
}

+ (id)bk_timerWithTimeInterval:(NSTimeInterval)inTimeInterval block:(void (^)(NSTimer *timer))block repeats:(BOOL)inRepeats
{
    NSParameterAssert(block != nil);
    return [self timerWithTimeInterval:inTimeInterval target:self selector:@selector(bk_executeBlockFromTimer:) userInfo:[block copy] repeats:inRepeats];
}

+ (void)bk_executeBlockFromTimer:(NSTimer *)aTimer {
    void (^block)(NSTimer *) = [aTimer userInfo];
    if (block) block(aTimer);
}
```

## 4BlocksKit源码分析（一）小结
本文介绍了BlocksKit源码的组织结构：Core、DynamicDelegate、MessageUI、UIKit。并简要分析了Core的源码。该部分代码由5个小部分组成：容器、关联对象、逻辑执行、KVO、定时器。由源码可以看出BlocksKit扩展更加符合自然语义，如容器相关代码；BlocksKit一般通过中间层来承载回调函数，并在中间层回调的时候调用相应block，如KVO的BlocksKit实现。总体来说，Core代码是这些代码的最基础部分，具有很强的通用性。

