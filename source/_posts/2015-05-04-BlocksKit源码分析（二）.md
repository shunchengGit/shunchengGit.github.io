---
layout: post_layout
author: 宿于松下
title:  BlocksKit源码分析（二）
date:   2015-05-04 00:00:00
categories: iOS源码分析
tags: BlocksKit
abstract: 本文介绍了BlocksKit的“动态代理”的实现方式。
---

## 1引言
在**《BlocksKit源码分析（一）》**中我们分析了BlocksKit源码组织结构以及第一部分Core的源码。在这里我们接着分析BlocksKit第二部分——DynamicDelegate（动态代理）。所谓动态代理，听起来挺玄乎。实际一言以蔽之，就是把delegate转为block的手段。

## 2动态代理样例
我们先从一个例子来看看动态代理的使用方式：

```objc
- (IBAction) annoyUser
{
	// 创建一个alert view
	UIAlertView *alertView = [[UIAlertView alloc]
							  initWithTitle:@"Hello World!"
							  message:@"This alert's delegate is implemented using blocks. That's so cool!"
							  delegate:nil
							  cancelButtonTitle:@"Meh."
							  otherButtonTitles:@"Woo!", nil];

	// 获取该alert view的动态代理对象（什么是动态代理对象稍后会说）
	A2DynamicDelegate *dd = alertView.bk_dynamicDelegate;
	
	// 调用动态代理对象的 - (void)implementMethod:(SEL)selector withBlock:(id)block;方法，使得SEL映射一个block对象(假设叫做block1)
	[dd implementMethod:@selector(alertViewShouldEnableFirstOtherButton:) withBlock:^(UIAlertView *alertView) {
		NSLog(@"Message: %@", alertView.message);
		return YES;
	}];

	// 同上，让映射-alertView:willDismissWithButtonIndex:的SEL到另外一个block对象(假设叫做block2)
	[dd implementMethod:@selector(alertView:willDismissWithButtonIndex:) withBlock:^(UIAlertView *alertView, NSInteger buttonIndex) {
		NSLog(@"You pushed button #%d (%@)", buttonIndex, [alertView buttonTitleAtIndex:buttonIndex]);
	}];

	// 把alertView的delegate设置为动态代理
	alertView.delegate = dd;
	
	[alertView show];
}
 // 那么，alert view在显示的时候收到alertViewShouldEnableFirstOtherButton:消息调用block1；alert view在消失的时候收到alertView:willDismissWithButtonIndex:消息，调用block2
```

从上面的代码我们可以直观地看到：dd（动态代理对象）直接被设置为alert view的delegate对象，那么该alert view的UIAlertViewDelegate消息直接传递向给了dd。然后dd又通过某种方式把对应的SEL调用转为对应的block调用。我们又可以作出如下猜测：
     
 1. dd内部可能有个dic一样的数据结构，key可能是SEL，value可能是与之对应的block，通过implementMethod:withBlock:这个方法把SEL和block以键值对的形式建立起dic映射
 2. Host对象（本例是alertView）向dd发delegate消息的时候传递了SEL，dd在内部的dic数据结构查找对应的block，找到后，调用该block。
 
基本思路是这样的，but还有两个问题需要解决：

 1. dd（动态代理对象）是如何创建的？
 2. 如UIAlertViewDelegate这样的各种delegate有不同的消息，任意一个消息都可能发给dd，dd是如何能接受任意消息的，显然实现每个消息是不现实的。

## 3消息转发机制
我们先来回答第二个问题，即动态代理是如何能接受任意消息的。
 当一个object收到它没实现的消息的时候，通常会发生如下的情况。
 
 1. 首先看是否为该 selector 提供了动态方法决议机制,如果提供了则转到2;如果没有提供则转到3;
 2. 如果动态方法决议真正为该 selector 提供了实现,那么就调用该实现,完成消息发送流程,消息转发就不会进行了;如果没有提供,则转到 3;
 3. 其次看是否为该 selector 提供了消息转发机制,如果提供了消息了则进行消息转发,此时,无论消息 转发是怎样实现的,程序均不会 crash。(因为消息调用的控制权完全交给消息转发机制处理,即使消息转发并没有做任何事情,运行也不会有错误,编译器更不会有错误提示。);如果没提供消息转发机制, 则转到 4;
 4. 运行报错:无法识别的 selector,程序 crash;

 通过以上描述我们可以知道。dd（A2DynamicDelegate类）支持任意消息发送，不是用了消息决议就是用了消息转发。实际上通过代码我们可以看出 A2DynamicDelegate是继承自NSProxy类，该类是专门做消息转发的。继承自该类的类必须要实现- (void)forwardInvocation:(NSInvocation *)outerInv方法。因为该方法是当该类的对象收到无法识别的消息的时候调用的方法。所以dd消息映射流程大概是这样的：当dd收到无法识别的消息时会调用forwardInvocation，该函数的参数outerInv可以拿到对应消息的SEL,再通过SEL在再内部的dic里面找到对应的block进行调用。（简要思路）

## 4动态代理对象的创建
A2DynamicDelegate这种对象是如何创建的？为什么从代码里面看dd（A2DynamicDelegate）好像是alertView的一个property。
 实际上dd是通过NSObject的(A2DynamicDelegate)分类来实现的。该分类用关联对象的方法，把protocal和A2DynamicDelegate对象关联起来。使得相应的protocal有对应的A2DynamicDelegate对象。某个类的bk_dynamicDelegate实际上是查找该类对应delegate的protocal，再根据这个protocal找到对应的A2DynamicDelegate对象。如果找不到A2DynamicDelegate对象就创建一个关联起来。比如alertView.bk_dynamicDelegate;会查找alertView的delegate UIAlertViewDelegate protocal所关联的对象，如果没有找到就创建一个，并且做关联。
 相关代码如下：
  
```objc
 // File:NSObject+A2DynamicDelegate.m
 - (id)bk_dynamicDelegateWithClass:(Class)cls forProtocol:(Protocol *)protocol
{
	/**
	 * Storing the dynamic delegate as an associated object of the delegating
	 * object not only allows us to later retrieve the delegate, but it also
	 * creates a strong relationship to the delegate. Since delegates are weak
	 * references on the part of the delegating object, a dynamic delegate
	 * would be deallocated immediately after its declaring scope ends.
	 * Therefore, this strong relationship is required to ensure that the
	 * delegate's lifetime is at least as long as that of the delegating object.
	 **/

	__block A2DynamicDelegate *dynamicDelegate;

	dispatch_sync(a2_backgroundQueue(), ^{
		dynamicDelegate = objc_getAssociatedObject(self, (__bridge const void *)protocol);
        
		if (!dynamicDelegate)
		{
			dynamicDelegate = [[cls alloc] initWithProtocol:protocol];
			objc_setAssociatedObject(self, (__bridge const void *)protocol, dynamicDelegate, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
		}
	});

	return dynamicDelegate;
}
```

## 5代码设计
A2DynamicDelegate是动态代理类，NSObject_A2DynamicDelegate是NSObject分类用来创建动态代理对象。A2BlockInvocation这个类我们之前没有说过，它是block的一层封装。A2DynamicDelegate里面的字典数据结构装得不是纯粹的block，而是A2BlockInvocation对象。

相关类图如下：
![类图](/assets/img/postImage/BlocksKit2/1.png)

相关序列图如下：
![序列图](/assets/img/postImage/BlocksKit2/2.png)

 
