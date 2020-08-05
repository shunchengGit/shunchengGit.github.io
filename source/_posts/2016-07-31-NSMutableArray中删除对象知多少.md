---
layout: post_layout
title: NSMutableArray中删除对象知多少
date: 2016-07-31 00:00:00
location: 北京
pulished: true
abstract: 本文是笔者对NSMutableArray删除对象的总结。
---

### 1 removeObjectAtIndex和removeObject的不同之处

`removeObjectAtIndex:`

	Removes the object at index .
	To fill the gap, all elements beyond index are moved by subtracting 1 from their index.
	
	Important:Important
	Raises an exception NSRangeException if index is beyond the end of the array.
	
删除指定NSMutableArray中指定index的对象，注意index不能越界。

`removeObject:`

	Removes all occurrences in the array of a given object.
	This method uses indexOfObject: to locate matches and then removes them by using removeObjectAtIndex:. Thus, matches are determined on the basis of an object’s response to the isEqual: message. If the array does not contain anObject, the method has no effect (although it does incur the overhead of searching the contents).
	
删除NSMutableArray中所有isEqual:待删对象的对象

从API文档可以看出，两者之间的主要区别是`removeObjectAtIndex:`最多只能删除一个对象，而`removeObject:`可以删除多个对象（只要符合`isEqual:`的都删除掉）。

### 2 在NSMutableArray循环中删除对象

#### 2.1 可能多删的做法

删除数组中的第一个@"remove"

```objc
- (void)removeObjectsUseFor
{   
    NSMutableArray *contents = [@[@"how", @"to", @"remove", @"remove", @"object"] mutableCopy];
    for (NSInteger i = 0; i != contents.count; ++i) {
        NSString *var = contents[i];
        if ([var isEqualToString:@"remove"]) {
            [contents removeObject:var];
            break;
        }
    }
    
    NSLog(@"%@", contents);
}
```

结果如下：

```
2016-07-31 21:14:13.541 RemoveObject[5862:310398] (
    how,
    to,
    object
)
```

从`removeObject:`的说明中可以看出，`removeObject:`不仅删除该对象本身，而且删除NSMutableArray中*所有*`isEqual:`待删对象的对象

#### 2.2 可能漏删的做法

删除数组中所有的@"remove"

```objc
- (void)removeObjectsUseFor
{   
    NSMutableArray *contents = [@[@"how", @"to", @"remove", @"remove", @"object"] mutableCopy];
    for (NSInteger i = 0; i != contents.count; ++i) {
        NSString *var = contents[i];
        if ([var isEqualToString:@"remove"]) {
            [contents removeObjectAtIndex:i];
        }
    }
    
    NSLog(@"%@", contents);
}
```

输出：

```
2016-07-31 21:19:59.615 RemoveObject[5886:315162] (
    how,
    to,
    remove,
    object
)
```

#### 2.3 引发崩溃的做法

删除数组中所有的@"remove"

```objc
- (void)removeObjectsUseForIn
{
    NSMutableArray *contents = [@[@"how", @"to", @"remove", @"object"] mutableCopy];
    for (NSString *var in contents) {
        if ([var isEqualToString:@"remove"]) {
            [contents removeObject:var];
        }
    }
    
    NSLog(@"%@", contents);
}
```

输出：崩溃

```
2016-07-31 21:27:40.337 RemoveObject[5915:321407] *** Terminating app due to uncaught exception 'NSGenericException', reason: '*** Collection <__NSArrayM: 0x7f9388c95580> was mutated while being enumerated.'
```

不要在for in 循环中删除数组内部对象。

#### 2.4 正确但别扭的做法

删除数组中所有的@"remove"

```objc
- (void)removeObjectsReversed
{
    NSMutableArray *contents = [@[@"how", @"to", @"remove", @"remove", @"object"] mutableCopy];
    for (NSInteger i = contents.count - 1; i >= 0; --i) {
        NSString *var = contents[i];
        if ([var isEqualToString:@"remove"]) {
            [contents removeObjectAtIndex:i];
        }
    }
    
    NSLog(@"%@", contents);
}
```

输出：

```
2016-07-31 21:31:37.655 RemoveObject[5934:325316] (
    how,
    to,
    object
)
```

倒序删除，正确但有点别扭！

#### 2.5 优雅的做法

```objc
- (void)removeObjectsUseEnumration
{
    NSMutableArray *contents = [@[@"how", @"remove", @"to", @"remove", @"object"] mutableCopy];
    NSIndexSet *indexSet =
        [contents indexesOfObjectsPassingTest:^BOOL(NSString *  _Nonnull var, NSUInteger idx, BOOL * _Nonnull stop) {
            return [var isEqualToString:@"remove"];
        }];
    [contents removeObjectsAtIndexes:indexSet];
    
    NSLog(@"%@", indexSet);
    NSLog(@"%@", contents);
}
```

输出：

```
2016-07-31 22:10:42.404 RemoveObject[6014:338210] <NSIndexSet: 0x7fb73a516040>[number of indexes: 2 (in 2 ranges), indexes: (1 3)]
2016-07-31 22:10:42.404 RemoveObject[6014:338210] (
    how,
    to,
    object
)
```

先通过`indexesOfObjectsPassingTest:`把待删除对象的index找出来，再调用`removeObjectsAtIndexes:`进行一次性删除。

### 3 总结

1. 不建议在NSMutableArray循环中使用`removeObject:`删除该NSMutableArray内部对象，此举可能引发误删，如2.1所示；
2. 不建议在NSMutableArray的for in 循环中删除对象，此举可能引发崩溃，如2.3所示；
3. 建议删除NSMutableArray内部对象时，先拿到待删对象的index，再进行一次性删除，如2.5所示。




