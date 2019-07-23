---
layout: post
title: 'ReactiveCocoa(二)'
date: 2017-04-30
author: Cheney
color: rgb(255,210,32)
cover: '../assets/test.png'
tags: ReactiveCocoa
---

# ReactiveCocoa(二)

## 常见用法

之前提到了 RAC 可以代替代理, KVO 等. 现在来看看具体用法. 

#### 代替代理:

* `rac_signalForSelector:`: 用于替代代理.

* 原理: 判断一个方法有没有调用, 如果调用了就会自动发送一个信号.

* 需求: 自定义 CustomView, 监听自定义 view 中按钮点击

* 之前都是需要通过代理监听, 给自定义 view 添加一个代理属性, 点击按钮的时候, 通知代理做事情.

* `rac_signalForSelector:`: 把调用某个对象的方法的信息转换成信号, 就要调用这个方法, 就会发送信号.

* 这里表示只要 CustomView 调用 btnClick, 就会发出信号, 订阅就好了.

代码:

```objc
// CustomView
- (void)buttonDidClick:(UIButton *)btn {
	NSLog(@"buttonDidClick");
}

// DelegateViewController
[[self.customView rac_signalForSelector:@selector(buttonDidClick:)] subscribeNext:^(id x) {
	NSLog(@"customView 点击了按钮");
}];
```

```objc
buttonDidClick
customView 点击了按钮
```

#### 代替KVO:

* `rac_valuesAndChangesForKeyPath:`: 用于监听某个对象的属性改变. 
* 方法调用者: 就是被监听的对象. 
* KeyPath: 监听的属性. 
* 把监听 redV 的 center 属性改变转换成信号, 只要值改变就会发送信号. 
* `observer`: 可以传入nil. 

```objc
- (void)kvo {
    self.person = Person.new;
    [[self.person rac_valuesAndChangesForKeyPath:@"age" options:NSKeyValueObservingOptionNew observer:self] subscribeNext:^(RACTwoTuple<id,NSDictionary *> * _Nullable x) {
        NSLog(@"%@", x);
    }];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    self.person.age = @(self.person.age.intValue + 1);
}
```
 
```objc
<RACTwoTuple: 0x6000012c2f60> (
    1,
        {
        kind = 1;
        new = 1;
    }
)
```

#### 监听事件:

* `rac_signalForControlEvents:`: 用于监听某个事件. 
* 把按钮点击事件转换为信号, 点击按钮, 就会发送信号. 

```objc
[[self.button rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(__kindof UIControl * _Nullable x) {
    NSLog(@"按钮被点击");
}];
```

```objc
按钮被点击
```

#### 代替通知:

* `rac_addObserverForName:`: 用于监听某个通知. 
* 只要发出这个通知, 又会转换成一个信号. 

```objc
[[NSNotificationCenter.defaultCenter rac_addObserverForName:UIKeyboardWillShowNotification object:nil] subscribeNext:^(NSNotification * _Nullable x) {
    NSLog(@"弹出键盘");
}];
```

```objc
弹出键盘
```

#### 监听文本框文字改变:

* `rac_textSignal`: 只要文本框发出改变就会发出这个信号. 
* 获取文本框文字改变的信号. 

```objc
[self.textfield.rac_textSignal subscribeNext:^(NSString * _Nullable x) {
    NSLog(@"%@", x);
}];
```

```objc
q
qq
qqq
```

#### `liftSelector:`

* 需求: 处理当界面有多次请求时, 需要都获取到数据时, 才能展示界面. 
* `rac_liftSelector:withSignalsFromArray:Signals:`当传入的`Signals`(信号数组), 每一个`signal`都至少`sendNext`过一次, 就会去触发第一个`selector`参数的方法. 
* 使用注意: 几个信号, 参数一的方法就几个参数, 每个参数对应信号发出的数据. 
* 不需要主动订阅`signalA`和`signalB`, 方法内部会自动订阅. 

```objc
- (void)liftSelector {
    RACSignal * signalA = [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
        uint64_t delayInSeconds = 2;
        dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, delayInSeconds * NSEC_PER_SEC);
        dispatch_after(popTime, dispatch_get_main_queue(), ^{
            [subscriber sendNext:@"singlaA"];
        });
        return nil;
    }];
    RACSignal * signalB = [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
        [subscriber sendNext:@"signalB"];
        [subscriber sendNext:@"another signalB"];
        [subscriber sendCompleted];
        return nil;
    }];
    [self rac_liftSelector:@selector(doA:WithB:) withSignals:signalA, signalB, nil];
}

- (void)doA:(NSString *)a WithB:(NSString *)b {
    NSLog(@"A: %@ and B: %@", a, b);
}
```

```objc
A: singlaA and B: another signalB
```

## RAC常见宏

#### `RAC(TARGET, ...)`

* 用于给某个对象的某个属性绑定. 把一个对象的某个属性绑定一个信号, 只要发出信号, 就会把信号的内容给对象的属性赋值. 
* 实现了 label 的内容跟随 textField 内容的改变而改变. 

```objc
RAC(self.label, text) = self.textField.rac_textSignal;
```

#### `RACObserve(TARGET, KEYPATH)`

* 用于给某个对象的某个属性绑定. 
* 快速的监听某个对象的某个属性改变. 
* 返回的是一个信号, 对象的某个属性改变的信号. 

```objc
[RACObserve(self.view, center) subscribeNext:^(id x) {
   NSLog(@"%@", NSStringFromCGRect(self.textField.frame));
}];
```

```objc
{ {198, 156}, {205, 30} }
```

#### `RACTuplePack`和`RACTupleUnpack`

* `RACTuplePack`: 把数据包装成`RACTuple`(元组类). 把包装的类型放在宏的参数里面, 就会自动包装. 
* `RACTupleUnpack`: 把`RACTuple`(元组类)解包成对应的数据. 等号的右边表示解析哪个元组. 宏的参数: 表示解析成什么类型. 

```objc
RACTuple * tuple = RACTuplePack(@1, @3);
NSLog(@"%@", tuple);
RACTupleUnpack(NSNumber * num1, NSNumber * num2) = tuple;
NSLog(@"%@, %@", num1, num2);
```

```objc
<RACTwoTuple: 0x6000024d42b0> (
    1,
    3
)
1, 3
```