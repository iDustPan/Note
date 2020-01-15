### 获取Cache资源目录

```
[[XBorderConst contentUrlBase] stringByAppendingString:@"/offline/manifest.json"]
```

资源目录: https://baleen-dev-cdn-g.bybieyang.com/offline/manifest.json

> `common`: 通用的资源文件, 很多H5都会复用.  
> `business`: 业务H5的资源, 比如文章页, lego等


### 根据资源目录下载资源文件

根据资源目录下载所有资源文件, 因为暂时不是返回资源包压缩文件的形式, 所以只能逐条下载:

```
// resource是资源目录的序列化对象, 也就是上述json结构转化成的OC对象
- (void)cacheResource:(BXLOfflineResource *)resource {
    [self downloadResource:resource.common];
    for (BXLOfflineBusinessResource *bResource in resource.business) {
        [self downloadResource:bResource.assets];
    }
}

- (void)downloadResource:(NSArray<BXLOfflineResourceItem *> *)resources {
    for (BXLOfflineResourceItem *item in resources) {
        [self downloadResourceFromResourceItem:item
                                    completion:^(BOOL success) {}];
    }
}

```

使用AFNetworking进行下载:

```
- (void)downloadResourceFromResourceItem:(BXLOfflineResourceItem *)item
                              completion:(void(^)(BOOL success))block {
    
    NSURL *url = [NSURL URLWithString:item.url];
    NSString *md5Url = item.url.md5String;
    NSString *cacheDir = [[self.cacheRootDir stringByAppendingPathComponent:md5Url]
                               stringByAppendingPathComponent:item.version];
    
    if ([[NSFileManager defaultManager] fileExistsAtPath:cacheDir]) {
        return;
    }
    
    [[NSFileManager defaultManager] createDirectoryAtPath:cacheDir
                              withIntermediateDirectories:YES
                                               attributes:nil
                                                    error:nil];
    
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration
                                                defaultSessionConfiguration];
    AFURLSessionManager *manager = [[AFURLSessionManager alloc]
                                    initWithSessionConfiguration:configuration];
    
    [[manager downloadTaskWithRequest:request
                             progress:nil
                          destination:^NSURL * _Nonnull(NSURL * _Nonnull targetPath, NSURLResponse * _Nonnull response) {
        NSString *fileName = response.suggestedFilename;
        return [NSURL fileURLWithPath:[cacheDir stringByAppendingPathComponent:fileName]];
    } completionHandler:^(NSURLResponse * _Nonnull response, NSURL * _Nullable filePath, NSError * _Nullable error) {
        // nothing for now
    }] resume];
}
```

可以看到每个资源的cache path是这样的 `{NSCachesDirectory}/h5_offline_resources/{md5(url)}/{version}` 即:

```
NSString *cacheDir = [[self.cacheRootDir stringByAppendingPathComponent:md5Url]
                               stringByAppendingPathComponent:item.version];
```

这样能保证缓存在有更新时才下载, 因为h5资源有更新时, version会变化:

```
if ([[NSFileManager defaultManager] fileExistsAtPath:cacheDir]) {
    return;
}
```

**如果缓存不存在, 要先为资源缓存创建目录**, 否则下载之后你会发现缓存目录里根本没有这个资源.
```
[[NSFileManager defaultManager] createDirectoryAtPath:cacheDir
                          withIntermediateDirectories:YES
                                           attributes:nil
                                                error:nil];
```

使用AFNetworking进行资源下载:

```
NSURLRequest *request = [NSURLRequest requestWithURL:url];
    NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration
                                                defaultSessionConfiguration];
    AFURLSessionManager *manager = [[AFURLSessionManager alloc]
                                    initWithSessionConfiguration:configuration];
    
    [[manager downloadTaskWithRequest:request
                             progress:nil
                          destination:^NSURL * _Nonnull(NSURL * _Nonnull targetPath, NSURLResponse * _Nonnull response) {
        NSString *fileName = response.suggestedFilename;
        // 记得拼接fileName
        return [NSURL fileURLWithPath:[cacheDir stringByAppendingPathComponent:fileName]];
    } completionHandler:^(NSURLResponse * _Nonnull response, NSURL * _Nullable filePath, NSError * _Nullable error) {
        // nothing for now
    }] resume];
```

### 使用 `WKURLSchemeHandler` 进行协议替换.

协议头注入:

```
BXLJSBridgeConfiguration *configuration = [BXLJSBridgeConfiguration configurationWithBridgeDelegate:self];
// feature gate控制随时可随时控制开关
if ([XBorderFeatureGateLocalManager isSupportOfflineCache]) {
    // 只支持iOS 11系统以上的版本
    if(@available(iOS 11, *)) {
        [configuration setURLSchemeHandler:[BXLSchemeHandler new] 
                              forURLScheme:kCustomScheme];
    }
}
WKWebView *webView = [[WKWebView alloc] initWithFrame:CGRectZero configuration:configuration];
```

对所有h5请求进行协议头替换

```
- (void)loadURL:(NSURL *)url {
    ...
    NSURL *urlReplacedScheme = url;
    // 注意 feature gate 和版本控制.
    if ([XBorderFeatureGateLocalManager isSupportOfflineCache]) {
        if(@available(iOS 11, *)) {
            urlReplacedScheme = [url URLByReplacingToScheme:kCustomScheme];
        }
    }
    ...
}
```

### `BXLSchemeHandler` 内部实现方法

```
@interface BXLSchemeHandler ()

// 用于缓存所有进行协议替换的url
@property (nonatomic, strong) NSMutableDictionary *urlSchemeHolds;

// 用于缓存所有不走cache的远程调用task, 便于在stop时cancel掉, 否则会Crash
@property (nonatomic, strong) NSMutableDictionary *requestCache;

// weak属性,也是用于stop之后判断webView是否还存在, 避免Crash
@property (nonatomic, weak) WKWebView *webView;

@end
```

因为scheme被替换后所有的请求都会走到下述方法, 所以我们需要检测该url是否被缓存下来了, 如果有本地缓存那么加载本地缓存, 如果没有, 那么走远程调用: 

```
- (void)webView:(WKWebView *)webView startURLSchemeTask:(id <WKURLSchemeTask>)urlSchemeTask API_AVAILABLE(ios(11.0)){
    self.webView = webView;

    // 将urlSchemeTask缓存下来, 其description的结果作为key, bool值为value, 设为yes,代表正在请求过程中.

    [self.urlSchemeHolds setObject:@(YES) forKey:urlSchemeTask.description];


    NSURL *url = urlSchemeTask.request.URL;
    NSString *exactLocalPath = [[BXLOffineCacheManager shareInstance]
                                fileCachedPathWithUrl:url];

    // 判断是否有被cache下来. 
    BOOL cached = [[NSFileManager defaultManager] fileExistsAtPath:exactLocalPath];

    if (cached) {

        // 如果被缓存下来了,直接走本地缓存.
        [self loadCache:exactLocalPath urlSchemeTask:urlSchemeTask];
    }else{

        // 如果未被缓存, 那么走远端请求, 网络请求时记得将协议头替换为https
        url = [url URLByReplacingToScheme:@"https"];
        [self requestRemote:url urlSchemeTask:urlSchemeTask];
    }
}
```

### 从本地加载
```
- (void)loadCache:(NSString *)exactLocalPath urlSchemeTask:(id<WKURLSchemeTask> _Nonnull)urlSchemeTask  API_AVAILABLE(ios(11.0)){

    dispatch_async(processing_queue(), ^{
        NSData *data = [NSData dataWithContentsOfFile:exactLocalPath
                                              options:NSDataReadingMappedIfSafe
                                                error:nil];
        NSString *mimeType = [self getMimeTypeWithFilePath:exactLocalPath];
        NSURLResponse *response = [[NSURLResponse alloc]
                                   initWithURL:urlSchemeTask.request.URL
                                   MIMEType:mimeType
                                   expectedContentLength:data.length
                                   textEncodingName:nil];
        dispatch_async(dispatch_get_main_queue(), ^{

            BOOL stopped = [[self.urlSchemeHolds objectForKey:urlSchemeTask.description] boolValue] == false;
            if (stopped) {
                return;
            }
            if (self.webView == nil) {
                return;
            }
            @try {
                [urlSchemeTask didReceiveResponse:response];
                [urlSchemeTask didReceiveData:data];
                [urlSchemeTask didFinish];
            } @catch (NSException *exception) {
                [self logExceptionToFabric:exception];
            } @finally {}
        });
    });
}
```

这几个判断很重要:
```
BOOL stopped = [[self.urlSchemeHolds objectForKey:urlSchemeTask.description] boolValue] == false;

if (stopped) {
    return;
}
if (self.webView == nil) {
    return;
}
```

具体原因请看下面几个方法的文档注释:

```
[urlSchemeTask didReceiveResponse:response];
[urlSchemeTask didReceiveData:data];
[urlSchemeTask didFinish];
```
### 拉取远程资源

```
- (void)requestRemote:(NSURL *)url urlSchemeTask:(id<WKURLSchemeTask> _Nonnull)urlSchemeTask  API_AVAILABLE(ios(11.0)) {

    NSMutableURLRequest * request = [NSMutableURLRequest requestWithURL:url
                                                            cachePolicy:NSURLRequestReturnCacheDataElseLoad
                                                        timeoutInterval:15];
    NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession *session = [NSURLSession sessionWithConfiguration:config];
    @weakify(self);
    NSURLSessionDataTask *dataTask = [session dataTaskWithRequest:request
                                                completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {

        @strongify(self);
        BOOL stopped = [[self.urlSchemeHolds objectForKey:urlSchemeTask.description] boolValue] == false;
        if (stopped) {
            return;
        }
        if (self.webView == nil) {
            return;
        }

        @try {
            [urlSchemeTask didReceiveResponse:response];
            [urlSchemeTask didReceiveData:data];
            if (error) {
                [urlSchemeTask didFailWithError:error];
            } else {
                [urlSchemeTask didFinish];
            }
        } @catch (NSException *exception) {
            [self logExceptionToFabric:exception];
        } @finally {}
    }];

    // 将正在进行的请求task cache下来, 如果请求过程中用户退出页面,需要取消这些task.
    [self.requestCache setObject:dataTask forKey:urlSchemeTask.description];
    
    
    BOOL stopped = [[self.urlSchemeHolds objectForKey:urlSchemeTask.description] boolValue] == false;
    if (stopped) {
        return;
    }
    if (self.webView == nil) {
        return;
    }
    
    [dataTask resume];
}
```

为了防止网络正在请求时, 用户点击返回退出页面, 导致App Crash, 这里进行了两次判断.发起前进行一次, 请求回调后通知task时,再进行一次.

### 停止时取消请求

```
- (void)webView:(WKWebView *)webView stopURLSchemeTask:(id <WKURLSchemeTask>)urlSchemeTask API_AVAILABLE(ios(11.0)){
    // 将被停止的urlSchemeTask标记, 用于远端请求时的判断.
    [self.urlSchemeHolds setObject:@(NO) forKey:urlSchemeTask.description];

    // 停止时如果正在进行请求, 需要将其cancel.
    NSURLSessionDataTask *task = [self.requestCache objectForKey:urlSchemeTask.description];
    if (task) {
        [task cancel];
    }
}
```

### 其他

1.相关Crash记录:
  
https://bugly.qq.com/v2/crash-reporting/crashes/18a2b44333/77205?pid=2

复现于1.90.0~1.90.3, 可能因为正在请求的过程中用户退出页面导致, 可以将网速设置为3G进行测试, 也可以对远端请求进行延时调用(只复现在iOS 12.x的系统).1.91版本在 `[dataTask resume];` 之前追加了判断, 需要观察是否还会复现.

2.参考资料:

- [iOS app秒开H5优化探索](https://juejin.im/post/5c9c664ff265da611624764d)
- [wkwebview离线化加载h5资源解决方案](https://juejin.im/post/5adac2766fb9a07a9a106c0b)

