---
layout: post
title: "A Discussion on Nested Scrolling in Practice"
date: 2019-02-26
comments: true
categories: [iOS]
tags: [nested scrolling]
keywords: [nested scrolling]
publish: true
## description: nested scrolling
---


This article discusses nested scrolling experiences like the ones you see on the home pages of [Jike](https://www.ruguoapp.com/), [Taopiaopiao](https://www.taopiaopiao.com/), and the personal profile pages of [Douyin](https://www.douyin.com/) and [Jianshu](https://www.jianshu.com/). In fact, there are already many related articles online, for example:


- [A solution to scroll conflicts in nested UIScrollView setups](https://www.jianshu.com/p/040772693872)

- [Another solution to scroll conflicts in nested UIScrollView setups on iOS](https://www.jianshu.com/p/df01610b4e73)

- [A solution for multi-level UIScrollView nested scrolling](https://jiar.me/article/Multi-tier-UIScrollView-nested-scrolling-solution)



Most of those articles focus on how to solve gesture conflicts and then present matching solutions, because the majority of them use a three-level ScrollView structure like the one below.

![image1](/assets/images/NestedScrolling/NestedScrolling-1.png)

​    

- Blue view: first-level ScrollView

- Red view: HeaderView

- Green view: MenuView

- Orange view: second-level ScrollView

- Black, dark black, light black: third-level ScrollView



You can see that both the first-level ScrollView and the third-level ScrollView need to scroll vertically, so the main problem is the scrolling conflict between them. I will not go into the implementation details here; you can also refer to [HGPersonalCenter](https://github.com/ArchLL/HGPersonalCenter), which contains detailed comments. The view hierarchy below is Taopiaopiao's home page, and it clearly shows a three-level ScrollView structure.

![image2](/assets/images/NestedScrolling/NestedScrolling-2.png)





- Top MVNestTableView: first-level ScrollView
- Middle UIScrollView: second-level ScrollView
- Bottom MVNestTableView: third-level ScrollView




The reason I gave four examples above is that, in addition to the three-level ScrollView approach used by Taopiaopiao and Jianshu, Douyin and Jike use a two-level ScrollView approach, and Jike's user experience is even better. I will explain that later. The rough structure of the two-level ScrollView solution is shown below.



![image-4](/assets/images/NestedScrolling/NestedScrolling-4.png)



- Blue view: first-level ScrollView
- Black, dark black, light black: second-level ScrollView
- Red view: HeaderView
- Green view: MenuView

Below is the structure of the Jike home page in version 5.x. You can clearly see that Jike uses the two-level ScrollView solution.

![image3](/assets/images/NestedScrolling/NestedScrolling-3.png)



You can also roughly judge the implementation by tapping the status bar. For example, after tapping the status bar, Taopiaopiao only scrolls to the top of the child ScrollView rather than the outermost ScrollView. Jianshu does scroll to the top of the outermost layer, but the effect is clearly not natural enough, because the three-level ScrollView does not extend all the way to the top vertically. Douyin and Jike, on the other hand, return to the top very naturally when the status bar is tapped.

From the overall structure, Jike only uses two levels of ScrollView. Vertically, the ChildScrollView fully takes over the gesture; horizontally, MainScrollView handles the scrolling. The benefit is that there is no need to worry about gesture conflicts. However, to achieve the effects mentioned above, you still need to handle the following issues:



- The positions of HeaderView and MenuView must change according to the ChildScrollView's scrolling

- When switching tabs, the next ChildScrollView's offset must be synchronized

- ChildScrollView must reserve blank space at the top equal to the combined height of HeaderView and MenuView

- HeaderView must not intercept scroll gestures



I will not go into implementation details here. At the end of the article, I provide open source libraries that implement both approaches. Feel free to give them a Star. Although both Jike and Douyin use this two-level ScrollView approach, Jike's experience is better. For example, on Douyin's profile page, if the point where the finger starts dragging is over an interactive control, such as the tab bar, the swipe gesture will fail. Also, after switching tabs, dragging the view down to the top and then returning to the previous tab, Douyin returns directly to the original position, while Jike can preserve the previous progress.



**Solution for disabled scrolling at the top**

To achieve a perfect experience, Jike adds HeaderView and MenuView to the top of each ChildScrollView. That way, the whole area can scroll vertically even if the initial touch starts on top of an interactive control. Then, when scrolling horizontally, the HeaderView and MenuView inside ChildScrollView are hidden, and when scrolling stops, the HeaderView and MenuView that originally lived in the outer ScrollView are shown.



**Solution for preserving scroll progress**

To preserve progress, the first thing to do is determine whether the current ChildScrollView is in a special state. That state is whether the value of offset.y is greater than the HeaderView offset. Then, by checking the current scrolling direction of ChildScrollView, you can decide whether the positions of HeaderView and MenuView should be adjusted.



The two solutions each have their own strengths and weaknesses.

#### Solution 1

Advantages:

- Works nicely with third-party pull-to-refresh libraries

- ChildViewController does not require extra setup



Disadvantages:

* More complex to implement
- Slight stutter during scrolling
- Tab switching cannot preserve progress
- Tapping the status bar cannot return to the top



#### Solution 2

Advantages:

- Simple to implement
- Smooth scrolling with no stutter
- Tab switching can preserve progress
- Tapping the status bar can return to the top



Disadvantages

- ChildViewController requires extra setup (ChildScrollView must leave space at the top equal to the height of HeaderView and MenuView)
- Pull-to-refresh can only be implemented inside ChildViewController



One thing to note is that, because MainScrollView does not scroll vertically in Solution 2, pull-to-refresh must be implemented inside ChildViewController. But since HeaderView and MenuView need to move according to ChildScrollView's offset, there is a noticeable bug when using [MJRefresh](https://github.com/CoderMJLee/MJRefresh): their offsets become incorrect. I did not investigate the solution in depth before publishing this article. Perhaps Jike adopted the workaround mentioned above for the same reason.



The open source libraries for the two solutions above are: **Solution 1: [Aquaman](https://github.com/bawn/Aquaman)** and **Solution 2: [Shazam](https://github.com/bawn/Shazam)**. MenuView is designed to be implemented by developers themselves, because even a MenuView with various built-in styles still cannot meet the endless variety of design requirements. With my demo, you can quickly build the exact effect you want.
