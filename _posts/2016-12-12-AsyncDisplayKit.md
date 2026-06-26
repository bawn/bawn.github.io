---
layout: post
title: "A New World: Texture"
date: 2016-12-12
comments: true
categories: [Texture]
tags: [iOS]
keywords: [Texture, iOS]
publish: true
description: Texture
---

APP performance optimization has always been a long and difficult road, and it is even more important for modern apps that need to handle more and more information. Fortunately, Apple has done at least a better job than Android in this area, making developers' lives easier. Although UIKit controls can satisfy smoothness requirements in most cases, sometimes they still fail to achieve the ideal result.

[AsyncDisplayKit](https://github.com/facebook/AsyncDisplayKit) (hereafter referred to as ASDK) gave developers another good option. After all, [Paper](https://en.wikipedia.org/wiki/Facebook_Paper) (although Facebook has already shut down the app) still managed to deliver cool effects while keeping good smoothness, thanks in part to ASDK. A few months after [Paper](https://en.wikipedia.org/wiki/Facebook_Paper) was released, Facebook split it out into an independent library. Just two days ago, ASDK released version 2.0. As far as I know, some of the better-known domestic apps using ASDK include [QingMang Reading](http://www.wandoujia.com/yilan?utm_source=homepage&utm_campaign=routine&utm_medium=internal&utm_content=header) (Wandoujia Yilan), [Jike](http://www.ruguoapp.com/), [Yep](https://itunes.apple.com/cn/app/id983891256?mt=8&l=cn), [Xiaohongshu](https://www.xiaohongshu.com/), and [Pipixia](https://itunes.apple.com/cn/app/%E7%9A%AE%E7%9A%AE%E8%99%BE-%E4%BB%8A%E6%97%A5%E5%A4%B4%E6%9D%A1%E5%AE%98%E6%96%B9%E7%88%86%E7%AC%91%E7%A4%BE%E5%8C%BA/id1393912676).

**At present, [AsyncDisplayKit](https://github.com/facebook/AsyncDisplayKit) has already migrated from Facebook to TextureGroup, and the new project address is [Texture](https://github.com/TextureGroup/Texture).**

## Controls

Texture covers almost all commonly used controls. Below is the mapping between `Texture` and `UIKit`, and some of the wrappers are really thoughtful.

**Nodes:**

| Texture              | UIKit                                |
| :------------------- | :----------------------------------- |
| ASDisplayNode        | UIView                               |
| ASCellNode           | UITableViewCell/UICollectionViewCell |
| ASTextNode           | UILabel                              |
| ASImageNode          | UIImageView                          |
| ASNetworkImageNode   | UIImageView                          |
| ASVideoNode          | AVPlayerLayer                        |
| ASControlNode        | UIControl                            |
| ASScrollNode         | UIScrollView                         |
| ASControlNode        | UIControl                            |
| ASEditableTextNode   | UITextView                           |
| ASMultiplexImageNode | UIImageView                          |

**Node Containers**

| Texture          | UIKit            |
| :--------------- | :--------------- |
| ASViewController | UIViewController |
| ASTableNode      | UITableView      |
| ASCollectionNode | UICollectionView |
| ASPagerNode      | UICollectionView |

Inheritance hierarchy:

* ASDisplayNode
  * ASCellNode
    * ASTextCellNode
  * ASCollectionNode
    * ASPagerNode
  * ASControlNode
    * ASButtonNode
    * ASImageNode
      * ASMapNode
      * ASMultiplexImageNode
      * ASNetworkImageNode
        * ASVideoNode
    * ASTextNode
    * ASTextNode2
  * ASEditableTextNode
  * ASScrollNode
  * ASTableNode
  * ASVideoPlayerNode

### ASDisplayNode:

Its role is equivalent to `UIView`. It is the parent class of all nodes, and `ASDisplayNode` also has a `view` property, so `ASDisplayNode` and its subclasses can add `UIKit` controls through that `view`.

Adding `UIKit` inside `ASDisplayNode`:

```
UIView *otherView = [[UIView alloc] init];
otherView.frame = ...;
[node.view addSubview:otherView];
```
or

```
ASDisplayNode *gradientNode = [[ASDisplayNode alloc] initWithViewBlock:^UIView * _Nonnull{
    UIView *view = [[UIView alloc] init];
    return view;
}];
```
The second initializer ultimately produces the `UIKit` object returned by the block, but what is exposed externally is still an `ASDisplayNode`. The advantage of this is layout, which I will talk about later.

Adding `ASDisplayNode` inside `UIKit`:

```
ASImageNode *imageNode = [[ASImageNode alloc] init];
imageNode.image = [UIImage imageNamed:@"iconShowMore"];
imageNode.frame = ...;
[self addSubnode:imageNode];
self.imageNode = imageNode;
```

### ASCellNode:

Its role is equivalent to `UITableViewCell` or `UICollectionViewCell`. It comes with an `indexPath` property, which can be very useful sometimes.

### ASTextNode

Its role is equivalent to `UILabel`. Unlike `UILabel`, `ASTextNode` must use `attributedText` to display text.

### ASTextNode2

It fixes some bugs based on ASTextNode.

### ASImageNode

Its role is equivalent to `UIImageView`, but it can only display static images. If you need to use network images, use `ASNetworkImageNode`.

### ASNetworkImageNode

Its role is equivalent to `UIImageView`. If you are using network images, you should use this class. Texture uses the third-party image loading library [PINRemoteImage](https://github.com/pinterest/PINRemoteImage). `ASNetworkImageNode` does not actually support GIFs. If you need to display GIFs, it is recommended to use [FLAnimatedImage](https://github.com/Flipboard/FLAnimatedImage).

### ASButtonNode

Its role is equivalent to `UIButton`. Pay attention to the following two properties:

```
@property (nonatomic, assign) CGFloat contentSpacing;// set the spacing between the image and text
@property (nonatomic, assign) ASButtonNodeImageAlignment imageAlignment;// set the arrangement of image and text
```
Honestly, this can be a little painful 😭. `imageAlignment` has two values:

```
ASButtonNodeImageAlignmentBeginning, // image first, text after
ASButtonNodeImageAlignmentEnd// text first, image after
```

### ASTableNode

Its role is equivalent to `UITableView`, but the implementation does not use the `UITableView` reuse mechanism. Instead, it adds views that need to be shown and removes those that do not as the user scrolls (this is my guess). Another important point: unlike `UITableView`, `ASTableNode` does not provide a `-tableView:heightForRowAtIndexPath:`-style delegate method to determine each cell's height. Instead, the height is determined by `ASCellNode` itself. **Another benefit is that implementing dynamic height becomes extremely easy.** For details, see the official demo [Kittens](https://github.com/facebook/AsyncDisplayKit/tree/master/examples/Kittens).


## Layout

Read [Texture Layout](https://bawn.github.io/posts/Texture-Layout/)


## Drawbacks

Of course, Texture is not perfect. After using it in a project, I summarized the following drawbacks:

1. Although it can be mixed with UIKit, some components still have to be wrapped again in Texture form
2. Because the underlying implementation is different, some third-party libraries are no longer applicable, such as [DZNEmptyDataSet](https://github.com/dzenbot/DZNEmptyDataSet)
3. Since rendering, image decoding, layout, and so on do not happen on the main thread, they inevitably bring issues such as thread safety, deadlocks, and difficult debugging
4. The layout system has a slightly higher learning curve and the layout is verbose
5. It is intrusive and difficult to remove later
6. [Flickering issues](https://juejin.im/post/5987cc536fb9a03c4b374bec)
7. It does not support Interface Builder
8. It is still not fully mature, for example [[Crash] ASDN::Mutex::~Mutex()](https://github.com/TextureGroup/Texture/issues/136) (reported on August 1, 2016 and not fixed until September 1, 2017)


## Other Topics

**Refreshing lists**



Whether it is `ASTableNode` or `ASCollectionNode`, when the list already has data and you call `reloadData`, you will notice that the list flashes. A common case is loading more data at the bottom and then calling `reloadData`, which creates a poor user experience. In fact, the official documentation provides a solution in the [[Batch Fetching API](http://asyncdisplaykit.org/docs/batch-fetching-api.html)]:

```objc
- (void)tableNode:(ASTableNode *)tableNode willBeginBatchFetchWithContext:(ASBatchContext *)context 
{
  // Fetch data most of the time asynchronously from an API or local database
  NSArray *newPhotos = [SomeSource getNewPhotos];

  // Insert data into table or collection node
  [self insertNewRowsInTableNode:newPhotos];

  // Decide if it's still necessary to trigger more batch fetches in the future
  _stillDataToFetch = ...;

  // Properly finish the batch fetch
  [context completeBatchFetching:YES];
}
```

After getting new data, insert it directly into the list instead of refreshing the entire list, for example:

```objc
- (void)insertSections:(NSIndexSet *)sections withRowAnimation:(UITableViewRowAnimation)animation;
```

and

```objc
- (void)insertRowsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths withRowAnimation:(UITableViewRowAnimation)animation;
```

**Loading more data**

Careful readers may have noticed that the related method was mentioned earlier:

```
- (void)tableNode:(ASTableNode *)tableNode willBeginBatchFetchWithContext:(ASBatchContext *)context;
```

Most apps now load more data by scrolling to the bottom of the list and then requesting more data before appending it to the list. Texture, however, provides another, more "reasonable" way. The original description is:

> By default, as a user is scrolling, when they approach the point in the table or collection where they are 2 “screens” away from the end of the current content, the table will try to fetch more data.

When the list scrolls to within two screen heights of the bottom, it requests new data. This threshold can be adjusted. Once the list reaches that two-screen distance from the bottom, the method mentioned earlier will be called. So it works roughly like this:

```
- (void)tableNode:(ASTableNode *)tableNode willBeginBatchFetchWithContext:(ASBatchContext *)context{
    [context beginBatchFetching];
    [listApi startWithBlockSuccess:^(HQHomeListApi *request) {
        @strongify(self);
        NSArray *array = [request responseJSONObject];
        [self.dataSourceArray addObjectsFromArray:array];
        [self.tableNode insertSections:[NSIndexSet indexSetWithIndexesInRange:rang] withRowAnimation:UITableViewRowAnimationNone];
        [self updateHavMore:array];
        [context completeBatchFetching:YES];
    } failure:NULL];
}

- (BOOL)shouldBatchFetchForTableNode:(ASTableNode *)tableNode{
    return self.haveMore;
}
```

`shouldBatchFetchForTableNode` is used to control whether more data should be fetched. The advantage of this approach is that when the network is good, users never feel that other data has already been loaded and displayed. The downside is that when the network is poor, even if the list has already been pulled all the way to the bottom, there is still no prompt.
