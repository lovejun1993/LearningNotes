---
layout: post
title: "Ping Host Select Best NetworkConnection"
date: 2015-01-05 11:11:46 +0800
comments: true
categories: 
---

###获取当前手机的dns

```
#import <netinet/in.h>
#import <netdb.h>
#import <ifaddrs.h>
#import <arpa/inet.h>
#import <net/ethernet.h>
#import <net/if_dl.h>
#include <resolv.h>

#include <dns.h>
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
#include<sys/socket.h>
#include<unistd.h>
```

```
+ (NSString *)getDNSInfos
{
    res_state res = (res_state)malloc(sizeof(struct __res_state));
    __uint32_t dwDNSIP = 0;
    int result = res_ninit(res);
    if (result == 0) {
        dwDNSIP = res->nsaddr_list[0].sin_addr.s_addr;
    }
    NSString *dns = [NSString stringWithUTF8String:inet_ntoa(res->nsaddr_list[0].sin_addr)];
    free(res);
    return dns;
}
```

如果报如下错:

```
Undefined symbols for architecture arm64: "_res_9_ns_initparse", referenced from: ....
```

在 build-settings中的`other linker flags`添加`-lresolv`

***

###通过ping某个host，查看其相应速度，决定选择最快的host进行网络连接

***

> 通过ping一个hostName，获取其解析域名服务器dns.

/**
 方法: 获取host的当前dns
 
 参数:
 hostName,必选
 shortName,可选
 allDNS,必选;yes,所有dns;no,当前dns
 pingDNSBlock,必选
 */

```
- (void)pingWithHostName:(NSString *)hostName
               shortName:(NSString *)shortName
                  allDNS:(BOOL)allDNS
            pingDNSBlock:(VSPingDNSBlock)pingDNSBlock
{
    //1. 没有网络 or hostName没有，停止ping
    if ([VSDevice deviceNetworkType] == VSDeviceNetworkTypeNone || [hostName length]<=0) {
        if (pingDNSBlock) {
            pingDNSBlock(nil);
        }
        return;
    }
    
    //2. 保存数据
    self.desript = shortName;
    self.allDNS = allDNS;
    self.pingDNSBlock = pingDNSBlock;
    
    //3. 开始ping hostName
    [self pingWithHostName:hostName];
}
```

```
- (void)pingWithHostName:(NSString *)hostName{
    
    if (hostName && ![hostName isEqualToString:@""])
    {
    	//1. 记录当前开始ping的起始时间
        self.requestTime = [[NSDate date] timeIntervalSince1970];
        
        //2. 进行ping
        [self pingHost:hostName];
    }
}
```

```
- (void)pingHost:(NSString *)hostName {
    
    //将ping的操作放到全局并发队列，异步执行，可能很耗时
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        
        //保存得到的所有dns
        NSArray *dns = nil;
        
        //是否需要得到所有的dns
        if (!self.allDNS) {
            
            //获取当前dns
            dns = [self getDNS:hostName];
            
        } else{
            
            //获取所有的dns
            dns = [self getAllDNS:hostName];
        }
        
        //当前ping操作响应的时间
        NSTimeInterval responeTime =[[NSDate date] timeIntervalSince1970];
        
        //响应时间 - 起始时间 = ping操作执行时间
        NSTimeInterval time = responeTime - self.requestTime;
        
        //组装hostName、dns、短域名、ping操作执行时间
        //回传给调用者
        VSPingInfo *pingInfo = [[VSPingInfo alloc] init];
        pingInfo.hostName = hostName;
        pingInfo.dns  = dns;
        pingInfo.shortName = self.desript;
        pingInfo.time = time;
        dispatch_async(dispatch_get_main_queue(), ^{
            [self completePingDNSBlock:pingInfo];
        });
    });
    
}
```

获取所有的dns

```
- (NSArray *)getAllDNS:(NSString *)hostName
{

    NSMutableArray *dns = nil;
    Boolean result = '\0';
    CFHostRef hostRef;
    CFArrayRef addresses = NULL;
    
    @try {
        
        hostRef = CFHostCreateWithName(kCFAllocatorDefault, (__bridge CFStringRef)hostName);
        if (hostRef) {
            result = CFHostStartInfoResolution(hostRef, kCFHostAddresses, NULL); // pass an error instead of NULL here to find out why it failed
            if (result == TRUE) {
                addresses = CFHostGetAddressing(hostRef, &result);
            }
        }
        
        if (result == TRUE) {
            dns = [[NSMutableArray alloc] init];
            for(int i = 0; i < CFArrayGetCount(addresses); i++){
                struct sockaddr_in* remoteAddr;
                CFDataRef saData = (CFDataRef)CFArrayGetValueAtIndex(addresses, i);
                remoteAddr = (struct sockaddr_in*)CFDataGetBytePtr(saData);
                
                if(remoteAddr != NULL){
                    NSString *strDNS =[NSString stringWithCString:inet_ntoa(remoteAddr->sin_addr) encoding:NSASCIIStringEncoding];
                    [dns addObject:strDNS];
                }
            }
        }

    }
    @catch (NSException *exception) {
        
        dns = nil;
    }
    @finally {
        
    }
    
    return dns;
}
```

获取当前dns

```
- (NSArray *)getDNS:(NSString *)hostName
{

    NSMutableArray *dns = nil;
    
    const char* szname = [hostName UTF8String];
    struct hostent* phot ;
    phot = gethostbyname(szname);
    struct in_addr ip_addr;
    if (phot == NULL){
        return nil;
    }
    memcpy(&ip_addr,phot->h_addr_list[0],4);
    
    char ip[20] = {0};
    inet_ntop(AF_INET, &ip_addr, ip, sizeof(ip));
    
    NSString *ipAddr = [NSString stringWithUTF8String:ip];
    
    if (ipAddr && ![ipAddr isEqualToString:@""]) {
        
        dns = [[NSMutableArray alloc] init];
        [dns addObject:ipAddr];
    }

    return dns;
}
```

