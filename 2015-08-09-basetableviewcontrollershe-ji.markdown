---
layout: post
title: "BaseTableViewController设计"
date: 2015-08-09 19:31:38 +0800
comments: true
categories: 
---



##提供一个Base TableViewController基类，提供一些公关的代码给其继承的子类。

* 各自的UITableViewCell类型
	* 各自的cell类型
	* 多个cell类型共存
	* BaseCell根据对应实体类对象进行初始化subviews显示数据
	* Cell高度的动态计算
* 数据源方法、代理回调方法
* 其他工具方法
	* UITableView类型
	* 没有List数据时默认的背景View
	* cell出现动画
	* 获取多个cell类型的对象

	
***

###No1. 定义子类可以重写用于获取指定类型的Cell方法

.h

```objc

- (Class)cellClass;

```


.m

```objc

- (Class)cellClass {
	//默认返回BaseCell的类型，子类重写返回自己的Cell的Class，用于后续使用指定的Cell
	return [ZSYBaseCell class];
}

```

BaseTableViewController创建时，注册返回的Cell的Class

```objc

- (void)registCell {
    [_tableView registerClass:[self cellClass] forCellReuseIdentifier:[self cellId]];
}


```


###No2. 封装通用的数据源方法、代理回调方法

* BaseTableViewController定义子类可以重写的数据源提供方法
	
```objc
	
		//cell Id
		- (NSString *)cellId;
	
		//组的个数
		- (NSInteger)sectionCount;

		//组内行数
		- (NSInteger)rowCountForSection:(NSInteger)section;

		//数据List
		- (NSArray *)dataSource;
	
```
	
```objc 
	
- (NSString *)cellId {
	//默认实现，可以重写		
    return NSStringFromClass([self class]);
}
	
- (NSInteger)sectionCount {
		//默认只返回一个section
	   return 1;
}

- (NSInteger)rowCountForSection:(NSInteger)section {
		//默认按照1个section形式，返回rowCount总数		
		 return [[self dataSource] count];
}

- (NSArray *)dataSource {
	//数据源默认实现
    return nil;
}	
			
```

	
```objc

	- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {

		//调用子类控制器的方法，返回section数
		return [self sectionCount];
	}

	- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {

		//调用子类控制器的方法，返回当前section的rowCount数
	    return [self rowCountForSection:section];
	}

	- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    ZSYBaseCell *cell = [tableView dequeueReusableCellWithIdentifier:[self cellId] forIndexPath:indexPath];
    
    cell.tableView = tableView;
    cell.indexPath = indexPath;
    cell.shouldShowLine = YES;
//    cell.hidden = NO;
    
    //数据源
    NSArray *dataList = [self dataSource];
    
    //按照一个section、多个section情况分析
    //1. 一维数组
    //2. 二维数组 ，[i]又是一个数组
    //3. 二维数组 ，[i]只是一个NSObject实体对象
    
    if ([self sectionCount] == 1) {//一维数组
        [cell setupDataItem:dataList[[indexPath row]]];
    } else {//二唯数组
        
        id sectionData = dataList[indexPath.section];
        
        if ([sectionData isKindOfClass:[NSArray class]]) {
            NSArray *sectionArray = (NSArray *)sectionData;
            id obj = sectionArray[indexPath.row];
            [cell setupDataItem:obj];
        } else {
            [cell setupDataItem:sectionData];
        }
    }
    
    	if (_OnConfigCell) {
       	 _OnConfigCell(indexPath, cell);
	    }
    
   		 return cell;
}

```

* Delegate方法

```objc 

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    UITableViewCell *cell = [tableView cellForRowAtIndexPath:indexPath];
    NSArray *dataList = [self dataSource];
    id data = nil;
    if ([self sectionCount] == 1) {//一维数组
        data = dataList[[indexPath row]];
    } else {
        
        id sectionData = dataList[indexPath.section];
        
        if ([sectionData isKindOfClass:[NSArray class]]) {
            NSArray *sectionArray = (NSArray *)sectionData;
            data = sectionArray[indexPath.row];
            
        } else {
            data = sectionData;
        }
    }
    
    if (_OnItemSelect) {
        _OnItemSelect(indexPath, data, cell);
    }
}


- (CGFloat)tableView:(UITableView *)tableView estimatedHeightForRowAtIndexPath:(NSIndexPath *)indexPath {
    CGFloat h = [self cellDefaultHeight];
    return h;
}

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    if (![[self cellClass] isDynamic]) {
        return [self cellDefaultHeight];
    }
        
    __weak typeof(self) weakSelf = self;
    
    //默认 size
    CGSize defaultSize = [[self cellClass] defaultCellSize];
    
    //计算后自适应的 size
    CGSize cellSize = [[self cellClass] sizeForCellWithDefaultSize:defaultSize
                                                    setupCellBlock:^id(id<ZSYAutoLayoutCellProtocol> cellToSetup){
        NSArray *dataList = [weakSelf dataSource];
        id obj = nil;
        
        if ([self sectionCount] == 1) {//一维数组
            obj = dataList[[indexPath row]];
            [cellToSetup setupDataItem:obj];
        } else {//二唯数组
            id sectionData = dataList[indexPath.section];
            
            if ([sectionData isKindOfClass:[NSArray class]]) {
                NSArray *sectionArray = (NSArray *)sectionData;
                id obj = sectionArray[indexPath.row];
                [cellToSetup setupDataItem:obj];
            }
        }
        return cellToSetup;
    }];
    
    return cellSize.height;

}



-(void)tableView:(UITableView *)tableView willDisplayCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath
{
    if (![self isPerDisplayAnimate]) {
        if ([_showIndexes containsObject:indexPath]) {
            return;
        }
    }
    
    switch ([self zsyAnimateType]) {
        case ZSYTableViewAnimateTypeNone: {
            //do nothings ...
        }
            break;
            
        case ZSYTableViewAnimateTypeScale: {
            [self addAnimationScale:cell];
        }
            break;
    }
    
    [_showIndexes addObject:indexPath];
    
}


```