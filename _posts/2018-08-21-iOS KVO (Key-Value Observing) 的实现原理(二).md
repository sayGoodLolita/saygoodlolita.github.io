---
layout: post
title: 'iOS KVO (Key-Value Observing) 的实现原理(二)'
date: 2018-08-21
author: Cheney
color: rgb(255,210,32)
cover: '../assets/test.png'
tags: iOS底层原理
---

# iOS KVO (Key-Value Observing) 的实现原理(二)

如果给一个对象添加了`KVO`, 则会利用`runtime`动态生成原类的`NSKVONotifying_`前缀的子类派生类, 使对象的`isa`指向这个派生类. 派生类重写了`set`方法, 通过对象调用`methodForSelector:`方法拿到`IMP`指针验证方法实现列表为
```objc
nm Foundation  | grep ValueAndNotify
__NSSetBoolValueAndNotify
__NSSetCharValueAndNotify
__NSSetDoubleValueAndNotify
__NSSetFloatValueAndNotify
__NSSetIntValueAndNotify
__NSSetLongLongValueAndNotify
__NSSetLongValueAndNotify
__NSSetObjectValueAndNotify
__NSSetPointValueAndNotify
__NSSetRangeValueAndNotify
__NSSetRectValueAndNotify
__NSSetShortValueAndNotify
__NSSetSizeValueAndNotify
```
这个方法的内部实现
```objc
__NSSetBoolValueAndNotify() {
	[self willChangeValueForKey:@""];
	// 调用父类的 set 方法, 然后
	[self didChangeValueForKey:@""];	// 这个方法里面调用 observeValueForKeyPath
}
```

派生类还重写了`dealloc`, `_isKVOA`以及`class`方法
重写`class`方法是为了隐藏派生类的存在, 使调用`class`方法总会返回最原始的类对象, 大概实现为
```objc
- (Class)class {
     // 得到类对象, 在找到类对象父类
     return class_getSuperclass(object_getClass(self));
}
```

手动调用`willChangeValueForKey`和`didChangeValueForKey`可以唤起`KVO`
本质上是重写`set`方法, 直接修改成员变量不会触发`KVO`

## KVC (Key-Value Coding)
使用`KVC`给对象的实例变量赋值会触发`KVO`

`setValue:forKey:`执行步骤
* 找到`Key`对应的`set`方法并执行
* 在`Key`对应的`set`方法前加`_`找到并执行
* 查看`accessInstanceVariablesDirectly`方法的返回值, 如果为`NO`则不允许访问实例变量
* 按照`_key`, `_isKey`, `key`, `isKey`顺序查找成员变量, 并直接赋值. 此时会调用`willChangeValueForKey`和`didChangeValueForKey`, 所以还是会触发`KVO`

`valueForKey:`执行步骤
* 按照`getKey`, `key`, `isKey`, `_key`顺序查找方法并调用
* 查看`accessInstanceVariablesDirectly`方法的返回值, 如果为`NO`则不允许访问实例变量
* 按照`_key`, `_isKey`, `key`, `isKey`顺序查找成员变量, 取出其中的值

`KVC`的方法在执行到`accessInstanceVariablesDirectly`返回值为`NO`或者最后一步没有找到实例变量会调用`setValue:forUndefinedKey:`并抛出异常`NSUnknownKeyException`