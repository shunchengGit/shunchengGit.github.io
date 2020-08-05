---
layout: post_layout
title: iOS Nimbus框架tableview部分解读(CSDN早期博客)
date: 2014-09-26 00:00:00
location: 北京
pulished: true
abstract: iOS Nimbus框架tableview部分解读。
---

### 一、引言

项目中很早就引入了Nimbus框架，一直在搞天气动画，没时间看这个东西。最近要做新的feature，跟table相关。于是花了一下午读了读相关部分的源码。Git上是这么介绍Nimbus的——The iOS framework that grows only as fast as its documentation.一看文档和注释，果然不错，读起来一气呵成，很舒服。

### 二、整体思路

众所周知，生成一个UITableView,一般要设置两个东西：dataSource和delegate。其中，dataSource来控制tableview上展示的内容，而delegate用来通知tableview上发生的各种事件，如cell被点击等等。通常我们把这两个委托设置为UITableViewController，让UITableViewController同时实现UITableViewDataSource和UITableViewDelegate protocol。而 Nimbus根据这两个特性造了两个类分别是NITableViewModel（实现UITableViewDataSource，用来设置数据源dataSource）和NITableViewActions（实现UITableViewDelegate，触发tableview上的各种事件）。

```objc
//testTableView is a UITableView
//_model is a NITableViewModel instance
//_actions is a NITableViewActions instance
self.testTableView.delegate = _actions;
self.testTableView.dataSource = _model;
```

### 三、从数据源到具体Cell

我们先来看看怎么从数据源生成具体的Cell。Nimbus把该过程分为2部分：数据管理和Cell生成。其中数据管理由NITableViewModel实现，而Cell生成是实现NITableViewModelDelegate protocol，可能有多种方式。

#### 1数据管理

首先来看看数据管理，NITableViewModel定义。

```objc
@interface NITableViewModel : NSObject <UITableViewDataSource>
    - (id)initWithDelegate:(id<NITableViewModelDelegate>)delegate;
    - (id)initWithListArray:(NSArray *)sectionedArray delegate:(id<NITableViewModelDelegate>)delegate;
    - (id)initWithSectionedArray:(NSArray *)sectionedArray delegate:(id<NITableViewModelDelegate>)delegate;
@end
```

其中NITableViewModelDelegate，是指定Cell的生成委托，对应第二步，我们先忽略。
其中sectionedArray是指定的数据内容。

```objc
/*
 * @code
 * NSArray* contents =
 * [NSArray arrayWithObjects:
 *  @"Section 1",
 *  [NSDictionary dictionaryWithObject:@"Row 1" forKey:@"title"],
 *  [NSDictionary dictionaryWithObject:@"Row 2" forKey:@"title"],
 *  @"Section 2",
 *  // This section is empty.
 *  @"Section 3",
 *  [NSDictionary dictionaryWithObject:@"Row 3" forKey:@"title"],
 *  [NITableViewModelFooter footerWithTitle:@"Footer"],
 *  nil];
 * [[NIStaticTableViewModel alloc] initWithSectionedArray:contents delegate:self];
 */
```

其中调用方式可能如上所示，如果sectionedArray里面的一项是NSString则认为这是一个section头部（标题），如果是一个NITableViewModelFooter对象则认为是一个section的尾部。其余每一项则认为是一个Cell上的内容。如上例，该sectionedArray展示了3个section，第一个section的标题是@"Section 1",内部有两个cell，具体内容分别由[NSDictionary dictionaryWithObject:@"Row 1" forKey:@"title"]和[NSDictionary dictionaryWithObject:@"Row 2" forKey:@"title"]表示。如此跟table视觉上的排布一致。得到这个sectionedArray后，NITableViewModel对该数组进行一顿解析，在内部用一些更加友好的数据管理起来。我们可以先把NITableViewModel简单的理解为：解析sectionedArray数组，了解里面有哪些section哪些cell，在UITableViewDataSource相应的函数里面返回解析过后的数值。

#### 2Cell生成

通过上文我们知道，NITableViewModel实现了UITableViewDataSource protocol，该protocol有个一函数用来生成cell的

`- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;`

NITableViewModel职责有限，只能管理数据源不管你生成Cell，所以他把这个东西抛给他的委托，NITableViewModelDelegate，让这个委托来生成Cell。我们前文提到生成Cell有多种方式，这些方式都是通过实现NITableViewModelDelegate得来的。
UITableViewController实现NITableViewModelDelegate

```objc
- (UITableViewCell *)tableViewModel: (NITableViewModel *)tableViewModel
                   cellForTableView: (UITableView *)tableView
                        atIndexPath: (NSIndexPath *)indexPath
                         withObject: (id)object
{
    id object = [tableViewModel objectAtIndexPath:indexPath];
    //从数据源获取cell上的数据
    //object就是该cell对应的数据，也就是你在model的sectionedArray里面设置的东西
    //如上例子，若indexPath为section0 row0则object是[NSDictionary dictionaryWithObject:@"Row 1" forKey:@"title"]
    //得到这些信息后，我们再通过平时的策略来生成cell
}
```

利用NICellFactory实现NITableViewModelDelegate

若利用此种方式实现Cell生成，则必须要求第一步sectionedArray表示cell内容的对象是一个NICellObject对象。其中NICellObject类定义如下：

```objc
@protocol NICellObject <NSObject>
@required
/** The class of cell to be created when this object is passed to the cell factory. */
- (Class)cellClass;
@optional
/** The style of UITableViewCell to be used when initializing the cell for the first time. */
- (UITableViewCellStyle)cellStyle;
@end

@interface NICellObject : NSObject <NICellObject>
// Designated initializer.
- (id)initWithCellClass:(Class)cellClass userInfo:(id)userInfo;
- (id)initWithCellClass:(Class)cellClass;
+ (id)objectWithCellClass:(Class)cellClass userInfo:(id)userInfo;
+ (id)objectWithCellClass:(Class)cellClass;
@property (nonatomic, strong) id userInfo;
@end
```

通过类定义，我们不难发现，NICellObject实际就是一个带cellClass信息的类，也就是说NICellFactory要求我们传到model的sectionedArray里面的cell内容对象都是带cell class信息的。通过改造，我们可以把indexPath为section0 row0的object变为[NICellObject objectWithCellClass:[UITableViewCell class] userInfo:[NSDictionary dictionaryWithObject:@"Row 1" forKey:@"title"] ];
具体生成函数如下所示：

```objc
+ (UITableViewCell *)tableViewModel:(NITableViewModel *)tableViewModel
               cellForTableView:(UITableView *)tableView
                    atIndexPath:(NSIndexPath *)indexPath
                     withObject:(id)object {
  UITableViewCell* cell = nil;

  // Only NICellObject-conformant objects may pass.
  if ([object respondsToSelector:@selector(cellClass)]) {
    //1拿cellClass
    Class cellClass = [object cellClass];
    //2根据cellClass来生成cell
    cell = [self cellWithClass:cellClass tableView:tableView object:object];

  } else if ([object respondsToSelector:@selector(cellNib)]) {
    UINib* nib = [object cellNib];
    cell = [self cellWithNib:nib tableView:tableView indexPath:indexPath object:object];
  }

  // If this assertion fires then your app is about to crash. You need to either add an explicit
  // binding in a NICellFactory object or implement the NICellObject protocol on this object and
  // return a cell class.
  NIDASSERT(nil != cell);

  return cell;
}

+ (UITableViewCell *)cellWithClass:(Class)cellClass
                     tableView:(UITableView *)tableView
                        object:(id)object {
  UITableViewCell* cell = nil;

  NSString* identifier = NSStringFromClass(cellClass);

  if ([cellClass respondsToSelector:@selector(shouldAppendObjectClassToReuseIdentifier)]
      && [cellClass shouldAppendObjectClassToReuseIdentifier]) {
    identifier = [identifier stringByAppendingFormat:@".%@", NSStringFromClass([object class])];
  }

  cell = [tableView dequeueReusableCellWithIdentifier:identifier];

  if (nil == cell) {
    UITableViewCellStyle style = UITableViewCellStyleDefault;
    if ([object respondsToSelector:@selector(cellStyle)]) {
      style = [object cellStyle];
    }
    cell = [[cellClass alloc] initWithStyle:style reuseIdentifier:identifier];
  }

  // Allow the cell to configure itself with the object's information.
  // 调用cell类的shouldAppendObjectClassToReuseIdentifier函数，给自定义的cell一个改变自己的机会
  if ([cell respondsToSelector:@selector(shouldUpdateCellWithObject:)]) {
    [(id<NICell>)cell shouldUpdateCellWithObject:object];
  }

  return cell;
}
```

Cell一气呵成

```objc
//自定义的cell
@interface TestCell : UITableViewCell <NICell>
@end

@interface TestCell ()
@property (nonatomic, weak) UILabel* lbl;
@end

@implementation TestCell
- (BOOL)shouldUpdateCellWithObject:(id)object
{
    if (!_lbl) {
        UILabel* tmp = [[UILabel alloc] initWithFrame:self.bounds];
        self.lbl = tmp;
        [self.contentView addSubview:_lbl];
    }
    NSDictionary* data = ((NICellObject*)object).userInfo;
    _lbl.text = data[@"title"];
    return YES;
}
@end

//一气呵成生成cell的相关代码
UITableView* tmpView = [[UITableView alloc] initWithFrame:self.view.bounds style:UITableViewStyleGrouped];
self.testView = tmpView;
[self.view addSubview:_testView];
NSArray* contentAry = @[ [NICellObject objectWithCellClass:[TestCell class] userInfo:@{ @"title" : @"hello" }],
                         [NICellObject objectWithCellClass:[TestCell class] userInfo:@{ @"title" : @"nimbus" }],
                         [NICellObject objectWithCellClass:[TestCell class] userInfo:@{ @"title" : @"what" }],
                         [NICellObject objectWithCellClass:[TestCell class] userInfo:@{ @"title" : @"are" }] ];

self.model = [[NITableViewModel alloc] initWithSectionedArray:contentAry
                                                     delegate:(id)[NICellFactory class]];
self.testView.dataSource = _model;
```

### 四、tableview事件处理

说完了数据源展示，我们在来探索一下table事件处理。Nimbus用到了NITableViewActions类，该类接口定义如下：

```objc
@interface NITableViewActions : NIActions <UITableViewDelegate>

#pragma mark Forwarding

- (id<UITableViewDelegate>)forwardingTo:(id<UITableViewDelegate>)forwardDelegate;
- (void)removeForwarding:(id<UITableViewDelegate>)forwardDelegate;

#pragma mark Object state

- (UITableViewCellAccessoryType)accessoryTypeForObject:(id)object;
- (UITableViewCellSelectionStyle)selectionStyleForObject:(id)object;

#pragma mark Configurable Properties

@property (nonatomic, assign) UITableViewCellSelectionStyle tableViewCellSelectionStyle;

@end
```

可以看出该类实现了UITableViewDelegate protocol，因此可以当作tableview的事件处理类。NITableViewActions继承自NIActions类，该类又定义如下

```objc
@interface NIActions : NSObject

// Designated initializer.
- (id)initWithTarget:(id)target;

#pragma mark Mapping Objects

- (id)attachToObject:(id<NSObject>)object tapBlock:(NIActionBlock)action;
- (id)attachToObject:(id<NSObject>)object detailBlock:(NIActionBlock)action;
- (id)attachToObject:(id<NSObject>)object navigationBlock:(NIActionBlock)action;

- (id)attachToObject:(id<NSObject>)object tapSelector:(SEL)selector;
- (id)attachToObject:(id<NSObject>)object detailSelector:(SEL)selector;
- (id)attachToObject:(id<NSObject>)object navigationSelector:(SEL)selector;

#pragma mark Mapping Classes

- (void)attachToClass:(Class)aClass tapBlock:(NIActionBlock)action;
- (void)attachToClass:(Class)aClass detailBlock:(NIActionBlock)action;
- (void)attachToClass:(Class)aClass navigationBlock:(NIActionBlock)action;

- (void)attachToClass:(Class)aClass tapSelector:(SEL)selector;
- (void)attachToClass:(Class)aClass detailSelector:(SEL)selector;
- (void)attachToClass:(Class)aClass navigationSelector:(SEL)selector;

#pragma mark Object State

- (BOOL)isObjectActionable:(id<NSObject>)object;

+ (id)objectFromKeyClass:(Class)keyClass map:(NSMutableDictionary *)map;

@end
```

从类接口可以直观的看出，该类是把block块或者selector绑定到一个对象上。应该是在内部实际维护了从对象到block块或者selector的关系。

```objc
@interface NIActions ()

@property (nonatomic, strong) NSMutableDictionary* objectToAction;
@property (nonatomic, strong) NSMutableDictionary* classToAction;
@property (nonatomic, strong) NSMutableSet* objectSet;

@end
```

再从这个类的extention，果断看出了端倪，证明我们的猜测不错。
我们来看一个attach方法

```objc
- (id)attachToObject:(id<NSObject>)object tapBlock:(NIActionBlock)action {
  [self.objectSet addObject:object];
  //获取该object对应的action数据结构，并且设置该数据结构对应项的值
  [self actionForObject:object].tapAction = action;
  return object;
}

- (NIObjectActions *)actionForObject:(id<NSObject>)object {
  //获取object的hash key
  id key = [self keyForObject:object];
  //通过object hash key在内部dic中查询NIObjectActions对象，如果没有找到，则新建一个
  NIObjectActions* action = [self.objectToAction objectForKey:key];
  if (nil == action) {
    action = [[NIObjectActions alloc] init];
    [self.objectToAction setObject:action forKey:key];
  }
  return action;
}

@interface NIObjectActions : NSObject

@property (nonatomic, copy) NIActionBlock tapAction;
@property (nonatomic, copy) NIActionBlock detailAction;
@property (nonatomic, copy) NIActionBlock navigateAction;

@property (nonatomic) SEL tapSelector;
@property (nonatomic) SEL detailSelector;
@property (nonatomic) SEL navigateSelector;

@end
```

通过以上函数可以看出NIActions内部用到了NIObjectActions这个数据结构来表示各种action，而NIActions维护了一个从object的hash key到NIObjectActions对象的关系。
我再回过头来看NITableViewActions类的UITableViewDelegate实现，具体函数如下：

```objc
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
  NIDASSERT([tableView.dataSource isKindOfClass:[NITableViewModel class]]);
  if ([tableView.dataSource isKindOfClass:[NITableViewModel class]]) {
    NITableViewModel* model = (NITableViewModel *)tableView.dataSource;
    //从model中获取cell内容对象
    id object = [model objectAtIndexPath:indexPath];

    if ([self isObjectActionable:object]) {
      //查询cell内容对象的action对象（NIObjectActions）
      NIObjectActions* action = [self actionForObjectOrClassOfObject:object];

      //如果action对象中设置了tapAction block ，调用之

      BOOL shouldDeselect = NO;
      if (action.tapAction) {
        // Tap actions can deselect the row if they return YES.
        shouldDeselect = action.tapAction(object, self.target, indexPath);
      }

      //如果action对象中设置了tapSelector，并且目标对象（由NITableViewActions对象保存）能响应tapSelector，调用之
      if (action.tapSelector && [self.target respondsToSelector:action.tapSelector]) {
        //多参数不能直接performSelector
        NSMethodSignature *methodSignature = [self.target methodSignatureForSelector:action.tapSelector];
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
        invocation.selector = action.tapSelector;
        if (methodSignature.numberOfArguments >= 3) {
          [invocation setArgument:&object atIndex:2];
        }
        if (methodSignature.numberOfArguments >= 4) {
          [invocation setArgument:&indexPath atIndex:3];
        }
        [invocation invokeWithTarget:self.target];

        NSUInteger length = invocation.methodSignature.methodReturnLength;
        if (length > 0) {
          char *buffer = (void *)malloc(length);
          memset(buffer, 0, sizeof(char) * length);
          [invocation getReturnValue:buffer];
          for (NSUInteger index = 0; index < length; ++index) {
            if (buffer[index]) {
              shouldDeselect = YES;
              break;
            }
          }
          free(buffer);
        }
      }
      if (shouldDeselect) {
        [tableView deselectRowAtIndexPath:indexPath animated:YES];
      }

      if (action.navigateAction) {
        action.navigateAction(object, self.target, indexPath);
      }
      //直接performSelector
      if (action.navigateSelector && [self.target respondsToSelector:action.navigateSelector]) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        [self.target performSelector:action.navigateSelector withObject:object withObject:indexPath];
#pragma clang diagnostic pop
      }
    }
  }

  // Forward the invocation along.
  for (id<UITableViewDelegate> delegate in self.forwardDelegates) {
    if ([delegate respondsToSelector:_cmd]) {
      [delegate tableView:tableView didSelectRowAtIndexPath:indexPath];
    }
  }
}
```

所以一个table的action可能是这样的

```objc
self.actions = [[NITableViewActions alloc] initWithTarget:self];
self.testView.delegate = _actions;

NSArray* contentAry = @[
    [_actions attachToObject:[NICellObject objectWithCellClass:[TestCell class] userInfo:@{ @"title" : @"hello" }]
                    tapBlock:^BOOL(id object, id target, NSIndexPath* indexPath) {
                        NSLog(@"%@", indexPath);
                        return YES;
                    }],
    [NICellObject objectWithCellClass:[TestCell class] userInfo:@{ @"title" : @"nimbus" }],
    [NICellObject objectWithCellClass:[TestCell class] userInfo:@{ @"title" : @"what" }],
    [NICellObject objectWithCellClass:[TestCell class] userInfo:@{ @"title" : @"are" }]
];

self.model = [[NITableViewModel alloc] initWithSectionedArray:contentAry
                                                     delegate:(id)[NICellFactory class]];
self.testView.dataSource = _model;
```

model里面表示cell内容的对象可以被attach到actions对象上。

