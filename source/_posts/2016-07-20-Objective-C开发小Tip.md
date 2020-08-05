---
layout: post_layout
title: Objective-C开发小Tip
date: 2016-07-20 00:00:00
location: 北京
pulished: true
abstract: 本文是2015年末笔者在开发美团APP外卖频道和美团外卖APP时总结的一些OC开发小Tip。
---

### 1 说好的格式呢

#### 1.1 static NSString *这种变量的命名尽量以k开头。

```objc
static NSString * const WMHomePageFlowPath = @"WMHomePageFlowPath";`

变为

static NSString * const kWMHomePageFlowPath = @"kWMHomePageFlowPath";
```

#### 1.2 定义常量最好用static const

```objc
#define TOP_VIEW_FIX_HEIGHT 116.0

变为

static const CGFloat kTopViewFixHeight = 116.0f;
```

#### 1.3 宏定义的名称全大写字母

#### 1.4 使用现代OC语法

```objc
[eventDictionary setObject:code forKey:@"code"];

变为

eventDictionary[@"code"] = code;
```

```objc
[cellIndexs objectAtIndex:cellIndexs.count - 1];

变为

cellIndexs[cellIndexs.count - 1];
```

### 2 并没有什么卵用

#### 2.1 函数传block参数，无意义的^{}不应该存在

```objc
[UIView animateWithDuration:0.1 animations:^{}];// 无意义
```

#### 2.2 IDE自动生成的无用函数应该删除

```objc
- (void)didReceiveMemoryWarning {// IDE自动生成的函数，没用到就删了
    [super didReceiveMemoryWarning];
}
```

#### 2.3 无意义的判断

```objc
if (bottomImageView) {
    bottomImageView.frame = imageViewRect;
}
```

### 3 良の实践

#### 3.1 字面量字典或者字面量数组语法使用要注意nil变量

```objc
NSDictionary *URLParameters = @{@"address" : address,
                                @"keyword" : keyword,
                                @"poi_page_index" : poiPageIndex,
                                @"poi_page_size" : poiPageSize,
                                @"food_page_index" : foodPageIndex,
                                @"food_page_size" : foodPageSize};
// 以上代码在address、keyword、poiPageIndex、poiPageSize、foodPageIndex、foodPageSize任意一个变量为nil的时候会crash
```
一种经常出现的crash是使用字面量字典或者字面量数组的时候误添加值为nil的变量，从而直接崩溃。

#### 3.2 各种type需要enum化

```objc
if (self.poi.deliveryType == 1) { //magic number，enum化
    deliveryTimeView = [self viewForMTDelivery];
} else {
    deliveryTimeView = [self viewForDeliveryTime];
}
```

#### 3.3 恰当运用三目操作符

```objc
if (poi.picture.length > 0) {
    [self.headImgView sd_setImageWithURL:[NSURL URLWithString:poi.picture]];
} else {
    [self.headImgView sd_setImageWithURL:[NSURL URLWithString:defaultImage]];
} 

变为

NSString *imgURLStr = poi.picture.length > 0 ? poi.picture : defaultImage;
[self.headImgView sd_setImageWithURL:[NSURL URLWithString:imgURLStr]];
```

#### 3.4 同一个类的不同函数多次用到同一个常量，该常量应该升级为.m文件static const常量

```objc
titleLabel.centerY = 22.5;// 45 / 2.0 ? 建议45可以提出来，好多地方都用到了
```

#### 3.5 一个函数内部有意义的量应该升级为const常量

```objc
例1
if (distance > 500000) {
    distance = 500000;
}
// 这些数字用 static const NSInteger命名一下？或者加个注释？

例2
if (mutebelArray.count > 10) {//只保留10个搜索历史
    NSRange removeRange = NSMakeRange(10, mutebelArray.count - 10);
    [mutebelArray removeObjectsInRange:removeRange];
}
// 10提出来，kMaxRetainedSearchKeyCount = 10
```

#### 3.6 protocol中申明为@optional的方法必须在调用处先用一下respondsToSelector判断

```objc
@protocol WMFoodListViewControllerDelegate <NSObject>

@optional
- (void)incFood:(WMOrderedSkuInfo *)orderedSku count:(NSInteger)count inBucket:(NSInteger)bucketNum animated:(BOOL)animated originalRect:(CGRect)originalRect;
- (void)decFood:(WMOrderedSkuInfo *)orderedSku count:(NSInteger)count;

@end
// 检查一下这些optional的代理方法。是否用respondsToSelector保护了
```

#### 3.7 常用UI量应该变为宏定义

```objc
//ui
NaviBar高度
#ifndef WM_NAVIBAR_HEIGHT
#define WM_NAVIBAR_HEIGHT  (WM_ABOVE_IOS(7) ? 64:44)
#endif

//1点的线宽
#ifndef WM_SINGLE_LINE_WEIGHT
#define WM_SINGLE_LINE_WEIGHT     (1 / [UIScreen mainScreen].scale)
#endif
```

#### 3.8 非必要不要在头文件里面直接#import其他头文件

```objc
某.h文件内容如下。。。

#import <Foundation/Foundation.h>
#import "WMOrder.h"
#import "WMDeliveryInfo.h"

@interface WMOrderPreviewInfo : NSObject

@property (nonatomic,strong) WMOrder *order;
@property (nonatomic, strong) WMDeliveryInfo *deliveryInfo;

@end

变为

@class WMOrder
@class WMDeliveryInfo

@interface WMOrderPreviewInfo : NSObject

@property (nonatomic,strong) WMOrder *order;
@property (nonatomic, strong) WMDeliveryInfo *deliveryInfo;

@end
```

尽量使用“向前声明”，减少类之间的编译耦合度。

#### 3.9 非必要不使用下划线访问变量

除了初始化函数，其他各处非必要不使用下划线访问变量。不要直接访问property的内置的私有变量，不管读写，都用self.propertyName。否则在把代码重构为block的时候很容易忘记添加self，而引起循环引用。**而通常使用self.propertyName访问变量的话，在代码进行block重构时，更容易发现错误。**

```objc
@interface Test ()

@property (nonatomic, assign) NSInteger age;

@end

@impliment Test

- (void)fooDelegate:(NSInteger)theAge
{
    _age = theAge;
}

@end

[WMAlert showWithAction:^(NSInteger theAge) {
    _age = theAge;  //maybe cause retain cycle
}];
```

#### 3.10 返回BOOL的方法设计和使用时要特别注意

```objc
设计一个方法，来表示对象中有没有内容

@interface Foo : NSObject

-(BOOL)isEmpty;

@end

如果判断这个对象是不是“空”很容易写出如下代码

if ([foo isEmpty]) {
	//当foo为“空”的时候执行一些动作
}

这种判断方式有个问题，即当foo为nil的时候会存在误判，很明显foo为nil时，该对象在语义上是“空”，但是它不会进入该if分支，因为[nil isEmpty]返回NO.

因此，正确的写法应该是：

if (foo == nil || [foo isEmpty]) {
	//当foo为“空”的时候执行一些动作
}

追根溯源还是这种签名设计方式有漏洞，试想OC的NSString并没有一个isEmpty方法，是否是基于这种考虑。

```
