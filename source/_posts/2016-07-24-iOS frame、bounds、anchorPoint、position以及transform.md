---
layout: post_layout
title: iOS frame、bounds、anchorPoint、position以及transform
date: 2016-07-24 00:00:00
location: 北京
pulished: true
abstract: 本文是笔者记录的CALayer中的关键属性包括:frame、bounds、anchorPoint、position、transform。
---

### 1 frame

layer相对其父坐标系的位置。包括矩形左上角点，矩形宽高。*值得注意的是layer被旋转后的宽高。*如下图所示，bounds是40\*50，frame是62\*64。

![CALayer-frame](/assets/img/postImage/iOS frame、bounds、anchorPoint、position以及transform/CALayer-frame.jpg)

### 2 bounds

layer相对其内部坐标系的位置。

### 3 anchorPoint

layer的锚点（默认是{0.5, 0.5}，即在layer的中部）相对其*内部单位坐标系*的位置。锚点就是layer旋转的中点。左上角是{0, 0}，右下角是{1,1}。值得注意的是*锚点的值可以比0小，比1大，例如{-0.5, 1.5}，如此layer旋转可以围绕外部某个点*

![CALayer-anchorPoint](/assets/img/postImage/iOS frame、bounds、anchorPoint、position以及transform/CALayer-anchorPoint.png)

### 4 position

layer的锚点相对其外部坐标系的位置。

```objc
CALayer *layer = [[CALayer alloc] init];
[self.view.layer addSublayer:layer];
layer.backgroundColor = [[UIColor redColor] CGColor];
layer.bounds = CGRectMake(0, 0, 200, 60);
// 在不改变锚点的情况下设置layer居中
layer.position = CGPointMake(layer.superlayer.frame.size.width / 2, layer.position.y);  // 水平居中
layer.position = CGPointMake(layer.position.x, layer.superlayer.frame.size.height / 2); // 垂直居中
```

### 5 transform (3D变换）

坐标变换，变换公式如下：

![CALayer-transform](/assets/img/postImage/iOS frame、bounds、anchorPoint、position以及transform/CALayer-transform.jpg)

样例:

scale变换

```objc
CALayer *layer = [[CALayer alloc] init];
[self.view.layer addSublayer:layer];
layer.backgroundColor = [[UIColor redColor] CGColor];
layer.bounds = CGRectMake(0, 0, 200, 60);
layer.position = CGPointMake(200, 200);
layer.transform = CATransform3DMakeScale(2, 0.5, 1); // x拉升为之前的2倍，y压缩为之前的1 / 2, z不变

NSLog(@"CALayer frame : %@", NSStringFromCGRect(layer.frame));
NSLog(@"CALayer bounds : %@", NSStringFromCGRect(layer.bounds));
NSLog(@"CALayer position : %@", NSStringFromCGPoint(layer.position));
NSLog(@"CALayer anchorPoint : %@", NSStringFromCGPoint(layer.anchorPoint));

//2016-07-27 21:10:38.034 CommonTest[47569:5204919] CALayer frame : { {0, 185}, {400, 30} }
//2016-07-27 21:10:38.034 CommonTest[47569:5204919] CALayer bounds : { {0, 0}, {200, 60} }
//2016-07-27 21:10:38.035 CommonTest[47569:5204919] CALayer position : {200, 200}
//2016-07-27 21:10:38.035 CommonTest[47569:5204919] CALayer anchorPoint : {0.5, 0.5}
```

旋转变换

```objc
CGFloat Radian(CGFloat angle)
{
    return angle * M_PI / 180.f;
}

CALayer *layer = [[CALayer alloc] init];
[self.view.layer addSublayer:layer];
layer.backgroundColor = [[UIColor redColor] CGColor];
layer.bounds = CGRectMake(0, 0, 200, 60);
layer.position = CGPointMake(200, 200);
layer.transform = CATransform3DMakeRotation(Radian(30), 0, 0, 1); //绕着Z轴顺时针旋转30°

NSLog(@"%@ %@ %@ %@", @(layer.transform.m11), @(layer.transform.m12), @(layer.transform.m13), @(layer.transform.m14));
NSLog(@"%@ %@ %@ %@", @(layer.transform.m21), @(layer.transform.m22), @(layer.transform.m23), @(layer.transform.m24));
NSLog(@"%@ %@ %@ %@", @(layer.transform.m31), @(layer.transform.m32), @(layer.transform.m33), @(layer.transform.m34));
NSLog(@"%@ %@ %@ %@", @(layer.transform.m41), @(layer.transform.m42), @(layer.transform.m43), @(layer.transform.m44));
NSLog(@"CALayer frame : %@", NSStringFromCGRect(layer.frame));
NSLog(@"CALayer bounds : %@", NSStringFromCGRect(layer.bounds));
NSLog(@"CALayer position : %@", NSStringFromCGPoint(layer.position));
NSLog(@"CALayer anchorPoint : %@", NSStringFromCGPoint(layer.anchorPoint));

//2016-07-27 21:24:13.989 CommonTest[48025:5281194] 0.8660254037844387 0.4999999999999999 0 0
//2016-07-27 21:24:13.990 CommonTest[48025:5281194] -0.4999999999999999 0.8660254037844387 0 0
//2016-07-27 21:24:13.990 CommonTest[48025:5281194] 0 0 1 0
//2016-07-27 21:24:13.990 CommonTest[48025:5281194] 0 0 0 1
//2016-07-27 21:24:13.990 CommonTest[48025:5281194] CALayer frame : { {98.397459621556123, 124.01923788646684}, {203.20508075688778, 151.96152422706632} }
//2016-07-27 21:24:13.990 CommonTest[48025:5281194] CALayer bounds : { {0, 0}, {200, 60} }
//2016-07-27 21:24:13.991 CommonTest[48025:5281194] CALayer position : {200, 200}
//2016-07-27 21:24:13.991 CommonTest[48025:5281194] CALayer anchorPoint : {0.5, 0.5}
```

### 6 CALayer的frame决定因素
view或者layer的frame其实不是一个独立的属性，它是由bounds、position和transform决定的虚拟属性。因此，一旦以上3个属性发生变化frame就会变化。相反地，一旦改变frame，那么bounds、position和transform也可能变化。

	The frame is not really a distinct property of the view or layer at all; it is a virtual property, computed from the bounds, position, and transform, and therefore changes when any of those properties are modified. Conversely, changing the frame may affect any or all of those values, as well.

### 7 参考文档

[Core Animation Programming Guide](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40004514-CH1-SW1)

《iOS Core Animation Advanced Techniques》



