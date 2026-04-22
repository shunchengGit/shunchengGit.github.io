---
layout: post_layout
title: 从C++到Objective-C的学习笔记(CSDN早期博客)
date: 2014-01-06 00:00:00
location: 北京
pulished: true
abstract: 从C++到Objective-C的学习笔记。
---


### 1头文件导入与新的基本类型

OC的头文件跟C和CPP一样都是.h，而OC的源文件是.m

导入头文件是用`#import`，这比`#include`好的地方在于默认设置了头文件保护，不需要在头文件中设置保护符

`#import <Foundation/Foundation.h>`

前面的Foundation是框架名，后面的Foundation.h是头文件名

OC引入了新的基本类型BOOL，类似于CPP中的bool，但是BOOL的值是YES /NO

语言          |C         | CPP   | ObjC
 -------------   | ------------- | ----------- | ----------- 
**源文件**      | .c | .cpp | .m
**.h导入符号**   | `#include`| `#include` | `#import`
**.h导入方式**       | 导入文件名  |  	 导入文件名 |  导入框架/文件名
**Log输出**          |   printf |  cout | NSLog

### 2面向对象

#### 类

创建一个OC类的关键字是@interface，实现一个类的关键词是@implementation

而CPP创建类是class，实现类没有关键词，只是函数前面都要加"类名::"。

所有的OC类必须要直接或者间接继承NSObject类。而CPP没有限制

OC在类的.h文件中不定义方法名字，直接在.m文件中写这些方法是没问题的。

而CPP必须要在头文件的class块中定义方法名字，才能在.CPP中实现。

#### 对象

OC的对象全部是new出来的，都存在heap上，而CPP的对象可能存在stack上，也可能在heap上

#### 消息

OC对对象的调用是基于消息的。这样即使在类的中没有明确指出该消息，也能调用该消息。编译是能过的。

CPP类要是没有写出成员函数，调用该成员函数是肯定编译不过的。

#### 类对象与OC对象模型

OC中存在类对象，每个OC对象中都有一个指针指向其类对象，以此来标识这个对象所从属的类。类对象也是一种对象。OC的面向对象模型是基于这种类对象，查表找对应的函数。

CPP中无类对象，面向对象的多态是基于vtbl

对象模型          | CPP   | ObjC
 -------------  | ----------- | ----------- 
 文档 |  《深度探索C++对象模型》 |  《深入浅出Cocoa教程》
 作者 | 侯捷 |  白云飘飘
 
OC多了类对象一层，程序多了一层间接性，因此效率比CPP要低一些，但是正因为这种间接性，所用其编程的灵活性应该比CPP要强，更加容易设计出易于扩展和维护的程序。因此OC中的设计模式可能更加灵活。

#### id
OC中的id类型泛指某个对象的指针，其实可以看作NSObject*
对应的CPP中相当于void*

#### 虚函数
OC中每个成员函数都可以当作虚函数，而CPP中标记为virtual的成员函数才是虚函数。

#### 继承
OC只支持单继承，而且没有继承方式
CPP支持多继承，继承方式有public、protected、private

#### Big3

**构造**

OC中以init开头的函数是构造函数，但是它不会像CPP的构造函数那样先帮你调用父类的构造函数。在OC中需要手动调用[super init];

**析构**

OC对象的析构函数统一为-(void) dealloc; 析构本类成员后需要手动调用[super dealloc];

**复制构造**

OC对象的复制构造以copy 或者 mutableCopy开头

### 3内存管理

OC的对象全部是alloc后init的存在heap上，stack上只是保存OC对象的指针。

不同的指针指向同一个heap对象的时候是以引用计数的方式进行统计的。

对象初创的时候，其计数是1， retain消息会把对象引用计数+1，release消息会把对象引用计数-1。当引用计数减到0的时候，自动调用对象的dealloc函数，对对象进行析构。

autorelease消息会把对象注册到距离最近的AutoreleasePool上去，一但这个AutoreleasePool被析构，它会向注册到其上的每个对象发一个release消息，使他们的对象引用计数-1。

### 4property（属性）

在类声明的时候以@property的方式来声明一个属性，

在类实现的时候以@synthesize来实现属性的存取方法

MRC模式下属性的参数

1复制
assign（默认），retain，copy

2读写
readwrite(默认)，readonly

3原子性
 atomic（默认）nonatomic
 
### 5category（类别）

```objc
@interface 宿主类名（类别名）
声明方式如下：
@interface HostClass(MyCategory)
-(void) categoryFunc;
@end
```

作用：

1. 分散类的实现，一个大类分拆成小的category
2. 为类添加前向声明（消除编译器的warning）
3. 当作非正式的协议，category中的方法不必每个都实现

### 6protocol（协议）
用@protocol来声明协议，若某个类需要实现协议必须把协议放在他父类的尖括号列表中，并且该类必须实现protocol中的所有声明的方法
如：

```objc
//protocol definition
@protocol Myprotocol
-(void) protocolFun;
@end
//use protocol
@interface MyClass : NSObject<Myprotocol>
@end
```

### 7delegate（委托）

委托是一种设计模式，就是把本类需要实现的东西，交给其委托类去实现，一般来说委托是基于protocol的，用protocol是声明某种接口，比如

```objc
@protocol MyDelegate
-(void) delegateFunc;
@end
@interface HostClass
{
NSObject<MyDelegate>  *delegate;
}
@end
```

实现MyDelegate协议的所有类都能当作HostClass的委托对象。

### 8 常用的Foundation Kit类

字符串

NSString：类似于string，但是！!!不能直接用＝＝比较两个字符是否相等，因为OC没有操作符重载,应该使用isEqualToString

集合

NSArray\NSMutableArray：类似于vector
NSDictionary\NSMutableDictionary： 类似于map！！！objectForKey和KVC的valueForKey；setObject：forKey：和KVC的setValue：forKey：不要混淆

数值

NSNumber：标量的类封装，如int，float等都可以使用NSNumber封装
NSValue：任意类型的类封装，以byte方式实现
NSNull：nil的类封装

### 9KVC

```objc
- (void)setValue:(id)value forKey:(NSString *)key
来设置对象中的属性
- (id)valueForKey:(NSString *)key
来获取对象中的属性
内部调用的是key名称的property，如果没有property，就就直接查找key名称或者_key名称的成员变量
路径
- (id) valueForKeyPath:(NSString *)key
获取路径上的属性值
整体操作
如果向NSArray请求一个键值，它实际上会查询数组中的每个对象来查找这个值，然后讲查询结果打包到另外一个数组并且返回给你
运算符
用- (id) valueForKeyPath:(NSString *)key可以引用一些运算符来进行运算
@count计算左侧的结果,用于计数
@sum计算总和,如:@"cars.@sum.mileage" @sum将结果拆成两部分
@agv计算平均值,如:@"cars.@avg.mileage"
@min取最小值
@max取最大值
```

