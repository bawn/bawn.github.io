---
layout: post
title: "Analytics Instrumentation"
date: 2019-01-11
comments: true
categories: [iOS]
tags: [analytics]
keywords: [analytics]
publish: true
description: analytics instrumentation
---

This article mainly shares some of the lessons I learned while implementing analytics instrumentation in a company project. In the early stage of the project, we relied entirely on third-party automatic analytics tracking. Client developers only needed to do a small amount of work to satisfy the BI team's data requirements. But as the business grew, the requirements for data accuracy and granularity became higher and higher, so we had to switch to manual tracking based on [Sensors Analytics](https://www.sensorsdata.cn).

In many cases, we need to choose an analytics approach based on the specific business. In the [Huoqiu Buyer](https://itunes.apple.com/cn/app/%E7%81%AB%E7%90%83%E4%B9%B0%E6%89%8B-%E5%B8%AE%E4%BD%A0%E9%80%89%E5%A5%BD%E8%B4%A7/id1070842761?mt=8) project, the BI team's requirement for tracking data can be summarized in one sentence: "from where to where." For example, if a user taps an article in the Timeline and enters the detail page, then the Timeline is the "from where" and the detail page is the "to where." Of course, "from where" is not always identifiable with just one dimension; sometimes it takes two or three dimensions to locate it precisely.

Here are some concrete examples. First, the "channel home page" is a common page in the project, and its corresponding model is Channel. When any event for entering a channel home page is triggered, the following data must be reported:

```
{
    module_name
    page_name
    channel_name
    channel_id
}
```

`page_name` refers to the current ViewController name. `module_name` is mainly used to distinguish different entry points on the same page, so that we can identify the "from where." `channel_name` and `channel_id` come from Channel. As for the "to where," we express it through the tracking key, such as ChannelClick.

The channel home page has many entry points in the app. Even without considering analytics, a unified entry method is necessary:

```swift
extension UIViewController {
    func pushToChannelDetailController(_ id: String?) {
        // ...
    }
}
```

Clearly, this method cannot satisfy analytics requirements, so let's refactor it:

```swift
extension UIViewController {
    func pushToChannelDetailController(_ model: Channel?) {
        // ...
        let value = [
            "module_name" : model.module_name,
            "channel_id" : model._id,
            "channel_name": model.name,
            "page_name": self.pageName
        ]
        SensorsAnalyticsSDK.sharedInstance()?.track(key, withProperties: value)
    }
}
```

In most cases, it does not work to make the entry method accept only one concrete parameter, because as the project evolves, other models will be introduced, and they may also contain an id that can navigate to the channel home page. I will come back to this later. Next is pageName:

```swift
extension UIViewController {
    var pageName: String {
        switch self {
			case is ChannelDetailController:
	            return "Channel Home"
        }
    }
}
```

Finally, set module_name at the specific navigation point. In fact, module_name does not belong to Channel. I will explain later why it needs to be bound to the model:

```swift
@objc func buttonAction(_ sender: Any) {
    model.module_name = "Header"
    pushToChannelDetailController(model)
}
```

Overall, this can handle analytics for entering the channel home page, but there are still the following problems:

#### The entry method is not abstract enough

In real development, it is common to receive different data models and navigate to the same page. Because analytics forced the entry method to replace a relatively abstract String with a concrete Channel, abstracting the model is the first thing to do. No matter what type of parameter is accepted, the actual navigation only needs one field: id. So it makes much more sense to constrain models with a protocol that only requires an id property.

```swift
protocol CommonModelType {
    var id: String { get }
}
```

Then make Channel conform to this protocol. Using an extension makes it look more decoupled:

```swift
extension Channel: CommonModelType{}
```

At this point, the entry method becomes:

```swift
func pushToChannellDetail(_ model: CommonModelType?)
```

It can accept any model with an id property. To make it even more abstract, you can even let String conform to this protocol:

```swift
extension String: CommonModelType {
    var id: String {
        return self
    }
}
```



#### The data provisioning approach is not elegant enough

Except for pageName, the analytics data belongs to the model itself. `module_name` is an extra field, but it is still part of the tracked data. So, just like the parameters, we can use a protocol to constrain Channel and give it a property that directly provides analytics data.

```swift
protocol AnalyticsModelType {
    var analytics: [String: Any] { get }
}
```

Let Channel conform to both protocols and add an analytics property:

```swift
extension Channel: CommonModelType, AnalyticsModelType {
    var analytics: [String : Any] {
        let value = [
            "module_name" : module_name,
            "channel_id" : id,
            "channel_name": name
        ]
        return value
    }
}
```

The final entry method looks like this:

```swift
extension UIViewController {

    func pushToChannellDetail(_ model: CommonModelType?) {
        guard let model = model else {
            return
        }
        let viewController = ChannelDetailViewController()
        viewController.id = model.id
        navigationController?.pushViewController(viewController, animated: true)
        
        guard let value = model as? AnalyticsModelType else {
            return
        }
        var properties = value.analytics
        properties["page_name"] = pageName
        SensorsAnalyticsSDK.sharedInstance.track(key: "ChannelClick", properties: properties)
    }
}
```

Do not forget `module_name`. As mentioned earlier, `module_name` is mainly used to distinguish different entry points on the same page. For example, entry points A and B on the page both trigger the ChannelClick event, and their module_name values are "A" and "B". Looking back at where `module_name` is set above, it is clear that module_name is tightly coupled to business logic:

```swift
@objc func buttonAction(_ sender: Any) {
    model.module_name = "You may want to follow"
    pushToChannelDetailController(model)
}
```

The most frustrating part in the early design of an analytics solution is how to handle module_name. Semantically, module_name should belong to the view layer and not be bound to the model. But after some thought, binding module_name to the model can still achieve low coupling, because in most cases module_name is already known after the data returns. For example, the module_name "Home Timeline" can be bound directly to the corresponding model after fetching server data. If you think about module division on a page from the perspective of the data structure, it is already clearly defined. In other words, module_name can even be provided directly by the server. For example, set module_name after the server response comes back:

```swift
    func brandFeed() -> [HQBrandList] {
        ...
        let listValue = Mapper<BrandList>().mapArray(JSONArray: timeline)
        listValue.forEach { (item) in
            item.reviews.forEach({$0.module_name = "xxxxx" + item.title})
        }
        return listValue
    }
```

In this way, module_name can be set like this in most cases. Yes, only most cases. I will mention the remaining cases later.



In short, the entry method is abstract enough. No matter how many model types are added later, as long as they conform to `CommonModelType`, navigation will still work. Even for someone unfamiliar with the project, passing in an id directly can still navigate correctly. The analytics details are hidden inside the entry method, and the data that needs to be reported is provided by the corresponding model as long as it conforms to `AnalyticsModelType`. module_name can be set where the corresponding data is returned.



### Browse events

In addition to the click events like ChannelClick described above, we also have browse events, such as browsing the channel detail page: ChannelBrowse. This event is triggered when the channel detail page is fully displayed, and it needs to carry the following information:

```
{
    channel_name
    channel_id
    previous_page_name
    previous_module_name
}
```

`previous_page_name` represents the name of the previous page, and `previous_module_name` is the module_name from the previous page. I will not discuss the exact difference between this and ChannelClick here. We can easily get previous_page_name from navigationController?.viewControllers. Based on this logic, I once made a wrong decision: I used an array to store module_name. The array was initialized with N empty module_name values. Whenever a navigation event occurred, I set the module_name at the current index. Later, whenever I needed it, I would use the current viewController's index to fetch previous_module_name from the array. However, not all navigation events refresh module_name, which means module_name was not reset to empty. There was also the issue of parentViewController. In short, the singleton-array approach had too many drawbacks.

In general, for events like ChannelBrowse, getting previous_module_name is the key. Besides the approach mentioned above, binding previous_module_name to the current viewController seems like a better solution. Specifically, when navigating between pages, directly bind the current module_name to the target viewController.

```swift
extension UIViewController {
    var previousValue: (String, String) {
        get {
            return (objc_getAssociatedObject(self, &AssociatedKeys.PreviousValue) as? (String, String)) ?? ("", "")
        }
        set {
            objc_setAssociatedObject(self, &AssociatedKeys.PreviousValue, newValue, .OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        }
    }
}
```

Bind data during navigation:

```swift
func pushToReviewDetailController(_ model: HQCommonModelType?) {
        guard let model = model else {
            return
        }
        SensorsAnalyticsSDK.track(key: .reviewClick(model), page: self)

        let viewController = ReviewDetailViewController.instantiateFromSB
        viewController.reviewId = model._id
			let module_name = (model as? AnalyticsModelType)?.module_name ?? ""
        viewController.previousValue = (pageName, module_name)
        navigationController?.pushViewController(viewController, animated: true)
    }
```

Trigger the ChannelBrowse browse event:

```swift
SensorsAnalyticsSDK.track(key: .channelBrowse(model), page: self)
```

Implementation:

```swift
static func track(key: SensorsAnalyticsKey, page: UIViewController? = nil) {
		case .channelBrowse(let model):
            guard let page = page else {
                return
            }
            var value = model.analytics
            value["previous_page_name"] = page.previousValue.0
            value["previous_module_name"] = page.page.previousValue.1
            value["page_name"] = page.pageName
            track(key.rawValue, value)
}
```