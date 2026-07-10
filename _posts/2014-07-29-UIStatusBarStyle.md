title: "UIStatusBarStyle"
---
layout: post
title: "UIStatusBarStyle"
date: 2014-07-29
comments: true
categories: iOS
tags: [UIStatusBarStyle]
published: true
keywords: UIStatusBarStyle
---
There are two ways to adjust the status bar style in iOS:

#### 1. Add View controller-based status bar appearance to the plist and set it to YES, then set the style with the following method

```
- (UIStatusBarStyle)preferredStatusBarStyle{
    return UIStatusBarStyleDefault;
}
```
However, this method often seems to stop working for no obvious reason. If you pay attention, you will notice that as long as you are using a `UINavigationController` and the `navigationBar` is not hidden, the `preferredStatusBarStyle` method will not be called for its `rootController` or any later pushed controllers. The reason is that `UINavigationController` determines the `StatusBarStyle` based on its own `navigationBar`'s `barStyle`.

Solution: extend `UINavigationController` ([venj blog](http://cocoa.venj.me/blog/view-controller-based-status-bar-style-and-uinavigationcontroller/))

> UINavigationController+StatusBar.h

```
#import <UIKit/UIKit.h>
@interface UINavigationController (StatusBar)
- (UIStatusBarStyle)preferredStatusBarStyle;
@end
```

>UINavigationController+StatusBar.m

```
#import "UINavigationController+StatusBar.h"
@implementation UINavigationController (StatusBar)
- (UIStatusBarStyle)preferredStatusBarStyle{
    return [self.topViewController preferredStatusBarStyle];
}
@end
```

Import `"UINavigationController+StatusBar.h"` and then call `- (UIStatusBarStyle)preferredStatusBarStyle`.

#### 2. Add View controller-based status bar appearance to the plist and set it to NO, then set the style with the following method

```
[[UIApplication sharedApplication] setStatusBarStyle:UIStatusBarStyleLightContent];
```

This lets you change the status bar style freely, regardless of whether you use a `UINavigationController`. However, this approach has one drawback: if you set the status bar to `UIStatusBarStyleLightContent` when pushing to a new VC, the status bar will still be in that state when you pop back. At that point, if you want to set it back to `UIStatusBarStyleDefault`, you must call it again. In other words, this method must be called manually.






