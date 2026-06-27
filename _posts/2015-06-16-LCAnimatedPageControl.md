---
layout: post
title: "A PageControl with Simple Animations"
date: 2015-06-16
comments: true
categories: iOS UIPageControl
tags: [iOS]
keywords: UIPageControl iOS
publish: true
description: An animated UIPageControl
---

An open source PageControl with simple animations, supporting Auto Layout. The project is available on [GitHub](https://github.com/bawn/LCAnimatedPageControl).

There are currently four styles to choose from:

* LCSquirmPageStyle
 ![image1](/assets/images/LCAnimatedPageControl/LCAnimatedPageControl1.gif)
* LCScaleColorPageStyle
 ![image2](/assets/images/LCAnimatedPageControl/LCAnimatedPageControl2.gif)
* LCDepthColorPageStyle
 ![image3](/assets/images/LCAnimatedPageControl/LCAnimatedPageControl3.gif)
* LCFillColorPageStyle
 ![image4](/assets/images/LCAnimatedPageControl/LCAnimatedPageControl4.gif)


### Example
```

self.pageControl.numberOfPages = 5; // number of indicators
self.pageControl.indicatorMargin = 5.0f; // spacing between indicators, default is 0
self.pageControl.indicatorMultiple = 1.6f; // indicator scale factor, default is 2
self.pageControl.indicatorDiameter = 10.0f; // indicator diameter
pageControl.pageIndicatorColor = [UIColor grayColor]; // color in normal state
pageControl.currentPageIndicatorColor = [UIColor redColor]; // color in current state
self.pageControl.pageStyle = LCScaleColorPageStyle; // style
self.pageControl.sourceScrollView = _collectionView; // bind the scroll view
[self.pageControl prepareShow]; // call this after all properties are set
[self.view addSubview:_pageControl];
```

Note that the spacing adjusted by `indicatorMargin` is the distance between two indicators when both are in the enlarged state. Diagram:
![2](/assets/images/LCAnimatedPageControl/LCAnimatedPageControl5.png)

In the `ScaleColorPageStyle` style, if the scroll view does not scroll to an adjacent position, you must implement the following protocol method and call `clearIndicators`:

```
- (void)scrollViewDidEndScrollingAnimation:(UIScrollView *)scrollView;{
    [self.pageControl clearIndicators];
}
```

Also, just like the native `UIPageControl`, listening for changes in the currently displayed indicator uses the `target-action` pattern:

```
[pageControl addTarget:self action:@selector(valueChanged:) forControlEvents:UIControlEventValueChanged];
```
