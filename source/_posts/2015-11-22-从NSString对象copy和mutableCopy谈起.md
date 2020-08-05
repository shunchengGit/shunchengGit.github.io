---
layout: post_layout
title: 从copy和mutableCopy谈起
date: 2015-11-22 00:00:00
location: 北京
pulished: true
abstract: 本文从Objective-C的copy和mutableCopy方法谈起，延伸了一些关于Objective-C内存管理的东西。
---

# 1 copy和mutableCopy

NSObject类有两个跟拷贝相关的方法——`copy`和`mutableCopy`。这两个方法都是返回一个id类型的对象，那么这两者之间有什么区别呢？根据官方文档解释，`copy`方法，返回`copyWithZone`方法返回的对象（Returns the object returned by copyWithZone:）。而`mutableCopy`方法，返回`mutableCopyWithZone`方法返回的对象(Returns the object returned by mutableCopyWithZone:)。读起来有点绕，一言以蔽之，调用`copy`就是调用`copyWithZone`，调用`mutableCopy`就是调用`mutableCopyWithZone`。**还是不够清楚！！！**接下来我们以NSString为例子，来说明`copy`和`mutableCopy`的区别。

## 1.1 NSString对象调用copy和mutableCopy

那么对NSString对象调用copy和mutableCopy究竟有什么效果呢？
在实验之前先定义如下通用宏

```objc
//打印方法名
#define LOG_METHOD_NAME {NSLog(@"%@", NSStringFromSelector(_cmd));}
//打印对象的类名，以及对象本身的地址
#define LOG_OBJ_ADDRESS(obj) {NSLog(@"%@ : %p",NSStringFromClass([obj class]), obj);}
//打印空行
#define LOG_END {NSLog(@"%@", @" ");}
```

测试代码如下：

```objc
- (void)stringCopyTest
{
    LOG_METHOD_NAME
    NSString *str = @"Hello";
    LOG_OBJ_ADDRESS(str);
    NSString *cpStr = [str copy];
    LOG_OBJ_ADDRESS(cpStr);
    NSMutableString *mutCpStr = [str mutableCopy];
    LOG_OBJ_ADDRESS(mutCpStr);
    LOG_END
}
```

输出结果如下：

```
2015-11-22 19:58:05.738 CopyTest[17230:1045019] stringCopyTest
2015-11-22 19:58:05.738 CopyTest[17230:1045019] __NSCFConstantString : 0x100fee0b0
2015-11-22 19:58:05.738 CopyTest[17230:1045019] __NSCFConstantString : 0x100fee0b0
2015-11-22 19:58:05.738 CopyTest[17230:1045019] __NSCFString : 0x7fabbaa00190
2015-11-22 19:58:05.738 CopyTest[17230:1045019]  
```

由以上输出可知，对一个NSString对象调用copy返回的还是该对象本身，因为str的地址和cpStr的地址是同一个。而调用mutableCopy，返回的是一个NSMutableString对象(注：__NSCFConstantString是常量串即NSString，而__NSCFString是可变串即NSMutableString）。

## 1.2 NSMutableString对象调用copy和mutableCopy

同理，引申到NSMutableString对象呢？
测试代码如下：

```objc
- (void)mutableStringCopyTest
{
    LOG_METHOD_NAME
    NSMutableString *mutStr = [@"OC" mutableCopy];
    LOG_OBJ_ADDRESS(mutStr);
    NSMutableString *cpMutStr = [mutStr copy];
    LOG_OBJ_ADDRESS(cpMutStr);
    NSMutableString *mutCpMutStr = [mutStr mutableCopy];
    LOG_OBJ_ADDRESS(mutCpMutStr);
    LOG_END
}
```

输出结果如下：

```
2015-11-22 19:58:05.738 CopyTest[17230:1045019] mutableStringCopyTest
2015-11-22 19:58:05.738 CopyTest[17230:1045019] __NSCFString : 0x7fabba9012c0
2015-11-22 19:58:05.739 CopyTest[17230:1045019] NSTaggedPointerString : 0xa0000000000434f2
2015-11-22 19:58:05.739 CopyTest[17230:1045019] __NSCFString : 0x7fabb840a860
2015-11-22 19:58:05.739 CopyTest[17230:1045019]  
```

由以上输出可知，对一个NSMutableString对象调用copy返回的是一个NSTaggedPointerString对象，该对象可认为是一个常量串。而调用mutableCopy返回的是另外一个可变对象__NSCFString，即NSMutableString（原NSMutableString对象的地址是0x7fabba9012c0，新NSMutableString对象地址是0x7fabb840a860）。

**针对NSArray、NSDictionary、NSSet等具有Mutable版本的类进行试验出现跟NSString类似的现象，不一一列举，有兴趣可以自己去试验。**

## 1.3 copy和mutableCopy调用小结

- 针对不可变对象调用copy返回该对象本身，调用mutableCopy返回一个可变对象（新的）；
- 针对可变对象调用copy返回一个不可变对象（新的），调用mutableCopy返回另外一个可变对象（新的）。


 class           | copy          | mutableCopy 
 -------------   | ------------- | ----------- 
 不可变（如，NSString）        | 返回本身（其他什么都不干） | 创建新的可变对象（如，创建一个NSMutableString对象，地址跟原对象不同）  
 可变（如，NSMutableString） | 创建新的不可变对象（如，创建一个NSTaggedPointerString对象，地址跟原对象不同 ）     |  创建新的可变对象（如，创建一个NSMutableString对象，地址跟原对象不同  ）     

再进一步从是否新建返回对象，返回对象是否可变两个角度总结如下：

- 只有不可变的copy是返回本身（其他什么都不干），其他都是创建一个新对象； 
- copy返回的是不可变对象，mutableCopy返回的是可变对象。

# 2 属性copy还是strong?

假设有两个id类型的属性如下:

```objc
@property (nonatomic, copy) id cpID;
@property (nonatomic, strong) id stID;
```

那么编译器把以上两属性分别实现为:

```objc
- (void)setCpID:(id)cpID {
    _cpID = [cpID copy];
}

- (id)cpID {
    return _cpID;
}

- (void)setStID:(id)stID {
    _stID = stID;
}

- (id)stID {
    return _stID;
}
```

从以上实现可以看出，strong和copy的属性主要是set方法有区别，strong的set是直接设置指定值，**而copy的set是设置指定值的copy版本**。接下来探索一下NSString、NSMutableString的copy和strong属性。

## 2.1 NSString属性copy还是strong?

测试代码如下:

```objc
@property (nonatomic, copy) NSString *cpStr;
@property (nonatomic, strong) NSString *stStr;

- (void)stringPropertyTest
{
    LOG_METHOD_NAME
    NSMutableString *mutStr = [@"123" mutableCopy];
    LOG_OBJ_ADDRESS(mutStr);
    self.cpStr = mutStr;
    LOG_OBJ_ADDRESS(self.cpStr);
    self.stStr = mutStr;
    LOG_OBJ_ADDRESS(self.stStr);
    
    NSLog(@"修改前");
    NSLog(@"mutStr：%@", mutStr);
    NSLog(@"copy：%@", self.cpStr);
    NSLog(@"strong：%@", self.stStr);
    
    [mutStr appendString:@"456"];
    NSLog(@"修改后");
    NSLog(@"mutStr：%@", mutStr);
    NSLog(@"copy：%@", self.cpStr);
    NSLog(@"strong：%@", self.stStr);
    
    LOG_END
}
```

输出结果如下：

```
2015-11-22 21:11:29.458 CopyTest[17977:1317284] stringPropertyTest
2015-11-22 21:11:29.458 CopyTest[17977:1317284] __NSCFString : 0x7fad23409b80
2015-11-22 21:11:29.458 CopyTest[17977:1317284] NSTaggedPointerString : 0xa000000003332313
2015-11-22 21:11:29.458 CopyTest[17977:1317284] __NSCFString : 0x7fad23409b80
2015-11-22 21:11:29.458 CopyTest[17977:1317284] 修改前
2015-11-22 21:11:29.458 CopyTest[17977:1317284] mutStr：123
2015-11-22 21:11:29.458 CopyTest[17977:1317284] copy：123
2015-11-22 21:11:29.458 CopyTest[17977:1317284] strong：123
2015-11-22 21:11:29.458 CopyTest[17977:1317284] 修改后
2015-11-22 21:11:29.458 CopyTest[17977:1317284] mutStr：123456
2015-11-22 21:11:29.459 CopyTest[17977:1317284] copy：123
2015-11-22 21:11:29.459 CopyTest[17977:1317284] strong：123456
2015-11-22 21:11:29.459 CopyTest[17977:1317284]  
```

由以上输出可知，假设两个NSString属性实际上指向的都是一个NSMutableString对象，那么在原NSMutableString对象修改后，strong版本的NSString属性跟着修改，而copy版本属性保持原状。self.cpStr实际上是一个NSTaggedPointerString对象，该对象正是NSMutableString对象执行copy的返回值。

## 2.2 NSMutableString属性copy还是strong

测试代码如下：

```objc
@property (nonatomic, copy) NSMutableString *cpMutStr;
@property (nonatomic, strong) NSMutableString *stMutStr;

- (void)mutableStringPropertyTest
{
    LOG_METHOD_NAME
    NSMutableString *mutStr = [@"123" mutableCopy];
    LOG_OBJ_ADDRESS(mutStr);
    self.cpMutStr = mutStr;
    LOG_OBJ_ADDRESS(self.cpMutStr);
    self.stMutStr = mutStr;
    LOG_OBJ_ADDRESS(self.stMutStr);
    
    NSLog(@"修改前");
    NSLog(@"mutStr：%@", mutStr);
    NSLog(@"copy：%@", self.cpMutStr);
    NSLog(@"strong：%@", self.stMutStr);
    
    [mutStr appendString:@"456"];
    NSLog(@"修改后");
    NSLog(@"mutStr：%@", mutStr);
    NSLog(@"copy：%@", self.cpMutStr);
    NSLog(@"strong：%@", self.stMutStr);
    LOG_END
}
```

输出结果如下：

```
2015-11-22 21:48:21.774 CopyTest[18508:1539502] mutableStringPropertyTest
2015-11-22 21:48:21.774 CopyTest[18508:1539502] __NSCFString : 0x7f806879b620
2015-11-22 21:48:21.774 CopyTest[18508:1539502] NSTaggedPointerString : 0xa000000003332313
2015-11-22 21:48:21.774 CopyTest[18508:1539502] __NSCFString : 0x7f806879b620
2015-11-22 21:48:21.774 CopyTest[18508:1539502] 修改前
2015-11-22 21:48:21.774 CopyTest[18508:1539502] mutStr：123
2015-11-22 21:48:21.774 CopyTest[18508:1539502] copy：123
2015-11-22 21:48:21.774 CopyTest[18508:1539502] strong：123
2015-11-22 21:48:21.775 CopyTest[18508:1539502] 修改后
2015-11-22 21:48:21.775 CopyTest[18508:1539502] mutStr：123456
2015-11-22 21:48:21.775 CopyTest[18508:1539502] copy：123
2015-11-22 21:48:21.775 CopyTest[18508:1539502] strong：123456
2015-11-22 21:48:21.775 CopyTest[18508:1539502]  
```

看起来没啥问题，strong版本的属性跟随原对象的变化而变化，copy版本的属性不变。但是，假设调用

```objc
[self.cpMutStr appendString:@"789"];
```

程序会崩溃。崩溃信息如下：

```
2015-11-22 21:51:37.282 CopyTest[18542:1545579] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[NSTaggedPointerString appendString:]: unrecognized selector sent to instance 0xa000000003332313'
*** First throw call stack:
```

原因很明显，是朝NSTaggedPointerString对象发了一个它不能识别的selector。原因是copy版本的NSMutableString属性本质上不是一个NSMutableString对象，而是一个NSTaggedPointerString对象，它是一个不可变对象。该对象是NSMutableString对象执行copy得来的，还记得我们上一节的结论吗？对一个对象执行copy得到的用于是一个不可变的对象。

**针对NSArray、NSDictionary、NSSet等具有Mutable版本的类进行试验出现跟NSString类似的现象。**

## 2.3 结论

- 不可变类型属性，**推荐使用copy**，因为假设该对象实际上指向的是一个mutable的对象，mutable对象的改变不会导致该对象的改变；假设指向的不是mutable的对象，那么copy和strong是等价，因为对一个不可变对象调用一次copy什么事情都不发生，_cpID = [cpID copy] 等价于_cpID = cpID。
- 可变类型属性，**不能使用copy**，因为copy产生的对象是一个不可变对象，跟属性描述是冲突的。

























