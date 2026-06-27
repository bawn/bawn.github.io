---
layout: post
title: "Networking with LCNetwork"
date: 2015-08-10
comments: true
categories: iOS AFNetworking
tags: [iOS]
keywords: AFNetworking iOS
publish: true
description: Network request wrapper library
---

Wrapping the network layer has always been one of the weaker parts of many projects. Not long ago, after reading Tang Qiao's [YTKNetwork](https://github.com/yuantiku/YTKNetwork) and then Yu's excellent [post](http://casatwy.com/iosying-yong-jia-gou-tan-wang-luo-ceng-she-ji-fang-an.html?utm_source=tuicool), I saw a typical example of discrete API wrapping in the former and a very good protocol-based design idea in the latter, along with a clear explanation of the pros and cons of inheritance-based wrapping. Combining the two, LCNetwork was born. The project is available on [GitHub](https://github.com/bawn/LCNetwork), and it is already adapted for AFNetworking 3.x.

If you encounter a demo crash, please delete the app and run it again. Also, thanks to [zdoz](http://api.zdoz.net/) for providing free test APIs.

LCNetwork mainly provides the following features:

1. Supports both `block` and `delegate` callbacks
2. Supports setting both a primary and a secondary server address
3. Supports `response` caching based on [TMCache](https://github.com/tumblr/TMCache)
4. Supports unified `argument` preprocessing
5. Supports unified `response` preprocessing
6. Supports sending multiple requests at the same time and setting their callbacks in a unified way
7. Supports showing a HUD in a plug-in-like manner
8. Supports retrieving real-time request progress

__A simple example of calling an API from a `ViewController` looks like this:__

```
Api2 *api2 = [[Api2 alloc] init];
api2.requestArgument = @{
                         @"lat" : @"34.345",
                         @"lng" : @"113.678"
                         };
[api2 startWithCompletionBlockWithSuccess:^(Api2 *api2) {
    self.weather2.text = api2.responseJSONObject[@"Weather"];
} failure:NULL];
```


## Integration

Cocoapods:

```
  pod 'LCNetwork'
```

## Usage

### Unified Configuration


```
LCNetworkConfig *config = [LCNetworkConfig sharedInstance];
config.mainBaseUrl = @"http://api.zdoz.net/";// set primary server address
config.viceBaseUrl = @"https://api.zdoz.net/";// set secondary server address
```

### Creating an API Call Class
Each request needs a corresponding class to execute it. The advantage of this is that all the information required by the API is encapsulated inside the API class and does not need to be exposed at the Controller layer.

To create an API class, inherit from `LCBaseRequest` and conform to the `LCAPIRequest` protocol. Below is the most basic example.

__Api1.h__

```
#import <LCNetwork/LCBaseRequest.h>
@interface Api1 : LCBaseRequest<LCAPIRequest>
@end
```

__Api1.m__

```
#import "Api1.h"
@implementation Api1
// API endpoint
- (NSString *)apiMethodName{
    return @"getweather.aspx";
}
// Request method
- (LCRequestMethod)requestMethod{
    return LCRequestMethodGet;
}
@end
```
`- (NSString *)apiMethodName` and `- (LCRequestMethod)requestMethod` are `@required` methods, so they must be implemented. This to some extent reduces the chance of crashing because a method was forgotten.

The following features are provided by `@optional` methods:

```
// Whether to use the secondary URL
- (BOOL)useViceUrl;

// Whether to cache response data
- (BOOL)cacheResponse;

// Custom timeout interval
- (NSTimeInterval)requestTimeoutInterval;

// Data block for multipart
- (AFConstructingBlock)constructingBodyBlock;

// Response processing
- (id)responseProcess:(id)responseObject;

// Whether to ignore unified response processing
- (BOOL)ignoreUnifiedResponseProcess;

// Return a fully custom API endpoint
- (NSString *)customApiMethodName;

// The server-side data encoding type, for example LCRequestSerializerTypeJSON for POSTing JSON data
- (LCRequestSerializerType)requestSerializerType;
```

### Setting Parameters

Request parameters can be set externally, for example:

```
Api2 *api2 = [[Api2 alloc] init];
api2.requestArgument = @{
                          @"lat" : @"34.345",
                          @"lng" : @"113.678"
                        };
```
If you do not want to expose the parameter keys outside, you can also define a custom initializer inside the API class, for example:

__Api2.h__

```
@interface Api2 : LCBaseRequest<LCAPIRequest>
- (instancetype)initWith:(NSString *)lat lng:(NSString *)lng;
@end
```

__Api2.m__

```
#import "Api2.h"
@implementation Api2
- (instancetype)initWith:(NSString *)lat lng:(NSString *)lng{
    self = [super init];
    if (self) {
                self.requestArgument = @{
                                 @"lat" : lat,
                                 @"lng" : lng
                                 };
    }
    return self;
}
```
Using `self.requestArgument = @{@"lat" : lat, lng" : lng}` directly in the initializer is actually not ideal. For the reason, please refer to [Tang Qiao](http://blog.devtang.com/blog/2011/08/10/do-not-use-accessor-in-init-and-dealloc-method/) and [jymn_chen](http://blog.csdn.net/jymn_chen/article/details/25000575). If you want to completely avoid this problem, please refer to the demo implementation.


### Unified Processing of `argument` and `response`

Here we need another protocol, `<LCProcessProtocol>`. For example, if every request needs to add a version parameter:

__LCProcessFilter.h__

```
#import "LCNetworkConfig.h"
@interface LCProcessFilter : NSObject <LCProcessProtocol>
@end
```
__LCProcessFilter.m__

```
#import "LCBaseRequest.h"
@implementation LCProcessFilter
- (NSDictionary *) processArgumentWithRequest:(NSDictionary *)argument{
     NSMutableDictionary *newParameters = [[NSMutableDictionary alloc] initWithDictionary:argument];
    [newParameters setObject:@"1.0.0" forKey:@"version"];
    return newParameters;
}
```

Or, for JSON data like this, where the server returns both `result` and `ok` keys:

```
{
  "result": {
      "_id": "564a931dbbb03c7002a2c0f3",
      "name": "clover",
      "count": 0
    },

  "ok": true,
  "message" : "success"
}
```

You can process it like this:

```
- (id) processResponseWithRequest:(id)response{
    if ([response[@"ok"] boolValue]) {
        return response[@"result"];
    }
    else{
        NSDictionary *userInfo = @{NSLocalizedDescriptionKey: response[@"message"]};
        return [NSError errorWithDomain:ErrorDomain code:0 userInfo:userInfo];
    }
}

```
That means when you use `api1.responseJSONObject` to get data, what you get is directly the value corresponding to `result`, or an error message.

Finally, assign the filter to `LCNetworkConfig`'s `processRule`:

```
LCProcessFilter *filter = [[LCProcessFilter alloc] init];
config.processRule = filter;
```

Of course, if you do not want a particular API's response to go through unified processing, you can implement this in the request subclass:

```
- (BOOL)ignoreUnifiedResponseProcess{
    return YES;
}
```
That way, the returned response will be the raw data.

### multipart/form-data

Usually, when uploading images or other files, we need `multipart/form-data`. Again, you only need to implement the `- (AFConstructingBlock)constructingBodyBlock;` protocol method, for example:

```
- (AFConstructingBlock)constructingBodyBlock {
    return ^(id<AFMultipartFormData> formData) {
        NSData *data = UIImageJPEGRepresentation([UIImage imageNamed:@"currentPageDot"], 0.9);
        NSString *name = @"image";
        NSString *formKey = @"image";
        NSString *type = @"image/jpeg";
        [formData appendPartWithFileData:data name:formKey fileName:name mimeType:type];
    };
}
```

For multi-image uploads, you may also need to know the progress. Since version 1.1.0, LCNetwork provides a method for observing progress. You only need to call:

```
- (void)startWithBlockProgress:(void (^)(NSProgress *progress))progress
                       success:(void (^)(id request))success
                       failure:(void (^)(id request))failure;
```
or the protocol method `- (void)requestProgress:(NSProgress *)progress`. Here is a concrete example:

```
MultiImageUploadApi *multiImageUploadApi = [[MultiImageUploadApi alloc] init];
    multiImageUploadApi.images = @[[UIImage imageNamed:@"test"], [UIImage imageNamed:@"test1"]];
    [multiImageUploadApi startWithBlockProgress:^(NSProgress *progress) {
        NSLog(@"%f", progress.fractionCompleted);
    } success:^(id request) {

    } failure:NULL];
```


### Reprocessing `response`

When the data looks like this:

```
{
  "result": {
      "_id": "564a931dbbb03c7002a2c0f3",
      "name": "clover",
      "count": 10
    },
  "ok": true,
  "message" : "success"
}
```

If the response has already gone through unified processing, for example if the successful return value is always the data inside `result`, then the returned value will be a `NSDictionary`, which may not meet your needs. In that case, it is more appropriate to hand the data over to the `LCBaseRequest` subclass for further processing. For example, if you need to directly obtain the `count` value, just implement the `- (id)responseProcess:(id)responseObject;` protocol method as follows:

```
- (id)responseProcess:(id)responseObject{
    return responseObject[@"count"];
}
```
__Note: you should not use `self.responseJSONObject` to process the data. Please use `responseObject` instead.__ When this protocol method is implemented, using `api1.responseJSONObject` to fetch the data will return the value of `count`.

### About HUDs

For how to display a "loading" HUD, please refer to the `LCRequestAccessory` class in the demo.
