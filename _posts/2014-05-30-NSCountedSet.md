---
layout: post
title: "NSCountedSet"
date: 2014-05-30
comments: true
categories: iOS
tags: [iOS]
published: true
keywords: NSCountedSet
description: Counting duplicate objects in an array
---

Someone in a group chat previously discussed how to calculate slot machine prize levels: the machine has four columns, and each column has four symbols. If all four are the same, that is level 1; if three are the same, that is level 2; and so on.

Using `if else` was said to be too cumbersome, so the question was whether there was a faster and more efficient method. The first thing I thought of was using `KVC` to remove duplicate data. For example:

```
NSArray *array = @[@2, @2, @2, @1];
NSArray *result = [array valueForKeyPath:@"@distinctUnionOfObjects.self"];
NSLog(@"%@", result);
```

This gives an array count of 2, which happens to be level 2. It looks reasonable. But if the array is `NSArray *array = @[@2, @1, @2, @1];`, the calculated array count is also 2, while the correct result should be level 3. Later I remembered a class that is used very rarely: `NSCountedSet`.

```
NSArray *array = @[@1, @2, @2, @1];
NSCountedSet *set = [[NSCountedSet alloc]initWithArray:array];

[set enumerateObjectsUsingBlock:^(id obj, BOOL *stop) {
	NSLog(@"%@ => %d", obj, [set countForObject:obj]);
}];
```

Output

```
2014-05-29 00:35:22.741 CoreData[3235:60b] 1 => 2
2014-05-29 00:35:22.742 CoreData[3235:60b] 2 => 2
```

You may notice that this class's superclass is `NSMutableSet`. Huh? Isn't `NSMutableSet` supposed to be unable to store duplicate objects? In fact, `NSCountedSet` also cannot store duplicate objects. Apple's documentation describes this class with the following sentence:

> Each distinct object inserted into an NSCountedSet object has a counter associated with it.
Each distinct object inserted into an NSCountedSet object has an associated counter (Google translation).

In other words, when a duplicate object is added, that object's counter is incremented by 1. So this class provides a method called `- (NSUInteger)countForObject:(id)object;` to count duplicate objects.
