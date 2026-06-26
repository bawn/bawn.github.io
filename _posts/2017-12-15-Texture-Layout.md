---
layout: post
title: "Texture Layout"
date: 2017-12-15
comments: true
categories: [Texture]
tags: [iOS]
keywords: [Texture, iOS]
publish: true
description: Texture
---

![image](/assets/images/Texture/textture-logo.png)

`Texture` has its own mature layout system. Although the learning curve is a bit steep, it is at least more pleasant to write than native `AutoLayout`, and the key point is that its performance is much better than `AutoLayout`. The Texture documentation also highlights the advantages of this layout system:

* **Fast**: As fast as manual layout code and significantly faster than Auto Layout
* **Asynchronous & Concurrent**: Layouts can be computed on background threads so user interactions are not interrupted.
* **Declarative**: Layouts are declared with immutable data structures. This makes layout code easier to develop, document, code review, test, debug, profile, and maintain.
* **Cacheable**: Layout results are immutable data structures so they can be precomputed in the background and cached to increase user perceived performance.
* **Extensible**: Easy to share code between classes.

First, this layout system is based on `Texture` components, so when you need to use native controls, wrapping a native component in a block is a very natural approach. For example:

```
ASDisplayNode *animationImageNode = [[ASDisplayNode alloc] initWithViewBlock:^UIView * _Nonnull{
    FLAnimatedImageView *animationImageView = [[FLAnimatedImageView alloc] init];
    animationImageView.layer.cornerRadius = 2.0f;
    animationImageView.clipsToBounds = YES;
    return animationImageView;
}];
[self addSubnode:animationImageNode];
self.animationImageNode = animationImageNode;
```


`ASDisplayNode` checks whether it has subviews after initialization, and if so it calls

```
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize
```

for layout, so you need to override this method to lay out a view. For example:

```objc
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize{
    ASInsetLayoutSpec *inset = [ASInsetLayoutSpec insetLayoutSpecWithInsets:UIEdgeInsetsZero child:_childNode];
    return insetLayout;
}
```

`_childNode` has zero margins relative to its parent view, which is equivalent to `top`, `bottom`, `left`, and `right` all being 0 in `AutoLayout`.

```
-----------------------------Parent View----------------------------
|  -------------------------_childNode---------------------  |
|  |                                                      |  |
|  |                                                      |  |
|  ---------------------------  ---------------------------  |
--------------------------------------------------------------------
```

You can see that `layoutSpecThatFits:` must return an `ASLayoutSpec`. `ASInsetLayoutSpec` is one of its subclasses. Below are all the subclasses and their relationships:

* ASLayoutSpec
  * ASAbsoluteLayoutSpec   // absolute layout
  * ASBackgroundLayoutSpec // background layout
  * ASInsetLayoutSpec      // inset layout
  * ASOverlayLayoutSpec    // overlay layout
  * ASRatioLayoutSpec      // ratio layout
  * ASRelativeLayoutSpec   // corner layout
    * ASCenterLayoutSpec   // center layout
  * ASStackLayoutSpec      // stack layout
  * ASWrapperLayoutSpec    // fill layout
  * ASCornerLayoutSpec  // badge layout

___

### ASAbsoluteLayoutSpec

The usage is similar to native absolute layout.

```
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize{
  self.childNode.style.layoutPosition = CGPointMake(100, 100);
  self.childNode.style.preferredLayoutSize = ASLayoutSizeMake(ASDimensionMake(100), ASDimensionMake(100));

  ASAbsoluteLayoutSpec *absoluteLayout = [ASAbsoluteLayoutSpec absoluteLayoutSpecWithChildren:@[self.childNode]];
  return absoluteLayout;
}
```
The important thing to note is that `ASAbsoluteLayoutSpec` is generally placed through `ASOverlayLayoutSpec` or `ASBackgroundLayoutSpec`, because only those two layouts can preserve the absolute-layout behavior of `ASAbsoluteLayoutSpec`. For example, if only one control in a view needs `ASAbsoluteLayoutSpec`, while the rest of the controls use `ASStackLayoutSpec` (introduced later), then once `absoluteLayout` is added to `ASStackLayoutSpec`, it loses its original meaning.

```
ASOverlayLayoutSpec *contentLayout = [ASOverlayLayoutSpec overlayLayoutSpecWithChild:stackLayout overlay:absoluteLayout];
```



However, the official documentation clearly says this layout style should be used as little as possible:

>Absolute layouts are less flexible and harder to maintain than other types of layouts.

___

### ASBackgroundLayoutSpec

```
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize{
  ASBackgroundLayoutSpec *backgroundLayout = [ASBackgroundLayoutSpec backgroundLayoutSpecWithChild:self.childNodeB background:self.childNodeA];
  return backgroundLayout;
}
```

Use `childNodeA` as the background of `childNodeB`, meaning `childNodeB` is on top. **Note** that `ASBackgroundLayoutSpec` does not actually change the view hierarchy. For example:

```
ASDisplayNode *childNodeB = [[ASDisplayNode alloc] init];
childNodeB.backgroundColor = [UIColor blueColor];
[self addSubnode:childNodeB];
self.childNodeB = childNodeB;

ASDisplayNode *childNodeA = [[ASDisplayNode alloc] init];
childNodeA.backgroundColor = [UIColor redColor];
[self addSubnode:childNodeA];
self.childNodeA = childNodeA;
```

Even if you use the layout above, `childNodeB` is still underneath.

___

### ASInsetLayoutSpec

This is one of the most commonly used classes. The diagram should make it clear at a glance (image from the [official documentation](http://localhost:4000/2016/12/AsyncDisplayKit/))

![image](/assets/images/Texture/ASInsetLayoutSpec-diagram.png)

```
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize{
    ASInsetLayoutSpec *inset = [ASInsetLayoutSpec insetLayoutSpecWithInsets:UIEdgeInsetsZero child:_childNode];
    return insetLayout;
}
```

`_childNode` has zero margins relative to the parent view, which is equivalent to filling the entire parent view. It and `ASOverlayLayoutSpec`, which will be mentioned later, are mostly used to combine two `Element`s.

___

### ASOverlayLayoutSpec

See `ASBackgroundLayoutSpec`.

___

### ASRatioLayoutSpec

![image](/assets/images/Texture/ASRatioLayoutSpec-diagram.png)

(image from the [official documentation](http://localhost:4000/2016/12/AsyncDisplayKit/))

This is also a commonly used class. Its purpose is to set the aspect ratio of itself, for example to create a square view.

```
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize{
    ASRatioLayoutSpec *ratioLayout = [ASRatioLayoutSpec ratioLayoutSpecWithRatio:1.0f child:self.childNodeA];
    return ratioLayout;
}
```

___

### ASRelativeLayoutSpec

Calling it **corner layout** may not be entirely accurate. In fact, it can place a view at `top-left`, `bottom-left`, `top-right`, and `bottom-right`, and it can also be set to center layout.

![image](/assets/images/Texture/ASRelativeLayoutSpec.jpg)

```
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize{
    self.childNodeA.style.preferredSize = CGSizeMake(100, 100);
    ASRelativeLayoutSpec *relativeLayout = [ASRelativeLayoutSpec relativePositionLayoutSpecWithHorizontalPosition:ASRelativeLayoutSpecPositionEnd verticalPosition:ASRelativeLayoutSpecPositionStart sizingOption:ASRelativeLayoutSpecSizingOptionDefault child:self.childNodeA];
    return relativeLayout;
}
```
The example above places `childNodeA` in the top-right corner.

___

### ASCenterLayoutSpec

In most cases, it is used to center a view.

```
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize{
    self.childNodeA.style.preferredSize = CGSizeMake(100, 100);
    ASCenterLayoutSpec *relativeLayout = [ASCenterLayoutSpec centerLayoutSpecWithCenteringOptions:ASCenterLayoutSpecCenteringXY sizingOptions:ASCenterLayoutSpecSizingOptionDefault child:self.childNodeA];
    return relativeLayout;
}
```

___

### ASStackLayoutSpec

This can be considered **the most commonly used class**, and in terms of functionality it is the closest to `AutoLayout`.
It is called **stack layout** because it is very similar to CSS `Flexbox`. For more about `Flexbox`, you can read Ruan Yifeng's [article](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html?utm_source=tuicool).

Let's look at an example first:

```
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize{
    self.childNodeA.style.preferredSize = CGSizeMake(100, 100);
    self.childNodeB.style.preferredSize = CGSizeMake(200, 200);
    ASStackLayoutSpec *stackLayout = [ASStackLayoutSpec stackLayoutSpecWithDirection:ASStackLayoutDirectionVertical
                                                                             spacing:12
                                                                      justifyContent:ASStackLayoutJustifyContentStart
                                                                          alignItems:ASStackLayoutAlignItemsStart
                                                                            children:@[self.childNodeA, self.childNodeB]];
    return stackLayout;
}
```
Here is a brief explanation of each parameter:

1. `direction`: the direction of the main axis. There are two options:
* Vertical: `ASStackLayoutDirectionVertical`
* Horizontal: `ASStackLayoutDirectionHorizontal`
2. `spacing`: the spacing between views along the main axis. For example, if there are four views, the three gaps between them should all be `spacing`.
3. `justifyContent`: the arrangement along the main axis. There are five options:
* `ASStackLayoutJustifyContentStart` arrange from start to end
* `ASStackLayoutJustifyContentCenter` center the items
    * `ASStackLayoutJustifyContentEnd` arrange from end to start
    * `ASStackLayoutJustifyContentSpaceBetween` distributed with no space at the ends
    * `ASStackLayoutJustifyContentSpaceAround` distributed with space at the ends
4. `alignItems`: the arrangement along the cross axis. There are five options:
* `ASStackLayoutAlignItemsStart` arrange from start to end
* `ASStackLayoutAlignItemsEnd` arrange from end to start
    * `ASStackLayoutAlignItemsCenter` center the items
    * `ASStackLayoutAlignItemsStretch` stretch the items
    * `ASStackLayoutAlignItemsBaselineFirst` align using the first text baseline (only available when the main axis is horizontal)
    * `ASStackLayoutAlignItemsBaselineLast` align using the last text baseline (only available when the main axis is horizontal)
5. `children`: the views to include. The order of elements in the array also represents their layout order, so this matters.

**The main-axis direction is especially important**. If the main axis is set to `ASStackLayoutDirectionVertical`, then the meaning of the `justifyContent` values becomes:

* `ASStackLayoutJustifyContentStart` arrange from top to bottom
* `ASStackLayoutJustifyContentCenter` center the items
* `ASStackLayoutJustifyContentEnd` arrange from bottom to top
* `ASStackLayoutJustifyContentSpaceBetween` distributed with no space at the ends
* `ASStackLayoutJustifyContentSpaceAround` distributed with space at the ends

`alignItems` then becomes:

* `ASStackLayoutAlignItemsStart` arrange from left to right
* `ASStackLayoutAlignItemsEnd` arrange from right to left
* `ASStackLayoutAlignItemsCenter` center the items
* `ASStackLayoutAlignItemsStretch` stretch the items
* `ASStackLayoutAlignItemsBaselineFirst` invalid
* `ASStackLayoutAlignItemsBaselineLast` invalid

A layout method for subviews with different spacing will be covered later in the practical examples.

___

### ASWrapperLayoutSpec

Fill the entire view.

```
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize{    
    ASWrapperLayoutSpec *wrapperLayout = [ASWrapperLayoutSpec wrapperWithLayoutElement:self.childNodeA];
    return wrapperLayout;
}
```



---

### ASCornerLayoutSpec

As the name suggests, `ASCornerLayoutSpec` is suitable for badge-like layouts.

```swift
override func layoutSpecThatFits(_ constrainedSize: ASSizeRange) -> ASLayoutSpec
{
  let cornerSpec = ASCornerLayoutSpec(child: avatarNode, corner: badgeNode, location: .topRight)
  cornerSpec.offset = CGPoint(x: -3, y: 3)
}
```

**The most important thing to note** is that `offset` is the offset of the control's center.


## Layout in Practice

### Example 1
![image](/assets/images/Texture/ASDKDemo1.png)

A simple title over an image, with the text centered.

```
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize{
    ASWrapperLayoutSpec *wrapperLayout = [ASWrapperLayoutSpec wrapperWithLayoutElement:self.coverImageNode];

    ASCenterLayoutSpec *centerSpec = [ASCenterLayoutSpec centerLayoutSpecWithCenteringOptions:ASCenterLayoutSpecCenteringXY sizingOptions:ASCenterLayoutSpecSizingOptionDefault child:self.textNode];
    ASOverlayLayoutSpec *overSpec = [ASOverlayLayoutSpec overlayLayoutSpecWithChild:wrapperLayout overlay:centerSpec];
    return overSpec;
}
```
1. `ASWrapperLayoutSpec` makes the image fill the entire view
2. `ASCenterLayoutSpec` centers the text
3. `ASOverlayLayoutSpec` overlays the text on the image

Note that the third step is the role mentioned earlier for `ASOverlayLayoutSpec` / `ASBackgroundLayoutSpec`: it is used to combine two `Element`s.

### Example 2

![image](/assets/images/Texture/ASDKDemo21.png)

This is the layout of an AppSo channel cell inside the [QingMang Reading](http://www.wandoujia.com/yilan?utm_source=homepage&utm_campaign=routine&utm_medium=internal&utm_content=header) (Wandoujia Yilan) app. It is also one of the more typical layouts. To make it easier to understand, let's name the elements from top to bottom and left to right as follows:

* coverImageNode // large image
* titleNode // title
* subTitleNode // subtitle
* dateTextNode // publish time
* shareImageNode // share icon
* shareNumberNode // share count
* likeImageNode // like icon
* likeNumberNode // like count

```objc
- (ASLayoutSpec *)layoutSpecThatFits:(ASSizeRange)constrainedSize{
    
    
    self.shareImageNode.style.preferredSize = CGSizeMake(15, 15);
    self.likeImageNode.style.preferredSize = CGSizeMake(15, 15);
    
    ASStackLayoutSpec *likeLayout = [ASStackLayoutSpec horizontalStackLayoutSpec];
    likeLayout.spacing = 4.0;
    likeLayout.justifyContent = ASStackLayoutJustifyContentStart;
    likeLayout.alignItems = ASStackLayoutAlignItemsCenter;
    likeLayout.children = @[self.likeImageNode, self.likeNumberNode];
    
    ASStackLayoutSpec *shareLayout = [ASStackLayoutSpec horizontalStackLayoutSpec];
    shareLayout.spacing = 4.0;
    shareLayout.justifyContent = ASStackLayoutJustifyContentStart;
    shareLayout.alignItems = ASStackLayoutAlignItemsCenter;
    shareLayout.children = @[self.shareImageNode, self.shareNumberNode];
    
    ASStackLayoutSpec *otherLayout = [ASStackLayoutSpec horizontalStackLayoutSpec];
    otherLayout.spacing = 12.0;
    otherLayout.justifyContent = ASStackLayoutJustifyContentStart;
    otherLayout.alignItems = ASStackLayoutAlignItemsCenter;
    otherLayout.children = @[likeLayout, shareLayout];
    
    ASStackLayoutSpec *bottomLayout = [ASStackLayoutSpec horizontalStackLayoutSpec];
    bottomLayout.justifyContent = ASStackLayoutJustifyContentSpaceBetween;
    bottomLayout.alignItems = ASStackLayoutAlignItemsCenter;
    bottomLayout.children = @[self.dateTextNode, otherLayout];
    
    self.titleNode.style.spacingBefore = 12.0f;
    
    self.subTitleNode.style.spacingBefore = 16.0f;
    self.subTitleNode.style.spacingAfter = 20.0f;
    
    ASRatioLayoutSpec *rationLayout = [ASRatioLayoutSpec ratioLayoutSpecWithRatio:0.5 child:self.coverImageNode];
    
    ASStackLayoutSpec *contentLayout = [ASStackLayoutSpec verticalStackLayoutSpec];
    contentLayout.justifyContent = ASStackLayoutJustifyContentStart;
    contentLayout.alignItems = ASStackLayoutAlignItemsStretch;
    contentLayout.children = @[
                               rationLayout,
                               self.titleNode,
                               self.subTitleNode,
                               bottomLayout
                               ];
    
    ASInsetLayoutSpec *insetLayout = [ASInsetLayoutSpec insetLayoutSpecWithInsets:UIEdgeInsetsMake(16, 16, 16, 16) child:contentLayout];
    return insetLayout;
}
```

Let's explain the layout in detail, but first it is important to be clear that Texture follows an **inside-out** layout principle. That is what makes it easy to use.

1. Following the layout principle, first use `ASStackLayoutSpec` to lay out the `share icon` and `share count`, as well as the `like icon` and `like count`.
2. Use `ASStackLayoutSpec` again to wrap the two layouts from step 1 and obtain the `otherLayout` object.
3. Use `ASStackLayoutSpec` again to wrap `otherLayout` and the `publish time`. Note that here the horizontal alignment is set to `ASStackLayoutJustifyContentSpaceBetween` to achieve edge-to-edge layout, and the final result is `bottomLayout`.
4. Since the `large image` is a network image, and for a cell the subview layout determines its height (the cell width defaults to the TableNode width), we must set the image height here. `ASRatioLayoutSpec` sets the image's aspect ratio.
5. Next, the layout should be a vertical stack of `large image`, `title`, `subtitle`, and `bottomLayout`. You can see that the spacing between these views is not the same. At this point, `spacingBefore` and `spacingAfter` become very useful, because they set the spacing before and after an element along the main axis. `self.titleNode.style.spacingBefore = 12.0f;` means the `title` has a spacing of 12 relative to the `large image`.
6. Finally, use `ASInsetLayoutSpec` to set padding.

You can see that not only `Node`, but `ASLayoutSpec` itself can also be used as a layout element, because any object that conforms to `<ASLayoutElement>` can be used as a layout element.

### Example 3

![image](/assets/images/Texture/Texture-1.jpg)



```swift
    override func layoutSpecThatFits(_ constrainedSize: ASSizeRange) -> ASLayoutSpec {
        
        self.node1.style.preferredSize = CGSize(width: constrainedSize.max.width, height: 136)
        
        
        self.node2.style.preferredSize = CGSize(width: 58, height: 25)
        self.node2.style.layoutPosition = CGPoint(x: 14.0, y: 95.0)
        
        self.node3.style.height = ASDimensionMake(37.0)
        self.node4.style.preferredSize = CGSize(width: 80, height: 20)
        self.node5.style.preferredSize = CGSize(width: 80, height: 20)
        
        self.node4.style.spacingBefore = 14.0
        self.node5.style.spacingAfter = 14.0
        
        let absoluteLayout = ASAbsoluteLayoutSpec(children: [self.node2])
        
        let overlyLayout = ASOverlayLayoutSpec(child: self.node1, overlay: absoluteLayout)
        
        let insetLayout = ASInsetLayoutSpec(insets: UIEdgeInsetsMake(0, 14, 0, 14), child: self.node3)
        insetLayout.style.spacingBefore = 13.0
        insetLayout.style.spacingAfter = 25.0
        
        
        let bottomLayout = ASStackLayoutSpec.horizontal()
        bottomLayout.justifyContent = .spaceBetween
        bottomLayout.alignItems = .start
        bottomLayout.children = [self.node4, self.node5]
        bottomLayout.style.spacingAfter = 10.0
//        bottomLayout.style.width = ASDimensionMake(constrainedSize.max.width)
        
        
        let stackLayout = ASStackLayoutSpec.vertical()
        stackLayout.justifyContent = .start
        stackLayout.alignItems = .stretch
        stackLayout.children = [overlyLayout, insetLayout, bottomLayout]
        
        return stackLayout
    }
```

To demonstrate `ASAbsoluteLayoutSpec`, here we use it for `node3`.

Key points:

1. Both nodes and layout specs can set the `style` property, because they both conform to `ASLayoutElement`
2. When `spaceBetween` does not achieve the desired edge-to-edge alignment, try setting the current layout spec's `width` (as shown in the comment) or the `alignItems` of its parent layout object. In this example, that is `stackLayout.alignItems = .stretch`
3. `ASAbsoluteLayoutSpec` must have an anchor point, unless the entire layout is absolute. In this example, the anchor point for `ASAbsoluteLayoutSpec` is `ASOverlayLayoutSpec`

### Example 4

![image](/assets/images/Texture/Texture-3.jpg)



This example is mainly used to demonstrate `flexGrow`. First, let's introduce what `flexGrow` does (from the Jianshu article [Nine-Color Platter](http://www.jianshu.com/p/0642dfe0e571)):

> This property controls how child elements allocate the remaining space of the parent when the parent is wider than the total width of all child elements.
>
> The default value of flex-grow is 0, which means the element does not claim any of the parent’s remaining space. If the value is greater than 0, the element does claim it. The larger the value, the more remaining space it claims. For example:
>
> Suppose the parent is 400px wide, and there are two child elements A and B. A is 100px wide and B is 200px wide, so the remaining space is 400 - (100 + 200) = 100px.
>
> If neither A nor B claims the remaining space, then there is 100px of empty space.
>
> If A claims the remaining space and flex-grow is set to 1 while B does not claim any, then the final size of A is its own width (100px) + the remaining space (100px) = 200px.
>
> If both A and B claim the remaining space, with A's flex-grow set to 1 and B's set to 2, then the final size of A is its own width (100px) + the remaining space it receives (100px * (1/(1+2))), and the final size of B is its own width (200px) + the remaining space it receives (100px * (2/(1+2))).

```swift
     override func layoutSpecThatFits(_ constrainedSize: ASSizeRange) -> ASLayoutSpec {
        
        self.node1.style.height = ASDimensionMake(20.0)
        var imageLayoutArray = [ASLayoutElement]()
        
        [self.node2, self.node3, self.node4].forEach { (node) in
            let layout = ASRatioLayoutSpec(ratio: 2.0/3.0, child: node)
            layout.style.flexGrow = 1 // equivalent to equal widths
            imageLayoutArray.append(layout)
        }
        
        let imageLayout = ASStackLayoutSpec.horizontal()
        imageLayout.justifyContent = .start
        imageLayout.alignItems = .start
        imageLayout.spacing = 14.0
        imageLayout.children = imageLayoutArray
        
        let contentLayout = ASStackLayoutSpec.vertical()
        contentLayout.justifyContent = .start
        contentLayout.alignItems = .stretch
        contentLayout.spacing = 22.0
        contentLayout.children = [self.node1, imageLayout]
        
        return ASInsetLayoutSpec(insets: UIEdgeInsetsMake(22.0, 16.0, 22.0, 16.0), child: contentLayout)
    }
```

In this example, the total width of `node2`, `node3`, and `node4` is smaller than the parent width. So to make them the same width, we only need to set the same `flexGrow` value for all three (all 1), and then use `ASRatioLayoutSpec` to fix each aspect ratio. That way, the final width of these three controls is determined.

### Example 5

![image](/assets/images/Texture/Texture-2.jpg)



This example is mainly used to demonstrate `flexShrink`. It also draws from the Jianshu article [Nine-Color Platter](http://www.jianshu.com/p/0642dfe0e571) for the explanation of `flexShrink`:

> This property controls how child elements shrink their width when the parent is narrower than the total width of all child elements.
>
> The default value of flex-shrink is 1. When the parent width is smaller than the sum of all child widths, the child widths will shrink. The larger the value, the more they shrink. If the value is 0, they do not shrink.
>
> For example: the parent is 400px wide, and there are two child elements A and B. A is 200px wide and B is 300px wide. Then the total overflow is (200 + 300) - 400 = 100px.
>
> If neither A nor B shrinks, that is, both have flex-shrink set to 0, then 100px will overflow the parent. If A does not shrink, with flex-shrink set to 0 and B does shrink, then the final size of B is its own width (300px) - the total overflow (100px) = 200px. If both A and B shrink, with A's flex-shrink set to 3 and B's set to 2, then the final size of A is its own width (200px) - the amount it shrinks (100px * (200px * 3/(200 * 3 + 300 * 2))) = 150px, and the final size of B is its own width (300px) - the amount it shrinks (100px * (300px * 2/(200 * 3 + 300 * 2))) = 250px.

At present, the most common use of this property is limiting text width. In the image above, `textNode` and `displayNode` are aligned at both ends, and the text's maximum width needs to be constrained. In that case, setting `flexShrink` is the most convenient approach.

```swift
override func layoutSpecThatFits(_ constrainedSize: ASSizeRange) -> ASLayoutSpec {
        self.displayNode.style.preferredSize = CGSize(width: 42.0, height: 18.0)
        self.textNode.style.flexShrink = 1
        
        let contentLayout = ASStackLayoutSpec.horizontal()
        contentLayout.justifyContent = .spaceBetween
        contentLayout.alignItems = .start
        contentLayout.children = [self.textNode, self.displayNode]
        
        let insetLayout = ASInsetLayoutSpec(insets: UIEdgeInsetsMake(16.0, 16.0, 16.0, 16.0), child: contentLayout)
        
        return insetLayout
        
    }
```

**As a side note, if `ASTextNode` shows unexplained text truncation issues, you can use `ASTextNode2` instead.**

### Example 6

A fairly typical example.

![image](/assets/images/Texture/ASDKDemo3.png)

```swift
override func layoutSpecThatFits(_ constrainedSize: ASSizeRange) -> ASLayoutSpec {

        let otherLayout = ASInsetLayoutSpec(insets: UIEdgeInsetsMake(10.0, 10.0, CGFloat(Float.infinity), CGFloat(Float.infinity)), child: topLeftNode)
        
        let contentLayout = ASOverlayLayoutSpec(child: coverImageNode, overlay: otherLayout)
        return contentLayout
    }
```

Using `ASInsetLayoutSpec` is the best solution. It is worth noting that for the red control, you only need to set the top and left spacing; the other directions can be replaced with `CGFloat(Float.infinity)` and do not need specific values.


Finally, the examples above have been uploaded to [TextureLayoutDemo](https://github.com/bawn/TextureLayoutDemo).
