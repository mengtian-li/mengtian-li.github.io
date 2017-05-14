---
layout: post
title:  "Mehtod Swizzling!"
date:   2017-05-14 15:43:59 +0800
categories: jekyll update
---
最近看`AFNetworking 3.1`源代码的时候，看到里面用到了`Method Swizzling`相关的技术，于是从网上查阅相关资料，做一个学习的记录。

先看`AFNetworking`里相关的代码实现，删除了源代码里的一些注释，加入了一些自己的注释，这里主要关注点在Method Swizzling相关技术。

```c
static inline void af_swizzleSelector(Class theClass, SEL originalSelector, SEL swizzledSelector) {
    //获取`theClass`的`originalSelector`对应方法的Method
    Method originalMethod = class_getInstanceMethod(theClass, originalSelector);
    //获取`theClass`的`swizzledSelector`对应方法的Method
    Method swizzledMethod = class_getInstanceMethod(theClass, swizzledSelector);
    //交换两个Method的实现
    method_exchangeImplementations(originalMethod, swizzledMethod);
}
```

查看文档，大概意思说`class_getInstanceMethod`这个方法，获取`aClass`中`aSelector`对应实例方法的实现，并且这个方法，若当前类不存在这个对应的实例方法，会一直向上搜寻父类中`aSelector`对应实例方法，直到返回NULL


### class_getInstanceMethod
> `Method class_getInstanceMethod(Class aClass, SEL aSelector);`  
>> The method that corresponds to the implementation of the selector specified by aSelector for the class specified by aClass, or NULL if the specified class or its superclasses do not contain an instance method with the specified selector.
>> Note that this function searches superclasses for implementations

查看文档，大概意思说`class_getInstanceMethod`这个方法，获取`aClass`中`aSelector`对应实例方法的实现，并且这个方法，若当前类不存在这个对应的实例方法，会一直向上搜寻父类中`aSelector`对应实例方法，直到返回NULL

### method_exchangeImplementations
> `void method_exchangeImplementations(Method m1, Method m2);`
> >Exchanges the implementations of two methods.

交换两个Mehtod的实现

```
static inline BOOL af_addMethod(Class theClass, SEL selector, Method method) {
    return class_addMethod(theClass, selector,  method_getImplementation(method),  method_getTypeEncoding(method));
}

```

### class_addMethod
> `BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);`
>> Adds a new method to a class with a given name and implementation.

### method_getImplementation
>`IMP method_getImplementation(Method m);`
>> Returns the implementation of a method.

### method_getTypeEncoding
>`const char * method_getTypeEncoding(Method m);`
>> Returns a string describing a method's parameter and return types.

再来看`AFNetworking`用到上述两个方法的地方


```c
+ (void)swizzleResumeAndSuspendMethodForClass:(Class)theClass {
    //获取af_resume af_suspend 方法
    Method afResumeMethod = class_getInstanceMethod(self, @selector(af_resume));
    Method afSuspendMethod = class_getInstanceMethod(self, @selector(af_suspend));

    //1.先将af_resume 加入到theClass里，再交换theClass resume 和 af_resumef方法的实现
    if (af_addMethod(theClass, @selector(af_resume), afResumeMethod)) {
        af_swizzleSelector(theClass, @selector(resume), @selector(af_resume));
    }

    if (af_addMethod(theClass, @selector(af_suspend), afSuspendMethod)) {
        af_swizzleSelector(theClass, @selector(suspend), @selector(af_suspend));
    }
}


- (void)af_resume {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    //这一步非常的关键，这里调用[self af_resume]，其实执行的是执行的 之前被交换的resume的实现
    [self af_resume];
    
    if (state != NSURLSessionTaskStateRunning) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidResumeNotification object:self];
    }
}

```

### 总结
AFNetworking 里是，像原有的类新加入了一个方法，然后和需要被交换的方法，交换实现。
1. 通过class_addMethod ，像原有类加入新的实例方法
2. 通过method_exchangeImplementations 交换方法
3. 若仍然需要原有被交换的方法的实现，只需要在新方法中，调用自己即可，因为自己已经和原有的方法实现进行了交换（= = 有点绕 自己看的懂就好）







