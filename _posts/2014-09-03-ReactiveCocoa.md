---
layout: post
title: "ReactiveCocoa"
date: 2015-03-26
comments: true
categories: iOS
tags: [ReactiveCocoa]
keywords: ReactiveCocoa functional programming reactive programming
description: Notes and takeaways about ReactiveCocoa
---
I started following [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) a long time ago, and recently decided to introduce it into a project for the following reasons:

* It was a good opportunity to get familiar with the reactive programming (functional programming) model
* After iterations from version 0.0.1 to 2.3.1, the framework had become relatively mature
* It was a chance to try the MVVM pattern


From the time I first started learning `ReactiveCocoa` until now, I sometimes felt that I was not using it to its full potential. For example:

```
@weakify(self);
[[self.nextButton rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(id x) {
     @strongify(self);
     [self performSegueWithIdentifier:@"Captcha" sender:nil];
}];
```
This kind of one-way signal delivery and subscription has no relationship with other signals, so it feels no different from the `action-target` pattern. It seems to drift away from the original purpose of reactive programming, although that may be a bit extreme.

## Countdown Feature

Let's start with the very common verification-code countdown feature. With RAC, it can be implemented almost in one go.
![Verification code countdown](/assets/images/ReactiveCocoa/ReactiveCocoa_1.jpg)
Clicking the button can restart the countdown.

```
  @weakify(self);
  RACSignal *timeSignal = [[[[[RACSignal interval:1.0f onScheduler:[RACScheduler mainThreadScheduler]] take:numberLimit] startWith:@(1)] map:^id(NSDate *date) {
      @strongify(self);
      if (number == 0) {
          [self.timeButton setTitle:@"Resend" forState:UIControlStateNormal];
          return @YES;
      }
      else{
          self.timeButton.titleLabel.text = [NSString stringWithFormat:@"%d", number--];
          return @NO;
      }
  }] takeUntil:self.rac_willDeallocSignal];
  self.timeButton.rac_command = [[RACCommand alloc]initWithEnabled:timeSignal signalBlock:^RACSignal *(id input) {
      number = numberLimit;
      return timeSignal;
  }];
```

* `+ (RACSignal *)interval:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler` returns a signal that fires once every second on the main thread. Internally, RAC uses a series of GCD APIs such as `dispatch_source_set_timer`, `dispatch_source_set_event_handler`, and `dispatch_resume` to implement it. Note that the value sent by this signal is an `NSDate` object representing the current date, but that value is not used by the feature itself.
* `- (instancetype)take:(NSUInteger)count` takes the first `numberLimit` `sendNext` values, which is equivalent to how long we want the countdown to last.
* `- (instancetype)startWith:(id)value` changes the first `sendNext` value and immediately sends that value. In theory, the actual `sendNext` value does not matter, because the signal returned by `+ (RACSignal *)interval:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler` waits one second before firing. `startWith` is used simply to start the countdown immediately, so the parameter value of `startWith:` is not important.
* `- (instancetype)map:(id (^)(id value))block` changes the final `sendNext` value to return `@YES` or `@NO`.
* `- (RACSignal *)takeUntil:(RACSignal *)signalTrigger` means that the countdown stops when this VC is about to deallocate. Internally, RAC just sends `completed`.
* Finally, bind `timeSignal` to `timeButton.rac_command` to determine whether the button is enabled, and start `timeSignal`. When the button is clicked, restore the value of `number`, and `return buttonSignal` to restart the countdown `timeSignal`.

____

## Replacing the Delegate Pattern

If RAC is used well, you can almost stop writing custom delegate protocols. Imagine this: when aVC `presentViewController`'s bVC, and then `dismissViewControllerAnimated` is called on bVC, aVC needs to do something afterward. A delegate protocol can solve this, but it is somewhat cumbersome. Using RAC makes the implementation much cleaner and clearer.

The key method here is `- (RACSignal *)rac_signalForSelector:(SEL)selector`, and basic usage looks like this:

```
[[self rac_signalForSelector:@selector(dismiss:)] subscribeNext:^(id x) {
	NSLog(@"%s", __func__);
}];

```

```
- (IBAction)dismiss:(id)sender{
    [self dismissViewControllerAnimated:YES completion:NULL];
}
```
When `dismiss:` is executed, the signal is subscribed to.


We get to decide when the signal is subscribed to. Since some work needs to happen in aVC, we should find a way to subscribe to the signal in aVC. At that point, the signal needs to be exposed as a property of bVC.

> bVC.h

```
#import <UIKit/UIKit.h>
@interface RACDelegateViewController : UIViewController
@property (nonatomic, strong) RACSignal *delegateSignal;
@end
```

Subscribing to the signal
> aVC.m

```
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender{
    if ([segue.destinationViewController isKindOfClass:[UINavigationController class]]) {
        UINavigationController *nav = (UINavigationController *)segue.destinationViewController;
        RACDelegateViewController *delegateVC = (RACDelegateViewController *)nav.topViewController;
        [delegateVC.delegateSignal subscribeNext:^(id x) {
            NSLog(@"%@", x);
        }];
    }
}
```
There is one last step: create `delegateSignal`.
> bVC.m

```
- (void)awakeFromNib{
    [super awakeFromNib];
    self.delegateSignal = [self rac_signalForSelector:@selector(dismiss:)];
}
```

Note: never create `delegateSignal` in `- (void)viewDidLoad;`. If you do, `delegateSignal` will still be `nil` when `subscribeNext` is executed.

It is still not perfect, right? The data received in aVC is always a `UIBarButtonItem` object. How can data from bVC be passed to aVC instead? We still need to adjust the signal. The `- (RACSignal *)then:(RACSignal * (^)(void))block;` method ignores all `next` values from the receiver until the signal completes, then returns a new `RACSignal`. Example:

```
- (void)awakeFromNib{
    [super awakeFromNib];
    @weakify(self);
    self.delegateSignal = [[self rac_signalForSelector:@selector(dismiss:)] then:^RACSignal *{
        @strongify(self);
        return [RACSignal return:self.array];
    }];
}
```

When the signal `[self rac_signalForSelector:@selector(dismiss:)]` sends `complete`, a block is executed, and that block must return a non-nil new `RACSignal`. Of course, there are other approaches as well, such as using `map` on the signal:

```
- (void)awakeFromNib{
    [super awakeFromNib];
    @weakify(self);
    self.delegateSignal = [[self rac_signalForSelector:@selector(dismiss:)] map:^id(id value) {
        @strongify(self);
        return self.array;
    }];
}
```

If you need to listen to protocol methods, you can use `- (RACSignal *)rac_signalForSelector:(SEL)selector fromProtocol:(Protocol *)protocol`. For details, see my [shared notes](https://www.evernote.com/shard/s147/sh/a01d60f5-36d3-43ad-9715-23fb19e4ae78/c4a9520d5d02767eea89a8c4cf005b4a).

____

## Automatic Data Updates

__Note: this feature is only suitable for UITableView and UICollectionView whose `row` and `section` counts remain unchanged.__

Some time ago, I read two objc.io articles about lighter [View Controllers](http://objccn.io/issue-1-1/) and cleaner [Table View](http://objccn.io/issue-1-1/) code. I also tried combining ReactiveCocoa with UITableView. Here are some of my experiments.

* __KVO data source array__

```
- (id)initWithCellIdentifier:(NSString *)aCellIdentifier
          configureCellBlock:(CellConfigureBlock)aConfigureCellBlock{
    self = [super init];
    if (self) {
        _cellIdentifier = aCellIdentifier;
        _configureCellBlock = [aConfigureCellBlock copy];
        _signal = RACObserve(self, dataSource);
    }
    return self;
}
```

* __Use `map` on the data source to obtain a single data model__

```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:_cellIdentifier
                                                            forIndexPath:indexPath];
    if (self.configureCellBlock) {
        @weakify(self);
        self.configureCellBlock(cell, [self.signal map:^id(NSArray *array) {
            @strongify(self);
            return [self itemAtIndexPath:indexPath];
        }]);
    }
    return cell;
}
```

* __Usage in the VC__

```
self.model = [[DataSourceGeneralModel alloc] initWithCellIdentifier:CelIdentifier
                                                 configureCellBlock:^(TableViewCell *cell, RACSignal *signal) {
        [cell configureCellWithSignal:signal];
    }];
```

* __Configuration in the cell__

```
- (void)configureCellWithSignal:(RACSignal *)signal{
    @weakify(self);
    [signal subscribeNext:^(Model *model) {
        @strongify(self);
        self.titleLabel.text = model.title;
        self.detailLabel.text = model.detail;
    }];
}
```

If the data source array changes, UITableView updates accordingly. There is no need to call `[self.tableView reloadData]`; just update it like this:

```
self.model.dataSource = @[model1];
```

#### You can download the examples above from [GitHub](https://github.com/bawn/RAC-Demo)

____

## Replacing `dispatch_group`

**Traditional `dispatch_group` setup and usage**

```

// Create a dispatch_group object
static dispatch_group_t home_request_operation_completion_group() {
    static dispatch_group_t http_request_operation_completion_group;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        http_request_operation_completion_group = dispatch_group_create();
    });
    return http_request_operation_completion_group;
}
// .......
- (void)viewDidLoad{
    [self updateData];
}
- (void)updateProfit{
// Add one asynchronous method
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_async(home_request_operation_completion_group(), queue, ^{
    dispatch_group_enter(home_request_operation_completion_group());
        [self update1];
    });
// Add another asynchronous method
    dispatch_group_async(home_request_operation_completion_group(), queue, ^{
    dispatch_group_enter(home_request_operation_completion_group());
        [self update2];
    });
// Stop pull-to-refresh when both asynchronous methods have finished
    dispatch_group_notify(home_request_operation_completion_group(), dispatch_get_main_queue(), ^{
        if ([self.tableView isHeaderRefreshing]) {
            [self.tableView headerEndRefreshing];
        });
};
```

**Using RAC instead of `dispatch_group`**

```
- (void)updateProfit{
    [[RACSignal zip:@[[self update1], [self update2]]] subscribeNext:^(RACTuple *x) {
        @strongify(self);
        if ([self.tableView isHeaderRefreshing]) {
            [self.tableView headerEndRefreshing];
        });
    }];
```

Of course, `[self update1]` and `[self update2]` need to be rewritten to return `RACSignal` objects here.

`+ (instancetype)zip:(id<NSFastEnumeration>)streams` waits until all bound signals have sent `next` before subscribing to the combined result.

For the difference between `combineLatest` and `zip`, see my [notes](https://www.evernote.com/l/AJMEYyw-2oxEQ7Pv0uxJ4faAgCyhXgjRnzs).


