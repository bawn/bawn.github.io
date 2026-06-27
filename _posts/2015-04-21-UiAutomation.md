---
layout: post
title: "UI Automation Testing"
date: 2015-04-21
comments: true
categories: iOS
tags: [iOS]
keywords: UI automation testing iOS
publish: false
description: iOS UI automation testing
---

Following the idea of making life easier and because testing has always had its share of criticism, I spent some free time researching Apple's UIAutomation. I thought this would let me just sit back and wait for the app to run through every flow by itself and then output a crash report. But reality is not as ideal as imagination. UiAutomation is not nearly as perfect as I had hoped.<br>

## Introduction

Press `⌘ + I` to open Instruments, choose UiAutomation, and the basic interface looks like this:

![image](/assets/images/UiAutomation/tool.png)
**Area overview:**<br>
1. Start/stop test buttons, device selection, and project menu<br>
2. JS script editor, Trace Log, and Editor Log display area<br>
3. Automation test timeline<br>
4. Other menu pages<br>
5. Script or log selection menu<br>
6. Start, stop, and pause buttons for auto-generated JS script code<br>

Let's look at a simple test example. On the app screen there is a UITextField, which we will call element `A`, and on the navigation bar there is a rightItem, which we will call element `B`. Tapping the rightItem pushes to another VC.
![image](/assets/images/UiAutomation/screen.png)

Choose an empty script in section ④ and write the following code. If you do not know JS, do not worry. The JS used for automation testing is very simple.

```
var target = UIATarget.localTarget();
var app = target.frontMostApp();
var window = app.mainWindow();
// print the element tree
app.logElementTree();
```

Press `⌘ + R` to run it, and section ② should automatically switch to the Log view.
![image](/assets/images/UiAutomation/log.png)

What gets printed in the log is the element (UIAElement) tree of the current screen. Elements at the same level are grouped into arrays. Simulating user actions is essentially operating on elements, so getting the right element is the key. The `logElementTree()` function is very useful. Remember to call it again when the page changes so that you can find the element you want.<br>

**Additional script:**


```
var target = UIATarget.localTarget();
var app = target.frontMostApp();
var window = app.mainWindow();
app.logElementTree();
var textField = window.textFields()[0];
// enter 122 into element A
textField.setValue("122");
// wait 1 second
target.delay(1);
// get element B
var rightButton = window.navigationBar().buttons()['Button'];
// tap element B
rightButton.tap();
```

The result after running it is: element A is filled with 122 -> then element B is tapped -> the app enters another page.

**Code explanation:**

* Get element A: `window.textFields()[0]`. There is only one textField on the screen, so naturally it is the first element in the textFields array.
* Get element B: `window.navigationBar().buttons()['Button']`. Since element B is on the navigationBar, you need to get the navigationBar first, then get element B from the buttons array. The value `Button` is retrieved through element B's `name` property (the default value).

The `name` value of an element can also be set manually. For example, set the `name` value of element A to `textField`. Note: do not set the element's accessibility property (`isAccessibilityElement`) to NO.
![image](/assets/images/UiAutomation/label.png)
Or set it in code:

```
self.aView.accessibilityLabel = @"textField";
```

Then you can retrieve it like this:

```
var textField = window.textFields()['textField'];
```

For executable element methods such as `tap()`, you can refer to Apple's [official documentation](https://developer.apple.com/library/ios/documentation/DeveloperTools/Reference/UIAutomationRef/).

**Element arrays include:**

```
1. buttons()
2. images()
3. scrollViews()
4. textFields()
5. webViews()
6. segmentedControls()
7. sliders()
8. staticTexts()
9. switches()
10. tabBar()
11. tableViews()
12. textViews()
13. toolbar()
14. toolbars()
15. secureTextFields() // encrypted UITextField
...
```

Apple also provides an accessibility inspection tool that makes it easy to view element information. Open Settings -- General -- Accessibility -- Accessibility Inspector.

![image](/assets/images/UiAutomation/accessibility.png)


## Recording Automation Scripts

Create a new script in section ④. In section ⑥, click the red button in the middle to start recording user actions and convert them into JS code. Click the button on the right to stop recording, and click the button on the left to run the script. At this point you may be thinking: if this feature exists, why do we still need to write scripts at all? Is that really true?<br>
Here is the script recorded for the same action: tap element A -> type 112 on the keyboard -> tap element B.

```
var target = UIATarget.localTarget();
target.frontMostApp().mainWindow().textFields()[0].textFields()[0].tap();
target.frontMostApp().keyboard().typeString("112");
target.frontMostApp().navigationBar().buttons()["Button"].tap();
```

This generated code has a problem: there is no delay between actions, meaning no `target.delay();`. For complex cross-page interactions, scripts recorded this way may fail when run again.

For example, if you tap a button, then present another page, and then tap the textField at the top of that page, the script might look like this:

```
var target = UIATarget.localTarget();
target.frontMostApp().mainWindow().buttons()[0].tap();
target.frontMostApp().mainWindow().textFields()[0].tap();
```

When execution reaches `target.frontMostApp().mainWindow().textFields()[0].tap();`, the interface may still be on the previous screen, so getting the `textfield` will definitely be a problem. If you run into this kind of issue, try adding `target.delay();` between actions. <br>
Another issue is the lack of logic checks, such as clicking one button in one case and another button in a different case.

So if you want to rely entirely on this method and avoid writing scripts, it basically will not work.

## Common Operations

Apple [official API documentation](https://developer.apple.com/library/ios/documentation/DeveloperTools/Reference/UIAutomationRef/)

* Screen taps

```
// single tap
UIATarget.localTarget().tap({x:100, y:200});
// double tap
UIATarget.localTarget().doubleTap({x:100, y:200});
// two-finger tap
UIATarget.localTarget().twoFingerTap({x:100, y:200});
```

* Zooming

```
// zoom in
UIATarget.localTarget().pinchOpenFromToForDuration({x:20, y:200},{x:300, y:200},2);
// zoom out
UIATarget.localTarget().pinchCloseFromToForDuration({x:20, y:200}, {x:300, y:200},2);
```

* Dragging and swiping

```
// drag
UIATarget.localTarget().pinchOpenFromToForDuration({x:20, y:200},{x:300, y:200},2);
// swipe
UIATarget.localTarget().pinchCloseFromToForDuration({x:20, y:200}, {x:300, y:200},2);
```

* Logging

```
UIALogger.logStart("xxx");
UIALogger.logFail("xxx");
UIALogger.logDebug("xxx");
```

## Running from the Command Line

The reason for using the command line is that Instruments is really slow for running tests. The command is:

```
instruments -t /Applications/Xcode.app/Contents/Applications/Instruments.app/Contents/PlugIns/AutomationInstrument.xrplugin/Contents/Resources/Automation.tracetemplate -w "iPhone 5s (8.3 Simulator)" /Users/xxxx/Library/Developer/CoreSimulator/Devices/A35F991E-425E-4F41-B76C-B7C176A06C36/data/Containers/Data/Application/324E3D2F-A0BC-4E96-97DF-97E791AB10A8/xxxx.app -e UIASCRIPT /Users/xxxx/Desktop/untitled.js -e UIARESULTSPATH /Users/xxxx/Desktop/tmp/
```

Parts you need to modify yourself:

```
// test device
-w "iPhone 5s (8.3 Simulator)"
```

```
// project directory; it is fine if the directory does not contain xxxx.app
/Users/xxxx/Library/Developer/CoreSimulator/Devices/A35F991E-425E-4F41-B76C-B7C176A06C36/data/Containers/Data/Application/324E3D2F-A0BC-4E96-97DF-97E791AB10A8/xxxx.app
```

```
// script directory
-e UIASCRIPT /Users/xxxx/Desktop/untitled.js
```

```
// test report output directory
-e UIARESULTSPATH /Users/xxxx/Desktop/tmp/
```

## Finally

The reason I say UiAutomation is not as perfect as it seems is because of the following:

1. JS script debugging is relatively troublesome
2. UiAutomation currently has quite a few bugs, such as manually setting `accessibilityLabel` not working, or random errors that disappear when you run it again
3. It is not complete enough. For example, it cannot determine whether a button is enabled, UIAlterView cannot be handled manually (the methods found online do not work), and when you have multiple JS scripts, you cannot automatically run the next one after finishing the current one
4. The generated test reports are very simple, containing only basic logs and screenshots, and the screenshot rules are not very clear (there may be an API for screenshots)

I also recommend an open source testing library, [tuneup_js](https://github.com/alexvollmer/tuneup_js), along with its [documentation](http://www.tuneupjs.org/installation.html).

#### Note: If you encounter the Error itms-90035 error when submitting your project, try removing the [tuneup_js](https://github.com/alexvollmer/tuneup_js) library.

__Reference: [Zhiping Software](http://www.cnblogs.com/vowei/archive/2012/08/10/2631949.html#3105924)__
