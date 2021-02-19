---
title: iOS之网络请求拦截与修改
date: 2021-2-2 20:16:06
category: iOS
tags: 
- 请求拦截
- NSURLProtocol
---
### 背景
有时候我们会有些特别的想法：
1. 查看或动态修改网络的请求与返回的参数
2. 实现自己的缓存规则


### 原理
NSURLProtocol可以用于数据请求和数据返回的拦截与修改。具体工作流程，如下。

### 具体工作流程
需要了解一下`URL Loading System`（ULS）工作原理。每当我们发出一个请求的时候，ULS会询问`NSURLProtocol`注册的子类是否需要拦截，需要拦截的话，处理完，再扔回ULS询问。为了避免这个问题，每次拦截完之后，需要标记一下已拦截过，下次再询问就不要再拦截了。 

### 注意事项
1. NSURLSession需要注意：对于基于NSURLSession的网络请求，需要通过配置NSURLSessionConfiguration对象的protocolClasses属性。
```objectivec
NSURLSessionConfiguration *sessionConfiguration = [NSURLSessionConfiguration defaultSessionConfiguration];
sessionConfiguration.protocolClasses = @[[NSClassFromString(@"CustomURLProtocol") class]];
```
2. NSURLConnection：直接registerClass就可以用

### 扩展
[iOS - 让WKWebView 支持 NSURLProtocol](https://www.cnblogs.com/junhuawang/p/8466128.html)

### 最后附上您可能用得着的代码
拦截修改请求的一个类
```
//RSURLProtocol.h
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface RSURLProtocol : NSURLProtocol
+ (void)requestConfig;
@end

NS_ASSUME_NONNULL_END
//RSURLProtocol.m
#import "RSURLProtocol.h"
#import "RSDataHandle.h"
#import <objc/runtime.h>
#import <UIKit/UIKit.h>
#import "ConsoleHelper.h"
#import "RSInfo.h"

static NSString * const RSURLConfig = @"http://xxx.com/config";
static NSString * const RSURLProtocolHandledKey = @"RSURLProtocolHandledKey";
@interface RSURLProtocol()<NSURLSessionDelegate>
@property (nonatomic,strong) NSURLSession *session;
@end
@implementation RSURLProtocol


+(void)load{
    [RSURLProtocol registerProtocol];
    [self injectNSURLSessionConfiguration];
    [self requestConfig];
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [[ConsoleHelper sharedInstance] show];
    });
}



+ (NSString *)finalConfigString{
    if ([RSInfo.sharedInstance.networkAppID length]>0) {
        return [NSString stringWithFormat:@"%@?appID=%@",RSURLConfig,RSInfo.sharedInstance.networkAppID];
    }
    return RSURLConfig;
}
+ (void)requestConfig{
    RSInfo.sharedInstance.configInit = false;
    NSURLSession *session = [NSURLSession sharedSession];

    NSURL *url = [NSURL URLWithString:[self finalConfigString]];

    NSURLRequest *request = [NSURLRequest requestWithURL:url];

    [[session dataTaskWithRequest:request completionHandler:^(NSData*_Nullable data, NSURLResponse*_Nullable response, NSError*_Nullable error) {

        if(!error) {

        // 数据请求成功
            NSDictionary *dic = [NSJSONSerialization JSONObjectWithData:data options:NSJSONReadingAllowFragments error:nil];
            if (dic && [dic[@"code"] intValue]==0) {
                [RSInfo sharedInstance].configs = dic[@"data"];
                RSInfo.sharedInstance.configInit = true;
            }

        } else {

        // 数据请求失败

        }
    }] resume];
}
+ (void)injectNSURLSessionConfiguration{
    Class cls = NSClassFromString(@"__NSCFURLSessionConfiguration") ?: NSClassFromString(@"NSURLSessionConfiguration");
    Method originalMethod = class_getInstanceMethod(cls, @selector(protocolClasses));
    Method stubMethod = class_getInstanceMethod([self class], @selector(rs_protocolClasses));
    if (!originalMethod || !stubMethod) {
        [NSException raise:NSInternalInconsistencyException format:@"Couldn't load NEURLSessionConfiguration."];
    }
    method_exchangeImplementations(originalMethod, stubMethod);
}

- (NSArray *)rs_protocolClasses {
    return @[[RSURLProtocol class]];
}

+ (void)registerProtocol
{
    [NSURLProtocol registerClass:[self class]];
}

+ (void)unregisterProtocol
{
    [NSURLProtocol unregisterClass:[self class]];
}
#pragma mark - 拦截处理

+ (BOOL)canInitWithRequest:(NSURLRequest *)request
{
    if ([NSURLProtocol propertyForKey:RSURLProtocolHandledKey inRequest:request]) {
        return NO;
    }
    
//    拦截http、https
    NSString * scheme = [[request.URL scheme] lowercaseString];
    if ([scheme isEqual:@"http"]||[scheme isEqual:@"https"]) {
        return YES;
    }
    return NO;
}
// 这个方法用来统一处理请求request 对象的，可以修改头信息，或者重定向。没有特殊需要，则直接return request。
//  如果要在这里做重定向以及头信息的时候注意检查是否已经添加，因为这个方法可能被调用多次，也可以在后面的方法中做。
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {
    
    return request;
}
//主要判断两个request是否相同，如果相同的话可以使用缓存数据，通常只需要调用父类的实现。
+ (BOOL)requestIsCacheEquivalent:(NSURLRequest *)a toRequest:(NSURLRequest *)b {
    return [super requestIsCacheEquivalent:a toRequest:b];
}
//在拦截到网络请求，并且对网络请求进行定制处理以后。我们需要将网络请求重新发送出去，就可以初始化一个NSURLProtocol对象了：
- (id)initWithRequest:(NSURLRequest *)request cachedResponse:(NSCachedURLResponse *)cachedResponse client:(id <NSURLProtocolClient>)client {
    
    return [super initWithRequest:request cachedResponse:cachedResponse client:client];
}


- (void)reStartRequest {
    [RSDataHandle handleRequest:[self request]];
    
    NSMutableURLRequest *mutableReqeust = [[self request] mutableCopy];
    //标示该request已经处理过了，防止无限循环
    [NSURLProtocol setProperty:@(YES) forKey:RSURLProtocolHandledKey inRequest:mutableReqeust];
    
    //使用NSURLSession继续把request发送出去
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSOperationQueue *mainQueue = [NSOperationQueue mainQueue];
    self.session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:mainQueue];
    NSURLSessionDataTask *task = [self.session dataTaskWithRequest:mutableReqeust];
    
    [task resume];
}

//实际请求
- (void)startLoading
{
    
    if (RSInfo.sharedInstance.configInit || [[RSURLProtocol finalConfigString] isEqualToString:[self request].URL.absoluteString]) {
        [self reStartRequest];
    }else{
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            [self startLoading];
        });
        
    }
    
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveResponse:(NSURLResponse *)response completionHandler:(void (^)(NSURLSessionResponseDisposition))completionHandler
{
    [self.client URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
    completionHandler(NSURLSessionResponseAllow);
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data
{
    // 打印返回数据
    NSString *dataStr = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    if (dataStr) {
        NSLog(@"***截取URL***: %@", dataTask.currentRequest.URL.absoluteString);
        NSLog(@"***截取数据***: %@", dataStr);
        dataStr = [RSDataHandle handleResponse:dataStr url:dataTask.currentRequest.URL];
        if (dataStr) {
            [self.client URLProtocol:self didLoadData:[dataStr dataUsingEncoding:NSUTF8StringEncoding]];
            return ;
        }
    }
    
    [self.client URLProtocol:self didLoadData:data];
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error
{
    if (error) {
        [self.client URLProtocol:self didFailWithError:error];
    } else {
        [self.client URLProtocolDidFinishLoading:self];
    }
}
- (void)stopLoading {
    [self.session invalidateAndCancel];
    self.session = nil;
}

@end
```

### 参考资料
[NSURLProtocol拦截 HTTP 请求](https://blog.csdn.net/u014600626/article/details/108195234)
