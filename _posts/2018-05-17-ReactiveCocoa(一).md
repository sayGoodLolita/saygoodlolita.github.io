---
layout: post
title: 'ReactiveCocoa(一)'
date: 2017-04-18
author: Cheney
color: rgb(255,210,32)
cover: '../assets/test.png'
tags: ReactiveCocoa
---
# ReactiveCocoa(一)

## ReactiveCocoa 简介

> ReactiveCocoa is inspired by functional reactive programming. Rather than using mutable variables which are replaced and modified in-place, RAC offers “event streams,” represented by the Signal and SignalProducer types, that send values over time.

ReactiveCocoa 的灵感来自函数式响应式编程(FRP). RAC 并不采用随时可变的变量, 而是用事件流(表现为`Signal`和`SignalProducer`)的方式来捕捉值的变化. 

> Event streams unify all of Cocoa’s common patterns for asynchrony and event handling, including: 

> * Delegate methods
> * Callback blocks
> * NSNotifications
> * Control actions and responder chain events
> * Futures and promises
> * Key-value observing(KVO)

在我们 iOS 开发过程中, 经常会响应某些事件来处理某些业务逻辑, 例如按钮的点击, 上下拉刷新, 网络请求, 属性的变化(通过 KVO)或者用户位置的变化(通过`CoreLocation`). 但是这些事件都用不同的方式来处理, 比如 action, delegate, KVO, callback 等.

其实这些事件, 都可以通过 RAC 处理, ReactiveCocoa 为事件提供了很多处理方法, 而且利用 RAC 处理事件很方便, 可以把要处理的事情, 和监听的事情的代码放在一起, 这样非常方便我们管理, 就不需要跳到对应的方法里, 非常符合我们开发中**高内聚, 低耦合**的思想. 

## ReactiveCocoa 常见类

### `RACSignal`

信号类, 一般表示将来有数据传递, 只要有数据改变, 信号内部接收到数据, 就会马上发出数据. 注意是, 数据发出, 并不是信号类发出. 

* 信号类(`RACSignal`), 只是表示当数据改变时, 信号内部会发出数据, 它本身不具备发送信号的能力, 而是交给内部一个订阅者去发出.

* 默认一个信号都是冷信号, 也就是值改变了, 也不会触发, 只有订阅了这个信号, 这个信号才会变为热信号, 值改变了才会触发.

* 如何订阅信号: 调用信号`RACSignal`的`subscribeNext`就能订阅.

**`RACSignal`使用步骤**: 

1. 创建信号`+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe`
2. 发送信号`- (void)sendNext:(id)value`
3. 订阅信号, 才会激活信号`- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock`

**`RACSignal`底层实现**：

1. 创建子类信号`RACDynamicSignal`, 首先把`didSubscribe`这个 block 保存到信号`RACDynamicSignal`中, 但是还不会触发(冷信号). 
2. 当信号被订阅, 也就是调用`signal`的`subscribeNext:nextBlock`. `subscribeNext`内部会创建订阅者`subscriber`, 并且把`nextBlock`保存到订阅者`subscriber`中, 此时为热信号. 
3. `subscribeNext`内部会调用`RACDynamicSignal`的`didSubscribe`这个block. 通常也就是在`RACDynamicSignal`的`didSubscribe`中调用`[subscriber sendNext:@1]`. 
4. `sendNext`底层其实就是执行订阅者`subscriber`的`nextBlock`. 
5. 标记信号发送完成或者取消订阅. 
6. 执行`RACDisposable`的`disposeBlock`中的代码. 

代码: 

```objc
// 1. 创建信号 createSignal:didSubscribe(block)
// RACDisposable: 取消订阅
// RACSubscriber: 发送数据
RACSignal * signal = [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
    // block 调用: 每当有订阅者订阅信号, 就会调用 block
    // block 作用: 描述当前信号那些数据需要发送
    
    // 3. 发送信号
    NSLog(@"调用了 didSubscribe");
    [subscriber sendNext:@1]; // 此处调用订阅者的 nextBlock
    
    // 5. 如果不在发送数据, 最好发送信号完成, 内部会自动调用 [RACDisposable disposable] 取消订阅信号. 或者信号想要被取消, 就必须返回一个 RACDisposable. 然后在后面 [disposable dispose]
    [subscriber sendCompleted];
    return [RACDisposable disposableWithBlock:^{
        // 6. 信号什么时候被取消: 1. 自动取消, 当一个信号的订阅者被销毁的时候, 就会自动取消订阅. 2. 主动取消
        // block 调用: 一旦一个信号被取消订阅就会调用
        // block 作用: 当信号取消订阅时清空一些资源
        NSLog(@"取消订阅");
    }];
}];

// subscribeNext: 创建订阅者, 然后把 nextBlock 保存到订阅者里面
// 2. 订阅信号: 只要订阅信号, 就会返回一个取消订阅信号的类
RACDisposable * disposable = [signal subscribeNext:^(id  _Nullable x) {
    // 4. block 调用: 每当有信号发出数据, 就会调用 block
    NSLog(@"接受到数据: %@", x);
}];
// 5. 取消订阅
// [disposable dispose];
```

```objc
2019-07-22 09:50:43.550264+0800 RACDemo(一)[95021:9298914] 调用了 didSubscribe
2019-07-22 09:50:43.550461+0800 RACDemo(一)[95021:9298914] 接受到数据: 1
2019-07-22 09:50:43.550595+0800 RACDemo(一)[95021:9298914] 取消订阅
```

#### `RACSubscriber`

表示订阅者的意思, 用于发送信号, 这是一个协议, 不是一个类, 只要遵守这个协议, 并且实现方法才能成为订阅者. 通过 create 创建的信号, 都有一个订阅者, 帮助他发送数据. 

#### `RACDisposable`

用于取消订阅或者清理资源, 当信号发送完成或者发送错误的时候, 就会自动触发它. 

### `RACSubject`

 信号提供者, 自己可以充当信号, 又能发送信号. 

 使用场景: 通常用来代替代理, 有了它, 就不必要定义代理了. 

**`RACSubject`使用步骤**

1. 创建信号`[RACSubject subject]`, 和`RACSignal`不一样, 创建信号时没有 block.
2. 订阅信号`- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock`. 
3. 发送信号`sendNext:(id)value`. 

`RACSubject`底层实现和`RACSignal`不一样. 

1. 调用`subscribeNext`订阅信号, 只是把订阅者保存起来, 并且订阅者的 nextBlock 已经赋值了. 
2. 调用`sendNext`发送信号, 遍历刚刚保存的所有订阅者, 一个一个调用订阅者的`nextBlock`, 所以一定要先订阅才能接受到数据. 

代码:

```objc
// RACSubject: 信号提供者
// 1. 创建信号
RACSubject * subject = [RACSubject subject];
// 2. 订阅信号
[subject subscribeNext:^(id  _Nullable x) {
    // 4. block 调用: 当有数据发出时就会调用
    // block 作用: 处理数据
    NSLog(@"第一个订阅者: %@", x);
}];
// 发送信号
[subject sendNext:@1];

// 2.1 第二次订阅信号
[subject subscribeNext:^(id  _Nullable x) {
    NSLog(@"第二个订阅者: %@", x);
}];
// 3. 发送信号
[subject sendNext:@2];
[subject sendNext:@3];
```

```objc
2019-07-22 10:10:42.191684+0800 RACDemo(一)[95217:9323244] 第一个订阅者: 1
2019-07-22 10:10:42.191868+0800 RACDemo(一)[95217:9323244] 第一个订阅者: 2
2019-07-22 10:10:42.191959+0800 RACDemo(一)[95217:9323244] 第二个订阅者: 2
2019-07-22 10:10:42.192053+0800 RACDemo(一)[95217:9323244] 第一个订阅者: 3
2019-07-22 10:10:42.192149+0800 RACDemo(一)[95217:9323244] 第二个订阅者: 3
```


#### `RACReplaySubject`

重复提供信号类, `RACSubject`的子类. 

使用场景一: 如果一个信号每被订阅一次, 就需要把之前的值重复发送一遍, 使用重复提供信号类. 
使用场景二: 可以设置`capacity`数量来限制缓存的 value 的数量, 即只缓充最新的几个值. 

1. 创建信号`[RACReplaySubject subject]`, 跟`RACSignal`不一样, 创建信号时没有 block. 
2. 可以先订阅信号, 也可以先发送信号. 
	1. 订阅信号`- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock`
	2. 发送信号`sendNext:(id)value`

`RACReplaySubject`底层实现和`RACSubject`不一样. 

1. 调用`sendNext`发送信号, 把值保存起来, 然后遍历刚刚保存的所有订阅者, 一个一个调用订阅者的nextBlock. 
2. 调用`subscribeNext`订阅信号, 遍历保存的所有值, 一个一个调用订阅者的`nextBlock`. 

如果想当一个信号被订阅, 就重复播放之前所有值, 需要先发送信号, 再订阅信号. **也就是先保存值，再订阅值**。

代码: 

```objc
// 1. 创建信号
RACReplaySubject * subject = [RACReplaySubject subject];
// 2. 订阅信号
[subject subscribeNext:^(id  _Nullable x) {
    NSLog(@"第一个订阅者: %@", x);
}];
// 3. 发送信号
[subject sendNext:@1];
[subject  sendNext:@2];
[subject subscribeNext:^(id  _Nullable x) {
    NSLog(@"第二个订阅者: %@", x);
}];
```

```objc
2019-07-22 10:21:01.192039+0800 RACDemo(一)[95313:9337583] 第一个订阅者: 1
2019-07-22 10:21:01.192158+0800 RACDemo(一)[95313:9337583] 第一个订阅者: 2
2019-07-22 10:21:01.194203+0800 RACDemo(一)[95313:9337583] 第二个订阅者: 1
2019-07-22 10:21:01.194310+0800 RACDemo(一)[95313:9337583] 第二个订阅者: 2
```

从输出中看出, 无论`sendNext`在订阅之前还是之后, 输出不变. 而`RACSubject`同样的代码顺序则会输出: 

```objc
2019-07-22 10:24:17.340032+0800 RACDemo(一)[95350:9342537] 第一个订阅者: 1
2019-07-22 10:24:17.340165+0800 RACDemo(一)[95350:9342537] 第一个订阅者: 2
```

也就是在`sendNext`后面订阅的信号已经不管用了. 

### `RACTuple`

元组类, 类似`NSArray`, 用来包装值. 

`RACTupleUnpack`宏: 专门用来解析元组. 等号右边: 需要解析的元组. 宏的参数: 填解析数据的类型. 元组里面有几个值, 宏的参数就必须填几个. 

代码:

```objc
// 遍历字典, 遍历出来的键值对会包装成 RACTuple (元组对象)
NSDictionary * dict = @{@"name":@"Cheney", @"age":@18};
[dict.rac_sequence.signal subscribeNext:^(id  _Nullable x) {
   // 解包元组, 会把元组的值按顺序给参数的变量赋值
    RACTupleUnpack(NSString * key, NSString * value) = x;
    // 相当于以下写法
    // NSString * key = x[0];
    // NSString * value = x[1];
    NSLog(@"%@ %@", key, value);
}];
```

```
2019-07-22 11:01:19.002974+0800 RACDemo(一)[95601:9382095] name Cheney
2019-07-22 11:01:19.003585+0800 RACDemo(一)[95601:9382095] age 18
```

### `RACSequence`

RAC 中的集合类, 用于代替`NSArray`和`NSDictionary`, 可以使用它来快速遍历数组和字典. 可以用来字典转模型等. (据说性能很差)

```objc
// 遍历数组
NSArray * array = @[@1, @2, @3];
// 把数组转换成集合 RACSequence: aray.rac_sequence
// 把集合转化为信号
// 订阅信号, 会把集合中的所有值遍历出来
[array.rac_sequence.signal subscribeNext:^(id  _Nullable x) {
    NSLog(@"%@", x);
}];
```

```objc
2019-07-22 11:04:48.837170+0800 RACDemo(一)[95656:9387035] 1
2019-07-22 11:04:48.837495+0800 RACDemo(一)[95656:9387035] 2
2019-07-22 11:04:48.837649+0800 RACDemo(一)[95656:9387035] 3
```

字典转模型：

```objc
NSDictionary * dict1 = @{@"name":@"Cheney", @"age":@18};
NSDictionary * dict2 = @{@"name":@"bobo", @"age":@18};
NSArray * dictArr = @[dict1, dict2];
// map: 映射, 吧原始的 value 值映射成一个新值
// array: 把集合转换成数组
// 当信号被订阅时, 遍历集合中的原始值, 映射成新值, 保存到新的数组
NSArray * perArr = [[dictArr.rac_sequence map:^id _Nullable(id  _Nullable value) {
    return [Person personWithDic:value];
}]array];
NSLog(@"%@", perArr);
```

### `RACCommand`

RAC 中用于处理事件的类, 可以把事件如何处理, 事件中的数据如何传递, 包装到这个类中, 他可以很方便的监控事件的执行过程. 

使用场景: 监听按钮点击, 网络请求. 

* `RACCommand`使用步骤:

1. 创建命令`initWithSignalBlock:(RACSignal * (^)(id input))signalBlock` 
2. 在`signalBlock`中, 创建`RACSignal`, 并且作为`signalBlock`的返回值 
3. 执行命令`- (RACSignal *)execute:(id)input` 


* `RACCommand`使用注意:

1. `signalBlock`必须要返回一个信号, 不能传 nil. 
2. 如果不想要传递信号, 直接创建空的信号`[RACSignal empty]`.
3. `RACCommand`中信号如果数据传递完, 必须调用`[subscriber sendCompleted]`, 这时命令才会执行完毕, 否则永远处于执行中. 
4. `RACCommand`需要被强引用, 否则接收不到`RACCommand`中的信号, 因此`RACCommand`中的信号是延迟发送的. 

* `RACCommand`设计思想: 内部`signalBlock`为什么要返回一个信号, 这个信号有什么用.

1. 在RAC开发中, 通常会把网络请求封装到`RACCommand`, 直接执行某个`RACCommand`就能发送请求. 
2. 当`RACCommand`内部请求到数据的时候, 需要把请求的数据传递给外界, 这时候就需要通过`signalBlock`返回的信号传递了.

* 如何拿到`RACCommand`中返回信号发出的数据.

1. `RACCommand`有个执行信号源`executionSignals`, 这个是`signal of signals`(信号的信号), 意思是信号发出的数据是信号, 不是普通的类型. 
2. 订阅`executionSignals`就能拿到`RACCommand`中返回的信号, 然后订阅`signalBlock`返回的信号, 就能获取发出的值. 

* 监听当前命令是否正在执行executing.

代码:

```objc
// 1. 创建命令
RACCommand * command = [RACCommand.alloc initWithSignalBlock:^RACSignal * _Nonnull(id  _Nullable input) {
    // block 调用: 当执行这个命令类的时候就会调用
    // block 作用: 描述如何处理事件, 网络请求
    NSLog(@"执行命令: %@", input);
    // 创建空信号, 必须返回信号
    // return RACSignal.empty;
    // 2. RACCommand 必须返回信号, 处理事件产生的数据就通过返回的信号发出. 注意: 数据传递完, 最好调用 sendCompleted, 这时命令才执行完毕.
    return [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
        // block 调用: 当信号被订阅时被调用
        // block 作用: 发送处理事件的信号
        [subscriber sendNext:@"信号发出的内容"];
        [subscriber sendCompleted];
        return nil;
    }];
}];
// executionSignals: 信号源, 包含事件处理的所有信号.
// executionSignals: signalOfSignals, 信号中信号, 就是信号发出的数据也是信号
// switchToLatest: 用于 signalOfSignals, 获取 signalOfSignals 发出的最新信号, 也就是直接拿到 RACCommand 的信号
// 3. 如果想要h订阅接受信号源的信号内容, 必须保证命令不被销毁
[command.executionSignals.switchToLatest subscribeNext:^(id  _Nullable x) {
    NSLog(@"%@", x);
}];
// 4. 执行命令, 调用 signalBlock
[command execute:@1];
// 5. 监听命令是否执行完毕, 默认会来一次, 可以直接跳过, skip 表示跳过第一次信号
[[command.executing skip:1] subscribeNext:^(NSNumber * _Nullable x) {
    if (x.boolValue == YES) {
        NSLog(@"正在执行");
    } else {
        NSLog(@"执行完成");
    }
}];
```

### `RACMulticastConnection`
用于当一个信号被多次订阅时, 为了保证创建信号时**避免多次调用创建信号中的 block**, 造成副作用, 可以使用这个类处理. 
* `RACMulticastConnection`使用步骤:

1. 创建信号`+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe`
2. 创建连接`RACMulticastConnection * connect = [signal publish];`
3. 订阅信号. 注意: 订阅的不在是之前的信号, 而是连接的信号. `[connect.signal subscribeNext:nextBlock]`
4. 连接`[connect connect]`

* `RACMulticastConnection`底层原理:

1. 创建`connect`, `connect.sourceSignal` -\> `RACSignal`(原始信号) `connect.signal` -\> `RACSubject`. 
2. 订阅`connect.signal`, 会调用`RACSubject`的`subscribeNext`, 创建订阅者, 而且把订阅者保存起来, 不会执行 block. 
3. `[connect connect]`内部会订阅`RACSignal`(原始信号), 并且订阅者是`RACSubject`. 
	1. 订阅原始信号, 就会调用原始信号中的`didSubscribe`. 
	2. `didSubscribe`, 拿到订阅者调用`sendNext`, 其实是调用`RACSubject`的`sendNext`.
4. `RACSubject`的`sendNext`, 会遍历`RACSubject`所有订阅者发送信号. 
	1. 因为刚刚第二步, 都是在订阅`RACSubject`, 因此会拿到第二步所有的订阅者, 调用他们的`nextBlock`.

需求: 假设在一个信号中发送请求, 每次订阅一次都会发送请求, 这样就会导致多次请求. 
解决: 使用 RACMulticastConnection 就能解决. 

```objc
// 发送请求, 用一个信号内包装, 不管有多少个订阅者, 只想要发送一次请求
RACSignal * signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    // didSubscribe(block)中的代码都统称为副作用 (Side Effects).
    // 发送请求
    NSLog(@"发送请求");
    [subscriber sendNext:@1];
    return nil;
}];
// 订阅信号
[signal subscribeNext:^(id x) {
    NSLog(@"接收数据: %@",x);
}];
[signal subscribeNext:^(id x) {
    NSLog(@"接收数据: %@",x);
}];
```

```objc
2019-07-22 11:45:00.916622+0800 RACDemo(一)[96081:9442511] 发送请求
2019-07-22 11:45:00.916781+0800 RACDemo(一)[96081:9442511] 接收数据: 1
2019-07-22 11:45:00.916923+0800 RACDemo(一)[96081:9442511] 发送请求
2019-07-22 11:45:00.917039+0800 RACDemo(一)[96081:9442511] 接收数据: 1
```
运行结果会执行两遍发送请求, 也就是每次订阅都会发送一次请求. 

```objc
// RACMulticastConnection: 解决重复请求问题
// 1. 创建信号
RACSignal * signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
    NSLog(@"发送请求");
    [subscriber sendNext:@1];
    return nil;
}];
// 2. 创建连接
RACMulticastConnection * connect = signal.publish;
// 3. 订阅信号
// 注意: 订阅信号也不能激活信号, 只是保存订阅者到数组, 必须通过连接, 当调用连接就会一次性调用所有订阅者的 sendNext:
[connect.signal subscribeNext:^(id x) {
    NSLog(@"订阅者一信号");
}];
[connect.signal subscribeNext:^(id x) {
    NSLog(@"订阅者二信号");
}];
// 4. 连接, 激活信号
[connect connect];
```

```objc
2019-07-22 11:46:55.626196+0800 RACDemo(一)[96118:9445261] 发送请求
2019-07-22 11:46:55.626338+0800 RACDemo(一)[96118:9445261] 订阅者一信号
2019-07-22 11:46:55.626440+0800 RACDemo(一)[96118:9445261] 订阅者二信号
```
### `RACSchedule`

RAC 中的队列, 用 GCD 封装的. 

### `RACUnit`

表⽰ stream 不包含有意义的值, 也就是看到这个, 可以直接理解为 nil. 

### `RACEvent`

把数据包装成信号事件(`signal event`). 它主要通过`RACSignal`的`-materialize`来使用. 