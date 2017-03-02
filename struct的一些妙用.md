## struct应用一、struct作为多个参数的打包器Context


## struct应用二、CoreFoundation Array/Set/Dic遍历时，struct传入作为公共访问的内存数据

```c
// 每次遍历传递进去的context
typedef struct CFDicApplyContext {
    void    *info;
}CFDicApplyContext;

// 每次遍历被回调的函数
void CFDictionaryApplyFunc(const void *key, const void *value, void *context) {
    
    //1.
    CFDicApplyContext *ctx = (CFDicApplyContext *)context;
    
    //2.
    NSMutableString *info = (__bridge NSMutableString*)(ctx->info);
    
    //3.
    NSString *tmp = (__bridge NSString*)(value);
    
    //4.
    [info appendFormat:@"___%@", tmp];
}

// 测试函数
void func8() {
    
    //1.
    static CFMutableDictionaryRef dic = NULL;
    dic = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 32, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
    
    //2.
    CFDictionarySetValue(dic, (__bridge void*)(@"key1"), (__bridge void*)(@"A"));
    CFDictionarySetValue(dic, (__bridge void*)(@"key2"), (__bridge void*)(@"B"));
    CFDictionarySetValue(dic, (__bridge void*)(@"key3"), (__bridge void*)(@"C"));
    CFDictionarySetValue(dic, (__bridge void*)(@"key4"), (__bridge void*)(@"D"));
    CFDictionarySetValue(dic, (__bridge void*)(@"key5"), (__bridge void*)(@"E"));
    
    //3.
    CFDicApplyContext ctx = {0};
    ctx.info = (__bridge_retained void*)([[NSMutableString alloc] init]);//__bridge_retained对oc对象retain，防止被释放废弃
    
    //4.
    CFDictionaryApplyFunction(dic,
                              CFDictionaryApplyFunc,
                              &ctx);
    
    //5.
    NSLog(@"%@", (__bridge_transfer NSMutableString*)(ctx.info));//因为之前retain后，所以此时需要release
    
}
```

输出结果

```
2017-02-10 23:08:50.708 CJiaJiaDemo[1959:55007] ___C___A___D___B___E
```


## struct应用三、位段结构体

> 位段结构体常用于OC中判断是否实现了声明的Delegate中的协议方法时

定义格式

```c
struct __touchDelegate {
    
    //格式:
    变量类型 成员名 : 分配的长度;
    
    //例子:
    unsigned int touchBegin : 1;
};
```

OC中delegate的定义

```objc
@protocol TouchDelegate : NSObject

- (void)touchBegin;
- (void)touchMoved;
- (void)touchEnd;

@end
```

对于上面Protocol对于的位段结构体定义，因为只需要判断TargetClass是否实现了如上的协议方法，所以只需要两种值: 

- 1)true/1 2)false/0
- 归根结底就是 0/1 >>> 只需要`一个二进制位`即可表示
- so，位段结构体成员长度全部分配为1

```c
struct __touchDelegate {
    unsigned int touchBegin : 1;
    unsigned int touchMoved : 1;
    unsigned int touchEnd   : 1;
};
```

然后在接收一个id<TouchDelegate>的setter方法内判断delegate目标类/对象是否已经实现了协议方法，后面直接根据位段结构体实体来判断是否已经实现.（因为一般不会出现之前实现了协议方法，而后面又没有实现的情况，除非是runtime时动态的移除了IMP）

```objc
- (void)setDelegate:(id<TouchDelegate>)delegate {
    _delegate = delegate;
    
    if ([delegate respondsToSelector:@selector(touchBegin)]) {
        __touchDelegate.touchBegin = 1;
    }
    
    if ([delegate respondsToSelector:@selector(touchMoved)]) {
        __touchDelegate.touchMoved = 1;
    }
    
    if ([delegate respondsToSelector:@selector(touchEnd)]) {
        __touchDelegate.touchEnd = 1;
    }
}
```

而后面调用delegate对应的方法时，根据上面的位段结构体实例的对应数据项进行回调协议方法.