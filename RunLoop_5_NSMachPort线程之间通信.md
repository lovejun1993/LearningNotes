## NSMachPort、完成不同线程之间的通信

### 苹果提供基于port端口事件的

NSPort是父类，有三个子类实现

```c
- NSPort
	- NSMachPort
	- NSMessagePort
	- NSSocketPort
```

NSPortDelegate

```c
@protocol NSPortDelegate <NSObject>
@optional

- (void)handlePortMessage:(NSPortMessage *)message;
	// This is the delegate method that subclasses should send
	// to their delegates, unless the subclass has something
	// more specific that it wants to try to send first
@end
```

NSMachPortDelegate

```c
@protocol NSMachPortDelegate <NSPortDelegate>
@optional

// Delegates are sent this if they respond, otherwise they
// are sent handlePortMessage:; argument is the raw Mach message
- (void)handleMachMessage:(void *)msg;

@end
```

NSMessagePort已经被屏蔽使用了。NSSocketPort好像没怎么看到使用过，暂时就懒得看了....

## 下面使用NSMachPort完成不同线程之间的通信

MachPort、MessagePort，在`iOS7`之后苹果不再面向开发者，因为其使用太过复杂、麻烦。

所以如下代码也只是伪代码，需要使用以前老本的Xcode才能编译通过。


- (1) ViewController中接收子线程发送的port消息后，向子线程发送一个port消息

```objc
@interface ViewController () <NSPortDelegate>
@end

@implementation ViewController {
    MyWorkTask *_task;
}

- (void)viewDidLoad {
    [super viewDidLoad];
	
	//1. 创建主线程自己的port
    NSPort *myPort = [NSMachPort port];
    
    //2. 指定接收到port消息时的回调
    myPort.delegate = self;
    
    //3. 把port事件源注册当当前线程的runloop，进行监听
    [[NSRunLoop currentRunLoop] addPort:myPort forMode:NSDefaultRunLoopMode];
    
    //4. 启动次线程,并传入主线程的port
    _task = [[MyWorkTask alloc] init];
    [_task startWithRemotePort:myPort];
}

#pragma mark - NSPortDelegate

- (void)handlePortMessage:(NSPortMessage *)message {
    
    NSLog(@"接到子线程传递的消息！");
    
    //1. 消息id
    NSUInteger msgId = [[message valueForKeyPath:@"msgid"] integerValue];
    
    //2. 当前主线程的port
    NSPort *localPort = [message valueForKeyPath:@"localPort"];
    
    //3. 发送消息的port
    NSPort *remotePort = [message valueForKeyPath:@"remotePort"];
    
    if (msgId == kOperation1)
    {
        //向子线的port发送消息
        [remotePort sendBeforeDate:[NSDate date]
                             msgid:kOperation2
                        components:nil
                              from:localPort
                          reserved:0];
        
    } else if (msgId == kOperation2){
        NSLog(@"操作2....\n");
    }

}

@end
```

- (2) 子线程runloop监听port消息，并向主线程发送port消息

```objc
NSUInteger kOperation1 = 100001;
NSUInteger kOperation2 = 100002;

@interface MyWorkTask : NSObject

- (void)startWithRemotePort:(NSPort *)remotePort;

@end
```

```objc
@interface MyWorkTask () <NSPortDelegate>
@end

@implementation MyWorkTask {
    NSPort      *_remotePort;
    NSPort      *_localPort;
    NSThread    *_curThread;
}

- (void)startWithRemotePort:(NSPort *)remotePort {
    
    //1. 保存主线程的port
    _remotePort = remotePort;
    
    //2. 创建当前内部子线程
    _curThread = [[NSThread alloc] initWithTarget:self
                                         selector:@selector(threadEntry)
                                           object:nil];
    
    //3. 开启线程
    [_curThread start];
}

- (void)threadEntry {
    
    @autoreleasepool {
        
        //1. 创建当前线程的port
        _localPort = [NSMachPort port];
        
        //2. 设置子线程名字
        [[NSThread currentThread] setName:@"MyWorkTaskThread"];

        //3. 设置port的代理回调对象，接收子线程发送过来的数据
        _localPort.delegate = self;
        
        //4. 获取当前`子线程`的runloop
        NSRunLoop *runloop = [NSRunLoop currentRunLoop];
        
        //5. 让当前线程的runloop监听发给这个localPort的事件源
        [runloop addPort:_localPort forMode:NSDefaultRunLoopMode];
        
        //6. 开启runloop
        [runloop run];
        
        //7. 向主线程port发送一条数据
        [self sendPortMessage];
    }
}

#pragma mark - 向传入的port所在的线程runloop发送消息

- (void)sendPortMessage {
    
    //发送消息到主线程，操作1
    [_remotePort sendBeforeDate:[NSDate date]
                          msgid:kOperation1
                     components:nil
                           from:_localPort
                       reserved:0];
    
    //发送消息到主线程，操作2
    [_remotePort sendBeforeDate:[NSDate date]
                          msgid:kOperation2
                     components:nil
                           from:_localPort
                       reserved:0];
}

#pragma mark - NSPortDelegate

- (void)handlePortMessage:(NSPortMessage *)message {
    NSLog(@"接收到主线程的消息...\n");
    
//    unsigned int msgid = [message msgid];
//    NSPort* distantPort = nil;
//
//    if (msgid == kCheckinMessage)
//    {
//        distantPort = [message sendPort];
//    }
//    else if(msgid == kExitMessage)
//    {
//        CFRunLoopStop((__bridge CFRunLoopRef)[NSRunLoop currentRunLoop]);
//    }
}

@end
```

小结下主要的步骤:

- (1) NSPort创建
- (2) 子线程的runloop创建，并给runloop注册一个NSPort事件端口
- (3) 向持有的NSPort对象发送消息
- (4) 指定线程的runloop接收到监听的NSPort事件之后，唤醒绑定的线程执行NSPortDelegate方法实现，处理接收到的NSPort消息

## 苹果推荐使用如下进行线程通信:

- (1) GCD DisPach Queue
- (2) NSOperation Queue
- (3) NSObject分类的各种的 performSelecter:onThread:…方法（不如使用上面两种）

## RunLoop 与 Thread 的关系:

### Thread

- (1) 就像苦力工，什么事情都干
- (2) 但是不知道什么时候该休息，什么时候该做事

### RunLoop

- (1) 就像领导，负责接收用户、老板的命令
- (2) 然后通知Thread苦力开始工作，工作完成时通知Thread睡觉