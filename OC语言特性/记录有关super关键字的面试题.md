# 问题一 
> `super`和`performSelector:`一起使用的问题，即`[super performSelector:@selector(xxx)]`执行过程是什么？

先看题目,请认真审题

```
@interface A : NSObject
- (void)myMethod;
@end

@implementation A
- (void)myMethod {
    NSLog(@"AAAA");
}
@end

@interface B : A
- (void)myMethod;
@end

@implementation B
- (void)myMethod {
    [super performSelector:@selector(myMethod)];
}
@end

执行
B *b = [[B alloc]init];
[b myMethod];
```
好的，题目看完了，给你半分钟的思考时间，请回答执行结果是什么？

> 什么？你的回答是执行打印AAAA，那么恭喜你，回答错了。先说答案吧，这个结果是死循环。为什么是死循环？接下来看我给你分析

**执行流程**

1. `[b myMethod]`调用`B`的`myMethod`
2. 执行 `[super performSelector:@selector(myMethod)]`
3. 编译器生成：`objc_msgSendSuper((struct objc_super){self, [A class]}, @selector(performSelector:))`
4. 从`A`类开始查找`performSelector:`方法
5. 找到`NSObject`的`performSelector:`方法
6. 在B的实例上调用`performSelector:`
7. `performSelector:`内部调用 `objc_msgSend(self, @selector(myMethod))`
8. 从`B`类开始查找`myMethod`
9. 找到`B`的`myMethod`，调用
10. 回到步骤2，无限循环！

**理解步骤3是关键`objc_msgSendSuper`的原理**

```
// objc_msgSendSuper的结构
struct objc_super {
    id receiver;        // 消息接收者（当前对象）
    Class super_class;  // 父类（从哪里开始查找方法）
};

// 当调用 [super performSelector:@selector(myMethod)] 时：
objc_msgSendSuper((struct objc_super){
    .receiver = self,        // self是B的实例
    .super_class = [A class] // 从A类开始查找
}, @selector(performSelector:));

// 关键：receiver仍然是B的实例，不是A的实例！
```
通过上边看源码，可以分析出，`objc_msgSendSuper`是从父类开始查找方法`performSelector:`,但是消息接收者还是自身,从继承链查到根类实现了`performSelector:`。这个时候就相当于实例b直接调用`performSelector:`了。

**可以再看下`performSelector:`的大概实现**

```
// NSObject中performSelector的实现（简化版）
@implementation NSObject

- (id)performSelector:(SEL)aSelector {
    if (!aSelector) {
        [NSException raise:NSInvalidArgumentException format:@"Selector cannot be nil"];
    }
    
    // 关键：这里调用objc_msgSend，从当前类开始查找
    return objc_msgSend(self, aSelector);
}

- (id)performSelector:(SEL)aSelector withObject:(id)object {
    if (!aSelector) {
        [NSException raise:NSInvalidArgumentException format:@"Selector cannot be nil"];
    }
    
    return objc_msgSend(self, aSelector, object);
}

@end
```

**`objc_msgSendSuper`和`objc_msgSend`的对比**

```
// objc_msgSend：从当前类开始查找，在当前对象上调用
objc_msgSend(self, @selector(myMethod));
// 1. 从B类开始查找myMethod
// 2. 在self(B的实例)上调用

// objc_msgSendSuper：从父类开始查找，在当前对象上调用
objc_msgSendSuper((struct objc_super){self, [A class]}, @selector(performSelector:));
// 1. 从A类开始查找performSelector:
// 2. 在self(B的实例)上调用
```
既然都看到这里了，那就再看看下边的会执行什么结果？
是在上边的题目上的扩展，B还是继承A

**扩展一**
```
@implementation B

- (void)myMethod {
    [self performSelector:@selector(myMethod)];
}

@end
```

**扩展二**

```
@implementation B

- (void)myMethod {
    NSLog(@"%@",[super class]);
    [super myMethod];
}

@end
```

