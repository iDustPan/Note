

### 背景

需要设计一个native和h5的通信机制, 达到能相互传递一些信息和相互发起操作的目的.

### 设计思路

为了避免所有bridge方法都写到 `ExternalLinkViewController` 里面, 也为了统一所有bridge方法.
将bridge的核心放在了 `BXLJSHandler` 类里面.

```
@protocol BXLJSMessageProtocol;
@interface BXLJSHandler : NSObject<WKScriptMessageHandler>

@property (nonatomic, weak, nullable) id<BXLJSMessageProtocol> delegate;


- (void)callJS:(nonnull NSString *)func args:(nullable NSArray *)args;
- (void)callJs:(nonnull NSString *)func args:(nullable NSArray *)args completionHandler:(nullable void (^)(id _Nullable value))completionHandler;

@end
```

可以看到该类即遵守了 `WKScriptMessageHandler` 协议, 也实现了 native call js 的接口.

另外我们还通过协议来统一所有bridge方法, 方便检索与维护:

```
@class BXLJSMessageAttributes;
@protocol BXLJSMessageProtocol <NSObject>

@optional
// 获取native的session信息
- (void)getSession:(BXLJSMessageAttributes *)data;
// 获取埋点的meta信息
- (void)getMetaInfo:(BXLJSMessageAttributes *)data;
// 某些情况下, native调用h5的方法后,需要给native一个回调, 通过该方法来返回一些回调数据
- (void)handleCallback:(BXLJSMessageAttributes *)data;
// 由h5直接调起分享
- (void)shareAction:(BXLJSMessageAttributes *)data;
// 控制导航栏的隐藏和显示, 决定导航栏上是否展示分享icon
- (void)onHeaderConfig:(BXLJSMessageAttributes *)data;
// 关闭h5
- (void)closeWindow:(BXLJSMessageAttributes *)data;
// 从h5登录后告知native用户的session信息
- (void)onLoginFromWeb:(BXLJSMessageAttributes *)data;
// 获取statusBar的高度
- (void)getStatusBarHeight:(BXLJSMessageAttributes *)data;
// 展示toast
- (void)showToast:(BXLJSMessageAttributes *)data;
// 刷新购物袋
- (void)reloadGroup:(BXLJSMessageAttributes *)data;
// 将h5统计的shopping tracing告诉native
- (void)recordPageState:(BXLJSMessageAttributes *)data;

@end

```

如有新增bridge方法, 请在此增加一个协议方法.



### Native Call JS

native call js主要采用了webView自带的方法:
`[self.webView evaluateJavaScript: completionHandler:^(id _Nullable result, NSError * _Nullable error) {}];` 

h5会在window对象中注入一个消息处理对象, 该对象会实现一个叫 `onJsCallback` 的方法.因为我们只需要传入


```
- (void)dispatchJavascriptCall:(BXLNativeMessage *)info {
    NSString *json = [info toJsonString];
    dispatch_async(dispatch_get_main_queue(), ^{
        NSString *js = [NSString stringWithFormat:@"window.%@.on(%@)", kBXLJSBridgeName, json];
        [self.webView evaluateJavaScript:js completionHandler:^(id _Nullable result, NSError * _Nullable error) {
            if (error) {
                NSLog(@"%@", [error localizedDescription]);
                return ;
            }
            NSLog(@"result: %@", result);
        }];
    });
}
```

目前项目中主要用来在h5向native发起获取某些信息请求时, 用来调起h5的方法来达到通知h5的目的, 比如h5像native发起获取埋点的meta信息时, native在准备好数据后通过调用h5里面的方法来告知h5.

```
- (void)getMetaInfo:(BXLJSMessageAttributes *)data {
    NSString *session = [XBorderAccount instance].sessionKey ?: @"";
    NSString *meta = [NSString stringWithFormat:@"%@?user_key=%@&%@",
                      kBXLJSBridgeProtocol,
                      session,
                      [XBorderAnalyticsWrapper trackingMetaInfoString]];
    [self callJS:kBXLNativeCallJS args:@[data.callId, meta]];
}
```


### JS Call Native



而js调用native是通过一个代理回调, 来向native传递参数的:

```
- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message {
    self.webView = message.webView;
    NSDictionary *dict;
    if ([message.body isKindOfClass:[NSDictionary class]]) {
        dict = message.body;
    }
    if (!dict.count) {
        return;
    }
    
    BXLJSMessage *msg = [BXLJSMessage modelWithDictionary:dict];
    NSString *handlerName = [msg handlerFunctionName];
   
    SEL selector = NSSelectorFromString(handlerName);

    if ([self.delegate respondsToSelector:selector]) {
        [self.delegate performSelector:selector withObject:msg.attributes];
    }else if ([self respondsToSelector:selector]){
        [self performSelector:selector withObject:msg.attributes];
    }
}
```

### Bridge Configuration的初始化

那么h5和native是如何注册通信协议的呢?

在初始化WKWebView时会传递一个配置参数: `WKWebViewConfiguration`, 我们继承该类, 实现一个自己的配置类:

```
@interface BXLJSBridgeConfiguration : WKWebViewConfiguration

+ (instancetype)configurationWithBridgeDelegate:(id<BXLJSMessageProtocol>)delegate;

- (instancetype)initWithBridgeDelegate:(id<BXLJSMessageProtocol>)delegate BXL_DESIGNATED_INITIALIZER;

- (instancetype)init BXL_UNAVAILABLE_INITIALIZER;

- (void)callJS:(NSString *)func args:(NSArray *)args;
- (void)callJs:(NSString *)func args:(NSArray *)args completionHandler:(void (^)(id _Nullable value))completionHandler;

@end

```

该类初始化时需要传递一个实现bridge协议方法的代理对象.

而该类将 `BXLJSHandler` 私有化, 可以看到会将代理对象传递给它.以达到即能将某些bridge方法统一处理, 又能将某些bridge方法外抛的目的.

```
- (instancetype)initWithBridgeDelegate:(id<BXLJSMessageProtocol>)delegate; {
    self = [super init];
    if (self) {
        WKUserContentController *controller = [[WKUserContentController alloc] init];
        BXLJSHandler *messageHandler = [[BXLJSHandler alloc] init];
        [controller addScriptMessageHandler:messageHandler name:kBXLNativeBridgeName];
        self.jsHandler = messageHandler;
        self.userContentController = controller;
        if (delegate) {
            self.jsHandler.delegate = delegate;
        }
    }
    return self;
}
```

看下下面这个方法的注释:
```
/*! @abstract Adds a script message handler.
 @param scriptMessageHandler The message handler to add.
 @param name The name of the message handler.
 @discussion Adding a scriptMessageHandler adds a function
 window.webkit.messageHandlers.<name>.postMessage(<messageBody>) for all
 frames.
 */
- (void)addScriptMessageHandler:(id <WKScriptMessageHandler>)scriptMessageHandler name:(NSString *)name;
```

`Adding a scriptMessageHandler adds a function
 window.webkit.messageHandlers.<name>.postMessage(<messageBody>) for all
 frames.`   

 意思就是调用该方法,会向webkit里面注入一个脚本消息的处理对象, 对象的名字就是第二个参数的名字, h5可以通过调用该对象的 `postMessage` 方法来像native发送消息, 而native一旦收到消息就会响应 `- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message` 这个代理方法.