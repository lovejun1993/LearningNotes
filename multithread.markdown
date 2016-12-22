
###示例1，创建一个`分离(detach)线程`，自己管理自己的声明周期

分离线程（detached），靠自己.

```
1. `一个分离的线程是不能被其他线程回收或杀死的.`
2. `它的存储器资源在线程自己终止时，由系统自动释放.`
```


```objc
void createPthread()
{
    //1. 线程属性（调度策略、优先级等）都在这里设置，如果为NULL则表示用默认属性
    pthread_attr_t attr;
    
    //2. 返回 创建线程id
    pthread_t posixThreadId;
    
    //3. 保存线程创建是否成功（0:成功，-1:失败）
    int returnVal;
    
    //4. 初始化线程属性
    returnVal = pthread_attr_init(&attr);
    
    //5. 如果线程创建失败
    assert(!returnVal);
    
    //5. 设置要创建出的线程 是 `分离线程`
    returnVal = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    
    //6. 创建出一个线程，传入参数:线程id、线程属性、线程入口函数
    int threadError = pthread_create(&posixThreadId, &attr, &threadFunc, NULL);
    
    //7.【重要】 不再使用线程属性，将其销毁
    returnVal = pthread_attr_destroy(&attr);
    
    assert(!returnVal);
    
    if (threadError != 0) {
        //线程创建错误
    }
}

//线程的入口函数
void* threadFunc(void* params)
{
    NSLog(@"%@", [NSThread currentThread]);
    
    return NULL;
}

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    createPthread();
}

@end
```

###示例2，创建一个`可结合(joinable)线程`，由创建这个线程的所在线程管理生命周期

类型二、可结合线程（joinable），靠别人.

```
1. `这个线程是由其他线程创建出来的.`
2. `只能被创建他的线程等待终止.`
3. `在被其他线程回收之前，它的存储器资源（如栈）是不释放的.`
4. `所在线程会一直阻塞，等待joinable线程执行完毕.`
```

> join()方法，使其他线程等待当前线程终止.

```objc
#import "ViewController.h"
#import <pthread.h>

// 打印线程信息
//static void printids()
//{
//    pid_t       pid;
//    pthread_t   tid;
//    
//    pid = getpid();//线程id
//    tid = pthread_self();//线程
//    
//    printf("pid %u tid %u (0x%x)\n", (unsigned int)pid,
//           (unsigned int)tid, (unsigned int)tid);
//}

void *thr_fn1(void *arg)
{
    printf("thread 1 returning.\n");
    return((void *)1);
}

void *thr_fn2(void *arg)
{
    printf("thread 2 exiting.\n");
    return((void *)2);
}

@implementation ViewController

- (void)exam1 {
    
    pthread_t tid1,tid2;
    
    void *tret;
    
    // 创建两个默认线程（joinable）
    pthread_create(&tid1, NULL, thr_fn1, NULL);
    pthread_create(&tid2, NULL, thr_fn2, NULL);
    
    NSLog(@"%@", [NSThread currentThread]);
    
    //【重要】在当前主线程上，等待 tid1 执行结束
    pthread_join(tid1, &tret);
    printf("thread 1 exit code %d\n",(int)tret);
    
    NSLog(@"%@", [NSThread currentThread]);
    
    //【重要】在当前主线程上，等待 tid2 执行结束
    pthread_join(tid2,&tret);
    printf("thread 2 exit code %d\n",(int)tret);
    
    NSLog(@"%@", [NSThread currentThread]);
    
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self exam1];
    
}

@end
```

输出信息

```
thread 1 returning.
thread 2 exiting.
2016-06-29 18:06:51.678 demo[9887:1715916] <NSThread: 0x7f80e2405b20>{number = 1, name = main}
thread 1 exit code 1
2016-06-29 18:06:51.679 demo[9887:1715916] <NSThread: 0x7f80e2405b20>{number = 1, name = main}
thread 2 exit code 2
2016-06-29 18:06:51.679 demo[9887:1715916] <NSThread: 0x7f80e2405b20>{number = 1, name = main}
```

###示例3，可结合线程之间，使用`pthread_exit()`函数结束线程并回传值的例子

```objc
#import "ViewController.h"
#import <pthread.h>

void thread1(char s[])
{
    printf("This is a pthread1.\n");
    printf("%s\n",s);
    pthread_exit("Hello first!");  //【重要】结束线程，返回一个值。
}

void thread2(char s[])
{
    printf("This is a pthread2.\n");
    printf("%s\n",s);
    pthread_exit("Hello second!");//【重要】结束线程，返回一个值。
}

@implementation ViewController

- (void)exam2 {
    
    pthread_t id1,id2;
    void *a1,*a2;
    int ret1,ret2;
    
    char s1[] = "传递给thread1的参数";
    char s2[] = "传递给thread2的参数";
    
    ret1 = pthread_create(&id1, NULL, (void *) thread1, s1);
    ret2 = pthread_create(&id2, NULL, (void *) thread2, s2);
    if (ret1 != 0 || ret2 != 0) {
        // 创建线程失败
        return;
    }
    
    // 在当前主线等待 线程1 执行完毕
    pthread_join(id1,&a1);
    printf("接收到线程1 的返回值: %s\n",(char*)a1);
    
    // 在当前主线等待 线程2 执行完毕
    pthread_join(id2,&a2);
    printf("接收到线程2 的返回值: %s\n",(char*)a2);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
//    [self exam1];
    [self exam2];
}

@end
```

输出信息

```
This is a pthread1.
传递给thread1的参数
This is a pthread2.
传递给thread2的参数
接收到线程1 的返回值: Hello first!
接收到线程2 的返回值: Hello second!
```

###在Cocoa程序上面使用POSIX线程注意

- 在Cocoa框架上，使用POSIX线程，Cocoa框架默认是不会创建锁机制去确保线程安全的.

- 为了让 Cocoa 知道你的程序正在使用用多线程，让Cocoa框架确保线程安全，那么可以这么做`生成一个NSThread线程，可以立即退出，线程的主体入口点不需要做任何事情.`这样后，Cocoa框架就会保证多线程安全执行.

- 检测程序是否是多线程环境? `+[NSThread isMultiThreaded]`来查看

- 不要混合 `POSIX` 和 `Cocoa` 的锁. `用POSIX API创建出的锁，就必须使用POSIX API操作这个锁。如果是Cocoa API创建出的锁，就必须使用Coacoa API 操作这个锁.`

###读取和修改，线程的`堆栈`空间大小

- 任何声明为 `static/extern` 的变量，可以被`进程内所有的线程`读与写. 

- 每个线程可以通过`暴露自己的堆栈地址`的方法，让`其他线程`进行共享自己的堆栈.

- 如果当前线程要执行一个`大数据量`的任务，那么就有必要给创建出的线程分配一个比较大的内存块.

- 但是线程`栈空间`的最大值，在线程创建的时刻就已经被固定了。如果访问超过栈空间地址，就会出现访问未授权的内存区域的错误.

下面给出demo示例代码，获取系统默认线程的栈大小、修改线程栈的默认大小、然后创建修改属性后的线程.

```objc
#import <pthread.h>
#include <limits.h>	一定要导入这个头文件

void* threadPoint(void* ptr)
{
    
    //线程入口函数
    
    return NULL;
}

void createThread()
{	
	//1. 线程Id
    pthread_t thread_id;
    
    //2. 线程属性
    pthread_attr_t thread_attr;
    
    //3. 保存线程栈大小的变量
    size_t stack_size;
    
    //4. 保存每一次POSIX Api操作的结果值（0:成功，-1:失败）
    int status;
    
    //5. 初始化线程属性
    status = pthread_attr_init(&thread_attr);
    
    if (status != 0) {
        perror("创建线程属性失败!\n");
    }
    
    //6. 设置要创建出的线程是【分离线程】
    status = pthread_attr_setdetachstate (&thread_attr, PTHREAD_CREATE_DETACHED);
    
    if (status != 0) {
        perror("创建线程属性失败!\n");
    }
    
    //7. 获取当前线程默认设置栈大小
    status = pthread_attr_getstacksize(&thread_attr, &stack_size);
    
    if (status != 0) {
        perror("获取线程的栈空间大小失败!\n");
    }
    
    printf ("Default stack size is %zu; minimum is %u\n",
            stack_size, PTHREAD_STACK_MIN);

    //8. 手动修改系统默认分配的栈栈大小
    status = pthread_attr_setstacksize(&thread_attr, PTHREAD_STACK_MIN * 1024);
    
    if (status != 0) {
        perror("设置线程的栈空间大小失败!\n");
    }
    
    //9. 获取修改之后的线程栈大小
    status = pthread_attr_getstacksize(&thread_attr, &stack_size);
    
    if (status != 0) {
        perror("获取线程的栈空间大小失败!\n");
    }
    
    printf ("Default stack size is %zu; minimum is %u\n",
            stack_size, PTHREAD_STACK_MIN);
    
    //10. 按照之前的设置，创建线程
    status = pthread_create(&thread_id, &thread_attr, &threadPoint, NULL);
    
    if (status != 0) {
        perror("创建线程失败!\n");
    }
    
    
}
```

###每个pthread线程都有自己的临时存储值得一个字典对象threadDictionary

- `NSThread`实例也有一个`threadDictionary`可变字典属性

```
NSThread *t = [[NSThread alloc] init];
    
[t.threadDictionary setObject:@"value" forKey:@"key"];

```

- POSIX线程，通过如下三个c函数

```objc
pthread_key_create()	// 创建字典
pthread_setspecific()	// 设置值到字典
pthread_getspecific()	// 从字典获取值
```

```objc
#include "Demo3.h"
#include<stdio.h>
#include<pthread.h>
#include<string.h>


//声明一个key
pthread_key_t p_key;

void *thread_func(void *args)
{
    
    //不同线程的读取自己的私有存储器中的p_key对应的值
    pthread_setspecific(p_key,args);
    
    //输出
    int *tmp = (int*)pthread_getspecific(p_key);
    printf("%d is runing in %s\n",*tmp,__func__);
    
    //修改p_key对应的值
    *tmp = (*tmp)*100;
    
    //输出
    int *tmp2 = (int*)pthread_getspecific(p_key);
    printf("%d is runing in %s\n",*tmp2,__func__);
    
    return (void*)0;
}

int main()
{
    //两个线程Id
    pthread_t pa, pb;
    
    //各自的参数
    int a=1;
    int b=2;
    
    //创建线程的私有存储有一个p_key
    pthread_key_create(&p_key,NULL);
    
    //创建线程
    pthread_create(&pa, NULL,thread_func,&a);
    pthread_create(&pb, NULL,thread_func,&b);
    
    //等待线程执行结束
    pthread_join(pa, NULL);
    pthread_join(pb, NULL);
    
    return 0;
}
```


