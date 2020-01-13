# 干净的code---策略模式

从一个实例讲起吧, 假定App有一个Address类,是填写地址页面的model层, 我们将其简化, 用3个字段举例:

```
@interface Address : NSObject

@property (nonatomic, copy) NSString *country;
@property (nonatomic, copy) NSString *lastName;
@property (nonatomic, copy) NSString *firstName;

@end
```

用户可以选择两个国家: US 和 China, 定义fullName为lastName和firstName的拼接. 但是我们知道中美两国的姓名格式是不一样的, 比如用户的姓名为: "Tony Zhang", 
当用户选择的国家为 `US` 时, 输出的fullName应该"Tony Zhang",
当用户选择的国家为 `China` 时, 输出fullName应该为"Zhang Tony".

如果不使用策略模式时, 我们先来看看if-else下的代码:

```
    if ([address.country isEqualToString:@"US"]) {
        NSLog(@"%@ %@", address.firstName, address.lastName);
    }else if ([address.country isEqualToString:@"China"]) {
        NSLog(@"%@ %@", address.lastName, address.firstName);
    }
```

在该例子下, if-else的逻辑还不算多, 设想如果在地址功能下还有其他的逻辑也需要根据不同国家来区别对待, 甚至增加一个国家地区的时候, 代码里就会充满各种各样零散的if-else逻辑, 维护的成本随之增高.

这个时候, 就是策略模式登场的时候.

### 什么是策略模式?

策略模式是对算法的封装，它把算法的责任和算法本身分割开，委派给不同的对象管理。策略模式通常把一个系列的算法封装到一系列的策略类里面，作为一个抽象策略类的子类。用一句话来说，就是“准备一组算法，并将每一个算法封装起来，使得它们可以互换”。

对于该例来说, 我们需要封装的算法简单明了, 就是 *根据不同的国家,返回不同的fullName, 每个国家fullName的生成逻辑由不同的类来管理*.

这样做的优点是什么呢?

1.可以避免大量的if-else逻辑;
2.新增地区的时候我只需要新增fullName算法的实现层,而不修改原有逻辑;

### 如何使用?

下面我们来看看反映到代码上如何来实现, 例子采用OC实现, 有兴趣的同学可以用Swift实现, 原理是一样的.

1. 用协议来抽象具体使用策略的算法. 在该例子当中,涉及到的算法也就是根据用户选择的国家返回不一样的fullName.那么我们可以把获取fullName这个算法定义成一个抽象接口:

```
@class Address;
@protocol AddressStrategy <NSObject>

- (NSString *)fullName:(Address *)address;

@end
```

2. 实现一个抽象的策略执行层, 执行层的作用是: 你给我一个策略, 我给你这个策略返回的结果:

```
@interface AddressHelper : NSObject

- (instancetype)initWithStrategy:(id<AddressStrategy>)strategy;
- (NSString *)getFullName:(Address *)address;

@end


@interface AddressHelper ()

@property (nonatomic, strong) id<AddressStrategy> strategy;

@end

@implementation AddressHelper

- (instancetype)initWithStrategy:(id<AddressStrategy>)strategy {
    if (self = [super init]) {
        self.strategy = strategy;
    }
    return self;
}

- (NSString *)getFullName:(Address *)address {
    return [self.strategy fullName:address];
}

@end
```

3. 在策略的实现层实现具体的策略算法:

中国地址策略实现层

```
#import "AddressStrategy.h"
@interface ChinaAddressStrategy : NSObject<AddressStrategy>
@end

@implementation ChinaAddressStrategy

- (NSString *)fullName:(Address *)address {
    return [address.lastName stringByAppendingFormat:@" %@", address.firstName];
}

@end

```

美国地址策略实现层

```

#import "AddressStrategy.h"
@interface USAddressStrategy : NSObject<AddressStrategy>
@end


#import "USAddressStrategy.h"
#import "Address.h"

@implementation USAddressStrategy

- (NSString *)fullName:(Address *)address {
    return [address.firstName stringByAppendingFormat:@" %@", address.lastName];
}

@end
```


那么,我在使用的时候,直接调用策略的实现层即可:

```
AddressHelper *helper = [[AddressHelper alloc] initWithStrategy:[USAddressStrategy new]];
NSLog(@"%@", [helper getFullName:address]);
    
    
AddressHelper *helper2 = [[AddressHelper alloc] initWithStrategy:[ChinaAddressStrategy new]];
NSLog(@"%@", [helper2 getFullName:address]);

```

你可能会觉得不就几行if-else逻辑而已, 干嘛要多此一举. 对该例来讲确实如此. 但是假如根据国籍需要做不同算法的行为不只是获取fullName呢? 甚至这些逻辑在不同的国家下都有所不同呢? 

### 策略模式的缺点

假设Address是由server返回的, 你需要把fullName显示在界面上. 这时候你会发现在选择策略的时候还是避免不了if-else来决定到底该使用哪种策略. 这就是策略模式的缺点了. 但是跟原来零散的逻辑相比, 已经好很多了.
而且,我们可以将选择策略的逻辑封装到 `AddressHelper` 当中:

```
@interface AddressHelper : NSObject

+ (instancetype)helperWithAddress:(Address *)address;

@end

@implementation AddressHelper

+ (instancetype)helperWithAddress:(Address *)address {
    NSDictionary *strategies = @{
        @"US": [USAddressStrategy new],
        @"CN": [ChinaAddressStrategy new]
    };
    if ([strategies objectForKey:address.country]) {
        AddressHelper *helper = [[AddressHelper alloc] initWithStrategy:strategies[address.country]];
        return helper;
    }
    return nil;
}

@end
```

最终优化过后的代码调用为:

```
AddressHelper *helper = [AddressHelper helperWithAddress:address];
NSLog(@"%@", [helper getFullName:address]);
```

完整demo地址见github: https://github.com/iDustPan/StrategyPattern 

#### 参考链接:

1. [策略模式](https://design-patterns.readthedocs.io/zh_CN/latest/behavioral_patterns/strategy.html)
2. [Strategy pattern in Swift](https://medium.com/flawless-app-stories/strategy-pattern-in-swift-1462dbddd9fe)



