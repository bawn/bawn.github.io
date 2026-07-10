---
layout: post
title: "NSPredicate"
date: 2014-05-07
comments: true
categories: iOS
tags: [iOS]
keywords: NSPredicate filtering
description: NSPredicate data filtering
---
In day-to-day development, everyone will more or less encounter data filtering, and that inevitably involves `NSPredicate`.

This class is just as powerful as [valueForKeyPath](http://bawn.github.io/2014/05/07/valueForKeyPath/) mentioned in my previous post. Its use mainly centers around two methods:

NSArray

```objc
- (NSArray *)filteredArrayUsingPredicate:(NSPredicate *)predicate;
```

NSMutableArray

```objc
- (void)filterUsingPredicate:(NSPredicate *)predicate;
```

`NSSet` and `NSMutableSet` can also be filtered with this class.
Below I will go through the usage of this class one by one. After reading this, I think you will agree with me that this class is really powerful.

>## Filtering Usage

### Using Instance Methods on Elements

* Filter strings whose length is greater than 3

```objc
NSArray *array = @[@"jim", @"cook", @"jobs", @"sdevm"];
NSPredicate *pre = [NSPredicate predicateWithFormat:@"length > 3"];
NSLog(@"%@", [array filteredArrayUsingPredicate:pre]);
```

Output

```
(
    cook,
    jobs,
    sdevm
)
```

`length` means calling `[xxxx length]` on each array element and then checking whether the returned `NSUInteger` value is greater than 3.

* Other NSString methods such as `integerValue`


```objc
NSArray *array = @[@"2", @"3", @"4", @"5"];
NSPredicate *pre = [NSPredicate predicateWithFormat:@"integerValue >= %@", @3];
NSLog(@"%@", [array filteredArrayUsingPredicate:pre]);
```

If you do not want to use any instance methods and want to filter on the element itself, you can use `self` instead.

```objc
NSPredicate *pre = [NSPredicate predicateWithFormat:@"self CONTAINS %@", @"3"];
```

The usage of `CONTAINS` will be discussed later.


### Extending to Models

```objc
@interface Test : NSObject
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSNumber *code;
@end
```
```objc
Test *test1 = [[Test alloc]init];
test1.name = @"West Lake";
test1.code = @1;

Test *test2 = [[Test alloc]init];
test2.name = @"Xixi Wetland";
test2.code = @2;

Test *test3 = [[Test alloc]init];
test3.name = @"Lingyin Temple";
test3.code = @3;

NSArray *array = @[test1, test2, test3];
```
Filter the array elements whose `[test code]` method (the getter for the `code` property) returns a value greater than or equal to 2. The comparison operators `==`, `!=`, `<=`, and `<` can also be used here.
```objc
NSPredicate *pre = [NSPredicate predicateWithFormat:@"code >= %@", @2];
NSLog(@"%@", [array filteredArrayUsingPredicate:pre]);
```

Actually, `==` can be used not only to compare `NSNumber` objects, but also to determine whether `NSString` objects are equal.
```objc
NSPredicate *pre = [NSPredicate predicateWithFormat:@"name == %@", @"West Lake"];
```
This filters the object array whose `name` is "West Lake".

#### Operations on NSString Objects

As mentioned above, the `==` comparison operator can achieve the same effect as `- (BOOL)isEqualToString:(NSString *)aString;` when determining whether strings are equal. So how do you check whether a string contains another string? In NSPredicate, you can use `CONTAINS` (case-insensitive in terms of keyword spelling) to represent containment.
```objc
 NSPredicate *pre = [NSPredicate predicateWithFormat:@"name CONTAINS %@", @"Lake"];
```
To ignore case when matching, you can use `[cd]`:
>[c] Ignore case
[d] Ignore diacritic marks
[cd] Ignore both case and diacritics.

Usage:
```objc
NSPredicate *pre = [NSPredicate predicateWithFormat:@"name CONTAINS[cd] %@", @"abc"];
```
For more complex queries, such as checking whether a string begins or ends with another string, or using wildcards:

**BEGINSWITH (begins with a certain string)**

```objc

NSString *targetString = @"h";
NSPredicate *pre = [NSPredicate predicateWithFormat:@"name BEGINSWITH
%@",targetString];

```

**ENDSWITH (ends with a certain string)**

```objc
NSString *targetString = @"ing";
NSPredicate *pre = [NSPredicate predicateWithFormat:@"name ENDSWITH %@",targetString];
```

**Wildcard LIKE**
>`*` represents one or more characters or an empty match, and `?` represents a single character.

```objc
Test *test1 = [[Test alloc]init];
test1.name = @"absr";
test1.code = @1;

Test *test2 = [[Test alloc]init];
test2.name = @"asb";
test2.code = @2;

Test *test3 = [[Test alloc]init];
test3.name = @"raskj";
test3.code = @3;

NSArray *array = @[test1, test2, test3];
```

```objc
NSPredicate *pre = [NSPredicate predicateWithFormat:@"name LIKE %@", @"?b*"];
NSLog(@"%@", [array filteredArrayUsingPredicate:pre]);
```
Result: only `test1` matches. `LIKE` can also accept `[cd]`, for example:
```objc
NSPredicate *pre = [NSPredicate predicateWithFormat:@"name LIKE[cd] %@", @"?b*"];
```

#### Relational Operators, Including IN, BETWEEN, AND, OR, and NOT

**IN**

```objc
NSPredicate *pre = [NSPredicate predicateWithFormat:@"code IN %@", @[@1, @3]];
```

Check whether `code` is @1 or @3, that is, whether it is in the array.

**OR**

`OR` can be used instead of `IN` to achieve the same effect, but `OR` is more flexible.

```objc
NSPredicate *pre = [NSPredicate predicateWithFormat:@"code == %@ OR code == %@ ", @1, @3];
```
The effect is the same as `IN`, but `OR` can test more than one property.

```objc
NSPredicate *pred = [NSPredicate predicateWithFormat:@"code == %@ OR name == %@ ", @1, @"asb"];
```

**BETWEEN**

Usually used when checking `NSNumber` objects.

```objc
NSPredicate *pred = [NSPredicate predicateWithFormat:@"code BETWEEN {1, 3}"];
```

Check whether `code` is >= 1 and <= 3.


**AND**

```objc
NSPredicate *pred = [NSPredicate predicateWithFormat:@"code >= %@ AND code <=%@", @1, @3];
```

**NOT**

The most common use of `NOT` is to remove the values of one array from another array. That may sound a bit confusing, but an example will make it clear.

```objc
NSArray *arrayFilter = @[@"abc1", @"abc2"];
NSArray *arrayContent = @[@"a1", @"abc1", @"abc4", @"abc2"];
NSPredicate *thePredicate = [NSPredicate predicateWithFormat:@"NOT (SELF in %@)", arrayFilter];
NSLog(@"%@",[arrayContent filteredArrayUsingPredicate:thePredicate]);
```

Output

```
(
    a1,
    abc4
)
```
Compared with looping through and adding items to a new array, this is much simpler.


The examples above all use `+ (NSPredicate *)predicateWithFormat:(NSString *)predicateFormat, ...;` to create predicates. There is also another commonly used method: `+ (NSPredicate*)predicateWithBlock:(BOOL (^)(id evaluatedObject, NSDictionary *bindings))block`, which creates a predicate using a block.

```objc
NSPredicate *pre = [NSPredicate predicateWithBlock:^BOOL(id evaluatedObject, NSDictionary *bindings) {
    Test *test = (Test *)evaluatedObject;
    if (test.code.integerValue > 2) {
        return YES;
    }
    else{
        return NO;
    }
}];
```

The parameter `evaluatedObject` represents an array element, and the block must return YES or NO to indicate a match or no match. Please ignore the `bindings` parameter; I have not figured out its exact purpose either.

### Multiple Filtering

If you need to filter by multiple properties, chaining conditions with `AND` or `OR` can be somewhat cumbersome. The `NSCompoundPredicate` class meets this need, because it can combine multiple `NSPredicate` objects, using either `AND` or `OR`.

```objc
	NSPredicate *pre1 = [NSPredicate predicateWithFormat:@"code >= %@", @3];
    NSPredicate *pre2 = [NSPredicate predicateWithFormat:@"code <= %@", @2];
    // Combine with AND
    NSPredicate *pre = [NSCompoundPredicate andPredicateWithSubpredicates:@[pre1,pre2]];
    // Combine with OR
    NSPredicate *pre = [NSCompoundPredicate orPredicateWithSubpredicates:@[pre1, pre2]];
```

___

## Matching Usage

Actually, NSPredicate can be used not only for filtering, but also to determine whether an object matches and return the result directly. The main method is `- (BOOL)evaluateWithObject:(id)object;`, used like this:

```objc
Test *test1 = [[Test alloc]init];
test1.name = @"absr";
test1.code = @1;

NSPredicate *pres = [NSPredicate predicateWithFormat:@"code == %@", @2];
BOOL match = [pres evaluateWithObject:test1];
```

Of course, the most common use is together with regular expressions. Here are a few common regex examples.

Whether it starts with a and ends with e

```objc
NSString *string=@"assdbfe";
NSString *targetString=@"^a.+e$";
NSPredicate *pres = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", targetString];
BOOL match = [pres evaluateWithObject:string];
```

Whether it is an email address

```objc
NSString *strRegex = @"[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{1,5}";
```

Whether it is a mobile phone number (including the 181, 1700, 1709, and 1705 prefixes)

```objc
+ (BOOL)isMobileNumber:(NSString *)mobileNum
{
    /**
    *  China Mobile
     *
     *  134[0-8],135,136,137,138,139,147,150,151,157,158,159,182,183,187,188,1705
     *
     */
    NSString * CM = @"^1((34[0-8]|(3[5-9]|5[017-9]|8[2378])|47\\d)|705)\\d{7}$";
    /**
    *  China Unicom
     *
     *  130,131,132,152,155,156,185,186,1709
     *
     */
    NSString * CU = @"^1((3[0-2]|5[256]|8[56])[0-9]|709)\\d{7}$";
    /**
    *  China Telecom
     *
     *  133,1349,153,180,189,181,177,1700
     *
     */
    NSString * CT = @"^1((33|53|77|8[019])[0-9]|349|700)\\d{7}$";
    NSPredicate *regextestcm = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", CM];
    NSPredicate *regextestcu = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", CU];
    NSPredicate *regextestct = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", CT];
    if (
        ([regextestcm evaluateWithObject:mobileNum] == YES)
        || ([regextestcu evaluateWithObject:mobileNum] == YES)
        || ([regextestct evaluateWithObject:mobileNum] == YES)
        ){
        return YES;
    }
    else{
        return NO;
    }
}
```

____

## Pitfalls


In some cases, strings like `code` in the example above are not very explicit, so you might create it like this:

```objc
    Test *test1 = [[Test alloc]init];
    test1.name = @"absr";
    test1.code = @1;

    Test *test2 = [[Test alloc]init];
    test2.name = @"asb";
    test2.code = @2;

    Test *test3 = [[Test alloc]init];
    test3.name = @"raskj";
    test3.code = @3;

    NSArray *array = @[test1, test2, test3];

    NSPredicate *pre = [NSPredicate predicateWithFormat:@"%@ == %@", @"code", @2];
    NSLog(@"%@", [array filteredArrayUsingPredicate:pre]);
```
Pay attention to the initialization style of the NSPredicate object. After execution, you will find that it is wrong. Printing `pre` shows `"code" == 2`, which means it is looking for the return value of the `"code"` method (note: with double quotes), which clearly will not work.

Solution:

```objc
NSPredicate *pre = [NSPredicate predicateWithFormat:@"%K == %@", @"code", @2];
```
Note: the `K` in `%K` must be uppercase.



Also note the handling of empty strings. `Person` is a class with a `name` property:

```objc
    Person *person = [[Person alloc] init];
    NSArray *array = @[person];
    NSPredicate *pre = [NSPredicate predicateWithFormat:@"name.length == 0"];
    NSLog(@"%@", [array filteredArrayUsingPredicate:pre]);
```

Using `name.length == 0` as a filter condition will inexplicably fail, while `name == nil` does work. You can also use:

```objc
NSPredicate *pre = [NSPredicate predicateWithBlock:^BOOL(Person *_Nullable evaluatedObject, NSDictionary<NSString *,id> * _Nullable bindings) {
        return [evaluatedObject.name length] == 0;
    }];
```

____

## Finally
NSPredicate can satisfy almost any form of query, and it is of course perfect for Core Data database queries. NSPredicate has even more uses than the ones covered here. If you are interested, you can read this [post](http://nshipster.com/nspredicate/) on nshipster, which covers some usage patterns I did not mention.
