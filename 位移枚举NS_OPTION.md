## 同时存在`多种类型`，而且每一种类型又可以有多种状态，并且任意类型的不同状态之间还可以相互组合

比如，在UIView布局方法autoresizingMask中就有`位移枚举`的使用

```objc
typedef NS_OPTIONS(NSUInteger, UIViewAutoresizing) {
    UIViewAutoresizingNone                 = 0,
    UIViewAutoresizingFlexibleLeftMargin   = 1 << 0,
    UIViewAutoresizingFlexibleWidth        = 1 << 1,
    UIViewAutoresizingFlexibleRightMargin  = 1 << 2,
    UIViewAutoresizingFlexibleTopMargin    = 1 << 3,
    UIViewAutoresizingFlexibleHeight       = 1 << 4,
    UIViewAutoresizingFlexibleBottomMargin = 1 << 5
};
```

看枚举值定义，可以看到分别占用二进制位的情况：

- (1) UIViewAutoresizingNone 左数第一个二进制位，表示0
- (2) UIViewAutoresizingFlexibleLeftMargin 左数第一个二进制位，表示1
- (3) UIViewAutoresizingFlexibleWidth 左数第二个二进制位，表示1
- (4) UIViewAutoresizingFlexibleRightMargin 左数第三个二进制位，表示1

.......依次类推。

一段测试代码明白如何使用位移枚举:

```objc
- (void)test2 {
    
    UIViewAutoresizing resizing = UIViewAutoresizingNone;
    
    // 左侧距离、宽度拉伸、顶部距离、底部距离
    resizing = UIViewAutoresizingFlexibleLeftMargin | \
    UIViewAutoresizingFlexibleWidth | \
    UIViewAutoresizingFlexibleTopMargin | \
    UIViewAutoresizingFlexibleBottomMargin;
    
    
    UIViewAutoresizing resizing_leftMargin = resizing & UIViewAutoresizingFlexibleLeftMargin;
    UIViewAutoresizing resizing_RightMargin = resizing & UIViewAutoresizingFlexibleRightMargin;
    
    UIViewAutoresizing resizing_topMargin = resizing & UIViewAutoresizingFlexibleTopMargin;
    UIViewAutoresizing resizing_bottomMargin = resizing & UIViewAutoresizingFlexibleBottomMargin;
    
    UIViewAutoresizing resizing_width = resizing & UIViewAutoresizingFlexibleWidth;
    UIViewAutoresizing resizing_height = resizing & UIViewAutoresizingFlexibleHeight;

//先断点fr v输出上面的所有值

    if (resizing_leftMargin == UIViewAutoresizingFlexibleLeftMargin) {
        NSLog(@"添加了left 适应");
    }
    
    if (resizing_RightMargin == UIViewAutoresizingFlexibleRightMargin) {
        NSLog(@"添加了right 适应");
    }
    
    if (resizing_topMargin == UIViewAutoresizingFlexibleTopMargin) {
        NSLog(@"添加了top 适应");
    }
    
    if (resizing_bottomMargin == UIViewAutoresizingFlexibleBottomMargin) {
        NSLog(@"添加了bottom 适应");
    }
    
    if (resizing_width == UIViewAutoresizingFlexibleWidth) {
        NSLog(@"添加了width 适应");
    }
    
    if (resizing_height == UIViewAutoresizingFlexibleHeight) {
        NSLog(@"添加了height 适应");
    }
}
```

运行后输出结果

```
(UIViewAutoresizing) resizing = 43
(UIViewAutoresizing) resizing_leftMargin = 1
(UIViewAutoresizing) resizing_RightMargin = 0
(UIViewAutoresizing) resizing_topMargin = 8
(UIViewAutoresizing) resizing_bottomMargin = 32
(UIViewAutoresizing) resizing_width = 2
(UIViewAutoresizing) resizing_height = 0
```

```
2016-07-20 19:44:23.751 Demo[13929:1404736] 添加了left 适应
2016-07-20 19:44:23.751 Demo[13929:1404736] 添加了top 适应
2016-07-20 19:44:23.751 Demo[13929:1404736] 添加了bottom 适应
2016-07-20 19:44:23.751 Demo[13929:1404736] 添加了width 适应
```

如上是`一种类型`下有不同的状态，不同的状态之间可以相互组合。但是有时候有可能需要`不同类型`的枚举，就需要使用到Mask掩码了。

使用`按位与上该位的Mask掩码`，即可得到当前位的数值。

### 截取自YYModel中定义的位移枚举来解析Class所有数据结构时使用，使用到了`Mask掩码`。这种情况适用于:

- (1) 枚举有不同的类型，每一种类型占用固定的二进制位，并使用一个唯一的Mask掩码来获取
	- 1 ~ 8 位，Mask掩码: `0xFF`（十六进制，一位代表4个二进制位）
	- 9 ~ 16 位，Mask掩码: `0xFF00`
	- 17 ~ 24 位，Mask掩码: `0xFF0000`
- (2) 不同的类型下又有不同的状态，且不同状态之间可以相加
- (3) 然后某一类的组合值通过某一类中每一位的状态值进行`&`操作

改变自YYModel中的type encoding枚举定义的简单例子:

```objc
typedef NS_OPTIONS(NSInteger, PersonState) {
	
	// 第一种类型: 占用1~8位的二进制位，掩码是 1111,1111
    PersonStateMask                     = 0xFF,//1-8位的掩码（十六进制数，一个数代表4位，F:1111，0:0000）
    PersonStateUnknown                  = 0,
    PersonStateAlive                    = 1,
    PersonStateWork                     = 2,
    PersonStateDead                     = 3,

	// 第二种类型: 占用9~16位的二进制位，掩码是 1111,1111,0000,0000   
    HouseStateMask                      = 0xFF00,//9-16位，左移8位
    HouseStateNone                      = 1 << 8,
    HouseStateSmall                     = 1 << 9,
    HouseStateBig                       = 1 << 10,
    
	// 第三种类型: 占用17~24位的二进制位，掩码是 1111,1111,0000,0000,0000,0000   
    CarStateMask                        = 0xFF0000,//17-24位，左移16位
    CarStateNone                        = 1 << 16,
    CarStateSmall                       = 1 << 17,
    CarStateBig                         = 1 << 18,
};
```

```
0xFF = 15*16^1 + 15*16^0 = 255 
或
0xFF = 1111,1111 = 2^8 - 1 = 255
```

如下就是分别获取得到 1~8位、9~16位、17~24位 这三个区段的所谓的Mask掩码

```
0xFF		>>> 1111,1111 >>> 获取低8位值
0xFF00 		>>> 1111,1111,0000,0000 >>> 获取9-16位值
0xFF0000  	>>> 1111,1111,0000,0000,0000,0000 >>> 获取17-24位值
```

```
- 通过 `按位&` 上 `FF、FF00、FF0000`最大值，得到其对应位上的值
	- & 上 `FF` 获取低8位的值
	- & 上 `FF00` 获取第9位到16位的值
	- & 上 `FF0000` 获取第17位到24位的值
- 通过 `按位|` 两个枚举值相加
```

简单的使用例子

```objc
//1.
PersonState state = PersonStateUnknown;

//2.
state = PersonStateDead;

//3.
state = state | HouseStateBig;
NSLog(@"state = %ld", state);
NSLog(@"person state = %ld", state & PersonStateMask);
NSLog(@"house state = %ld", state & HouseStateMask);

//4.
state = state | CarStateBig;
NSLog(@"state = %ld", state);
NSLog(@"person state = %ld", state & PersonStateMask);
NSLog(@"house state = %ld", state & HouseStateMask);
NSLog(@"car state = %ld", state & CarStateMask);
```