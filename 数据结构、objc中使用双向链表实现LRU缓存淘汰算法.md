

```c
#import <pthread.h>
```

## 双向链表节点

```objc
@interface __XZHNetworkLRUNode : NSObject {
    @package
    __unsafe_unretained __XZHNetworkLRUNode *_prev;     // 前一个node（注意：node之间不需要存在持有的关系）
    __unsafe_unretained __XZHNetworkLRUNode *_next;     // 后一个node
    NSUInteger                              _cost;      // node存储数据的长度
    NSTimeInterval                          _lastTime;  // node最后使用时间
    NSTimeInterval                          _aliveTime; // 存活时间
    NSString                                *_key;      // node的key
    id                                      _value;     // node的value
    NSNumber                                *_version;  // node的版本号
}
@end

@implementation __XZHNetworkLRUNode

-(NSString *)description {
    
#pragma clang diagnostic push
#pragma clang diagnostic ignored"-Wformat"
    return [NSString stringWithFormat:@"<%@ - %p> _key = %@", [self class], self, _key];
#pragma clang diagnostic pop
}

- (NSString *)debugDescription {
    return [self description];
}

#if DEBUG
- (void)dealloc {
    NSLog(@"【dealloc】%@:【thread: %@】", [self description], [NSThread currentThread]);
}
#endif

@end
```

## 双向链表结构、（不带头结点）

```
  LRU淘汰对象策略:
 *  - 最新使用的对象，使用【头插法】查到表头
 *  - 【表末尾】的节点是每次被【淘汰】的缓存对象
 *  - 中间的节点被【使用】后，重新插入到【表头】
```

```objc
@interface __XZHNetworkLRUCache : NSObject {
    @package
    CFMutableDictionaryRef                  _dic;                       // 所有的 __XZHNetworkLRUNode对象，都只由这个dic来持有
    __unsafe_unretained __XZHNetworkLRUNode *_head;                     // 记录整个链表的 第一个 数据节点
    __XZHNetworkLRUNode                     *_tail;                     // 记录整个链表的 最后一个 数据节点（注意：需要持有，在removeTailNode时，需要返回给调用者继续使用）
    NSUInteger                              _totalCost;                 // 记录链表中所有node占用内存的总大小
    NSUInteger                              _totalCount;                // 记录链表中node总个数
    BOOL                                    _releaseNodeOnMainThread;   // 是否在主线程执行释放对象（1、主线程  2、子线程）
    BOOL                                    _releaseAsynchronously;     // 是否异步释放容器对象（1、同步完成释放与废弃 2、异步完成释放与废弃）
}

// 1. 添加新的node，并使用头插法
- (void)addNodeToHead:(__XZHNetworkLRUNode *)node;

// 2. 将已经存在于链表中的node，调整为链表的第一个数据节点
- (void)bringNodeToHead:(__XZHNetworkLRUNode *)node;

// 3. 删除链表中某一个位置的node（缓存项被淘汰）
- (void)removeNode:(__XZHNetworkLRUNode *)node;

// 4. 删除链表中最后一个node（最久未被使用）
- (__XZHNetworkLRUNode *)removeTailNode;

// 5. 清除所有缓存
- (void)removeAllNode;

//6. 打印链表中所有节点
- (void)debugAllNode;

@end
```

```objc
@implementation __XZHNetworkLRUCache

- (instancetype)init {
    if (self = [super init]) {
        _dic = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 1, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        _head = _tail = nil;
        _totalCost = _totalCount = 0;
        _releaseNodeOnMainThread = NO;
        _releaseAsynchronously = YES;
    }
    return self;
}

- (void)addNodeToHead:(__XZHNetworkLRUNode *)node {
    if (!node) return;
    
    //1. 插入的节点统一由 _dic 来持有
    NSString *key = node->_key ? node->_key : @"";
    CFDictionarySetValue(_dic, (__bridge const void *)key, (__bridge const void *)(node));
    
    //2. 统计
    _totalCost += node->_cost;
    _totalCount++;
    
    //3. 区分链表是否存在数据节点
    if (_head) {
        node->_next = _head;
        _head->_prev = node;
        _head = node;
    } else {
        _head = _tail = node;
    }
}

- (void)bringNodeToHead:(__XZHNetworkLRUNode *)node {
    if (!node) return;
    
    //1. 链表只存在一个数据节点
    if (node == _head) return;
    
    //2. 区分调整的是否是链表最后一个节点，将被调整节点从链表中单独取出
    if (_tail == node) {
        _tail = node->_prev;
        _tail->_next = nil;
    } else {
        node->_next->_prev = node->_prev;
        node->_prev->_next = node->_next;
    }
    
    //3. 将取出的节点重新调整到链表第一个位置
    node->_next = _head;
    node->_prev = nil;
    _head->_prev = node;
    _head = node;
}

- (void)removeNode:(__XZHNetworkLRUNode *)node {
    if (!node) return;
    
    //1. 从唯一持有node的 _dic 中解除持有，此时node.retainCount == 0
    NSString *key = node->_key ? node->_key : @"";
    CFDictionaryRemoveValue(_dic, (__bridge const void *)(key));//retainCount==1
    
    //2. 统计
    _totalCost -= node->_cost;
    _totalCount--;
    
    //3. 将删除的节点从链表中进行移除
    if (node->_next) node->_next->_prev = node->_prev;
    if (node->_prev) node->_prev->_next = node->_next;
    if (_head == node) _head = node->_next;
    if (_tail == node) _tail = node->_prev;
}

- (__XZHNetworkLRUNode *)removeTailNode {
    if (!_tail) return nil;
    
    //1. node.retainCount == 2
    __XZHNetworkLRUNode *tail = _tail;
    
    //2. node.retainCount == 1
    NSString *key = tail->_key ? tail->_key : @"";
    CFDictionaryRemoveValue(_dic, (__bridge const void *)(key));
    
    //3.
    _totalCost -= _tail->_cost;
    _totalCount--;
    
    //4. 将node从链表中进行移除
    if (_head == _tail) {
        _head = _tail = nil;
    } else {
        _tail = _tail->_prev;
        _tail->_next = nil;
    }
    
    /**
     *  5. 返回node，此时node.retainCount == 1，
     *     如果之前使用 __unsafe_unretained修饰 _tail，
     *     那么此时node.retainCount == 0，返回出去使用可能导致崩溃
     */
    return tail;
}

- (void)removeAllNode {
    
    //1.
    _totalCost = 0;
    _totalCount = 0;
    
    //2. 分别执行[_head release]与[_tail release]，此时 _head与_tail指向的node对象.retainCount == 1，只有 _dic 持有
    _head = nil;
    _tail = nil;
    
    //3. 异步子线程释放所有的node对象
    if (CFDictionaryGetCount(_dic) > 0) {
        
        //3.1 所有的 node.retainCount == 2
        CFMutableDictionaryRef _releaseDic = _dic;
        
        //3.2 所有的 node.retainCount == 1
        _dic = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);//retainCount==1，并使用新的缓存字典，因为内部对象是异步释放
        
        //3.3 是否主线程释放废弃nodes？ 是否异步释放废弃nodes？
        if (_releaseAsynchronously) {
            
            // 确定是异步释放废弃，还需要确定是 1)主线程 2)主线程 上进行
            dispatch_queue_t queue = _releaseNodeOnMainThread ? dispatch_get_main_queue() : dispatch_get_global_queue(0, 0);
            
            // 将所有nodes的释放与废弃的执行，带入到【子线程/主线程】上异步完成
            dispatch_async(queue, ^{
                CFRelease(_releaseDic);
            });
            
        } else if (_releaseNodeOnMainThread && !pthread_main_np()) {
            
            // 将所有nodes的释放与废弃的执行，带入到【主线程】上异步完成
            dispatch_async(dispatch_get_main_queue(), ^{
                CFRelease(_releaseDic);//retainCount==0
            });
            
        } else {
            
            // 将所有nodes的释放与废弃的执行，直接在【当前线程】上同步执行
            CFRelease(_releaseDic);//retainCount==0
        }
    }
}

- (void)debugAllNode {
    NSLog(@"====================begin====================");
    __XZHNetworkLRUNode *node = _head;
    while (node) {
        NSLog(@"node.key = %@", node->_key);
        node = node->_next;
    }
}

@end
```

## 测试1

```objc
- (void)test1 {
    
    //1.
    __XZHNetworkLRUCache *lru = [[__XZHNetworkLRUCache alloc] init];
    
    //2.
    NSMutableArray *nodes = [NSMutableArray new];
    for (int i = 0; i < 10; i++) {
        __XZHNetworkLRUNode *newNode = [[__XZHNetworkLRUNode alloc] init];
        newNode->_key = [NSString stringWithFormat:@"key_%d", i + 1];
        [lru addNodeToHead:newNode];
        [nodes addObject:newNode];
    }
    
    //3.
    [lru debugAllNode];
    
    //4.
    for (int i = 9; i >= 0; i--) {
        [lru bringNodeToHead:nodes[i]];
    }
    
    //5.
    [lru debugAllNode];
    
    //6.
    [lru removeTailNode];
    [lru removeTailNode];
    [lru removeTailNode];
    [lru removeTailNode];
    [lru debugAllNode];
    
    //6.
    [lru removeAllNode];
    
    //7. 注意，nodes数组，会继续持有所有的node对象，导致(6)之后，node对象都没有被释放废弃
}
```

输出信息

```
2017-02-09 13:04:53.304 LRUCacheDemo[4156:119288] ====================begin====================
2017-02-09 13:04:53.305 LRUCacheDemo[4156:119288] node.key = key_10
2017-02-09 13:04:53.305 LRUCacheDemo[4156:119288] node.key = key_9
2017-02-09 13:04:53.305 LRUCacheDemo[4156:119288] node.key = key_8
2017-02-09 13:04:53.306 LRUCacheDemo[4156:119288] node.key = key_7
2017-02-09 13:04:53.306 LRUCacheDemo[4156:119288] node.key = key_6
2017-02-09 13:04:53.306 LRUCacheDemo[4156:119288] node.key = key_5
2017-02-09 13:04:53.306 LRUCacheDemo[4156:119288] node.key = key_4
2017-02-09 13:04:53.306 LRUCacheDemo[4156:119288] node.key = key_3
2017-02-09 13:04:53.307 LRUCacheDemo[4156:119288] node.key = key_2
2017-02-09 13:04:53.307 LRUCacheDemo[4156:119288] node.key = key_1
2017-02-09 13:04:53.307 LRUCacheDemo[4156:119288] ====================begin====================
2017-02-09 13:04:53.307 LRUCacheDemo[4156:119288] node.key = key_1
2017-02-09 13:04:53.307 LRUCacheDemo[4156:119288] node.key = key_2
2017-02-09 13:04:53.307 LRUCacheDemo[4156:119288] node.key = key_3
2017-02-09 13:04:53.308 LRUCacheDemo[4156:119288] node.key = key_4
2017-02-09 13:04:53.308 LRUCacheDemo[4156:119288] node.key = key_5
2017-02-09 13:04:53.308 LRUCacheDemo[4156:119288] node.key = key_6
2017-02-09 13:04:53.308 LRUCacheDemo[4156:119288] node.key = key_7
2017-02-09 13:04:53.308 LRUCacheDemo[4156:119288] node.key = key_8
2017-02-09 13:04:53.309 LRUCacheDemo[4156:119288] node.key = key_9
2017-02-09 13:04:53.309 LRUCacheDemo[4156:119288] node.key = key_10
2017-02-09 13:04:53.309 LRUCacheDemo[4156:119288] ====================begin====================
2017-02-09 13:04:53.309 LRUCacheDemo[4156:119288] node.key = key_1
2017-02-09 13:04:53.309 LRUCacheDemo[4156:119288] node.key = key_2
2017-02-09 13:04:53.309 LRUCacheDemo[4156:119288] node.key = key_3
2017-02-09 13:04:53.310 LRUCacheDemo[4156:119288] node.key = key_4
2017-02-09 13:04:53.310 LRUCacheDemo[4156:119288] node.key = key_5
2017-02-09 13:04:53.310 LRUCacheDemo[4156:119288] node.key = key_6

2017-02-09 13:04:53.310 LRUCacheDemo[4156:119288] 【dealloc】<__XZHNetworkLRUNode - 0x6080002800a0> _key = key_7:【thread: <NSThread: 0x600000260380>{number = 1, name = main}】
2017-02-09 13:04:53.310 LRUCacheDemo[4156:119454] 【dealloc】<__XZHNetworkLRUNode - 0x60800009f810> _key = key_1:【thread: <NSThread: 0x6080002727c0>{number = 3, name = (null)}】
2017-02-09 13:04:53.311 LRUCacheDemo[4156:119288] 【dealloc】<__XZHNetworkLRUNode - 0x60800009edc0> _key = key_8:【thread: <NSThread: 0x600000260380>{number = 1, name = main}】
2017-02-09 13:04:53.311 LRUCacheDemo[4156:119454] 【dealloc】<__XZHNetworkLRUNode - 0x60800009dec0> _key = key_3:【thread: <NSThread: 0x6080002727c0>{number = 3, name = (null)}】
2017-02-09 13:04:53.311 LRUCacheDemo[4156:119288] 【dealloc】<__XZHNetworkLRUNode - 0x60800009eaa0> _key = key_9:【thread: <NSThread: 0x600000260380>{number = 1, name = main}】
2017-02-09 13:04:53.311 LRUCacheDemo[4156:119454] 【dealloc】<__XZHNetworkLRUNode - 0x6080002817c0> _key = key_5:【thread: <NSThread: 0x6080002727c0>{number = 3, name = (null)}】
2017-02-09 13:04:53.311 LRUCacheDemo[4156:119288] 【dealloc】<__XZHNetworkLRUNode - 0x60800009e8c0> _key = key_10:【thread: <NSThread: 0x600000260380>{number = 1, name = main}】
2017-02-09 13:04:53.312 LRUCacheDemo[4156:119454] 【dealloc】<__XZHNetworkLRUNode - 0x608000282940> _key = key_2:【thread: <NSThread: 0x6080002727c0>{number = 3, name = (null)}】
2017-02-09 13:04:53.312 LRUCacheDemo[4156:119454] 【dealloc】<__XZHNetworkLRUNode - 0x608000282530> _key = key_4:【thread: <NSThread: 0x6080002727c0>{number = 3, name = (null)}】
2017-02-09 13:04:53.312 LRUCacheDemo[4156:119454] 【dealloc】<__XZHNetworkLRUNode - 0x60800009fa40> _key = key_6:【thread: <NSThread: 0x6080002727c0>{number = 3, name = (null)}】
```

可以看到，全部的node对象，都是在`主线程`上完成释放废弃的，这是一个问题。

## 测试2

```objc
- (void)test2 {
    
    //1.
    __XZHNetworkLRUCache *lru = [[__XZHNetworkLRUCache alloc] init];
    
    //2.
    for (int i = 0; i < 10; i++) {
        __XZHNetworkLRUNode *newNode = [[__XZHNetworkLRUNode alloc] init];
        newNode->_key = [NSString stringWithFormat:@"key_%d", i + 1];
        [lru addNodeToHead:newNode];
    }
    
    //3.
    [lru debugAllNode];
    
    //4.
    [lru removeTailNode];
    [lru removeTailNode];
    [lru removeTailNode];
    [lru removeTailNode];
    [lru debugAllNode];
    
    //5.
    [lru removeAllNode];
    
    //6. 这里没有nodes数组持有node对象，所有此时都会被释放废弃
}
```

输出信息

```
2017-02-09 13:05:45.894 LRUCacheDemo[4174:120413] ====================begin====================
2017-02-09 13:05:45.894 LRUCacheDemo[4174:120413] node.key = key_10
2017-02-09 13:05:45.894 LRUCacheDemo[4174:120413] node.key = key_9
2017-02-09 13:05:45.894 LRUCacheDemo[4174:120413] node.key = key_8
2017-02-09 13:05:45.894 LRUCacheDemo[4174:120413] node.key = key_7
2017-02-09 13:05:45.895 LRUCacheDemo[4174:120413] node.key = key_6
2017-02-09 13:05:45.895 LRUCacheDemo[4174:120413] node.key = key_5
2017-02-09 13:05:45.895 LRUCacheDemo[4174:120413] node.key = key_4
2017-02-09 13:05:45.895 LRUCacheDemo[4174:120413] node.key = key_3
2017-02-09 13:05:45.895 LRUCacheDemo[4174:120413] node.key = key_2
2017-02-09 13:05:45.895 LRUCacheDemo[4174:120413] node.key = key_1

2017-02-09 13:05:45.929 LRUCacheDemo[4174:120413] ====================begin====================
2017-02-09 13:05:45.930 LRUCacheDemo[4174:120413] node.key = key_10
2017-02-09 13:05:45.930 LRUCacheDemo[4174:120413] node.key = key_9
2017-02-09 13:05:45.930 LRUCacheDemo[4174:120413] node.key = key_8
2017-02-09 13:05:45.930 LRUCacheDemo[4174:120413] node.key = key_7
2017-02-09 13:05:45.930 LRUCacheDemo[4174:120413] node.key = key_6
2017-02-09 13:05:45.931 LRUCacheDemo[4174:120413] node.key = key_5

2017-02-09 13:05:45.896 LRUCacheDemo[4174:120413] 【dealloc】<__XZHNetworkLRUNode - 0x608000099af0> _key = key_2:【thread: <NSThread: 0x600000068e80>{number = 1, name = main}】
2017-02-09 13:05:45.896 LRUCacheDemo[4174:120413] 【dealloc】<__XZHNetworkLRUNode - 0x6080000998c0> _key = key_3:【thread: <NSThread: 0x600000068e80>{number = 1, name = main}】
2017-02-09 13:05:45.896 LRUCacheDemo[4174:120413] 【dealloc】<__XZHNetworkLRUNode - 0x6080000991e0> _key = key_4:【thread: <NSThread: 0x600000068e80>{number = 1, name = main}】
2017-02-09 13:05:45.931 LRUCacheDemo[4174:120413] 【dealloc】<__XZHNetworkLRUNode - 0x608000094ff0> _key = key_1:【thread: <NSThread: 0x600000068e80>{number = 1, name = main}】

2017-02-09 13:05:45.931 LRUCacheDemo[4174:120459] 【dealloc】<__XZHNetworkLRUNode - 0x608000098f10> _key = key_8:【thread: <NSThread: 0x608000266f00>{number = 4, name = (null)}】
2017-02-09 13:05:45.931 LRUCacheDemo[4174:120459] 【dealloc】<__XZHNetworkLRUNode - 0x608000099460> _key = key_10:【thread: <NSThread: 0x608000266f00>{number = 4, name = (null)}】
2017-02-09 13:05:45.931 LRUCacheDemo[4174:120459] 【dealloc】<__XZHNetworkLRUNode - 0x6080000992d0> _key = key_5:【thread: <NSThread: 0x608000266f00>{number = 4, name = (null)}】
2017-02-09 13:05:45.931 LRUCacheDemo[4174:120459] 【dealloc】<__XZHNetworkLRUNode - 0x608000099000> _key = key_7:【thread: <NSThread: 0x608000266f00>{number = 4, name = (null)}】
2017-02-09 13:05:45.932 LRUCacheDemo[4174:120459] 【dealloc】<__XZHNetworkLRUNode - 0x608000098600> _key = key_9:【thread: <NSThread: 0x608000266f00>{number = 4, name = (null)}】
2017-02-09 13:05:45.932 LRUCacheDemo[4174:120459] 【dealloc】<__XZHNetworkLRUNode - 0x608000099140> _key = key_6:【thread: <NSThread: 0x608000266f00>{number = 4, name = (null)}】
```

可以看到，一部分node对象在`主线程`，一部分node对象在`子线程`完成释放废弃。
