---
layout: post
title: "valueForKeyPath"
date: 2014-05-07
comments: true
categories: iOS
tags: [iOS]
keywords: iOS KVC valueForKeyPath
description: The power of the valueForKeyPath method
---
Perhaps many people do not use `- (id)valueForKeyPath:(NSString *)keyPath` very often in day-to-day development.

However, this method is very powerful. For example:

```objc
NSArray *array = @[@"name", @"w", @"aa", @"jimsa"];
NSLog(@"%@", [array valueForKeyPath:@"uppercaseString"]);
```

Output

```
(
    NAME,
    W,
    AA,
    JIMSA
)
```

This is equivalent to calling `uppercaseString` on each element in the array, then returning the results as a new array. Since `uppercaseString` works, other NSString methods work too, for example:

```
[array valueForKeyPath:@"length"]
```

This returns an array containing the length of each string. Any instance method you can think of can be used this way.

If you think this method only does that much, you are wrong. Let's look at some concrete examples.

### Quickly Calculating Sum, Average, Maximum, and Minimum for an NSArray of NSNumbers

```
NSArray *array = @[@1, @2, @3, @4, @10];
NSNumber *sum = [array valueForKeyPath:@"@sum.self"];
NSNumber *avg = [array valueForKeyPath:@"@avg.self"];
NSNumber *max = [array valueForKeyPath:@"@max.self"];
NSNumber *min = [array valueForKeyPath:@"@min.self"];
```

Or specify the output type:

```
NSNumber *sum = [array valueForKeyPath:@"@sum.floatValue"];
NSNumber *avg = [array valueForKeyPath:@"@avg.floatValue"];
NSNumber *max = [array valueForKeyPath:@"@max.floatValue"];
NSNumber *min = [array valueForKeyPath:@"@min.floatValue"];
```

One thing to note: the collection operators `@sum` and `@avg` first convert objects to `double` values,
and then output an `NSNumber` containing a `double`, so do not make this mistake:

```
NSArray *arr = @[@(0),@(10),@(40)];
NSInteger avg = [[arr valueForKeyPath:@"@avg.self"] integerValue];
NSLog(@"---%ld----",avg);
```
After calling `integerValue`, the result becomes 0 directly.



### Removing Duplicate Data

```objc
NSArray *array = @[@"name", @"w", @"aa", @"jimsa", @"aa"];
NSLog(@"%@", [array valueForKeyPath:@"@distinctUnionOfObjects.self"]);
```
```
(
    name,
    w,
    jimsa,
    aa
)
```

### Quick Value Extraction

```objc
    NSArray *array = @[@{@"name" : @"cookeee",
                         @"code" : @1},
                       @{@"name" : @"sswwre",
                         @"code" : @2}];
    NSLog(@"%@", [array valueForKeyPath:@"name"]);
```

This directly returns an array containing the values for the `name` key in each dictionary. Clearly, this is more convenient and faster than looping through the values and adding them to a new array.

```
(
    cookeee,
    jim,
    jim,
    jbos
)
```

### Nested Usage

Remove duplicate values from the `name` field and then extract the values.

```objc

NSArray *array = @[@{@"name" : @"cookeee",@"code" : @1},
                    @{@"name": @"jim",@"code" : @2},
                    @{@"name": @"jim",@"code" : @1},
                    @{@"name": @"jbos",@"code" : @1}];

NSLog(@"%@", [array valueForKeyPath:@"@distinctUnionOfObjects.name"]);
```

```
(
    cookeee,
    jim,
    jbos
)
```

Get the maximum value of the `number` field.

```
NSArray *array = @[@{@"number" : @1, @"title" : @""},
                   @{@"number" : @2, @"title" : @""},
                   @{@"number" : @20, @"title" : @""},
                   @{@"number" : @4, @"title" : @""},
                   @{@"number" : @5, @"title" : @""}
];
NSNumber *max = [array valueForKeyPath:@"@max.number"];
```

### Changing the Placeholder Color of a UITextField

```
    [searchField setValue:[UIColor whiteColor] forKeyPath:@"_placeholderLabel.textColor"];
```

This is much more convenient than overriding `- (void)drawPlaceholderInRect:(CGRect)rect;`.



### Practical Example

```objc
NSArray *array = @[
                   @[@1, @2],
                   @[@1, @2, @3, @4],
                   @[@1, @2, @3]
                   ];
NSNumber *count = [[array valueForKeyPath:@"@unionOfObjects.@count"] valueForKeyPath:@"@sum.self"];
```

Calculate the total number of elements in a nested array. For example, this returns 9.
