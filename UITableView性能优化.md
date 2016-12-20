##UITableView性能优化

今天面试问道了这个东西，平时搞UI也没太在乎这些性能方面的东西，补补....

###UITableView delegate函数调用顺序

- (1) 显示5个cell，就先调用5次`tableView:heightForRowAtIndexPath:`确定如下值
	- contentSize(继承自UIScrollView): 当前ScrollView内部显示内容的总高度

- (2) 计算当前ScrollView.contentSize处于当前屏幕可见的部分，再调用`tableView:heightForRowAtIndexPath:`，知道cell需要显示多高

- (2) 最后调用`tableView:cellForRowAtIndexPath:`，获取显示在ScrollView上的Cell

###一、Frame异步计算与Cell.subviews.frame缓存、Cell高度缓存

- (1) FrameModel的概念
	- `tableView:cellForRowAtIndexPath:`方法只负责赋值
		- 包含一个Model实体类对象
	- `tableView:heightForRowAtIndexPath:`方法只负责计算高度
		- 包含所有Cell.subviews的frame
- (2) 从服务器加载一部分（一页）的List Item数据
- (3) 异步`dispatch_async(){}`将这部分List Item的frame计算出来
	- 每一个Item数据对应Cell.subviews的frame使用FrameModel对象保存		- 然后将FrameModel使用对应的NSIndexPath缓存起来
- (4)异步frame计算完毕之后，通知tableview走delegate函数，进行cell显示
	- `-[UITableView reloadRowsAtIndexPaths:withRowAnimation:]`
	- `-[UITableView reloadData]`

###二、使用CoreGraphics/CoreText进行文字绘制、图像绘制、背景颜色绘制，因为CoreGraphics/CoreText提供的所有的绘制函数都是线程安全的，那就可以在子线程上进行`异步绘制`

- UIKit的`单线程`天性意味着寄宿图通畅要在`主线程`上更新
	- (1) 这意味着绘制会打断用户交互
	- (2) 甚至让整个app看起来处于无响应状态
	- (3) 那么可以让某一些耗时的绘制代码，在子线程上`异步`执行绘制
	- (4) 绘制完毕之后，回到主线程将绘制出来的内容设置给`CALayer.contents`属性值

- 往Cell上添加各种View（UILabel、UIButton、UIImageView）底层实现:
	- (1) 每一个我们添加的View，都是在一个CALayer中进行渲染
	- (2) 然后将所有subviews的CALayer，进行合并、重叠处理，最终合并成一张图片

- 但是全部手动绘制，也不太现实，既麻烦还有CALayer也不具备事件响应
	- (1) 如果只是UI展示，那么可以考虑使用CALayer/CoreGraphics/CoreText进行异步绘制
	- (2) 如果需要具备事件响应，仍然还是使用UIKit。但是也可以重写`-[UIView hitTest:withEvent:]`实现来接收触摸事件

异步绘制demo代码:


```objc
@interface VVeboTableViewCell : UITableViewCell
- (void)draw;
@end
@implementation VVeboTableViewCell
- (void)draw{
	
	// 子线程异步进行绘制
	dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
	
		//1. 获取到异步计算完毕的frame
		CGRect rect = [_data[@"frame"] CGRectValue];
		
		//2. 以frame大小，创建一个绘图上下文
		UIGraphicsBeginImageContextWithOptions(rect.size, YES, 0);
		CGContextRef context = UIGraphicsGetCurrentContext();
        
       //3. 填充背景色
       [[UIColor colorWithRed:250/255.0 green:250/255.0 blue:250/255.0 alpha:1] set];
        CGContextFillRect(context, rect);
        
        //4. 绘制文字
        [_data[@"name"] drawInRect:_postContentTextView.frame];
	});
		
		//5. 绘制图像
		UIImage *temp = UIGraphicsGetImageFromCurrentImageContext();
      	UIGraphicsEndImageContext();
      	dispatch_async(backgroundQueue, ^{
			CGContextRef ctx = CGBitmapContextCreate(...);
				// draw in context...
				CGImageRef img = CGBitmapContextCreateImage(ctx);
				CFRelease(ctx);
				
				// 回到主线程设置CALayer.contents
				dispatch_async(mainQueue, ^{
				   _imgeLayer.contents = nil;
				   _imgeLayer.contents = img;
				});
       });
}
@end

- (void)drawRect:(CGRect)rect {
	// 也可以在这里写绘制代码
	// 这个方法也是异步执行，只是在主线程上
}
```

使用hitTest:withEvent:来代替UIView的事件响应demo代码:

```objc
- (UIView*)hitTest:(CGPoint)point withEvent:(UIEvent*)event{

    UIView*result = [superhitTest:pointwithEvent:event];

    CGPointbuttonPoint = [_subjectButtonconvertPoint:pointfromView:self];

    if ([_subjectButtonpointInside:buttonPointwithEvent:event]){

        return _subjectButton;

    }

    returnresult;

}
```

###三、滑动UITableView时，按需加载对应的内容

https://github.com/johnil/VVeboTableViewDemo/blob/master/VVeboTableViewDemo/VVeboTableView.m

###四、尽量的减少Cell.subviews个数，即减少CALayer的数量

这点感觉然并卵...

###五、提前在子线程解压图片

- (1) 将图片的一个像素绘制成一个像素大小的CGContext
- (2) 将整张图片绘制到CGContext中，丢弃原始的图片，并且用一个从上下文内容中新的图片来代替

如果不使用`+[UIImage imageNamed:]`，那么把整张图片绘制到CGContext可能是最佳的方式代码如下:

```objc
- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView
                  cellForItemAtIndexPath:(NSIndexPath *)indexPath
?{
    //dequeue cell
    UICollectionViewCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:@"Cell" forIndexPath:indexPath];
	//...
	
    //switch to background thread
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0), ^{
    
        //1. 使用读取文件方式读取压缩JPEG格式图片文件
        NSInteger index = indexPath.row;
        NSString *imagePath = self.imagePaths[index];
        UIImage *image = [UIImage imageWithContentsOfFile:imagePath];
        
        
        //2. 将压缩图片绘制到Context，在主线程上提前进行解压缩
        UIGraphicsBeginImageContextWithOptions(imageView.bounds.size, YES, 0);
        [image drawInRect:imageView.bounds];
        image = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        
        //3. 得到最终解压缩后的图像，回到主线程进行设置
        dispatch_async(dispatch_get_main_queue(), ^{
            imageView.image = image;
        });
    });
    return cell;
}
```