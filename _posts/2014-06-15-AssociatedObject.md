---
layout: post
title: "Associating Objects with a Class"
date: 2014-06-15
comments: true
categories: iOS
tags: [iOS]
published: true
keywords: objc_setAssociatedObject objc_getAssociatedObject
description: Associated objects
---

I believe most people have seen the two methods `objc_setAssociatedObject` and `objc_getAssociatedObject` to some extent.

What are they actually useful for? If we only say "associated objects," that does not really show their practical value. Here is a more practical use case.
When using `UIAlertView`, have you ever felt it was cumbersome? The logic for button taps has to be written in delegate methods. If a VC contains multiple `UIAlertView` instances, things become even more complicated, because you have to determine which `alertView` object was passed into the delegate method. We can use the two methods above to solve this kind of problem once and for all.

In the end, creating and showing a `UIAlertView` can look like this:

```
	[self showAlertViewWithTitle:@"Are you sure you want to close the case?"
                             message:nil
                   cancelButtonTitle:@"Cancel"
                   otherButtonTitles:@[@"Return", @"Accept"]
                            onCancel:^{  
            NSLog(@"Cancel button tapped");
        }onOther:^(NSInteger index) {
            if (index == 0) {
                NSLog(@"First other button tapped");
            }
            else{
                NSLog(@"Second other button tapped");
            }
        }];
```

You can see that when the `UIAlertView` is created, the logic for handling each button has already been defined.

Let's start writing this extension. We will create a `UIViewController` category called `Addons`.

>UIViewController+Addons.h

```
#import <Foundation/Foundation.h>
typedef void(^HandleBlock)(void);
typedef void(^OtherHandleBlock)(NSInteger index);
@interface UIViewController(Addons)<UIAlertViewDelegate>
/**
 *  Show an alert view
 *
 *  @param title             title
 *  @param message           message
 *  @param cancelButtonTitle title for the cancel button
 *  @param otherButtonTitles array of titles for other buttons
 *  @param cancelBlock       block executed when the cancel button is tapped
 *  @param otherBlock        block executed when another button is tapped
 */
- (void)showAlertViewWithTitle:(NSString *) title
                      message:(NSString *) message
            cancelButtonTitle:(NSString *) cancelButtonTitle
            otherButtonTitles:(NSArray *) otherButtonTitles
                     onCancel:(CancelBlock) cancelBlock
                      onOther:(OtherHandleBlock) otherBlock;
```

Two `block` types are defined. The first is used when the cancel button is tapped, and the second is used when another button is tapped. Because there may be multiple other buttons, an `index` parameter is added to determine which button was tapped.

>UIViewController+Addons.m

```
#import "UIViewController+Addons.h"
#import <objc/runtime.h>
static char kVCActionHandleAlertCancelBlockKey;
static char kVCActionHandleAlertOtherBlockKey;
@implementation UIViewController(Addons)
-(void)showAlertViewWithTitle:(NSString *)title
                      message:(NSString *)message
            cancelButtonTitle:(NSString *)cancelButtonTitle
            otherButtonTitles:(NSArray *)otherButtonTitles
                     onCancel:(CancelBlock)cancelBlock
                      onOther:(OtherHandleBlock)otherBlock{
     UIAlertView *alertView = [[UIAlertView alloc]initWithTitle:title
                                                       message:message
                                                      delegate:self
                      	                   cancelButtonTitle:cancelButtonTitle
                                             otherButtonTitles:nil];
    for(NSString *buttonTitle in otherButtonTitles){
        [alertView addButtonWithTitle:buttonTitle];
    }
    if (cancelBlock) {
        objc_setAssociatedObject(self, &kVCActionHandleAlertCancelBlockKey, cancelBlock, OBJC_ASSOCIATION_COPY_NONATOMIC);
    }
    if (otherBlock) {
        objc_setAssociatedObject(self, &kVCActionHandleAlertOtherBlockKey, otherBlock, OBJC_ASSOCIATION_COPY_NONATOMIC);
    }
    [alertView show];
}
```

```
-(void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex{
	if(buttonIndex == alertView.cancelButtonIndex){
        CancelBlock cancelBlock = objc_getAssociatedObject(self, &kVCActionHandleAlertCancelBlockKey);
        if(cancelBlock){
            cancelBlock();
        }
    }
    else{
        OtherHandleBlock otherBlock = objc_getAssociatedObject(self, &kVCActionHandleAlertOtherBlockKey);
        if(otherBlock){
            otherBlock(buttonIndex - 1);
        }
    }
}
```

Remember to import the `<objc/runtime.h>` header, create the `UIAlertView`, and then associate the two incoming `block`s with the `UIViewController` object, which is this step:

```
 if (cancelBlock) {
        objc_setAssociatedObject(self, &kVCActionHandleAlertCancelBlockKey, cancelBlock, OBJC_ASSOCIATION_COPY_NONATOMIC);
    }
    if (otherBlock) {
        objc_setAssociatedObject(self, &kVCActionHandleAlertOtherBlockKey, otherBlock, OBJC_ASSOCIATION_COPY_NONATOMIC);
    }

```

Note that the association policy here should preferably use `OBJC_ASSOCIATION_COPY_NONATOMIC` so that ARC can manage the `block`. The delegate method later simply retrieves the `block` from `self` and executes it.

Of course, `UIActionSheet` and even `UIImagePickerController` can also be extended this way.
