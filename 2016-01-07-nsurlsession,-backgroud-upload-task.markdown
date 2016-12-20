---
layout: post
title: "NSURLSession、Backgroud Upload Task"
date: 2016-01-07 16:01:43 +0800
comments: true
categories: 
---

###demo1、App在前台时，`不借助第三方框架`，只使用NSURLSession完成文件上传

> 最主要的是要自己拼接NSData二进制参数

```objc
//字符串转换成NSData
#define ConvertData(str) [str dataUsingEncoding:NSUTF8StringEncoding]
```

```objc
- (NSString *)boundary {
    return @"一个字符串标示符";
}

- (NSString *)beginStr {
    return [NSString stringWithFormat:@"--%@\r\n", [self boundary]];
}

- (NSString *)endStr {
    return [NSString stringWithFormat:@"--%@--\r\n", [self boundary]];
}

- (NSString *)line {
    return @"\r\n";
}
```

```objc
- (NSString *)contentDispositionWithName:(NSString *)name FileName:(NSString *)filename {
    if (name && filename) {
        return [NSString stringWithFormat:@"Content-Disposition: form-data; name=\"%@\"; filename=\"%@\"\r\n", name, filename];
    } else {
        return @"";
    }
}

- (NSString *)paramDispositionWithKey:(NSString *)key {
    if (key) {
        return [NSString stringWithFormat:@"Content-Disposition: form-data; name=\"%@\"\r\n", key];
    } else {
        return @"";
    }
}
```

```objc
- (NSData *)mimeTypeDataWithFileName:(NSString *)fileName MimeType:(NSString **)mimeType_p
{
    NSURL *fileurl = [[NSBundle mainBundle] URLForResource:fileName withExtension:nil];
    NSURLRequest *request = [NSURLRequest requestWithURL:fileurl];
    NSURLResponse *repsonse = nil;
    NSData *data = [NSURLConnection sendSynchronousRequest:request returningResponse:&repsonse error:nil];
    *mimeType_p = repsonse.MIMEType;
    return data;
}
```

```objc
//【最重要】拼接上传需要的数据参数NSData

- (NSData *)httpBodyDataWithName:(NSString *)name
                        Filename:(NSString *)filename
                       ParamDict:(NSDictionary *)param
{
    //1. 创建一个最重要上传的bodyData
    __block NSMutableData *bodyData = [NSMutableData data];
    
    //***************2. 设置上传文件的参数***************
    //2.1 参数开始的标志
    [bodyData appendData:ConvertData([self beginStr])];
    
    //2.2 上传文件描述，注意其中的name参数和filename参数写法
    // name : 指定参数名(必须跟服务器端保持一致)
    // filename : 文件名
    NSString *contentDispose = [self contentDispositionWithName:name FileName:filename];
    [bodyData appendData:ConvertData(contentDispose)];
    
    //2.3 设置mimeType
    NSString *mimeType = nil;
    NSData *mimeData = [self mimeTypeDataWithFileName:filename MimeType:&mimeType];
    [NSString stringWithFormat:@"Content-Type: %@\r\n", mimeType];
    [bodyData appendData:ConvertData(mimeType)];
    
    //2.4 分界线
    [bodyData appendData:ConvertData([self line])];
    
    //2.5
    [bodyData appendData:mimeData];
    
    //2.6 分界线
    [bodyData appendData:ConvertData([self line])];

    //***************3. 设置上传需要的普通参数 ***************
    __weak __typeof(self)weakSelf = self;
    [param enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
        // 参数开始的标志
        [bodyData appendData:ConvertData([self beginStr])];
        
        //添加key
        NSString *disposition = [weakSelf paramDispositionWithKey:key];
        [bodyData appendData:ConvertData(disposition)];
        
        //分界线
        [bodyData appendData:ConvertData([self line])];
        
        //添加value（value必须是字符串）
        [bodyData appendData:ConvertData(obj)];
        
        //分界线
        [bodyData appendData:ConvertData([self line])];
    }];
    
    //***************参数结束***************
    
    [bodyData appendData:ConvertData([self endStr])];
    
    return bodyData;
}

```

如上参数拼接完成的最终上传请求数据格式如下所示:

![](http://i13.tietuku.com/ebc2a91ade9ac9e2.png)

```objc
//使用NSURLSession上传文件

- (void)uploadFile:(NSString *)filename Param:(NSDictionary *)param {
    
    //1. 构造request，并设置为POST请求
    NSURL *url = [NSURL URLWithString:@"http://192.168.1.200:8080/YYServer/upload"];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    
    //2. 设置request的body
    NSData *body = [self httpBodyDataWithName:@"这个name要和服务器协商必须一致"
                                     Filename:filename
                                    ParamDict:param];
//    @{
//      @"username" : @"999",
//      @"type" : @"XML"
//      }
    
    request.HTTPBody = body;
    
    //3. 设置请求头
    
    //3.1 请求体的长度
    [request setValue:[NSString stringWithFormat:@"%zd", body.length] forHTTPHeaderField:@"Content-Length"];
    
    //3.2 声明这个POST请求是个文件上传
    NSString *value = [NSString stringWithFormat:@"multipart/form-data; boundary=%@", [self boundary]];
    [request setValue:value forHTTPHeaderField:@"Content-Type"];
    
    //4. 使用NSURLSession Api发起上传操作
    NSURLSession *session=[NSURLSession sharedSession];
    
    NSURLSessionUploadTask *uploadTask=[session uploadTaskWithRequest:request fromData:body completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (!error) {
            NSString *dataStr=[[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding];
            NSLog(@"%@",dataStr);
        }else{
            NSLog(@"error is :%@",error.localizedDescription);
        }
    }];
    
    [uploadTask resume];
}
```

####所以，上传其实就是几个关键点:

- `拼接要上传的参数`与`上传文件数据`，最终得到一个`NSData数据`，也就是最终要上传的数据，作为http body.

- `name`参数一定要和服务器协商好.

- 设置额外的参数到请求头
	- Content－length
	- Content－Type
	- 用户名与密码
	- 等等额外参数

***

###demo2、使用AFNetworking这样类似类库，会帮我完成二进制参数拼接的过程

- 那么看一下使用`AFNetworking`完成上传文件

```objc
//1. 使用AFHTTPRequestSerializer，快速构造要上传的NSMutableURLRequest
NSMutableURLRequest *request = [[AFHTTPRequestSerializer serializer] multipartFormRequestWithMethod:@"POST"
                                                                                          URLString:@"http://example.com/upload"
                                                                                         parameters:nil
                                                                          constructingBodyWithBlock:^(id<AFMultipartFormData> formData)
{
    
    //向Block回传的NSMuatbleData实例，append我们自己要上传文件数据
    //【重点】name: 一定要和服务器协商，name值一定要一致，才能完成上传
    //fileName: 磁盘上要上传的文件名
    //mimeType: 搜索mimeTypeDataWithFileName:方法，获取文件的MimeType
    [formData appendPartWithFileURL:[NSURL fileURLWithPath:@"file://path/to/image.jpg"]
                               name:@"file"
                           fileName:@"filename.jpg"
                           mimeType:@"image/jpeg"
                              error:nil];
} error:nil];
    
//2. 创建SessionManager
AFURLSessionManager *manager = [[AFURLSessionManager alloc] initWithSessionConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration]];
    
//3. 记录进度的变量
NSProgress *progress = nil;
    
//4. 创建Task
NSURLSessionUploadTask *uploadTask = [manager uploadTaskWithStreamedRequest:request progress:&progress completionHandler:^(NSURLResponse *response, id responseObject, NSError *error) {
    if (error) {
        NSLog(@"Error: %@", error);
    } else {
        NSLog(@"%@ %@", response, responseObject);
    }
}];
    
//5. 执行Task
[uploadTask resume];
```

***

###NSURLSession三种上传

- 使用文件NSData数据

```objc
NSURLSessionUploadTask *task = self.backgroundSession uploadTaskWithRequest:<#(nonnull NSURLRequest *)#> fromData:<#(nonnull NSData *)#>
```

- 使用文件路径

```objc
NSURLSessionUploadTask *task = [self.backgroundSession uploadTaskWithRequest:<#(nonnull NSURLRequest *)#> fromFile:<#(nonnull NSURL *)#>]
```

- 使用数据流

```objc
NSURLSessionUploadTask *task = [self.backgroundSession uploadTaskWithStreamedRequest:<#(nonnull NSURLRequest *)#>]
```

***

###前面两种上传回调 NSURLSessionTaskDelegate 定义函数.

```objc
/* Sent periodically to notify the delegate of upload progress.  This
 * information is also available as properties of the task.
 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                                didSendBodyData:(int64_t)bytesSent
                                 totalBytesSent:(int64_t)totalBytesSent
                       totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend;

/* Sent as the last message related to a specific task.  Error may be
 * nil, which implies that no error occurred and this task is complete. 
 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                           didCompleteWithError:(nullable NSError *)error;
```

***


##使用Stream上传并设置时，必须实现两个协议

- 必须实现 协议一、NSURLSessionTaskDelegate如下方法，让用户`提供要上传文件的数据源（输入流）`

```objc
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                              needNewBodyStream:(void (^)(NSInputStream * __nullable bodyStream))completionHandler
{
	//1. 得到一个文件的输入流
	NSInputStream *input = ...;
	
	//2. 回调告诉NSURLSession这个输入流
	completionHandler(input);
}
```

- 可选实现 协议二、NSURLSessionStreamDelegate 

```objc
/* Indiciates that the read side of a connection has been closed.  Any
 * outstanding reads complete, but future reads will immediately fail.
 * This may be sent even when no reads are in progress. However, when
 * this delegate message is received, there may still be bytes
 * available.  You only know that no more bytes are available when you
 * are able to read until EOF. */
- (void)URLSession:(NSURLSession *)session readClosedForStreamTask:(NSURLSessionStreamTask *)streamTask;

/* Indiciates that the write side of a connection has been closed.
 * Any outstanding writes complete, but future writes will immediately
 * fail.
 */
- (void)URLSession:(NSURLSession *)session writeClosedForStreamTask:(NSURLSessionStreamTask *)streamTask;

/* A notification that the system has determined that a better route
 * to the host has been detected (eg, a wi-fi interface becoming
 * available.)  This is a hint to the delegate that it may be
 * desirable to create a new task for subsequent work.  Note that
 * there is no guarantee that the future task will be able to connect
 * to the host, so callers should should be prepared for failure of
 * reads and writes over any new interface. */
- (void)URLSession:(NSURLSession *)session betterRouteDiscoveredForStreamTask:(NSURLSessionStreamTask *)streamTask;

/* The given task has been completed, and unopened NSInputStream and
 * NSOutputStream objects are created from the underlying network
 * connection.  This will only be invoked after all enqueued IO has
 * completed (including any necessary handshakes.)  The streamTask
 * will not receive any further delegate messages.
 */
- (void)URLSession:(NSURLSession *)session streamTask:(NSURLSessionStreamTask *)streamTask
                                 didBecomeInputStream:(NSInputStream *)inputStream
                                         outputStream:(NSOutputStream *)outputStream;
```

可以看到 既可以输入，也可以输出.


***

###后台上传类似后台下载，最后实现的几个函数步骤差不多.