
## AFNetworking基于NSURLSession实现中遇到类簇的问题，

在AFURLSessionManager.m中，有一个私有分类`_AFURLSessionTaskSwizzling`，用来swziizle系统NSURLSessionTask对象的`state`属性，然后进行Task的对象改变通知告诉框架调用者进行回调处理。

```objc
@interface _AFURLSessionTaskSwizzling : NSObject

@end

@implementation _AFURLSessionTaskSwizzling

+ (void)load {
    /**
     WARNING: Trouble Ahead
     https://github.com/AFNetworking/AFNetworking/pull/2702
     */

    if (NSClassFromString(@"NSURLSessionTask")) {
        /**
         iOS 7 and iOS 8 differ in NSURLSessionTask implementation, which makes the next bit of code a bit tricky.
         Many Unit Tests have been built to validate as much of this behavior has possible.
         Here is what we know:
            - NSURLSessionTasks are implemented with class clusters, meaning the class you request from the API isn't actually the type of class you will get back.
            - Simply referencing `[NSURLSessionTask class]` will not work. You need to ask an `NSURLSession` to actually create an object, and grab the class from there.
            - On iOS 7, `localDataTask` is a `__NSCFLocalDataTask`, which inherits from `__NSCFLocalSessionTask`, which inherits from `__NSCFURLSessionTask`.
            - On iOS 8, `localDataTask` is a `__NSCFLocalDataTask`, which inherits from `__NSCFLocalSessionTask`, which inherits from `NSURLSessionTask`.
            - On iOS 7, `__NSCFLocalSessionTask` and `__NSCFURLSessionTask` are the only two classes that have their own implementations of `resume` and `suspend`, and `__NSCFLocalSessionTask` DOES NOT CALL SUPER. This means both classes need to be swizzled.
            - On iOS 8, `NSURLSessionTask` is the only class that implements `resume` and `suspend`. This means this is the only class that needs to be swizzled.
            - Because `NSURLSessionTask` is not involved in the class hierarchy for every version of iOS, its easier to add the swizzled methods to a dummy class and manage them there.
        
         Some Assumptions:
            - No implementations of `resume` or `suspend` call super. If this were to change in a future version of iOS, we'd need to handle it.
            - No background task classes override `resume` or `suspend`
         
         The current solution:
            1) Grab an instance of `__NSCFLocalDataTask` by asking an instance of `NSURLSession` for a data task.
            2) Grab a pointer to the original implementation of `af_resume`
            3) Check to see if the current class has an implementation of resume. If so, continue to step 4.
            4) Grab the super class of the current class.
            5) Grab a pointer for the current class to the current implementation of `resume`.
            6) Grab a pointer for the super class to the current implementation of `resume`.
            7) If the current class implementation of `resume` is not equal to the super class implementation of `resume` AND the current implementation of `resume` is not equal to the original implementation of `af_resume`, THEN swizzle the methods
            8) Set the current class to the super class, and repeat steps 3-8
         */
        NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        NSURLSession * session = [NSURLSession sessionWithConfiguration:configuration];
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wnonnull"
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];
#pragma clang diagnostic pop
        IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));
        Class currentClass = [localDataTask class];
        
        while (class_getInstanceMethod(currentClass, @selector(resume))) {
            Class superClass = [currentClass superclass];
            IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
            IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));
            if (classResumeIMP != superclassResumeIMP &&
                originalAFResumeIMP != classResumeIMP) {
                [self swizzleResumeAndSuspendMethodForClass:currentClass];
            }
            currentClass = [currentClass superclass];
        }
        
        [localDataTask cancel];
        [session finishTasksAndInvalidate];
    }
}

+ (void)swizzleResumeAndSuspendMethodForClass:(Class)theClass {
    Method afResumeMethod = class_getInstanceMethod(self, @selector(af_resume));
    Method afSuspendMethod = class_getInstanceMethod(self, @selector(af_suspend));

    if (af_addMethod(theClass, @selector(af_resume), afResumeMethod)) {
        af_swizzleSelector(theClass, @selector(resume), @selector(af_resume));
    }

    if (af_addMethod(theClass, @selector(af_suspend), afSuspendMethod)) {
        af_swizzleSelector(theClass, @selector(suspend), @selector(af_suspend));
    }
}

- (NSURLSessionTaskState)state {
    NSAssert(NO, @"State method should never be called in the actual dummy class");
    return NSURLSessionTaskStateCanceling;
}

- (void)af_resume {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_resume];
    
    if (state != NSURLSessionTaskStateRunning) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidResumeNotification object:self];
    }
}

- (void)af_suspend {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_suspend];
    
    if (state != NSURLSessionTaskStateSuspended) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidSuspendNotification object:self];
    }
}

@end
```

从上面一大坨的注释中，有一个很大的问题，就是NSURLSessionTask的实现在`iOS7`与`iOS8、iOS8+`下有着很大的差别，所以在swizzle的时候也就需要分别对待。

- (1) NSURLSessionTask 最低必须在 `iOS7` 系统上运行

```objc
if (NSClassFromString(@"NSURLSessionTask")) {
	// execute swizzle NSURLSessionTask ... 
}
```

- (2) 系统提供的一系列NSURLSessionTask、NSURLSessionDataTask、NSURLSessionUploadTask、NSURLSessionDownloadTask、NSURLSessionStreamTask、这些task类统统都是`类簇类`

即这些类并不是最终系统真正使用的真实类型。而要进行swizzle目标类的方法实现，就必须要要找到这个类型的真实类型，继而得到他真实的`objc_class`实例，继而进行SEL的指向交换，才能起作用。

- (3) iOS7与iOS7+系统下，这些Task类最终私有内部类，继承结构有所不同

![](http://i4.buimg.com/4851/ff9ad2b9a4ee550c.png)

On iOS 7，`__NSCFLocalDataTask > __NSCFLocalSessionTask > __NSCFURLSessionTask`

On iOS 8，`__NSCFLocalDataTask > __NSCFLocalSessionTask > NSURLSessionTask`

- (4) iOS7，`__NSCFLocalSessionTask`与`__NSCFURLSessionTask`都实现了`resume`与`suspend`。但是`__NSCFLocalSessionTask`中这两个方法实现，都没有调用`super`，即没有调用`__NSCFLocalSessionTask`的两个方法实现

所以在iOS7时，`__NSCFLocalSessionTask`与`__NSCFURLSessionTask`这两个类，都需要进行一次swizzle。

- (5) iOS8、iOS8+，只有`NSURLSessionTask`实现了`resume`与`suspend`，所以只需要swizzle`NSURLSessionTask`的两个方法实现

- (6) 随便获取一个dataTask对象，然后找到如上情况的Task私有类，进行swizzle两个方法的实现

## AFN获取真实类型的步骤

获取一个真实task对象

```objc
NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        NSURLSession * session = [NSURLSession sessionWithConfiguration:configuration];
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wnonnull"
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];
#pragma clang diagnostic pop
```

找到这个task对象的真实私有的内部类型（`__NSCFLocalDataTask`）

```objc
IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));
Class currentClass = [localDataTask class];
```

根据iOS7和iOS7+区别，iOS7需要swizzle两个类的实现，iOS8只需要swizzle一个类的实现:

```objc
while (class_getInstanceMethod(currentClass, @selector(resume))) {
	// __NSCFLocalSessionTask、__NSCFURLSessionTask、NSURLSessionTask	
		
    Class superClass = [currentClass superclass];
    IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
    IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));
    if (classResumeIMP != superclassResumeIMP &&
        originalAFResumeIMP != classResumeIMP) {
        [self swizzleResumeAndSuspendMethodForClass:currentClass];
    }
    currentClass = [currentClass superclass];
}
```
