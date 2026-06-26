---
layout: post
title: "React Native: A Few Thoughts"
date: 2016-04-15
comments: true
categories: [React Native]
tags: [react-native]
keywords: [React Native, iOS]
publish: true
description: React Native
---
![image](/assets/images/ReactNative/react_native.png)

I had been following [react-native](https://github.com/facebook/react-native) for a long time, just like [reactivecocoa](https://github.com/ReactiveCocoa/ReactiveCocoa). The reason I only started learning it recently is because of a few factors:

- The project is already relatively mature
- The community is active both in China and abroad
- The learning curve for JavaScript and React is not as high as I had imagined

I think the first two reasons deserve a more detailed explanation, so from here on I will use RN as shorthand for react-native.

RN updates very frequently. From what I can see, there is roughly one new version every half month. Since the release of `0.1.0` at the end of March last year, a total of 82 versions have been released up to `0.25.0`, and that number includes many release candidates. That alone shows how much importance Facebook places on RN.

In fact, a large part of an open source project's popularity depends on how active its community is. RN's community is so active now largely thanks to Facebook's promotion, such as the F8 conferences in recent years, the open source Atom-based IDE [Nuclide](https://github.com/facebook/nuclide), the third-party package search site [js.coach](https://js.coach/react-native), and Facebook's own projects such as `Facebook Ads Manager` and `Facebook Groups`, both of which were developed with RN. There are also good communities in China, such as the [React Native Chinese community](http://reactnative.cn/). In addition, two useful GitHub collections for RN learning resources and components, [awesome-react-native](https://github.com/jondot/awesome-react-native) and [react-native-guide](https://github.com/ele828/react-native-guide), are both good references.

I picked a few questions that many people care about and shared my own thoughts and understanding below.

### Learning Curve

When learning a new skill, the first thing most people do is figure out what the prerequisites are. For RN, those prerequisites are naturally JavaScript and React. But I did not deliberately study JavaScript and React first; instead, I started using RN directly while learning both along the way. Personally, I think that for developers with experience, syntax is not really a problem. That is true even for JavaScript, which I consider a very "casual" language. So if you want to learn RN but do not have JavaScript or React experience, there is no need to worry that the lack of a foundation will block you too much.

### Performance

There are already many articles analyzing performance, so I will only talk about my subjective impression. On iOS, as long as the page is not extremely complex, performance is basically close to native. Sometimes you may notice a slight stutter during page transitions. Android may have platform issues or RN optimization issues, and its performance is currently not ideal, but it is still completely acceptable. One thing to mention is that animations on RN do affect performance to some extent, so I do not recommend using a large number of animations. The official website also has [documentation](https://facebook.github.io/react-native/docs/performance.html#content) explaining performance issues and their causes. If you run into performance problems, I strongly recommend reading it carefully.

### Real-World Usage

If you try to build a commercial project entirely with RN, I think you will run into the following issues:

* You will inevitably need some native platform code unless your project is very simple and does not involve third-party platforms (sharing, login), complex pages, complex logic, or complex animations. For example, if the app has a clear cache feature, you must write two sets of code. Personally, I think that no matter how far RN evolves, there will still be cases where it cannot satisfy both platforms at the same time, and at those times you will need to rely on third-party libraries.
* UI differences between platforms. RN's philosophy is `write once, run anywhere`, so you still need to prepare for platform adaptation. That is almost unavoidable. Also, the same control may behave differently on different platforms. I once ran into a case where the `placeholder` position of `TextInput` was different on iOS and Android.
* Animations. I mentioned earlier that animations can affect performance to some extent, and complex animations are very difficult for RN.
* Bugs in RN itself. This may have been especially prominent in the early versions. Once you encounter one, all you can do is hope the RN team fixes it quickly, and that can be inconvenient for commercial projects.

During the learning process, I also found a few interesting things:

1. [ListView](https://facebook.github.io/react-native/docs/listview.html#content) on iOS is wrapped around `UIScrollView`
2. [Text](https://facebook.github.io/react-native/docs/text.html#content) on iOS is `UIView`, not `UILabel` or `UITextView`. RN draws text onto the view through the Draw method because that gives better performance. By the way, the Calendar app that comes with the iPhone uses the same approach. Details: [Jianshu](http://www.jianshu.com/p/1881c12ae33e)
3. [TouchableHighlight](http://reactnative.cn/docs/0.24/touchablehighlight.html#content) and similar components on iOS are actually wrappers around `UIGestureRecognizer`, not `UIButton`

### Tools

For beginners on macOS, I recommend the editor [Atom](https://atom.io/), together with the Atom-based IDE [Nuclide](https://github.com/facebook/nuclide) mentioned earlier, plus a few Atom plugins:

* [language-babel](https://github.com/gandm/language-babel)
* [language-javascript-jsx](https://github.com/subtleGradient/language-javascript-jsx)
* [react](https://github.com/orktes/atom-react)

If you think autocomplete is important, you can also try these two plugins: [react-snippets](https://atom.io/packages/react-snippets) and [atom-react-native-autocomplete](https://atom.io/packages/atom-react-native-autocomplete)

### Notes

If you use the international version of Evernote, you can add my [RN notebook](https://www.evernote.com/pub/lingchen621/reactnative). I will periodically add my notes there.
