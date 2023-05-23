---
layout: post
title: "网络请求LCNetwork"
date: 2015-08-10
comments: true
categories: iOS AFNetworking
tags: [iOS]
keywords: AFNetworking iOS
publish: true
description: 网络请求封装库
---
网络层的封装一直是项目中不足之处，前不久看了唐巧大神的[YTKNetwork](https://github.com/yuantiku/YTKNetwork)后又拜读了宇大神的这篇[博客](http://casatwy.com/iosying-yong-jia-gou-tan-wang-luo-ceng-she-ji-fang-an.html?utm_source=tuicool)，前者让我看到了离散型API封装的典型例子，后者恰好又提供了用 protocol 封装的很好思路以及说明了继承方式的封装的优缺点，于是结合两者 LCNetwork 就诞生了。项目地址[github](https://github.com/bawn/LCNetwork)，目前已经适配 AFNetworking 3.x

若遇到 Demo 闪退问题，请删除 APP 重新运行，另外感谢[zdoz](http://api.zdoz.net/)提供免费的测试接口。

LCNetwork 主要功能有

1. 支持`block`和`delegate`的回调方式
2. 支持设置主、副两个服务器地址
3. 支持`response`缓存，基于[TMCache](https://github.com/tumblr/TMCache)
4. 支持统一的`argument`加工
5. 支持统一的`response`加工
6. 支持多个请求同时发送，并统一设置它们的回调
7. 支持以类似于插件的形式显示HUD
8. 支持获取请求的实时进度

__最终在 `ViewController` 中调用一个接口请求的例子如下__

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


## 集成

Cocoapods:

```
  pod 'LCNetwork'
```

## 使用

### 统一配置


```
LCNetworkConfig *config = [LCNetworkConfig sharedInstance];
config.mainBaseUrl = @"http://api.zdoz.net/";// 设置主服务器地址
config.viceBaseUrl = @"https://api.zdoz.net/";// 设置副服务器地址
```

### 创建接口调用类
每个请求都需要一个对应的类去执行，这样的好处是接口所需要的信息都集成到了这个API类的内部，不在暴露在Controller层。

创建一个API类需要继承`LCBaseRequest`类，并且遵守`LCAPIRequest`协议，下面是最基本的API类的创建。

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
// 接口地址
- (NSString *)apiMethodName{
    return @"getweather.aspx";
}
// 请求方式
- (LCRequestMethod)requestMethod{
    return LCRequestMethodGet;
}
@end
```
`- (NSString *)apiMethodName` 和 `- (LCRequestMethod)requestMethod` 是 @required 方法，所以必须实现，这在一定程度上降低了因漏写方法而crash的概率。

另外 @optional 方法提供了如下的功能

```
// 是否使用副Url
- (BOOL)useViceUrl;

// 是否缓存数据 response 数据
- (BOOL)cacheResponse;

// 自定义超时时间
- (NSTimeInterval)requestTimeoutInterval;

// 用于 multipart 的数据block
- (AFConstructingBlock)constructingBodyBlock;

// response处理
- (id)responseProcess:(id)responseObject;

// 是否忽略统一的参数加工
- (BOOL)ignoreUnifiedResponseProcess;

// 返回完全自定义的接口地址
- (NSString *)customApiMethodName;

// 服务端数据接收类型，比如 LCRequestSerializerTypeJSON 用于 post json 数据
- (LCRequestSerializerType)requestSerializerType;
```

### 参数设置

请求的参数可以在外部设置，例如：

```
Api2 *api2 = [[Api2 alloc] init];
api2.requestArgument = @{
                          @"lat" : @"34.345",
                          @"lng" : @"113.678"
                        };
```
如果不想把参数的 key 值暴露在外部，也可以在 API 类中自定义初始化方法，例如:

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
直接在初始化方法中使用 `self.requestArgument = @{@"lat" : lat, lng" : lng}` 其实不妥，原因请参考 [唐巧](http://blog.devtang.com/blog/2011/08/10/do-not-use-accessor-in-init-and-dealloc-method/) 和 [jymn_chen](http://blog.csdn.net/jymn_chen/article/details/25000575)，如果想完全规避这样的问题，请参考demo中的实现


### 统一处理`argument` 和 `response`

这里需要用到另外一个协议 ` <LCProcessProtocol>`，比如我们的每个请求需要添加一个关于版本的参数：

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

或者是处理类似于下面的 json 数据，服务端返回的数据结构都会带`result` 和 `ok` 的 key

```
{
  "result": {
      "_id": "564a931dbbb03c7002a2c0f3",
      "name": "clover",
      "count": 0
    },

  "ok": true,
  "message" : "成功"
}
```
那么可以这样处理

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
也就是说当使用 `api1.responseJSONObject` 获取数据时，返回的直接是 `result` 对应的值，或者是错误信息。

最后，赋值给 `LCNetworkConfig` 的 `processRule`

```
LCProcessFilter *filter = [[LCProcessFilter alloc] init];
config.processRule = filter;
```

当然，如果你某个接口的 response 你不想做统一的处理，可以在请求子类中实现

```
- (BOOL)ignoreUnifiedResponseProcess{
    return YES;
}
```
这样返回的 response 就是原始数据

### multipart/form-data

通常我们会用到上传图片或者其他文件就需要用到 `multipart/form-data`，同样的只需要实现`- (AFConstructingBlock)constructingBodyBlock;`协议方法即可，比如

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

对于多图上传可能还会需要知道进度情况，LCNetwork 在 1.1.0 版本之后提供了监听进度的方法，只需要调用

```
- (void)startWithBlockProgress:(void (^)(NSProgress *progress))progress
                       success:(void (^)(id request))success
                       failure:(void (^)(id request))failure;
```
或者 `- (void)requestProgress:(NSProgress *)progress` 的协议方法，下面是一个具体例子：

```
MultiImageUploadApi *multiImageUploadApi = [[MultiImageUploadApi alloc] init];
    multiImageUploadApi.images = @[[UIImage imageNamed:@"test"], [UIImage imageNamed:@"test1"]];
    [multiImageUploadApi startWithBlockProgress:^(NSProgress *progress) {
        NSLog(@"%f", progress.fractionCompleted);
    } success:^(id request) {

    } failure:NULL];
```


### response 再加工

当类似于

```
{
  "result": {
      "_id": "564a931dbbb03c7002a2c0f3",
      "name": "clover",
      "count": 10
    },
  "ok": true,
  "message" : "成功"
}
```
这样的数据，如果已经对 response 做了统一的加工，比如成功后统一返回的数据是 result 中的数据，那么返回的也是一个 `NSDictionary`，可能无法满足需求，这时候再把数据交给 `LCBaseRequest` 子类处理再合适不过了。比如需要直接获取`count`值，那么只需要实现 `- (id)responseProcess:(id)responseObject;` 协议方法，具体如下

```
- (id)responseProcess:(id)responseObject{
    return responseObject[@"count"];
}
```
__注意，不应该调用`self.responseJSONObject`作为处理数据，请使用`responseObject`来处理__，当实现这个协议方法后，使用 `api1.responseJSONObject` 获取数据时，返回将是 `count` 的值。


### 关于HUD

如何显示 "正在加载"的 HUD，请参考Demo中的 `LCRequestAccessory` 类
