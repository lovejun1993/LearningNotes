---
layout: post
title: "DesignMode、Fade"
date: 2014-01-08 00:09:03 +0800
comments: true
categories: 
---

###外观模式、为内部子系统的复杂逻辑，提供一个简单的入口.

- 为 `复杂的子系统` 向外提供一个简单的接口
- 给每一层的子系统提供一个统一的入口

***

###模拟照相、灯光、传感器三种硬件互相配合完成一个复杂的功能.

***

###照相机

```objc
#import <Foundation/Foundation.h>

/**
 *  照相机
 */
@interface Camera : NSObject

- (void)turnON;

- (void)turnOff;

@end
```

```objc
#import "Camera.h"

@implementation Camera

- (void)turnON {
    
}

- (void)turnOff {
    
}

@end
```

###灯光

```objc
#import <Foundation/Foundation.h>

/**
 *  灯光
 */
@interface Light : NSObject

- (void)turnON;

- (void)turnOff;

@end
```

```objc
#import "Light.h"

@implementation Light

- (void)turnON {
    
}

- (void)turnOff {
    
}

@end
```

###传感器

```objc
#import <Foundation/Foundation.h>

/**
 *  传感器
 */
@interface Sensor : NSObject

- (void)activate;

- (void)deactivate;

@end
```

```objc
#import "Sensor.h"

@implementation Sensor

- (void)activate {
    
}

- (void)deactivate {
    
}

@end
```

###向外提供一个简单的入口，来操作照相机、灯光、传感器之间的复杂逻辑.

```objc
#import <Foundation/Foundation.h>

@interface Fade : NSObject

- (void)activate;

- (void)deactivate;

@end
```

```objc
#import "Fade.h"

#import "Camera.h"
#import "Light.h"
#import "Sensor.h"

@interface Fade ()

@property (nonatomic, strong) Camera *camera1;
@property (nonatomic, strong) Camera *camera2;
@property (nonatomic, strong) Camera *camera3;

@property (nonatomic, strong) Light * light1;
@property (nonatomic, strong) Light * light2;

@property (nonatomic, strong) Sensor *sensor1;
@property (nonatomic, strong) Sensor *sensor2;

@end

@implementation Fade

- (instancetype)init {
    self = [super init];
    if (self) {
        
        _camera1 = [Camera new];
        _camera2 = [Camera new];
        _camera3 = [Camera new];
        
        _light1 = [Light new];
        _light2 = [Light new];
        
        _sensor1 = [Sensor new];
        _sensor2 = [Sensor new];
    }
    return self;
}

- (void)activate {
    [_camera1 turnON];
    [_camera2 turnON];
    
    [_light1 turnON];
    [_light2 turnON];
    
    [_sensor1 activate];
    [_sensor2 activate];
}

- (void)deactivate {
    [_camera1 turnOff];
    [_camera2 turnOff];
    
    [_light1 turnOff];
    [_light2 turnOff];
    
    [_sensor1 deactivate];
    [_sensor2 deactivate];
}

@end
```