---
layout: post
title: NSMutableArray 排序以及NSComparator和NSSortOptions的解释说明
date: 2016-12-11
categories: iOS
---
最近为 APP 做一个数据库版本升级模块，用到了数组排序吗，同时由于特殊需求，要求使用稳定的排序算法。

这种需求在商城类 APP 上对商品排序也经常出现。问了度娘，没有看到非常好的答案，现在把自己研究的记录一下。

NSMutableArray 排序通常使用

`- (void)sortUsingComparator:(NSComparator)cmptr;`

有时也需要用到

`- (void)sortWithOptions:(NSSortOptions)opts usingComparator:(NSComparator)cmptr;`

其余的例如:

`- (void)sortUsingFunction:(NSInteger (*)(ObjectType,  ObjectType, void * __nullable))compare context:(nullable void *)context;`

`- (void)sortUsingSelector:(SEL)comparator;`

没有用过,暂时也不研究怎么用

先来看一个例子:
```
NSMutableArray* testArray = [@[@1,@4,@3,@2] mutableCopy];
    [testArray sortUsingComparator:^NSComparisonResult(id  _Nonnull obj1, id  _Nonnull obj2) {
        if ([obj1 intValue] < [obj2 intValue]) {
            return NSOrderedAscending;
        }else if ([obj1 intValue] > [obj2 intValue]){
            return NSOrderedDescending;
        }else{
            return NSOrderedSame;
        }
    }];
    NSLog(@"%@",testArray);
```
上述代码运行结果
(
    1,
    2,
    3,
    4
)

NSOrderedAscending 表示 obj1 应该排在 obj2 前面
NSOrderedDescending 表示 obj2 应该排在 obj1 前面
NSOrderedSame 表示 obj1 和 obj2 是一样的
在 Foundation 中，有些类已经实现了默认的排序方法，例如 NSString

`- (NSComparisonResult)compare:(NSString *)string;`

按照 ASCII 表排序

再来看一个例子:
```
NSMutableArray *array = [NSMutableArray arrayWithObjects:@"one", @"two", @"three", @"four", nil];
[array sortWithOptions:0 usingComparator:^NSComparisonResult(id obj1, id obj2) {
    if ( [obj1 length] < [obj2 length] )
        return NSOrderedAscending;
    if ( [obj1 length] > [obj2 length] )
        return NSOrderedDescending;
    return NSOrderedSame;
}];
```

有一个参数是 NSSortOptions 这是一个 NS_OPTIONS 枚举值如下:

```
typedef NS_OPTIONS(NSUInteger, NSSortOptions) {
    NSSortConcurrent = (1UL << 0),
    NSSortStable = (1UL << 4),
};
```

文档上给的解释

**NSSortConcurrent**
**Specifies that the Block sort operation should be concurrent.**
**This option is a hint and may be ignored by the implementation under some circumstances; the code of the Block must be safe against concurrent invocation.**

**NSSortStable**
**Specifies that the sorted results should return compared items have equal value in the order they occurred originally.**
**If this option is unspecified equal objects may, or may not, be returned in their original order.**

我们都知道,排序算法有稳定性,并且,稳定的排序算法时间复杂度都比较高,那么文档上的解释应该这样理解:
NSSortConcurrent 是高效的但不稳定的排序算法,例如:快速排序
NSSortStable 是稳定的排序算法,例如:冒泡排序 插入排序

所以上面的代码:
如果使用 NSSortStable 正确的结果是 @"one", @"two", @"four", @"three"
如果使用 NSSortConcurrent 正确的结果是 @"one", @"two", @"four", @"three" 或者 @"two", @"one", @"four", @"three"

那么 NSSortStable 这个选项就容易选了.没有特殊要求就使用 NSSortConcurrent 速度快,有特殊要求就使用 NSSortStable.

参考:
http://stackoverflow.com/questions/9794957/documentation-for-nssortstable-is-ungrammatical-what-is-i