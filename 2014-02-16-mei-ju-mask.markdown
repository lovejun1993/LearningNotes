---
layout: post
title: "枚举Mask"
date: 2014-02-16 15:36:05 +0800
comments: true
categories: 
---

###Mask 掩码，用于截取获得低n位的枚举值，让枚举值之间可以使用运算符`|`相加

- 带Mask掩码的枚举值定义

```
typedef NS_OPTIONS(NSInteger, Test) {

    TestIvarMask                = 0xFF,//1到8位
    TestIvar1                   = 1,
    TestIvar2                   = 2,
    
    TestMethodMask              = 0xFF00,//9到16位
    TestMethod1                 = 1 << 8,//左移8位
    TestMethod2                 = 2 << 8,//左移8位
    TestMethod3                 = 3 << 8,//左移8位
    
    TestPropertyMask            = 0xFF0000,//17到24位
    TestProperty1               = 1 << 16,
    TestProperty2               = 2 << 16,
    TestProperty3               = 3 << 16,
};
```

- 如上的Mask枚举值，没有实际作用，只是用来获取低位的枚举数值
- 使用`NS_OPTIONS`来定义具有`位移`的枚举类型

- 使用demo

```
Test test1 = TestIvar1;
Test test2 = TestIvar2;
Test test3 = TestIvar1 | TestIvar2;

Test test4 = test3 | TestMethod1;
Test Mask8 = test4 & TestIvarMask;
Test Mask16 = test4 & TestMethodMask;

Test test5 = test3 | TestMethod1 | TestProperty1;
Test Mask8_2 = test5 & TestIvarMask;
Test Mask16_2 = test5 & TestMethodMask;
Test Mask24 = test5 & TestPropertyMask;
```

输出

```
(Test) test1 = TestIvar1
(Test) test2 = TestIvar2
(Test) test3 = 3
(Test) test4 = 259
(Test) Mask8 = 3
(Test) Mask16 = TestMethod1
(Test) test5 = 65795
(Test) Mask8_2 = 3
(Test) Mask16_2 = TestMethod1
(Test) Mask24 = TestProperty1
```

也就是说一个枚举定义中，存在不同类型的定义，而不同的类型又需要组合.