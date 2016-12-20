##App进程启动后，查看主线程的RunLoop对象

```objc
@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
    NSRunLoop *runloop = [NSRunLoop currentRunLoop];
//断点.... 
    return YES;
}

@end
```

断点`po runloop`得到如下

```
(lldb) po runloop
<CFRunLoop 0x7ff5c1605ff0 [0x1109ea7b0]>{wakeup port = 0x1403, stopped = false, ignoreWakeUps = false, 
current mode = UIInitializationRunLoopMode,
common modes = <CFBasicHash 0x7ff5c16070f0 [0x1109ea7b0]>{type = mutable set, count = 2,
entries =>
	0 : <CFString 0x111d8e270 [0x1109ea7b0]>{contents = "UITrackingRunLoopMode"}
	2 : <CFString 0x110a0ab60 [0x1109ea7b0]>{contents = "kCFRunLoopDefaultMode"}
}
,
common mode items = <CFBasicHash 0x7ff5c16072e0 [0x1109ea7b0]>{type = mutable set, count = 16,
entries =>
	0 : <CFRunLoopSource 0x7ff5c17004a0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x114ecb779)}}
	1 : <CFRunLoopSource 0x7ff5c1500e40 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x7ff5c1607dd0 [0x1109ea7b0]>{valid = Yes, port = 1e03, source = 0x7ff5c1500e40, callout = __IOHIDEventSystemClientAvailabilityCallback (0x112e30a2f), context = <CFMachPort context 0x7ff5c1407f80>}}
	2 : <CFRunLoopObserver 0x7ff5c1606810 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x11542e320), context = <CFRunLoopObserver context 0x0>}
	8 : <CFRunLoopObserver 0x7ff5c16179f0 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x111013c4e), context = <CFArray 0x7ff5c16177e0 [0x1109ea7b0]>{type = mutable-small, count = 0, values = ()}}
	9 : <CFRunLoopObserver 0x7ff5c1617950 [0x1109ea7b0]>{valid = Yes, activities = 0x1, repeats = Yes, order = -2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x111013c4e), context = <CFArray 0x7ff5c16177e0 [0x1109ea7b0]>{type = mutable-small, count = 0, values = ()}}
	10 : <CFRunLoopObserver 0x7ff5c16175d0 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 1999000, callout = _beforeCACommitHandler (0x111046a54), context = <CFRunLoopObserver context 0x7ff5c1408cd0>}
	11 : <CFRunLoopObserver 0x7ff5c1617810 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2001000, callout = _afterCACommitHandler (0x111046a99), context = <CFRunLoopObserver context 0x7ff5c1408cd0>}
	12 : <CFRunLoopSource 0x7ff5c1605ef0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x7ff5c1500cb0 [0x1109ea7b0]>{valid = Yes, port = 1007, source = 0x7ff5c1605ef0, callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_ (0x11543e8a9), context = <CFMachPort context 0x0>}}
	13 : <CFRunLoopSource 0x7ff5c1408900 [0x1109ea7b0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 1, info = 0x2203, callout = PurpleEventCallback (0x114ecdcb0)}}
	14 : <CFRunLoopSource 0x7ff5c1617b50 [0x1109ea7b0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x7ff5c1408cd0, callout = _UIApplicationHandleEventQueue (0x1110142db)}}
	15 : <CFRunLoopSource 0x7ff5c1500f20 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 1, context = <CFMachPort 0x7ff5c1607960 [0x1109ea7b0]>{valid = Yes, port = 1b03, source = 0x7ff5c1500f20, callout = __IOMIGMachPortPortCallback (0x112e38fce), context = <CFMachPort context 0x7ff5c16077d0>}}
	16 : <CFRunLoopSource 0x7ff5c1505df0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 17675, subsystem = 0x111d73920, context = 0x7ff5c160c9e0}}
	19 : <CFRunLoopObserver 0x7ff5c1711f20 [0x1109ea7b0]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x1114f36ab), context = <CFRunLoopObserver context 0x0>}
	20 : <CFRunLoopSource 0x7ff5c16176a0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 14855, subsystem = 0x111d60fe0, context = 0x0}}
	21 : <CFRunLoopSource 0x7ff5c1700350 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x7ff5c1607ad0 [0x1109ea7b0]>{valid = Yes, port = 1d03, source = 0x7ff5c1700350, callout = __IOHIDEventSystemClientQueueCallback (0x112e3087e), context = <CFMachPort context 0x7ff5c1407f80>}}
	22 : <CFRunLoopSource 0x7ff5c161a460 [0x1109ea7b0]>{signalled = Yes, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x7ff5c161a1e0, callout = FBSSerialQueueRunLoopSourceHandler (0x11448af60)}}
}
,
modes = <CFBasicHash 0x7ff5c1607130 [0x1109ea7b0]>{type = mutable set, count = 5,
entries =>
	2 : <CFRunLoopMode 0x7ff5c1403fb0 [0x1109ea7b0]>{name = UITrackingRunLoopMode, port set = 0x1707, timer port = 0x1803, 
	sources0 = <CFBasicHash 0x7ff5c1403a80 [0x1109ea7b0]>{type = mutable set, count = 3,
entries =>
	0 : <CFRunLoopSource 0x7ff5c17004a0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x114ecb779)}}
	1 : <CFRunLoopSource 0x7ff5c161a460 [0x1109ea7b0]>{signalled = Yes, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x7ff5c161a1e0, callout = FBSSerialQueueRunLoopSourceHandler (0x11448af60)}}
	2 : <CFRunLoopSource 0x7ff5c1617b50 [0x1109ea7b0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x7ff5c1408cd0, callout = _UIApplicationHandleEventQueue (0x1110142db)}}
}
,
	sources1 = <CFBasicHash 0x7ff5c1403ac0 [0x1109ea7b0]>{type = mutable set, count = 7,
entries =>
	0 : <CFRunLoopSource 0x7ff5c1500e40 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x7ff5c1607dd0 [0x1109ea7b0]>{valid = Yes, port = 1e03, source = 0x7ff5c1500e40, callout = __IOHIDEventSystemClientAvailabilityCallback (0x112e30a2f), context = <CFMachPort context 0x7ff5c1407f80>}}
	4 : <CFRunLoopSource 0x7ff5c1700350 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x7ff5c1607ad0 [0x1109ea7b0]>{valid = Yes, port = 1d03, source = 0x7ff5c1700350, callout = __IOHIDEventSystemClientQueueCallback (0x112e3087e), context = <CFMachPort context 0x7ff5c1407f80>}}
	8 : <CFRunLoopSource 0x7ff5c1605ef0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x7ff5c1500cb0 [0x1109ea7b0]>{valid = Yes, port = 1007, source = 0x7ff5c1605ef0, callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_ (0x11543e8a9), context = <CFMachPort context 0x0>}}
	9 : <CFRunLoopSource 0x7ff5c16176a0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 14855, subsystem = 0x111d60fe0, context = 0x0}}
	10 : <CFRunLoopSource 0x7ff5c1408900 [0x1109ea7b0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 1, info = 0x2203, callout = PurpleEventCallback (0x114ecdcb0)}}
	11 : <CFRunLoopSource 0x7ff5c1505df0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 17675, subsystem = 0x111d73920, context = 0x7ff5c160c9e0}}
	12 : <CFRunLoopSource 0x7ff5c1500f20 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 1, context = <CFMachPort 0x7ff5c1607960 [0x1109ea7b0]>{valid = Yes, port = 1b03, source = 0x7ff5c1500f20, callout = __IOMIGMachPortPortCallback (0x112e38fce), context = <CFMachPort context 0x7ff5c16077d0>}}
}
,
	observers = <CFArray 0x7ff5c1617670 [0x1109ea7b0]>{type = mutable-small, count = 6, values = (
	0 : <CFRunLoopObserver 0x7ff5c1617950 [0x1109ea7b0]>{valid = Yes, activities = 0x1, repeats = Yes, order = -2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x111013c4e), context = <CFArray 0x7ff5c16177e0 [0x1109ea7b0]>{type = mutable-small, count = 0, values = ()}}
	1 : <CFRunLoopObserver 0x7ff5c1711f20 [0x1109ea7b0]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x1114f36ab), context = <CFRunLoopObserver context 0x0>}
	2 : <CFRunLoopObserver 0x7ff5c16175d0 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 1999000, callout = _beforeCACommitHandler (0x111046a54), context = <CFRunLoopObserver context 0x7ff5c1408cd0>}
	3 : <CFRunLoopObserver 0x7ff5c1606810 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x11542e320), context = <CFRunLoopObserver context 0x0>}
	4 : <CFRunLoopObserver 0x7ff5c1617810 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2001000, callout = _afterCACommitHandler (0x111046a99), context = <CFRunLoopObserver context 0x7ff5c1408cd0>}
	5 : <CFRunLoopObserver 0x7ff5c16179f0 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x111013c4e), context = <CFArray 0x7ff5c16177e0 [0x1109ea7b0]>{type = mutable-small, count = 0, values = ()}}
)},
	timers = (null),
	currently 501932102 (8175174882013) / soft deadline in: 1.84467359e+10 sec (@ -1) / hard deadline in: 1.84467359e+10 sec (@ -1)
},

	3 : <CFRunLoopMode 0x7ff5c14086c0 [0x1109ea7b0]>{name = GSEventReceiveRunLoopMode, port set = 0x2003, timer port = 0x2103, 
	sources0 = <CFBasicHash 0x7ff5c1408600 [0x1109ea7b0]>{type = mutable set, count = 1,
entries =>
	0 : <CFRunLoopSource 0x7ff5c17004a0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x114ecb779)}}
}
,
	sources1 = <CFBasicHash 0x7ff5c1408770 [0x1109ea7b0]>{type = mutable set, count = 1,
entries =>
	1 : <CFRunLoopSource 0x7ff5c1408a20 [0x1109ea7b0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 1, info = 0x2203, callout = PurpleEventCallback (0x114ecdcb0)}}
}
,
	observers = (null),
	timers = (null),
	currently 501932102 (8175176340712) / soft deadline in: 1.84467359e+10 sec (@ -1) / hard deadline in: 1.84467359e+10 sec (@ -1)
},

	4 : <CFRunLoopMode 0x7ff5c1607170 [0x1109ea7b0]>{name = kCFRunLoopDefaultMode, port set = 0x1503, timer port = 0x1603, 
	sources0 = <CFBasicHash 0x7ff5c1607340 [0x1109ea7b0]>{type = mutable set, count = 3,
entries =>
	0 : <CFRunLoopSource 0x7ff5c17004a0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x0, callout = PurpleEventSignalCallback (0x114ecb779)}}
	1 : <CFRunLoopSource 0x7ff5c161a460 [0x1109ea7b0]>{signalled = Yes, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x7ff5c161a1e0, callout = FBSSerialQueueRunLoopSourceHandler (0x11448af60)}}
	2 : <CFRunLoopSource 0x7ff5c1617b50 [0x1109ea7b0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 0, info = 0x7ff5c1408cd0, callout = _UIApplicationHandleEventQueue (0x1110142db)}}
}
,
	sources1 = <CFBasicHash 0x7ff5c1607380 [0x1109ea7b0]>{type = mutable set, count = 7,
entries =>
	0 : <CFRunLoopSource 0x7ff5c1500e40 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x7ff5c1607dd0 [0x1109ea7b0]>{valid = Yes, port = 1e03, source = 0x7ff5c1500e40, callout = __IOHIDEventSystemClientAvailabilityCallback (0x112e30a2f), context = <CFMachPort context 0x7ff5c1407f80>}}
	4 : <CFRunLoopSource 0x7ff5c1700350 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x7ff5c1607ad0 [0x1109ea7b0]>{valid = Yes, port = 1d03, source = 0x7ff5c1700350, callout = __IOHIDEventSystemClientQueueCallback (0x112e3087e), context = <CFMachPort context 0x7ff5c1407f80>}}
	8 : <CFRunLoopSource 0x7ff5c1605ef0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFMachPort 0x7ff5c1500cb0 [0x1109ea7b0]>{valid = Yes, port = 1007, source = 0x7ff5c1605ef0, callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_ (0x11543e8a9), context = <CFMachPort context 0x0>}}
	9 : <CFRunLoopSource 0x7ff5c16176a0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 14855, subsystem = 0x111d60fe0, context = 0x0}}
	10 : <CFRunLoopSource 0x7ff5c1408900 [0x1109ea7b0]>{signalled = No, valid = Yes, order = -1, context = <CFRunLoopSource context>{version = 1, info = 0x2203, callout = PurpleEventCallback (0x114ecdcb0)}}
	11 : <CFRunLoopSource 0x7ff5c1505df0 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 0, context = <CFRunLoopSource MIG Server> {port = 17675, subsystem = 0x111d73920, context = 0x7ff5c160c9e0}}
	12 : <CFRunLoopSource 0x7ff5c1500f20 [0x1109ea7b0]>{signalled = No, valid = Yes, order = 1, context = <CFMachPort 0x7ff5c1607960 [0x1109ea7b0]>{valid = Yes, port = 1b03, source = 0x7ff5c1500f20, callout = __IOMIGMachPortPortCallback (0x112e38fce), context = <CFMachPort context 0x7ff5c16077d0>}}
}
,
	observers = <CFArray 0x7ff5c16177b0 [0x1109ea7b0]>{type = mutable-small, count = 6, values = (
	0 : <CFRunLoopObserver 0x7ff5c1617950 [0x1109ea7b0]>{valid = Yes, activities = 0x1, repeats = Yes, order = -2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x111013c4e), context = <CFArray 0x7ff5c16177e0 [0x1109ea7b0]>{type = mutable-small, count = 0, values = ()}}
	1 : <CFRunLoopObserver 0x7ff5c1711f20 [0x1109ea7b0]>{valid = Yes, activities = 0x20, repeats = Yes, order = 0, callout = _UIGestureRecognizerUpdateObserver (0x1114f36ab), context = <CFRunLoopObserver context 0x0>}
	2 : <CFRunLoopObserver 0x7ff5c16175d0 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 1999000, callout = _beforeCACommitHandler (0x111046a54), context = <CFRunLoopObserver context 0x7ff5c1408cd0>}
	3 : <CFRunLoopObserver 0x7ff5c1606810 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x11542e320), context = <CFRunLoopObserver context 0x0>}
	4 : <CFRunLoopObserver 0x7ff5c1617810 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2001000, callout = _afterCACommitHandler (0x111046a99), context = <CFRunLoopObserver context 0x7ff5c1408cd0>}
	5 : <CFRunLoopObserver 0x7ff5c16179f0 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler (0x111013c4e), context = <CFArray 0x7ff5c16177e0 [0x1109ea7b0]>{type = mutable-small, count = 0, values = ()}}
)},
	timers = <CFArray 0x7ff5c1504c50 [0x1109ea7b0]>{type = mutable-small, count = 1, values = (
	0 : <CFRunLoopTimer 0x7ff5c1504b50 [0x1109ea7b0]>{valid = Yes, firing = No, interval = 0, tolerance = 0, next fire date = 501932074 (-27.326308 @ 8147852878634), callout = (Delayed Perform) UIApplication _accessibilitySetUpQuickSpeak (0x10fe279f7 / 0x11143a9a6) (/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/UIKit.framework/UIKit), context = <CFRunLoopTimer context 0x7ff5c1504b10>}
)},
	currently 501932102 (8175176403374) / soft deadline in: 1.8446744e+10 sec (@ 8147852878634) / hard deadline in: 1.8446744e+10 sec (@ 8147852878634)
},

	5 : <CFRunLoopMode 0x7ff5c161a650 [0x1109ea7b0]>{name = UIInitializationRunLoopMode, port set = 0x291b, timer port = 0x2d13, 
	sources0 = <CFBasicHash 0x7ff5c161a700 [0x1109ea7b0]>{type = mutable set, count = 1,
entries =>
	0 : <CFRunLoopSource 0x7ff5c161a460 [0x1109ea7b0]>{signalled = Yes, valid = Yes, order = 0, context = <CFRunLoopSource context>{version = 0, info = 0x7ff5c161a1e0, callout = FBSSerialQueueRunLoopSourceHandler (0x11448af60)}}
}
,
	sources1 = <CFBasicHash 0x7ff5c161a740 [0x1109ea7b0]>{type = mutable set, count = 0,
entries =>
}
,
	observers = <CFArray 0x7ff5c161f2a0 [0x1109ea7b0]>{type = mutable-small, count = 1, values = (
	0 : <CFRunLoopObserver 0x7ff5c1606810 [0x1109ea7b0]>{valid = Yes, activities = 0xa0, repeats = Yes, order = 2000000, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv (0x11542e320), context = <CFRunLoopObserver context 0x0>}
)},
	timers = (null),
	currently 501932102 (8175178141033) / soft deadline in: 1.84467359e+10 sec (@ -1) / hard deadline in: 1.84467359e+10 sec (@ -1)
},

	6 : <CFRunLoopMode 0x7ff5c1707c00 [0x1109ea7b0]>{name = kCFRunLoopCommonModes, port set = 0x3c03, timer port = 0x3d03, 
	sources0 = (null),
	sources1 = (null),
	observers = (null),
	timers = (null),
	currently 501932102 (8175178241519) / soft deadline in: 1.84467359e+10 sec (@ -1) / hard deadline in: 1.84467359e+10 sec (@ -1)
},

}
}


(lldb) 
```

其大致结构如下

```c
CFRunLoop {

    //1. 当前 runloop mode
    current mode = UIInitializationRunLoopMode,//私有的runloop mode

    //2. commom runloop modes 默认包含的两种mode
    common modes = [
        UITrackingRunLoopMode,
        kCFRunLoopDefaultMode,
    ],

    //3. 所有的 runloop mode下的 source0/source1/timers/observers
    common mode items = {

	    //3.1 所有的 source0 (manual) 事件
	    CFRunLoopSource {order =-1, {callout = _UIApplicationHandleEventQueue}},
	    CFRunLoopSource {order =-1, {callout = PurpleEventSignalCallback }},
	    CFRunLoopSource {order = 0, {callout = FBSSerialQueueRunLoopSourceHandler}},
	
	    //3.2 所有的 source1 (mach port) 事件
	    CFRunLoopSource {order = 0,  {port = 17923}},
	    CFRunLoopSource {order = 0,  {port = 12039}},
	    CFRunLoopSource {order = 0,  {port = 16647}},
	    CFRunLoopSource {order =-1, { callout = PurpleEventCallback}},
	    CFRunLoopSource {order = 0, {port = 2407, callout = _ZL20notify_port_callbackP12__CFMachPortPvlS1_}},
	    CFRunLoopSource {order = 0, {port = 1c03, callout = __IOHIDEventSystemClientAvailabilityCallback}},
	    CFRunLoopSource {order = 0, {port = 1b03, callout = __IOHIDEventSystemClientQueueCallback}},
	    CFRunLoopSource {order = 1, {port = 1903, callout = __IOMIGMachPortPortCallback}},
	
	    //3.3 所有的 runloop Ovserver
	    CFRunLoopObserver {order = -2147483647, activities = 0x1, callout = _wrapRunLoopWithAutoreleasePoolHandler}// Entry
	    CFRunLoopObserver {order = 0, activities = 0x20, callout = _UIGestureRecognizerUpdateObserver}// BeforeWaiting
	    CFRunLoopObserver {order = 1999000, activities = 0xa0, callout = _afterCACommitHandler}// BeforeWaiting | Exit
	    CFRunLoopObserver {order = 2000000, activities = 0xa0, callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv}// BeforeWaiting | Exit
	    CFRunLoopObserver {order = 2147483647, activities = 0xa0, callout = _wrapRunLoopWithAutoreleasePoolHandler}// BeforeWaiting | Exit
	
	    //3.4 所有的 Timer 事件
	    CFRunLoopTimer {firing = No, interval = 3.1536e+09, tolerance = 0, next fire date = 453098071 (-4421.76019 @ 96223387169499), callout = _ZN2CAL14timer_callbackEP16__CFRunLoopTimerPv (QuartzCore.framework)}
	 },

    //4. 所有的运行模式modes
    modes ＝ {

        // 4.1 UITrackingRunLoopMode
        CFRunLoopMode  {
            name = UITrackingRunLoopMode,
            sources0 =  [/* same as 'common mode items' */],
            sources1 =  [/* same as 'common mode items' */],
            observers = [/* same as 'common mode items' */],
            timers =    [/* same as 'common mode items' */],
        },


        // 4.2 GSEventReceiveRunLoopMode
        CFRunLoopMode  {
            name = GSEventReceiveRunLoopMode,
            sources0 =  [/* same as 'common mode items' */],
            sources1 =  [/* same as 'common mode items' */],
            observers = [/* same as 'common mode items' */],
            timers =    [/* same as 'common mode items' */],
        },

        // 4.3 kCFRunLoopDefaultMode
        CFRunLoopMode  {
            name = kCFRunLoopDefaultMode,
            sources0 = [
                    CFRunLoopSource {order = 0, {callout = FBSSerialQueueRunLoopSourceHandler}},
                ],
            sources1 = (null),
            observers = [
                CFRunLoopObserver {activities = 0xa0, order = 2000000,callout = _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv}},
                ],
            timers = (null),
        },

        // 4.4 UIInitializationRunLoopMode
        CFRunLoopMode  {
            name = UIInitializationRunLoopMode,
            sources0 = [
                CFRunLoopSource {order = -1, {callout = PurpleEventSignalCallback}}
            ],
            sources1 = [
                CFRunLoopSource {order = -1, callout = PurpleEventCallback}}
            ],
            observers = (null),
            timers = (null),
        },

        //4.5 kCFRunLoopCommonModes
        CFRunLoopMode  {
            name = kCFRunLoopCommonModes,
            sources0 = (null),
            sources1 = (null),
            observers = (null),
            timers = (null),
        },
    }
}
```

查看之前的runloop输出可以得到如下:

- (1) RunLoop刚创建的时候处于的模式

```
current mode = UIInitializationRunLoopMode
```

启动完成后就不再使用`UIInitializationRunLoopMode `，其他大部分时刻都处于`kCFRunLoopDefaultMode`

- (2) RunLoop刚创建的时候 commom modes 包含两种 mode

```
0 : <CFString 0x10684b210 [0x107f02a40]>{contents = "UITrackingRunLoopMode"}
2 : <CFString 0x107f235e0 [0x107f02a40]>{contents = "kCFRunLoopDefaultMode"}
```

- (3) `UITrackingRunLoopMode`对应的runloop mode下有如下几个重要的observer

```c
repeats = Yes, order = -2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler
```

```c
repeats = Yes, order = 2147483647, callout = _wrapRunLoopWithAutoreleasePoolHandler
```

看到上面两个observer都是回调执行`_wrapRunLoopWithAutoreleasePoolHandler()`函数实现，可以参看`_wrapRunLoopWithAutoreleasePoolHandler()`的函数实现，就可以发现这两个observer在`不同的runloop状态`时回调执行，那么理所当然做不同的事情:

第一个 Observer 监视的事件是 `Entry(即将进入Loop)`，其回调内会调用 `_objc_autoreleasePoolPush()` 用来`创建自动释放池`。其 order 是-2147483647，`优先级最高`，保证`创建`释放池发生在其他所有回调之前。


第二个 Observer 监视了两个事件: `BeforeWaiting(准备进入休眠)` 时调用`_objc_autoreleasePoolPop()` 和 `_objc_autoreleasePoolPush()` 释放旧的池并创建新池；`Exit(即将退出Loop)` 时调用 `_objc_autoreleasePoolPop()` 来释放自动释放池。这个 Observer 的 order 是 2147483647，`优先级最低`，保证其`释放`池子发生在其他所有回调之后。

- (4) `GSEventReceiveRunLoopMode`用于接收事件的模式，这个mode是内部使用的

- (5) `kCFRunLoopCommonModes`默认情况下，里面不存在任何的事件

```
sources0 = (null),
sources1 = (null),
observers = (null),
timers = (null),
```

- (6) 还可以看到其他的runloop observer的回调callout函数:

```
- (1) 用于UI事件响应 `_UIApplicationHandleEventQueue`
- (2) 用于手势识别 `_UIGestureRecognizerUpdateObserver`
- (3) 用于界面更新 `_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv`，这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面
```

##kCFRunLoopCommonModes并不是一个RunLoopMode，其实是一个类似NSSet的容器对象

### commom modes 可以看做是一个set集合，我们可以将其他的RunLoopMode实例 添加到 这个set集合中

比如我们使用了一个NSTimer事件加入到runloop去调度执行，基本上是如下这两种写法:

```objc
NSTimer *timer = .....;
    
//1.
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
    
//2. 
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

后面会说的RunLoop有几种运行模式，处于某一种模式时，就只会去处理`当前这个模式`下的 sources/observers/timers事件。

###最简单的一个问题就是，NSTimer在发生UI事件的时候，可能会暂时停止执行，那么调度NSTimer的代码，就是使用的上面的(1)中的代码，将NSTimer添加到RunLoop。

```objc
//1. 
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
```

这样的Timer就只会在RunLoop处于`NSDefaultRunLoopMode`这个模式下时，才会去处理这个Timer的事件。

```objc
//2.
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

这样的Timer就会在common modes这个set集合中所有的mode模式下时，都回去处理这个Timer时间。

###默认的common modes这个set集合中，默认包含`UITrackingRunLoopMode`和`kCFRunLoopDefaultMode`这两个mode模式

而RunLoop大部分时间都处于`kCFRunLoopDefaultMode`，只有发生UI事件时才会处于`UITrackingRunLoopMode`，还有几种都是内部私有mode，我们也使用不了。

runloop common modes 并不是一个模式，而是一个`__CFRunLoopMode`实例的数组容器，处于这个数组容器中的模式的所有的事件，都会被runloop去处理。


### runloop的mode是不断变化，那么注册到runloop mode下的 timers/observers/sources 如何处理了？

- (1) 处理当前runloop处于某一个mode下的 timers/observers/sources 

- (2) 去`kCFRunLoopCommonModes`这个Set容器保存的`所有的__CFRunLoopMode实例`的timers/observers/sources 

###如果将一个事件源（NSTimer/Observer/Source）添加到runloop了，但是没有执行，可能的原因:

- 原因一、检查线程是否start、检查runloop是否正常运行
- 原因二、事件源添加到runloop时，指定的mode是不是错误

###RunLoopMode、RunLoop存在几种不同的状态mode ，而不同的mode下只接收处理对应的mode类型的事件源

会根据不同的mode进行`事件过滤`，只去处理这个mode下的 sources/timers/observers。

- `NSDefaultRunLoopMode`: RunLoop大多时候都处于这个模式，即空闲模式。
	- 当系统产生`UI事件`时，RunLoopMode就会切换成`UITrackingRunLoopMode`模式，就会转向处理`UI事件
	- 就会导致RunLoop之前的`NSTimer事件源`被遗弃，一直等到UI事件处理完毕
	- 所以，如果需要NSTimer在任何时刻都需要执行，最好使用`NSRunLoopCommonModes`这个mode

- `UITrackingRunLoopMode`: 系统触发`UI事件`时的模式
	- 当手指按住UITableView拖动时就会处于此模式
	- 为了来保证滑动事件的优先处理

- `UIInitializationRunLoopMode`: 苹果自己使用的，非公开的模式，App启动时RunLoop状态

- `NSRunLoopCommonModes`: 主要是组合如下几种RunLoop Mode模式下都进行工作
	- **这并不是一个RunLoopMode运行模式**
	- **其实就是一个类似NSSet的容器**
	- 默认存在于这个set中的两个RunLoopMode
		- 模式一、`NSDefaultRunLoopMode` 系统处于空闲
		- 模式二、`UITrackingRunLoopMode` 系统产生UI事件
	- 我们还可以将其他的runloop mode添加到这个NSRunLoopCommonModes

	```c
	void CFRunLoopAddCommonMode(CFRunLoopRef rl, CFStringRef modeName);
	```


##RunLoop有CoreFoundation与CoacaFoundation两个平台的实现版本

- CFRunLoopRef
	- 由CoreFoundation类库提供，提供的全部都是纯C语言的Api
	- 这些C语言的Api，都是`线程安全`的

- NSRunLoop
	- 是基于 CFRunLoopRef 的封装，提供了面向对象的 API
	- OC的API `不是线程安全`的
	- 所以需要避免在`其他线程`上使用`当前线程`的RunLoop

Objective-C 中的一切 `可变`对象 基本上都是 `非线程安全`。CoreFoundation的RunLoop版本是`开源`的，可以参考

```
http://opensource.apple.com/source/CF/CF-855.17/CFRunLoop.h
```

由于CF版本是开源的，所以学习时候大部分都是使用CF版本的代码，下面就去苹果的CF版本的RunLoop实现中去学习下吧.

##CoreFoundation中关于RunLoop体系结构中主要的几个组件

```
- (1) RunLoop本身的抽象 >>>> CFRunLoopRef
- (2) RunLoop运行模式的抽象 >>> CFRunLoopModeRef
- (3) 注册到RunLoop事件循环的`事件source` 的抽象 >>> CFRunLoopSourceRef
- (4) 注册到RunLoop事件循环的`观察者observer` 的抽象 >>> CFRunLoopObserverRef
- (5) 注册到RunLoop事件循环的`定时器timer` 的抽象 >>> CFRunLoopTimerRef
```


一个RunLoop对象在内存中包含如上所有东西的组成结构:

```
- 某一个线程的RunLoop对象
	- UIInitializationRunLoopMode(私有mode)
		- sources0
			- source1
			- source2
			- .....
			- sourceN
		- sources1
			- source1
			- source2
			- .....
			- sourceN
		- timers
			- timer1
			- timer2
			- ....
			- timerN
		- observers
			- observer1
			- observer2
			- ....
			- observerN
	- NSDefaultRunLoopMode(大部分时间都处于这个mode)
		- sources0
			- source1
			- source2
			- .....
			- sourceN
		- sources1
			- source1
			- source2
			- .....
			- sourceN
		- timers
			- timer1
			- timer2
			- ....
			- timerN
		- observers
			- observer1
			- observer2
			- ....
			- observerN
.......................................
```

图示 RunLoop、mode、source、timer、observer之间的组成关系

![](http://i4.piimg.com/567571/cfb8344751587e7d.png)

- (1) 一个RunLoop对象具备`四种`运行模式
	
```
NSDefaultRunLoopMode
UITrackingRunLoopMode
UIInitializationRunLoopMode（内部私有mode，无法使用）
GSEventReceiveRunLoopMode（内部私有mode，无法使用）
```
	
- (2) 当RunLoop对象处理一个事件时，会自动切换到与该事件匹配的mode模式下，这个Mode被称作`CurrentMode`

- (3) 每一种运行模式下，都可以注册不同的事件源
	- sources
		- sources0
		- sources1
	-  timers
	-  observers

- (4) 当RunLoop切换到某一个mode模式下时，暂时只会处理该mode模式下的 timers/observers/sources，一直到事件处理完毕切换到其他的mode模式下

##CFRunLoopRef、封装了事件循环的处理功能

###CoreFoundation中的代码结构定义:

```objc
typedef struct CF_BRIDGED_MUTABLE_TYPE(id) __CFRunLoop * CFRunLoopRef
```

所以CFRunLoopRef实际上是`__CFRunLoop`这个结构体类型实例的一个指针变量类型。

```c
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    
    // 【重要】接收事件的Port
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    
    // 【重要】RunLoop对应的线程对象
    pthread_t _pthread;
    
    uint32_t _winthread;
    
    // 【重要】组合在NSRunLoopCommomModes中的其他子mode
    CFMutableSetRef _commonModes;
    
    // 【重要】NSRunLoopCommomModes下的 timers/observers/sources
    CFMutableSetRef _commonModeItems;
    
    // 【重要】当前RunLoop对象的雨欣模式
    CFRunLoopModeRef _currentMode;
    
    // 【重要】RunLoop注册的所有运行模式
    CFMutableSetRef _modes;
    
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};
```

###开启当前所在线程的RunLoop

```c
void CFRunLoopRun(void) {	/* DOES CALLOUT */
    int32_t result;
    
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}
```

- (1) 可以看到该函数开启了一个`do-wihile(条件){}` 一个循环

- (2) 然后不断的执行`CFRunLoopRunSpecific()`这个c函数实现，并获取其执行的返回值

- (3) 而结束runloop执行的条件就是返回值是如下两个值
	- `result == kCFRunLoopRunStopped`
	- `result == kCFRunLoopRunFinished`


###但是我就有一个疑问，这样一个死循环会不会导致线程的卡顿了？

从前面一片文章中，一个全局dic保存thread与runloop之间的关系可以得到:

- (1) runloop和thread并不是一个东西
- (2) 但是之间是具有 `1:1` 的映射关系，保存在全局缓存dic中
- (3) 开启runloop时，其实只会卡主thread的init函数实现，并不会卡主其他的地方

可以从CFRunLoop开源代码中，查看`__CFRunLoopRun()`最终负责开启runloop函数的逻辑（代码太长就不贴了）:

- (1) 判断当前runloop是否还有items（timers/sources0/sources1）

- (2) 调用`bool ret = __CFRunLoopModeIsEmpty();`函数
	- 如果返回值 == false，就会return结束掉`do()while(){}`这个死循环，即退出结束runloop


可以查看`__CFRunLoopModeIsEmpty()`函数实现:

```c
static Boolean __CFRunLoopModeIsEmpty(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFRunLoopModeRef previousMode) {
	....
	
	// sources0事件items对象
	if (NULL != rlm->_sources0 && 0 < CFSetGetCount(rlm->_sources0)) return false;
	
	// sources1事件items对象
    if (NULL != rlm->_sources1 && 0 < CFSetGetCount(rlm->_sources1)) return false;
    
    // timer事件items对象
    if (NULL != rlm->_timers && 0 < CFArrayGetCount(rlm->_timers)) return false;
    
    ....
}
```

所以可以看到，当一个`__CFRunLoopMode`实例中的如下这三个数组容器中是否存在事件源:

- (1) `CFMutableSetRef _sources0;` 
- (2) `CFMutableSetRef _sources1;`
- (3) `CFMutableArrayRef _timers;`

这三个集合对象都没有item子对象时，这个RunLoop对象就会退出执行。


##CFRunLoopModeRef

首先看下这个mode的c结构体定义

```objc
typedef struct CF_BRIDGED_MUTABLE_TYPE(id) __CFRunLoopMode * CFRunLoopModeRef
```

```c
struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
    CFStringRef _name;
    Boolean _stopped;
    char _padding[3];
    
    // 【重要】_sources0类型的事件
    CFMutableSetRef _sources0;
    
    // 【重要】_sources1类型的事件
    CFMutableSetRef _sources1;
    
    // 【重要】该模式下所有的观察者
    CFMutableArrayRef _observers;
    
    // 【重要】该模式下所有的计时器
    CFMutableArrayRef _timers;
    
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet;
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```

- (1) RunLoop有多种运行模式，由系统自动变化
- (2) 每一个mode实例，有4个`Set`类型的容器，来保存不同类型的事件源
	- `_sources0`
	- `_sources1`
	- `_observers`
	- `_timers`

###开发者没有办法直接去操作`__CFRunLoopMode`对象

只有通过这些`kCFRunLoopDefaultMode`、`UITrackingRunLoopMode`、`kCFRunLoopCommonModes`等等标示去间接操作RunLoop对象内部的`__CFRunLoopMode`对象.

所以在默认情况下`schedule`一个Timer对象，实际上都是将这个timer事件加入到runloop对象的`kCFRunLoopDefaultMode`这个标示对于的`__CFRunLoopMode`对象内部的`_timers`数组中。

当发生UI操作时，也就是主线程RunLoop对象接收到一个UI事件，接着就会将这个timer事件加入到runloop对象的`UITrackingRunLoopMode`对应的`__CFRunLoopMode`对象中`_timers`数组内的timer对象。


##CFRunLoopSourceRef

描述一个注册到runloop接收的事件源

```objc
typedef struct CF_BRIDGED_MUTABLE_TYPE(id) __CFRunLoopSource * CFRunLoopSourceRef;
```

###runloop source结构体定义如下:

```c
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;			/* immutable */
    
     // 【重要】source被哪一些运行模式持有
    CFMutableBagRef _runLoops;
    
     // 【重要】包含两种类型事件
    union {
		CFRunLoopSourceContext version0;	/* immutable, except invalidation */
		CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};
```

###version0事件Context结构体定义

```c
typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
    void	(*schedule)(void *info, CFRunLoopRef rl, CFStringRef mode);
    void	(*cancel)(void *info, CFRunLoopRef rl, CFStringRef mode);
    void	(*perform)(void *info);
} CFRunLoopSourceContext;
```


###version1事件Context结构体定义

```c
typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
#if (TARGET_OS_MAC && !(TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)) || (TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)

	// 【区别】source1 会使用 mach_port_t 这个结构
	// 这个回调函数，获取得到一个mach port实例，用于线程之间的通信
    mach_port_t	(*getPort)(void *info);
    
    void *	(*perform)(void *msg, CFIndex size, CFAllocatorRef allocator, void *info);
#else
    void *	(*getPort)(void *info);
    void	(*perform)(void *info);
#endif
} CFRunLoopSourceContext1;
```


###对比CFRunLoopSourceContext与CFRunLoopSourceContext1最大的区别: `mach_port_t`

- (1) source1类型事件:

```c
- 使用到了 `mach_port_t` 这个结构实例，而`mach_port_t`可以用于多个线程之间的通信

- 假设A线程创建了一个source1，而B线程注册监听这个source1，那么对于B线程:
	- 调用`source1->getPort`这个回调函数
	- 来获取得到一个`mach_port_t`实例
	- 从而可以向`A线程`发送消息说，你的事情做完了，该你执行了

- 也就是 能够 `主动唤醒` 某个线程对象去处理这个事件（这个是和source0事件的根本区别）
```

- (2) source0类型事件:

```c
- 结构体有关于`schdule`调度事件、`cancel`取消事件、`perform`执行事件的`函数实现指针`

- 因为都是函数指针，所以对于source0这类事件，必须是通过一些函数来设置这些函数指针指向具体的某一个函数

- 对于创建的这类事件对象，首先必须通过`CFRunLoopSourceSignal(source)` 将这个 Source 标记为`待处理`

- 然后通过`CFRunLoopWakeUp(runloop)` 唤醒RunLoop 让其处理这个事件
```


##CFRunLoopObserverRef

每当runloop状态改变的时候，通知observer回调执行处理点什么。


```objc
typedef struct CF_BRIDGED_MUTABLE_TYPE(id) __CFRunLoopObserver * CFRunLoopObserverRef;
```

其observer的c结构体定义

```c
struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    
    // 【重要】监测的是哪一个runloop
    CFRunLoopRef _runLoop;
    
    CFIndex _rlCount;
    
    // 【重要】变化状态
    CFOptionFlags _activities;		/* immutable */
    CFIndex _order;			/* immutable */
    
    // 【重要】runloop状态改变时，回调执行的回调c函数
    CFRunLoopObserverCallBack _callout;	/* immutable */
    CFRunLoopObserverContext _context;	/* immutable, except invalidation */
};
```

###对于runloop具备如下变化状态

```c
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {

	// 状态一、即将进入Loop
    kCFRunLoopEntry = (1UL << 0),
    
    // 状态二、 即将处理 Timer
    kCFRunLoopBeforeTimers = (1UL << 1),
    
    // 状态三、 即将处理 Source
    kCFRunLoopBeforeSources = (1UL << 2),
    
    // 状态四、 即将进入休眠
    kCFRunLoopBeforeWaiting = (1UL << 5),
    
    // 状态五、 刚从休眠中唤醒
    kCFRunLoopAfterWaiting = (1UL << 6),
    
    // 状态六、 即将退出Loop
    kCFRunLoopExit = (1UL << 7),
    
    // Mask掩码
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

RunLoop对象状态变化图示（来自YY博客文章）:

![](http://i4.piimg.com/567571/cdb4164e313d96fd.png)

RunLoop就是一个`do{}while()`这样的一个循环，如果条件成立会一直执行`do{}`中的代码块，而`do{}`中做的事情就是不断重复的执行上面图示的事情.

第一件事、RunLoop开始执行时，通知observer执行回调。

第二件事 ~ 第九件事，属于`do{}while()`循环不断执行的代码，不断的等待时间处理时间、休眠唤醒，然后通知线程对象。

第十件事、RunLoop对象退出执行。


RunLoop对象退出执行的情况:


```c
一、手动执行 void CFRunLoopStop(CFRunLoopRef rl); 退出runloop
```

```c
二、Runloop对象中没有任何的 timer/source0/source1，自动退出
```

```c
三、Runloop对象运行超时，创建RunLoop对象时，指定了默认的超时时间为 `seconds = 9999999999.0;` 
```


##CFRunLoopTimerRef

就是经常使用的NSTimer

```objc
typedef struct CF_BRIDGED_MUTABLE_TYPE(NSTimer) __CFRunLoopTimer * CFRunLoopTimerRef;
```

timer的c结构体定义

```c
struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;
    pthread_mutex_t _lock;
    
    // 【重要】被哪一个runloop使用
    CFRunLoopRef _runLoop;
    CFMutableSetRef _rlModes;
    CFAbsoluteTime _nextFireDate;
    
    // 【重要】timer的间隔时间
    CFTimeInterval _interval;		/* immutable */
    CFTimeInterval _tolerance;          /* mutable */
    
    uint64_t _fireTSR;			/* TSR units */
    CFIndex _order;			/* immutable */
    
    // 【重要】指定timer被runloop回调执行的函数实现
    CFRunLoopTimerCallBack _callout;	/* immutable */
    CFRunLoopTimerContext _context;	/* immutable, except invalidation */
};
```


##代码参考资源

```
https://github.com/opensource-apple/CF
https://github.com/opensource-apple/objc4
```