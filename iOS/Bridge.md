

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



### Native Call JS

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


### JS Call Native

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

```
@interface BXLJSBridgeConfiguration : WKWebViewConfiguration

+ (instancetype)configurationWithBridgeDelegate:(id<BXLJSMessageProtocol>)delegate;

- (instancetype)initWithBridgeDelegate:(id<BXLJSMessageProtocol>)delegate BXL_DESIGNATED_INITIALIZER;

- (instancetype)init BXL_UNAVAILABLE_INITIALIZER;

- (void)callJS:(NSString *)func args:(NSArray *)args;
- (void)callJs:(NSString *)func args:(NSArray *)args completionHandler:(void (^)(id _Nullable value))completionHandler;

@end

```



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