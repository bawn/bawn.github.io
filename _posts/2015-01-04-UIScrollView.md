---
layout: post
title: "UIScrollView Relative Layout"
date: 2015-01-04
comments: true
categories: iOS
tags: [iOS]
keywords: UIScrollView autolayout
description: UIScrollView relative layout, including a video tutorial
---

Relative layout for UIScrollView in Interface Builder has always been a headache. The two problems people usually run into are:

1. How to determine the correct contentSize
2. How to design an interface larger than one screen

First, there is one important thing to know: after using Auto Layout, contentSize does not need to be set manually. The system determines contentSize from the content added to the UIScrollView. Let's start with a simple example.

### How to determine contentSize correctly

1. Drag a `UIScrollView` onto the view controller and set its top, bottom, left, and right constraints to 0 relative to the parent view.
![image](/assets/images/UIScrollView/UIScrollView-1.png)

2. Drag a `UILabel` into the center of the `UIScrollView` and also set its top, bottom, left, and right constraints.
![image](/assets/images/UIScrollView/UIScrollView-2.png)

3. The final result:
![image](/assets/images/UIScrollView/UIScrollView-4.png)

__Why does the label layout need explicit top, bottom, left, and right constraints?__

As mentioned above, the system determines `contentSize` from the content added to the UIScrollView. Imagine removing the right and bottom constraints. Then, for the UIScrollView to contain this label, the conditions for `contentSize` would be:

* Horizontal width of at least 139 + 42 = 181 pixels
* Vertical height of at least 273 + 21 = 294 pixels

That means `contentSize` must be at least {181, 294}. Of course, it could also be {200, 300}, and so on, so the `contentSize` value is ambiguous. When either of these constraints is removed, IB also warns that the content size is ambiguous.

> warning: Ambiguous Layout: Scrollable content size is ambiguous for "Scroll View".

__Also note that for labels with a fixed number of lines and fixed text, the width and height are already determined, so there is no need to set them explicitly.__

Now look at the case where the bottom and right constraints are added. The conditions for `contentSize` become:

* Horizontal: 139 + 139 + 42 = 320
* Vertical: 274 + 273 + 21 = 568

At this point, `contentSize` is determined as `{320, 568}`. So when laying out the content view inside a UIScrollView, always remember one thing: you must make UIScrollView know its `contentSize`, not leave it ambiguous. That is what it means to finish laying out a UIScrollView.

Looking back at the UIScrollView layout: its top, bottom, left, and right constraints relative to the parent view are all 0. That means its width and height are actually determined by the device. A 3.5-inch device is `{320, 480}`, and a 4.0-inch device is `{320, 568}`. So if you run it on an iPhone 4, the UIScrollView needs vertical scrolling because the required vertical size (568) is greater than 480. On an iPhone 5, vertical scrolling is not needed because the required vertical size (568) equals 568.

What happens in practice? If you run this in the iPhone 5 (iOS 7.1) simulator, you will find that the UIScrollView can actually scroll vertically. Why? Remember the extra 64 pixels of scrollable area reserved by iOS 7 for UIScrollView and its subclasses? That is what causes the height of the UIScrollView's `contentSize` to be reduced by 64 pixels. So the required vertical size (568) is greater than 568 - 64, which is why the UIScrollView scrolls vertically. You can try changing the label's top constraint to `274 - 64 = 210`, or unchecking the view controller's `Adjust Scroll View Insets` in IB, and scrolling will no longer work.

### How to design an interface larger than one screen

For a simple two-screen-wide UI layout, I also uploaded a step-by-step video to [Tudou](http://www.tudou.com/programs/view/RK6niTubChI).

1. Set the view controller's width
![image](/assets/images/UIScrollView/UIScrollView-5.png)
Set Size to Freeform so you can lay out a view wider than two screens.
2. Drag two views onto the UIScrollView and set their colors. view1 represents the red view, and view2 represents the black view.
![image](/assets/images/UIScrollView/UIScrollView-6.png)
3. Set view1's left, top, and bottom constraints to 0, and its right constraint to 0 relative to view2. Set view2's right, top, and bottom constraints to 0.
![image](/assets/images/UIScrollView/UIScrollView-7.png)
Ignore the warnings for now. At this point, both the width and height of `contentSize` are still undetermined because the widths and heights of view1 and view2 are undetermined.
4. Select view1 and constrain it to have the same width and height as the UIScrollView, then do the same for view2, and finally choose Update Frames.
![image](/assets/images/UIScrollView/UIScrollView-8.png)
Horizontal: 0 + view1's width + 0 + view2's width = twice the width of the UIScrollView
Vertical: 0 + view1's height + 0 = the height of the UIScrollView, or 0 + view2's height + 0 = the height of the UIScrollView
5. Set the view controller's size back to Fixed
