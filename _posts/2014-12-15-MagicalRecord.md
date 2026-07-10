---
layout: post
title: "MagicalRecord"
date: 2014-12-15
comments: true
categories: iOS
tags: [iOS]
published: ture
keywords: MagicalRecord
description: Using MagicalRecord with Mantle
---

![image](/assets/images/MagicalRecord/bang.png)

Before we begin, let's create an entity named MemberManaged.

![image](/assets/images/MagicalRecord/entity.png)

>MemberManaged.h

```
@interface MemberManaged : NSManagedObject
@property (nonatomic, retain) NSString * memberID;
@property (nonatomic, retain) NSString * mobilePhone;
@property (nonatomic, retain) NSDate * createDate;
@property (nonatomic, retain) NSNumber * goldNumber;
@property (nonatomic, retain) NSNumber * age;
@property (nonatomic, retain) NSNumber * isVip;
@property (nonatomic, retain) NSString * url;
```
All of the examples below operate on this entity in the database.

## Quick Start

### Setup

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    [MagicalRecord setupAutoMigratingCoreDataStack];
    // ...
    return YES;
}
- (void)applicationWillTerminate:(UIApplication *)application
{
    [MagicalRecord cleanUp];
}
```

### Fetching Data

```
// Return the first record in the MemberManaged table
MemberManaged *memberManaged = [MemberManaged MR_findFirst];
```

```
// Return all records in the MemberManaged table
NSArray *array = [[MemberManaged MR_findAll];
```

```
// Find by key-value condition and return all matching records
NSArray *array = [MemberManaged MR_findByAttribute:@"memberID" withValue:@"1"];
```

```
// Sort by the specified field
NSArray *array = [MemberManaged MR_findAllSortedBy:@"age" ascending:YES];
```

```
// Use a custom NSPredicate to fetch all matching records
NSPredicate *pre = [NSPredicate predicateWithFormat:@"age > 18"];
MemberManaged *memberManaged = [MemberManaged MR_findAllWithPredicate:pre];
```

If you are not familiar with NSPredicate, you can read my earlier [post](http://bawn.github.io/2014/05/07/NSPredicate/) introducing it. I will not go through the other fetch methods one by one here.

### Inserting Data
```
    [MagicalRecord saveWithBlock:^(NSManagedObjectContext *localContext) {
        MemberManaged *memberManaged = [MemberManaged MR_createInContext:localContext];
        memberManaged.memberID = @"1";
        memberManaged.mobilePhone = @"xxxxxxxx";
        memberManaged.createDate = [NSDate date];
        memberManaged.goldNumber = @2;
        memberManaged.age = @18;
        memberManaged.url = @"http://bawn.github.io/";
        memberManaged.isVip = @YES;

    } completion:^(BOOL success, NSError *error) {
        // ...
    }];
```

### Deleting Data

```
	// Delete a single record
   [MagicalRecord saveWithBlock:^(NSManagedObjectContext *localContext) {
        MemberManaged *member = [MemberManaged MR_findFirstInContext:localContext];
        [member MR_deleteEntity];
    } completion:^(BOOL success, NSError *error) {
        // ...
    }];
```

```
	// Delete all records in the table
    [MagicalRecord saveWithBlock:^(NSManagedObjectContext *localContext) {
        [MemberManaged MR_truncateAllInContext:localContext];
    } completion:^(BOOL success, NSError *error) {
        // ...
    }];
```

### Updating Data
```
    MemberManaged *member = [MemberManaged MR_findFirst];
    [MagicalRecord saveWithBlock:^(NSManagedObjectContext *localContext) {
        MemberManaged *localMember = [member MR_inContext:localContext];
        localMember.age = @22;
    } completion:^(BOOL success, NSError *error) {
        // ...
    }];
```



## Working with Mantle

### Basic Transformation

In my previous [post](http://bawn.github.io/2014/12/11/Mantle/), I mentioned Mantle's [MTLManagedObjectAdapter](https://github.com/Mantle/MTLManagedObjectAdapter) class. In version 2.0, the developers split it out from Mantle into a separate repository. This class uses a protocol called `MTLManagedObjectSerializing`, which has two required methods:

```
// Return the entity class name corresponding to this class
+ (NSString *)managedObjectEntityName;
```

```
// Return the mapping between this class and the entity's properties
+ (NSDictionary *)managedObjectKeysByPropertyKey;
```

In addition, because the `url` property has different types in the `Member` and `MemberManaged` classes, another protocol method is needed to convert `NSURL` to `NSString` (the `age` and `isVip` fields do not need conversion; don't ask me why).

```
// Transform property values
+ (NSValueTransformer *)entityAttributeTransformerForKey:(NSString *)key;
```

Let's look at the implementation.

>Member.h

```
@interface Member : MTLModel<MTLJSONSerializing, MTLManagedObjectSerializing>
@property (nonatomic, retain) NSString   * memberID;
@property (nonatomic, retain) NSString   * mobilePhone;
@property (nonatomic, retain) NSDate     * createDate;
@property (nonatomic, retain) NSNumber   *goldNumber;
@property (nonatomic, assign) NSUInteger age;
@property (nonatomic, assign) BOOL       isVip;
@property (nonatomic, retain) NSURL      *url;
```

>Member.m

```
// Indicates that the entity class corresponding to Member is MemberManaged
+ (NSString *)managedObjectEntityName{
    return @"MemberManaged";
}
```

```
// Indicates the field mapping when converting Member to MemberManaged; this must also be fully specified
+ (NSDictionary *)managedObjectKeysByPropertyKey{
    return @{
             @"memberID" : @"memberID",
             @"mobilePhone" : @"mobilePhone",
             @"createDate" : @"createDate",
             @"goldNumber" : @"goldNumber",
             @"age" : @"age",
             @"isVip" : @"isVip",
             @"url" : @"url"
             };
}
```

Usage:

```
    NSDictionary *dic = @{
						  @"id" : @"2",
                          @"phone" : @"xxxxxxxx",
                          @"date" : @"2014-09-09",
                          @"goldNumber" : @2,
                          @"age" : @"18",
                          @"url" : @"http://bawn.github.io/",
                          @"isVip" : NSNull.null
                          };
    Member *member = [MTLJSONAdapter modelOfClass:[Member class] fromJSONDictionary:dic error:nil];

    [MagicalRecord saveWithBlock:^(NSManagedObjectContext *localContext) {
        [MTLManagedObjectAdapter managedObjectFromModel:member insertingIntoContext:localContext error:nil];
    } completion:^(BOOL success, NSError *error) {
        NSLog(@"%lu", (unsigned long)[MemberManaged MR_findAll].count);
    }];
```


1. `Member *member = [MTLJSONAdapter modelOfClass:[Member class] fromJSONDictionary:dic error:nil];` completes the transformation from NSDictionary to Member and returns a Member instance.
2. `[MTLManagedObjectAdapter managedObjectFromModel:member insertingIntoContext:localContext error:nil];` completes the Member -> MemberManaged conversion and returns a MemberManaged instance, but we do not need it.
3. Use MagicalRecord's save method `+ (void) saveWithBlock:(void(^)(NSManagedObjectContext *localContext))block;`

**Note: for the MemberManaged class, we do not need to do anything to it.**


### Uniqueness Check

This is also implemented through a method in `<MTLManagedObjectSerializing>`:

```
+ (NSSet *)propertyKeysForManagedObjectUniquing{
    return [NSSet setWithObject:@"memberID"];
}
```

This means that when inserting new data, the `memberID` value of the record being inserted is compared with the values already in the database. If there is a match, the existing record is updated. If not, a new one is inserted. The convenience of this is obvious.

Updating Data

```

- (IBAction)updateData:(id)sender{
    NSDictionary *dic = @{
						  @"id" : @"2",
                          @"phone" : @"xxxxxxxx",
                          @"date" : @"2015-12-09",
                          @"goldNumber" : @2,
                          @"age" : @"19",
                          @"url" : @"http://bawn.github.io/",
                          @"isVip" : NSNull.null
                          };
    Member *member = [MTLJSONAdapter modelOfClass:[Member class] fromJSONDictionary:dic error:nil];

    [MagicalRecord saveWithBlock:^(NSManagedObjectContext *localContext) {
        [MTLManagedObjectAdapter managedObjectFromModel:member insertingIntoContext:localContext error:nil];
    } completion:^(BOOL success, NSError *error) {
        NSLog(@"%lu", (unsigned long)[MemberManaged MR_findAll].count);
    }];
}
```

There is only one record in the database because the inserted records all have a `memberID` of 2.

## Summary

Using MagicalRecord together with Mantle makes the code look much cleaner. There is no longer any need to struggle with Core Data's complicated APIs, and there is no need to write `if/else` logic for field conversion.

Demo: [MagicalRecord-Mantle](https://github.com/bawn/MagicalRecord-Mantle)
