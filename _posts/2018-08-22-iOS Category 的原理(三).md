---
layout: post
title: 'iOS Category 的原理(三)'
date: 2018-08-22
author: Cheney
color: rgb(255,210,32)
cover: '../assets/test.png'
tags: iOS底层原理
---

# iOS Category 的原理(三)

`Category`会在编译时转化为以下结构体
```objc
struct _category_t {
  const char *name; 								// 类名
  struct _class_t *cls;			
  const struct _method_list_t *instance_methods;	// 对象方法列表
  const struct _method_list_t *class_methods;		// 类方法列表
  const struct _protocol_list_t *protoc ols;		// 协议列表
  const struct _prop_list_t *properties;			// 属性列表
};
```

  1. 通过`runtime`加载类的全部`Category`
2. 把全部`Category`的方法, 属性, 协议合并到一个大数组中, 后编译的数组会出现在数组前面
3. 然后会扩容类一开始的数组内存, 通过`memmove`, `memcpy`来把数组插到类原来数据的前面. 所以`Category`的方法会优先调用
而`Extension`在编译时就会合并到类对象里面, 和`Category`不同

`memcpy`做单位copy, 不会做判断, `memmove`可以保证移动的完整性

在分类里面写属性, 系统只会生成`set`, `get`方法的声明, 不会生成成员变量和方法的实现, 且编译器不允许分类里面存放成员变量, 但是可以间接实现分类里添加成员变量
`objc_setAssociatedObject`可以添加关联对象
* 关联的对象
* 关联属性的`key`
* 关联属性的`value`
* 关联策略
`objc_getAssociatedObject`可以获取关联对象
* 获取的对象
* 关联属性的`key`
`objc_removeAssociatedObjects`移除所有的关联对象
设置关联属性的值为`nil`, 相当于移除关联对象

关联属性的`key`可以直接用`get`方法的地址, 每个方法都会默认传两个参数`self`和`_cmd`, 后者就是当前方法的地址

`objc_setAssociatedObject`方法内部实现结构使用以下关键对象
* `AssociationsManager`为全局统一的管理者
* `AssociationsHashMap`为对应不同类的对象
* `ObjectAssociationMap`为对应不同的属性
* `ObjcAssociation`为存放的值
`AssociationsManager`对象里面有`AssociationsHashMap`对象, `AssociationsHashMap`对象的`key`就是传入的`object`, `value`为`ObjectAssociationMap`对象, `ObjectAssociationMap`对象的`key`就是传入的`key`, `value`为`ObjcAssociation`对象, `ObjcAssociation`里面有传入的`value`对应的`_value`以及`policy`对应的`_policy`.

使用`objc_setAssociatedObject`添加的关联对象不是存储在被关联对象本身内存中, 不会影响被关联对象的实例变量列表, 而是存储在全局统一的`AssociationsManager`中

如果对象被销毁, 关联属性也会被自动移除

## `load`和`initialize`方法

#### `load`
`load`方法在`runtime`加载类或者分类的时候直接找到方法地址调用, 类和分类的`load`方法都会被调用且只调用一次
而其他的方法是用`objc_msgSend`消息机制执行
`load`方法的执行顺序
* 先调用类的`load`方法, 
	1. 根据编译顺序调用`load`方法
	2. 调用子类的`load`方法之前会调用父类的`load`方法
* 再调用`Category`的`load`方法
	1. 根据编译顺序调用`load`方法

如果手动调用`load`方法, 等同于`objc_msgSend`, 所以存在继承

#### `initialize`
`initialize`会在类第一次接收到消息时使用消息发送机制调用, 会在`lookUpImpOrForward`寻找方法里判断是否初始化过, 然后通过`objc_msgSend`方法调用
`initialize`方法的执行顺序
* 调用子类的`initialize`方法之前会调用父类的`initialize`方法

### `load`方法和`initialize`的区别
`initialize`是通过`objc_msgSend`调用, `load`方法找到地址直接调用, 所以`initialize`存在以下特点
1. 如果子类没有实现`initialize`, 则会调用父类的`initialize`, 也就是说父类的`initialize`会调用多次
2. 如果分类实现了`initialize`, 则会覆盖掉类本身的`initialize`













