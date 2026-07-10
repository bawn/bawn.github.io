---
layout: post
title: "Miscellaneous Notes"
date: 2014-11-24
update: 2015-04-7
comments: true
categories: iOS
tags: [iOS]
published: ture
keywords: iOS
description: Miscellaneous iOS notes
---

It has been a long time since I last updated the blog, because the project has been a bit busy.

Back to the point: over nearly a year of development work, I have been recording things I learned using [Evernote](https://www.yinxiang.com/). This is indeed a good learning habit. However, Evernote does not support Markdown, and there are many ways online to save notes into Evernote in Markdown format. At the moment, I use [MaKeFeiXiang](http://marxi.co/), which is quite convenient, but the professional version is paid.

Below are some uncategorized miscellaneous notes from the past two years. Some of them may be helpful.

### Joining an Array into a String

```
NSArray *array=[[NSArray alloc]initWithObjects:@"apple",@"banana",@"strawberry", @"pineapple", nil];
NSString *newString=[array componentsJoinedByString:@","];    
NSLog(@"%@", newString);
```


```
2013-10-29 15:38:23.372 Nurse[4001:c07] apple,banana,strawberry,pineapple
```

___

### Splitting a String into an Array

```
- (void)viewDidLoad
  {
    NSString *a = [[NSString alloc] initWithString : @"winter melon, watermelon, dragon fruit, big head, puppy" ];
    NSArray *b = [a componentsSeparatedByString:@"，"];
  }
```
___


### Getting RGB Values from UIColor

```
CGFloat R, G, B;
const CGFloat *components = CGColorGetComponents(color.CGColor);
R = components[0];
G = components[1];
B = components[2];
```

___

### Anti-Aliasing

Sometimes, after an image is rotated, it may show obvious aliasing.

Solution: add a key named `UIViewEdgeAntialiasing` to the project's plist file and set it to `YES`, but this may have a serious impact on performance.

Apple's explanation:

> Use antialiasing when drawing a layer that is not aligned to pixel boundaries. This option allows for more sophisticated rendering in the simulator but can have a noticeable impact on performance.

Simple solution: add a 1-pixel transparent border around the original image.

___

### Screen Capture

After iOS 7:

```

UIGraphicsBeginImageContextWithOptions(_sourceViewController.view.frame.size, YES, 0.0);
[self.sourceViewController.view.window drawViewHierarchyInRect:_sourceViewController.view.frame afterScreenUpdates:NO];
UIImage *snapshot = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();

```

____

### Hiding the Keyboard by Tapping Anywhere on the Screen

Override this in the view controller:


```
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event;

```

Method

```
  -(void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event{

    [super touchesBegan:touches withEvent:event];

    [self.view endEditing:YES];

  }
```
___

### Adding a Tap Gesture to a View Containing a UITableView

If you add a tap gesture to a view containing a UITableView, that tap gesture will block the UITableView's


```
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath;
```

You can use the `Delegate` method of `UIGestureRecognizer`:

```
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer;
```

Cancel the gesture when the tap occurs inside the UITableView.


```
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer{
    CGPoint point = [gestureRecognizer locationInView:self];
    if(CGRectContainsPoint(menuTableView.frame, point)){
      return NO;
    }
    return YES;
  }
```

The simpler solution is to set the tap gesture's `cancelsTouchesInView` to `NO`.

```
singleTap.cancelsTouchesInView = NO;
```

___

### `URLWithString:` Returns `nil`

Solution

```
NSString *tempString = [urlString stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
NSURL *url = [NSURL URLWithString:tempString];
```

___

### Checking Whether a String Contains Chinese Characters

```
NSRegularExpression *regularExpression = [NSRegularExpression regularExpressionWithPattern:@"[\u4e00-\u9fa5]" options:0 error:nil];
// Return the number of Chinese characters in numberOfMatches
NSUInteger numberOfMatches = [regularExpression numberOfMatchesInString:string options:0 range:NSMakeRange(0, [string length])];
```

___

### Converting Degrees to Radians

`#import <GLKit/GLKit.h>`


```objectivec
float GLKMathDegreesToRadians(float degrees)
```
___

### Quickly Locating a Cell

Use a subview inside the cell, in this example a button.

```
NSIndexPath *path = [_collectionView indexPathForItemAtPoint:[button convertPoint:CGPointZero toView:self.collectionView]];
```

Works for UITableView (with different APIs) and UICollectionView.

___

### Checking Whether One View Is a Subview of Another

```
- (BOOL)isDescendantOfView:(UIView *)view;
```

___

### Fixing `%d` and `%u` Warnings on 64-bit

```
# if __LP64__
# define NSI "ld"
# define NSU "lu"
# else
# define NSI "d"
# define NSU "u"
# endif
```

Usage

```
NSInteger row = 2;
NSLog(@"i=%"NSI@" other text", row);
```

___

### The `?:` Syntax

```
NSLog(@"%@", @"a" ?: @"b"); // @"a"
```

If the condition is true, it returns the message receiver on the left side of `?:`, which here is `@"a"`. Common usage:

```
cell.titleLabel.text = self.name ?:nil;
```

___

### Opening the System Settings Screen (iOS 8 Only)

```
[[UIApplication sharedApplication] openURL:[NSURL URLWithString:UIApplicationOpenSettingsURLString]];
```

___

### Opening the App Store (iOS 7 and Later)

Open the product details page.

```
[[UIApplication sharedApplication] openURL:[NSURL URLWithString:@"itms-apps://itunes.apple.com/app/idxxxxxx"]];
```

Open the product review page.

```
[[UIApplication sharedApplication] openURL:[NSURL URLWithString:@"http://itunes.apple.com/WebObjects/MZStore.woa/wa/viewContentsUserReviews?id=xxxxxx&pageNumber=0&sortOrdering=2&type=Purple+Software&mt=8"]];
```

`xxxxx` is the Apple ID.

___

### Removing the One-Pixel Bottom Line of UINavigationBar

```
[self.navigationController.navigationBar setBackgroundImage:[[UIImage alloc] init] forBarMetrics:UIBarMetricsDefault];
self.navigationController.navigationBar.shadowImage = [[UIImage alloc] init];
```

Restore

```
[self.navigationController.navigationBar setBackgroundImage:[[UIImage alloc] init] forBarMetrics:UIBarMetricsDefault];
self.navigationController.navigationBar.shadowImage = nil;
```

___

### Determining Gesture Direction

* Vertical

```
  CGPoint velocity = [sender velocityInView:self];

  BOOL isVerticalGesture = fabsf(velocity.y) >= fabsf(velocity.x);
```

* Horizontal

```
  CGPoint velocity = [self.panGestureRecognizer velocityInView:self.panGestureRecognizer.view];
  BOOL isHorizontalGesture = fabs(velocity.y) <= fabs(velocity.x);
```

___

### Scrolling a UIScrollView to the Top

```
self.tableView.contentOffset = CGPointMake(0, 0 - self.tableView.contentInset.top);
```

___

### Disabling Third-Party Keyboards in the App

Add this in `AppDelegate`:

```
- (BOOL)application:(UIApplication *)application shouldAllowExtensionPointIdentifier:(NSString *)extensionPointIdentifier {
    if ([extensionPointIdentifier isEqualToString: UIApplicationKeyboardExtensionPointIdentifier]) {
  ​	  return NO;
    }
    return YES;
  }
```
___

### Formatting File Size

No need to use the very low-level `/1024/1024` conversion to MB anymore. Try this instead:

```
NSString *storage = [NSByteCountFormatter stringFromByteCount:size countStyle:NSByteCountFormatterCountStyleFile];
```
