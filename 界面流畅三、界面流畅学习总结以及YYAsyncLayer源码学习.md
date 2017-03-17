
## 开始

学习计划:

```
1. 看了一遍 CoreAnimation 高级编程（基础的基础，必须看一遍）
2. YYKit那篇界面流畅的文章（以前也看过，但是一定要细看，多看，才会有收获）
3. 仔细看了VVeboTableView的优化代码（这个最好动手做一遍）
4. 复习了下CoreText文本、图文绘制（这个就是一些基础拉）
5. 仔细看了YYAsyncLayer源码（源自大牛，必须得精看、看懂、利用、变为自己的）
```

一些简单的东西、以及YYKit文章中提及的优化点我就不说了。主要说下VVeboTableView里面那个优化列表滚动的demo。

可能很多人也都看过，我问过我很多朋友、同事，都说看过。但是，知道具体有什么优化吗？都说不太清楚...

于是，我拿着VVeboTableView的代码，然后自己一点点抠，一行一行的看，跟着做了一个demo，然后又自己优化了一下的效果图:

![demo.gif](http://upload-images.jianshu.io/upload_images/1829660-78ebbdb3f3335a81.gif?imageMogr2/auto-orient/strip)

我这个是模拟器跑的，用真机跑基本是在60FPS。可以看到一个最明显的效果，就是快速滚动多个cell的时候，出现的都是空白，也就是并没有立刻去进行绘制，而是等到列表停止滚动时，才去进行cell的绘制。

YYKit作者也说过，这个空白的效果，可能是这个代码的最大的一个缺点，但是仅此而已，但是对于整个TableView滚动的性能优化，还是不错的。

对于大多数的滚动界面，按照这个思路去做优化，基本上不会遇到卡顿的问题，只是需要直接使用CoreText、CoreGraphics等直接进行绘制，还是略显麻烦。

主要的优化逻辑就是，将中间快速滚动滚出当前可见区域的cell，直接忽略，不进行绘制。这样一来，就会减少很多的不可见cell的绘制，不管是对CPU还是GPU都节约了很多的时间。

自然而然，FPS就很高了。

### 可能还有一些初学者对`FPS`不太明白是个什么东西，我科普下，因为我也是最近才弄懂的...

- (1) 我们在屏幕显示器看到的东西，其实每一秒种，都在不停的刷新。即不停的擦除上一秒显示的数据，然后又绘制下一秒要显示的数据。

- (2) FPS就是显示器，`一秒钟`刷新屏幕的次数。

- (3) 对于我们的PC，比如我玩CF的时候，低于30就有点卡了。我在网吧玩，都是在60~100左右，相当的流畅。

- (4) 对于移动设备（手机）的显示器来说，正常情况下，最好保持在`60`次左右。但是也分情况:
	- 一般情况下，最好保持60左右
	- 如果有复杂的动画，可能最低保持到30

- (5) 关于FPS如何计算出来的:
	- 首先要搞清楚，是计算主线程的FPS（绘制显示都是在主线程）
	- 向主线程的Runloop注册一个CADisplayLink定时器事件源
	- 然后系统每当进行屏幕刷新重绘的时候，都会回调CADisplayLink指定的回调函数
	- 在CADisplayLink回调函数中，让一个时间间隔内的`总刷新次数`除以`时间间隔`，即可得到当前一秒内的刷新次数，这个就是FPS


### FPS低，为何就会出现界面卡顿的效果？

我们写的每一个UIView对其设置的文本、图片、背景色、字体、边框样式...等等。最终显示到屏幕上，都会经历如下这些步骤:

- (1) 对象的创建、调整、计算
	- UIView对象的创建、frame的计算、frame的设置
	- 文件的读取
	- 自定义绘制

- (2) 由CoreAnimation对UIView对象提取成一个个的CALayer，然后发送给渲染服务`Render Server`（一个独立的进程，专门进行CALayer的数据）`Render Server`处理后的数据，再交给GPU进行最终的处理。

- (3) GPU将接收到的数据，处理为显示器能够识别的数据格式（纹理？），确切的说是用于`某一帧`显示的数据。然后GPU将最终的数据，扔到`帧数据缓冲池`。

- (4) 显示器，每隔一段时间，就从`帧数据缓冲池`取出帧数据，显示到屏幕上。

简单的可以理解为这么几个步骤，要往深了说那可就有点复杂，我就不展开了，因为我也不懂那么深（哈哈~~）。

针对上面的几个步骤，基于硬件，可以分为两个大方向:

- (1) CPU
- (2) GPU

摘抄下，YYKit文章中记录的这个两个硬件主要做的事情:

#### CPU做的事情

- (1) 对象的创建、调整、废弃
- (2) UI对象的frame计算、文本size的计算、autolayout的计算
- (3) 文本的渲染、图片的渲染、以及其他自定义的绘制渲染
- (4) 各种磁盘文件的读取、写入。
- (5) 对于图片文件，图片的解压缩、图片的解码

#### GPU做的事情

- (1) 纹理的渲染
- (2) 多图层的混合 (Composing) 
- (3) 特效的离屏渲染（border、圆角、阴影、遮罩（mask）...）

总之，我想表达的时，必须等待CPU先处理完毕，然后再交给GPU处理完毕，最后通知显示器去去数据显示，这么一个过程。

那如果因为CPU货GPU处理时间超时，就会造成显示器一直没有刷新屏幕。而长时间未刷新屏幕，屏幕就一直显示之前的一帧数据，就给用户形成一种卡死了的感觉。

而突然GPU处理完毕，通知显示器去读取数据显示。此时，屏幕上又突然刷新了下一帧的数据显示，也会出现一种不太平和的过度。

总结起来，就是突然卡主了，又突然出现另一个出面，没有平和的过度，这就是卡顿感的产生。

YYkit文章中，也分别记录对应如上这些事情的具体优化的办法，我就不列举了。

## 我总结下这个代码的优化点把:

第一个重头戏、使用CoreText进行文本绘制，这个是进行后续优化的基础。实现起来是有点麻烦，但是必须这么做，否则无法完成后面的的优化。

```
1. 自定义UILabel，内部使用CoreText文本绘制
2. 子线程异步完成文本的绘制，并渲染得到图像
3. 最终将图像设置给UILabel的backing layer显示
4. 涉及到的特殊文本的高亮、点击效果，通过预先正则式切割，然后保存高亮文本出现的frame
5. 将绘制渲染的代码，全部放到子线程
```

第二个重头戏、TableView滚动时，监听scrollView的状态

```
1. 开始滚动时，保存一个状态
2. 停止滚动时，保存一个状态，并且计算出最终停止时的indexpath
3. 过滤掉中间快速滚动过的indexpath
4. 将最终停止出现的可见indexpath保存到一个数组，并额外添加附近的三个indexpath
5. 最终列表停止后，将保存在数组中的indexpath，挨个取出对应的cell，进行内容的绘制
```

cell里面的文本部分就交给上面自定义的UILabel完成，其他很小不能复用的的图像、文本的绘制，可以使用UIKit的绘制，然后同样从绘图上下文获取得到渲染的图像，塞给cell内部的一个backgroundView或者backing layer显示即可。

这是我在这个demo中学习到的列表优化思路，简单的列举下片段代码吧。比如，上面使用CoreText文本绘制，渲染图像，设置给layer显示:

```c
@implementation XXXLabel
- (void)asyncDraw {
    
    /**
     *  渲染生成图像的过程，全部都在子线程异步完成
     */
    
    XZHDispatchQueueAsyncBlockWithQOSBackgroud(^{
        
        //1. 设置画布的大小
        CGSize size = self.frame.size;
        size.height += 10;
        
        //2. 创建一个画布
        UIGraphicsBeginImageContextWithOptions(size, ![self.backgroundColor isEqual:[UIColor clearColor]], 0);
        CGContextRef context = UIGraphicsGetCurrentContext();
        if (context==NULL) {return;}
        
        //3. 填充画板的背景色，否则默认是黑色的
        if (![self.backgroundColor isEqual:[UIColor clearColor]]) {
            [self.backgroundColor set];
            CGContextFillRect(context, CGRectMake(0, 0, size.width, size.height));
        }
        
        //3. 翻转上下文的y坐标轴，因为CoreText与UIKit的，坐标系y轴是`相反`的
        CGContextSetTextMatrix(context,CGAffineTransformIdentity);
        CGContextTranslateCTM(context,0,size.height);
        CGContextScaleCTM(context,1.0,-1.0);
        
        //4. 尝试使用 [绘制原始文字的 MD5] 值，从缓存进行读取CTFrameRef的
        NSString *md5 = [_text xzh_MD5];
        CTFrameRef ctFrame = CTFrameForKey(md5);
        
        //5. 绘制时产生的临时变量，需要在绘制结束后进行释放废弃
        CTFontRef font;
        CTFramesetterRef framesetter;

        //6. 判断是否使用缓存的CTFrameRef，进行直接绘制
        CGRect rect = CGRectMake(0, 5,(size.width),(size.height-5));
        if (!_highlighting && ctFrame) {
            //6.1 使用缓存的CTFrame进行绘制，不必再进行文本的解析、渲染
            [self drawWithCTFrame:ctFrame inRect:rect context:context];
        } else {
            //6.2 重新对文本进行富文本设置、解析、CTFrameRef渲染
            
            //6.2.1 准备要绘制的富文本内容 >>> NSMutableAttributedString
            UIColor* textColor = self.textColor;
            CGFloat minimumLineHeight = self.font.pointSize,maximumLineHeight = minimumLineHeight, linespace = self.lineSpace;
            font = CTFontCreateWithName((__bridge CFStringRef)self.font.fontName, self.font.pointSize,NULL);
            CTLineBreakMode lineBreakMode = kCTLineBreakByWordWrapping;
            CTTextAlignment alignment = CTTextAlignmentFromUITextAlignment(self.textAlignment);
            CTParagraphStyleRef style = CTParagraphStyleCreate((CTParagraphStyleSetting[6]){
                {kCTParagraphStyleSpecifierAlignment, sizeof(alignment), &alignment},
                {kCTParagraphStyleSpecifierMinimumLineHeight,sizeof(minimumLineHeight),&minimumLineHeight},
                {kCTParagraphStyleSpecifierMaximumLineHeight,sizeof(maximumLineHeight),&maximumLineHeight},
                {kCTParagraphStyleSpecifierMaximumLineSpacing, sizeof(linespace), &linespace},
                {kCTParagraphStyleSpecifierMinimumLineSpacing, sizeof(linespace), &linespace},
                {kCTParagraphStyleSpecifierLineBreakMode,sizeof(CTLineBreakMode),&lineBreakMode}
            },6);
            
            NSDictionary* attributes = [NSDictionary dictionaryWithObjectsAndKeys:(__bridge id)font,(NSString*)kCTFontAttributeName,
                                        textColor.CGColor,kCTForegroundColorAttributeName,
                                        style,kCTParagraphStyleAttributeName,
                                        nil];
            NSMutableAttributedString *attributedStr = [[NSMutableAttributedString alloc] initWithString:_text
                                                                                              attributes:attributes];
            
            //6.2.2 NSMutableAttributedString >>> CTFramesetterRef
            CFAttributedStringRef attributedString = (__bridge CFAttributedStringRef)[self highlightText:attributedStr];
            framesetter = CTFramesetterCreateWithAttributedString((CFAttributedStringRef)attributedString);
            
            //6.2.3 CGPath + CTFramesetterRef >>> CTFrameRef。设置要绘制文字的路径:
            CGMutablePathRef path = CGPathCreateMutable();
            CGPathAddRect(path, NULL, rect);
            
            //6.2.4 生成该区域绘制文本的数据 CTFrameRef
            CTFrameRef ctFrame = CTFramesetterCreateFrame(framesetter,
                                                        CFRangeMake(0, _text.length),
                                                        path,
                                                        NULL);
            
            //6.2.5 将CTFrameRef实例，缓存起来，避免重复对同一段文本进行解析
            CacheCTFrameWithKey(ctFrame, md5);
            
            //6.2.6 将重新解析CTFrame进行绘制
            [self drawWithCTFrame:ctFrame inRect:rect context:context];
            
            //6.2.7 不对frame废弃，在内存中缓存起来
            //CFRelease(ctFrame);
        }
        
        //7. 继续翻转y轴，得到UIKit的y轴方向
        CGContextSetTextMatrix(context,CGAffineTransformIdentity);
        CGContextTranslateCTM(context,0,size.height);
        CGContextScaleCTM(context,1.0,-1.0);
        
        //8. 从上下文获取渲染得到的图像
        UIImage *screenShotimage = UIGraphicsGetImageFromCurrentImageContext();
        
        //9. 结束绘图上下文
        UIGraphicsEndImageContext();
        
        //10. 回到主线程，将渲染得到的图像，给给layer显示
        dispatch_async(dispatch_get_main_queue(), ^{
//            if (font) {CFRelease(font);}
//            if (framesetter) {CFRelease(framesetter);}

            if (_highlighting) {
                _highlightImageView.image = nil;
                if (_highlightImageView.width!=screenShotimage.size.width) {
                    _highlightImageView.width = screenShotimage.size.width;
                }
                if (_highlightImageView.height!=screenShotimage.size.height) {
                    _highlightImageView.height = screenShotimage.size.height;
                }
                _highlightImageView.image = screenShotimage;
            } else {
                if (_labelImageView.width!=screenShotimage.size.width) {
                    _labelImageView.width = screenShotimage.size.width;
                }
                if (_labelImageView.height!=screenShotimage.size.height) {
                    _labelImageView.height = screenShotimage.size.height;
                }
                
                // 清空高亮view的图像
                _highlightImageView.image = nil;
                
                _labelImageView.image = nil;
                _labelImageView.image = screenShotimage;
            }
            
//            [self debugDraw];//绘制可触摸区域
        });
    });
}

- (void)drawWithCTFrame:(CTFrameRef)frame
                 inRect:(CGRect)rect
                context:(CGContextRef)ctx
{
    //1.
    if (NULL == frame) {return;}
    if (NULL == ctx) {return;}
    
    //2. 所有行
    CFArrayRef lines = CTFrameGetLines(frame);
    NSInteger numberOfLines = CFArrayGetCount(lines);
    
    //3.
    CGPoint lineOrigins[numberOfLines];
    CTFrameGetLineOrigins(frame, CFRangeMake(0, numberOfLines), lineOrigins);
    
    //4.
    for (CFIndex lineIndex = 0; lineIndex < numberOfLines; lineIndex++) {
        
        //4.1
        CGPoint lineOrigin = lineOrigins[lineIndex];
        lineOrigin = CGPointMake(CGFloat_ceil(lineOrigin.x), CGFloat_ceil(lineOrigin.y));
        
        //4.2
        CGContextSetTextPosition(ctx, lineOrigin.x, lineOrigin.y);
        
        //4.3
        CTLineRef line = CFArrayGetValueAtIndex(lines, lineIndex);
        
        //4.4
        CGFloat descent = 0.0f;
        CGFloat ascent = 0.0f;
        CGFloat lineLeading;
        CTLineGetTypographicBounds((CTLineRef)line, &ascent, &descent, &lineLeading);
        
        //4.5 Adjust pen offset for flush depending on text alignment
        CGFloat flushFactor = NSTextAlignmentLeft;
        CGFloat penOffset;
        CGFloat y;
        
        //4.6
        penOffset = (CGFloat)CTLineGetPenOffsetForFlush(line, flushFactor, rect.size.width);
        y = lineOrigin.y - descent - self.font.descender;
        CGContextSetTextPosition(ctx, penOffset, y);
        CTLineDraw(line, ctx);
        
        //4.7
        if (!_highlighting && (self.superview != nil)) {
            CFArrayRef runs = CTLineGetGlyphRuns(line);
            for (int j = 0; j < CFArrayGetCount(runs); j++) {
                CGFloat runAscent;
                CGFloat runDescent;
                CTRunRef run = CFArrayGetValueAtIndex(runs, j);
                
                NSDictionary* attributes = (__bridge NSDictionary*)CTRunGetAttributes(run);
                
                if (!CGColorEqualToColor((__bridge CGColorRef)([attributes valueForKey:@"CTForegroundColor"]), self.textColor.CGColor)
                    && _clickRangeFramesDict!=nil) {
                    CFRange range = CTRunGetStringRange(run);
                    CGRect runRect;
                    runRect.size.width = CTRunGetTypographicBounds(run, CFRangeMake(0,0), &runAscent, &runDescent, NULL);
                    float offset = CTLineGetOffsetForStringIndex(line, range.location, NULL);
                    float height = runAscent;
                    runRect = CGRectMake(lineOrigin.x + offset, (self.height+5)-y-height+runDescent/2, runRect.size.width, height);
                    NSRange nRange = NSMakeRange(range.location, range.length);
                    [_clickRangeFramesDict setValue:[NSValue valueWithCGRect:runRect] forKey:NSStringFromRange(nRange)];
                }
            }
        }
    }
}
@end
```

然后是TableView监控快速滚动的逻辑

```objc
#pragma mark - UIScrollViewDelegate

// 【开始滚动时】、清除缓存的所有cell
- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView{
    
    //1. 标记正在滚动ing
    _isScrolling = YES;
    
    //2. 清除之前保存的绘制cell的indexPath
    [_drawableIndexPaths removeAllObjects];
}

// 【手指离开屏幕】、如果【最终停止的indexpath】与【当前indexpath】相差超过指定行数
//那么只在目标滚动范围的前后指定3行的cell进行内容数据的绘制
- (void)scrollViewWillEndDragging:(UIScrollView *)scrollView
                     withVelocity:(CGPoint)velocity
              targetContentOffset:(inout CGPoint *)targetContentOffset
{
    //1. 发生滚动之前，当前可见区域的最上面的一个cell的indexPath
    NSIndexPath *curIndexPath = [_tableView xzh_firstVisbledCellIndexPath];
    
    //2. 最终停止滚动时，出现在屏幕上可见区域的最下方的坐标点(x,y)
    CGPoint stoppedPoint = CGPointMake(0, targetContentOffset->y);
    
    //3. 通过stoppedPoint，查询到所处的IndexPath
    NSIndexPath *stopedIndexPath = [_tableView indexPathForRowAtPoint:stoppedPoint];
    
    //4.
    NSLog(@"curIndexPath = %@, stopedIndexPath = %@", curIndexPath, stopedIndexPath);
    
    //5. 设置 【滚动前的row】 到 【滚动停止时的row】之前最大相差的【行数】
    NSInteger skipCount = 8;
    
    /**
     *  6. 如果 【滚动前的row】 距离 【滚动停止时的row】，超过了 skipCount
     *  - (1) 则忽略中间的 skipCount个 cell的绘制
     *  - (2) 只在停止滚动的【前后】指定的 3行 cell进行绘制
     */
    BOOL isOverSkipCount = labs(stopedIndexPath.row - curIndexPath.row) > skipCount;
    
    //7. 如果超过了skipCount，则完成上的(1)、(2)
    if (isOverSkipCount) {

        //7.1 获取最终停止滚动位置时，在屏幕上可见的cell对应的IndexPath
        NSArray *stoppedVisbleIndexpaths = [_tableView indexPathsForRowsInRect:CGRectMake(0,
                                                                                          targetContentOffset->y,
                                                                                          _tableView.width,
                                                                                          _tableView.height)];
        
        //7.2
        NSMutableArray *mutableIndexPaths = [NSMutableArray arrayWithArray:stoppedVisbleIndexpaths];
        
        //7.3 判断是继续正方向滚动或反方向滚动
        if (velocity.y > 0) {
            // 7.3.1 正方向滚动
            // 获取【最后】一个可见的cell的IndexPath
            NSIndexPath *idx = [mutableIndexPaths lastObject];
            
            // 添加下方的顺数三个cell的IndexPath
            if ((idx.row + 3) < _tweetList.count) {
                NSIndexPath *next1 = [idx xzh_nextRow];
                NSIndexPath *next2 = [next1 xzh_nextRow];
                NSIndexPath *next3 = [next2 xzh_nextRow];
                [mutableIndexPaths addObject:next1];
                [mutableIndexPaths addObject:next2];
                [mutableIndexPaths addObject:next3];
            }
            
        } else {
            //7.3.2 反方向滚动
            // 获取【最前面】一个可见的cell的IndexPath
            NSIndexPath *idx = [mutableIndexPaths firstObject];
            
            // 添加上方的倒数三个cell的IndexPath
            if ((idx.row - 3) >= 0) {
                NSIndexPath *prev1 = [idx xzh_previousRow];
                NSIndexPath *prev2 = [prev1 xzh_previousRow];
                NSIndexPath *prev3 = [prev2 xzh_previousRow];
                [mutableIndexPaths addObject:prev1];
                [mutableIndexPaths addObject:prev2];
                [mutableIndexPaths addObject:prev3];
            }
        }
        
        //7.4 保存需要进行绘制的cell的indexPath
        [_drawableIndexPaths addObjectsFromArray:mutableIndexPaths];
        
    } else {
        
        /**
         *  走到这里，不会走scrollview下面的几个delegate函数，
         *  所以，直接标记停止滚动，并绘制当前scrollview的可见区域的subviews
         */
        
        //7.1 标记停止滚动
        _isScrolling = NO;
        
        //7.2 绘制当前可见区域的cell
        [self drawVisbledCells];
    }
}

// 【是否允许滚动到顶部】
- (BOOL)scrollViewShouldScrollToTop:(UIScrollView *)scrollView{
    
    //1. 标记正在滚动ing
    _isScrolling = YES;
    
    //2. 允许滚动到顶部
    return YES;
}

// 【已经滚动到顶部】
- (void)scrollViewDidScrollToTop:(UIScrollView *)scrollView{
    
    //1. 标记已经停止滚动
    _isScrolling = NO;
    
    //2. 绘制当前可见的cell
    [self drawVisbledCells];
}


// 【停止滚动】
- (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView {
    
    //1. 标记已经停止滚动
    _isScrolling = NO;
    
    //2. 绘制当前可见的cell
    [self drawVisbledCells];
}
```

这个demo给我学习到的主要就是子线程异步使用CoreText文本渲染得到图像直接塞给layer显示，再就是scrollview快速滚动时过滤掉被滚动超过一定数的cell不进行绘制。

这个demo，还重写SDWebImage的部分代码，就是下载到图片后，直接将图像绘制到context，然后从context获取得到渲染的图像，然后塞给layer显示。

## 我还加入了自己的一些优化点

```
1. dispatch_queue_t 的缓存
2. CTFrameRef 的缓存
```

因为`dispatch_get_global_queue()`获取的是并发队列，也就是说可能会创建n个线程，这个是我们控制不了的，完全看GCD底层的心情。

而YYKit作者，列举出了使用并发队列进行异步绘制时，很可能会出现某一个线程长时间被锁住的情况。而一旦出现这种情况，GCD底层就会去创建新的线程来分配当前其他等待执行的绘制代码。

一旦等待的线程越来越多，就会出现n多个新线程被无限制的创建（可能有点夸张），但还是会有一定的CPU影响的。

YYKit作者写了一个`dispatch_queue_t 串行`实例的缓存容器，并且使用iOS8推荐的QOS来搭建。每一种QOS对应的一个Context，每一个Context下保存当前CPU激活核心数相等的`dispatch_queue_t 串行`实例个数。其Pool的结构图:

```
- Pool
	- (1) QOS_CLASS_USER_INITIATED Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
	- (2) QOS_CLASS_DEFAULT Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
	- (3) QOS_CLASS_UTILITY Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
	- (4) QOS_CLASS_BACKGROUND Dispatch Context 对象
		- 缓存的dispatch_queue_t 实例1
		- 缓存的dispatch_queue_t 实例1
		- ....
		- 缓存的dispatch_queue_t 实例n
```

这样一来，线程能够复用，控制线程数不太多，又够让CPU核心数全部泡满，真划算。然后。期初我还想过，为何不直接对`NSThread`对象进行缓存，就像AFNetworking那样做一个后台服务的NSThread那样进行缓存了？

后来我试了一下，主要就一个原因，麻烦...呵呵，实在是太麻烦，难度也有点大。首先一个问题就是，让NSThread对象一直保活。并不是不让其释放废弃，而是让这个NSThread一直能接受事件进行执行。

你们可以尝试下，保存一个NSThread对象，然后后续不断的给其分配任务执行，看看有啥问题就知道了。再就是还需要做大量的同步互斥的操作，并且线程池需要复用，但是当不够的时候还需要去创建，总之难度还是大大的，于是放弃了。再而言之，已经有`dispatch_queue_t`这么好的东西，就不要走原始路了把...

我对其又精简了一下，其主要的API如下:

```c
// 创建pool
void XZHDispatchQueuePoolCreate();
```


```c
// 将一个绘制代码block，分配到一个QOS下的缓存queue

- (void)test2 {
    
   XZHDispatchQueueAsyncBlockWithQOSUserInteractive(^{
        NSLog(@"task1 : %@", [NSThread currentThread]);
    });

    XZHDispatchQueueAsyncBlockWithQOSUserInitiated(^{
        NSLog(@"task2 : %@", [NSThread currentThread]);
    });

    XZHDispatchQueueAsyncBlockWithQOSUtility(^{
        NSLog(@"task3 : %@", [NSThread currentThread]);
    });

    XZHDispatchQueueAsyncBlockWithQOSBackgroud(^{
        NSLog(@"task4 : %@", [NSThread currentThread]);
    });
```

```c
// 不再使用的时候，全部废弃掉。但是我觉得还是不要废弃，全局就使用这一个缓存池

void XZHDispatchQueuePoolRelease();
```


这样一来，就按照不同的QOS等级，来分配不同的绘制任务，这样一来可以按照轻重缓急来进行绘制任务的执行了，并且使用的是`串行`队列，保证不会出现创建多个线程进行绘制的情况。

然后就是对CoreText对文本最终计算出来的CTFrameRef实例进行缓存：

```objc
- (void)asyncDraw {
    
    /**
     *  渲染生成图像的过程，全部都在子线程异步完成
     */
    
    XZHDispatchQueueAsyncBlockWithQOSBackgroud(^{
        ..........................     
       
        // 对文本的md5
         NSString *md5 = [_text xzh_MD5];

        // 使用MD5从内存缓存取出CTFrameRef
         CTFrameRef ctFrame = CTFrameForKey(md5);

        // 判断是否使用缓存的CTFrameRef，进行直接绘制
        if (!_highlighting && ctFrame) {
            // 使用缓存的CTFrame进行绘制，不必再进行文本的解析、渲染
            [self drawWithCTFrame:ctFrame inRect:rect context:context];
        } else {
            // 重新走解析CTFrameRef的流程
            CTFrameRef ctFrame = ............;

            // 将CTFrameRef实例，缓存起来，避免重复对同一段文本进行解析
            CacheCTFrameWithKey(ctFrame, md5);

            ...........

      }
}
```

避免对同一段相同文字的重复性的解析，也是一个小小的优化吧。

## YYAsyncLayer的源码学习

开始之前，我有一个疑问。系统已经提供了很多种的高效绘图的专用CALayer子类，还需要再自己写一个这样的CALayer吗？

```
- (1) CAShapeLayer 矢量绘图
- (2) CATextLayer 文本绘制
- (3) CATransformLayer 形变绘制
- (4) CAGradientLayer 渐变绘制
- (5) CAReplicatorLayer 重复多个样式的绘制
- (6) CAScrollLayer 类似ScollView
- (7) CATiledLayer 切割大图为n个小图，按需加载
- (8) CAEmitterLayer 不常用
- (9) CAEAGLLayer 不常用
- (10) AVPlayerLayer 播放视频，不算为一种高效layer
```

利用这些专用的CALayer，已经是可以完成大部分的高效绘图代码了，还需要自定义吗？

### CALayer 提供了 `drawsAsynchronously` 这个异步绘制的属性，还需要再写一个异步绘制的CALayer吗？


下面个简单的例子，使用自带的这个异步绘制属性完成简单的CoreGraphics图像绘制

```objc
#import <QuartzCore/QuartzCore.h>
#import <UIKit/UIKit.h>

@interface CALayerSub : CALayer

@end
@implementation CALayerSub

- (void)drawInContext:(CGContextRef)ctx {
    NSLog(@"thread = %@", [NSThread currentThread]);
    
    // 绘制一个图像
    //CGContextDrawImage(ctx, self.bounds, [UIImage imageNamed:@"demo"].CGImage);
    
    // 绘制一个椭圆
    CGContextAddEllipseInRect(ctx, self.bounds);
    CGContextSetFillColorWithColor(ctx, [UIColor orangeColor].CGColor);
    CGContextFillPath(ctx);
    
    NSLog(@"thread = %@", [NSThread currentThread]);
}

@end
```


VC里面测试

```objc
#import "ViewController.h"
#import "CALayerSub.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)test1 {
    CALayerSub *layer = [CALayerSub layer];
    layer.drawsAsynchronously = YES;
    layer.frame = CGRectMake(50, 100, 200, 100);
    [self.view.layer addSublayer:layer];
    [layer setNeedsDisplay];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self test1];
}

@end
```

打印输出

```
2017-03-09 18:21:48.673 XZHAsyncLayerDemo[21807:1380124] thread = <NSThread: 0x60000006acc0>{number = 1, name = main}
2017-03-09 18:21:48.674 XZHAsyncLayerDemo[21807:1380124] thread = <NSThread: 0x60000006acc0>{number = 1, name = main}
```

发现仍然是在`主线程`，那不是然并卵。那就奇怪了，不是说异步子线程的吗？

最后在一篇国外技术文章中解释是，`drawInContext:`仍然确实还是在主线程执行，但是最终的CoreGraphics等绘制代码，是在子线程完成的。

即使CoreGraphics等绘制代码，是在子线程完成，但是在绘制之前的一些代码仍然是在`主线程`。比如:

- (1) 图片文件、文本文件等读取
- (2) 图片的解压缩
- (3) 以及各种绘制涉及的辅助对象创建废弃

仍然都是在主线程完成。也就是说异步的不够彻底，这个就是系统CALayer的一个不足之处。

### 尝试在子线程上完成创建CALayer对象、设置CALayer对象，最终回到主线程添加CALayer到VC.view.layer

```objc
@implementation ViewController

- (void)test1 {
    CALayerSub *layer = [CALayerSub layer];
//    layer.drawsAsynchronously = YES;
    layer.frame = CGRectMake(50, 100, 200, 100);
    layer.borderWidth = 1;
    layer.contents = (__bridge id)([UIImage imageNamed:@"demo"].CGImage);
    
    dispatch_async(dispatch_get_main_queue(), ^{
        [self.view.layer addSublayer:layer];
    });
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
          [self test1];
    });
}

@end
```

运行后，没有崩溃，只是图片显示的稍微慢一点，其原因是没等到子线程没有对CALayer内部数据进行渲染完毕，立马就回到了主线程添加显示了CALayer。

我修改为尝试让子线程渲染完毕之后，再回到主线程添加CALayer:

```objc
@interface ViewController () {
@property (weak, nonatomic) IBOutlet UIImageView *imageview;
@end
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    self.imageview.image = [UIImage imageNamed:@"demo"];
}

- (void)testScreenShotAsync {
    CALayer *layer = self.imageview.layer;
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        UIImage *image = [layer xzh_screenShot];//网上找到一个CALayer数据渲染的代码
        CALayer *bottomLayer = [CALayer layer];
        bottomLayer.contentsScale = [UIScreen mainScreen].scale;
        bottomLayer.frame = CGRectMake(50, 200, 200, 150);
        bottomLayer.borderWidth = 1;
        bottomLayer.borderColor = [UIColor redColor].CGColor;
        bottomLayer.contents = (__bridge id)(image.CGImage);
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.view.layer addSublayer:bottomLayer];
        });
    });
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
      [self testScreenShotAsync];
}

@end
```

下面是运行后的效果

![demo.gif](http://upload-images.jianshu.io/upload_images/1829660-a96ce12a20280e2c.gif?imageMogr2/auto-orient/strip)

那么对于之前的例子，没有等待子线程将图片渲染完毕，就立刻回到了主线程时，我觉得可能会把图片的渲染过程带回到`主线程`继续完成。

因为后面，压根就没有操作`layer`这个局部对象的代码了，后续的图片渲染，肯定是在主线程去完成。

这两个例子的代码，比较起来，很显然是后面的这一个更好。因为图片的渲染都是在子线程全部处理完毕，最后只是回到主线程进行显示，对主线程节约了很多的时间。

还可以做的更变态，连`图片`的`解压缩`、`解码`、`圆角化、阴影、遮罩...`都放在子线程去完成，然后将其在内存中缓存起来。想想如果这么做之后，主线程会省去多少事情...

最后，我看了下CALayer的头文件，发现CALayer（其他的专用Layer）的所有的属性修饰符，基本上没有带`nonatomic`，那就是都是使用`atomic`。那这样意味着使用原子属性默认进行多线程访问排队，那应该是可以在多线程环境任意使用的。只是最终操作UIView对象时，必须回到主线程。

我估计应该是可以在一个子线程上，去单独操作某一个CALayer对象的，其正确性还有待验证。

### CALayer的 setNeedsDisplay 、display

```objc
- (void)test1 {
    CALayerSub *layer = [CALayerSub layer];
//    layer.drawsAsynchronously = YES; 注释不注释不影响
    
    layer.frame = CGRectMake(50, 100, 200, 100);
    [self.view.layer addSublayer:layer];
    
    
    for (int i = 0; i < 10; i++) {
        layer.backgroundColor = [UIColor randomColor].CGColor;
//        [layer display]; 会强制重绘10次
        [layer setNeedsDisplay]; // 只会进行最后的一次重绘
    }
}
```

如果是 `[layer setNeedsDisplay]`的输出

```
2017-03-09 19:16:35.469 XZHAsyncLayerDemo[22421:1428266] thread = <NSThread: 0x6080000680c0>{number = 1, name = main}
```

如果是`[layer display]`的输出

```
2017-03-09 19:17:28.803 XZHAsyncLayerDemo[22472:1429927] thread = <NSThread: 0x60000006a3c0>{number = 1, name = main}
2017-03-09 19:17:28.803 XZHAsyncLayerDemo[22472:1429927] thread = <NSThread: 0x60000006a3c0>{number = 1, name = main}
2017-03-09 19:17:28.803 XZHAsyncLayerDemo[22472:1429927] thread = <NSThread: 0x60000006a3c0>{number = 1, name = main}
2017-03-09 19:17:28.804 XZHAsyncLayerDemo[22472:1429927] thread = <NSThread: 0x60000006a3c0>{number = 1, name = main}
2017-03-09 19:17:28.804 XZHAsyncLayerDemo[22472:1429927] thread = <NSThread: 0x60000006a3c0>{number = 1, name = main}
2017-03-09 19:17:28.804 XZHAsyncLayerDemo[22472:1429927] thread = <NSThread: 0x60000006a3c0>{number = 1, name = main}
2017-03-09 19:17:28.804 XZHAsyncLayerDemo[22472:1429927] thread = <NSThread: 0x60000006a3c0>{number = 1, name = main}
2017-03-09 19:17:28.804 XZHAsyncLayerDemo[22472:1429927] thread = <NSThread: 0x60000006a3c0>{number = 1, name = main}
2017-03-09 19:17:28.805 XZHAsyncLayerDemo[22472:1429927] thread = <NSThread: 0x60000006a3c0>{number = 1, name = main}
2017-03-09 19:17:28.805 XZHAsyncLayerDemo[22472:1429927] thread = <NSThread: 0x60000006a3c0>{number = 1, name = main}
```

区别就是：`每一次都会重绘` 和 `只会绘制最终的一次数据`。

说明，`setNeedsDisplay`，会将前一次开始或准备开始的绘制操作，临时停止掉结束绘制，而只对最后一次进行绘制。

这个是YYAsyncLayer模拟实现的一个点，用于`频繁很多很多次`的执行重新绘时的优化，即只对`最后一次设置的数据`进行绘制。

并且YYAsyncLayer完全异步子线程，直接将渲染得到的Image塞给CALayer的contents属性，完全可以绕过UIView对象，这个可能是相比系统CALayer更高的一个优化点把。

### 为何YYAsyncLayer继承自CALayer实现，而不是继承自专用的CALayer做实现？

YYAsyncLayer继承自CALayer实现，但并没有继承于任何一种专用Layer，我想可能是一旦继承于某一种专用的Layer就只能干这一类的事了，所以继承自CALayer做一个通用性的异步子线程CALayer。

YYAsyncLayer在执行内部绘制时，只是创建了一个Context，然后回传出去给外部进行绘制。那也就是说，外部在接收的Context中，可以任意的进行绘制，比如：文本、图片、自定义图形、路径...。完全可以不是要其他的各种UIView对象，手工的进行分区域的绘制。

最终，YYAsyncLayer将外界对Context中进行的所有的绘制，都在`子线程`直接渲染成为一个`CGImageRef`实例，然后直接塞给了`YYAsyncLayer.contents`属性值进行显示。

将一切的文件读取、图片解压缩、图片的解码、各种特效的绘制，全部放到了子线程上进行，这就是比系统CALayer更加高效的地方了。

### YYAsyncLayer并不是作为一个单独使用的CALayer设计的

由于CALayer单独使用是很麻烦的，不具备事件响应，当屏幕旋转时候也无法响应，也没有像UIView那样容易管理层级关系。

所以，YYAsyncLayer设计的目的也并不是作为一个单独使用的Layer，而是作为某一个View（比如，自定义文本绘制的UILabel）的`backing layer`而存在。

而这个`backing layer`不像系统的CALayer将全部操作都是放到`主线程`上，而是将文件读取、绘制、渲染全部放到子线程，来充分释放`主线程`的压力。

所以，最好在外面套一个UIView容器，而内部的所有绘制，直接通过YYAsyncLayer回传出来的Context中进行绘制渲染生成图像。

## YYAsyncLayer源码学习

如其名，就是一个异步xxx的CALayer。从大体简介来说，就是在异步子线程上进行CALayer数据的渲染得到最终塞给CALayer显示的图像。这个可能是这一套代码，最核心的作用了。

比如，如下就是一个最简单的异步子线程绘制渲染得到图像然后直接显示的demo:

```objc
@implementation CoreTextDemoVC {
    UIImageView *_labelImageView;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    [self drawText2_3];
}

- (void)drawText2_3 {
    CGRect rect = CGRectMake(10, 100, 300, 300);
    _labelImageView = [[UIImageView alloc] initWithFrame:rect];
    _labelImageView.layer.borderWidth = 1;
    _labelImageView.contentMode = UIViewContentModeScaleAspectFit;
    _labelImageView.clipsToBounds = YES;
    [self.view addSubview:_labelImageView];
    
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        UIGraphicsBeginImageContextWithOptions(rect.size, NO, 0);
        CGContextRef context = UIGraphicsGetCurrentContext();
        [[UIColor whiteColor] set];
        CGContextFillRect(context, rect);
        CGContextSetTextMatrix(context, CGAffineTransformIdentity);
        CGContextTranslateCTM(context, 0, rect.size.height);
        CGContextScaleCTM(context, 1.0, -1.0);
        
        NSMutableAttributedString *attrString = [[NSMutableAttributedString alloc] initWithString:@"iOS程序在启动时会创建一个主线程，而在一个线程只能执行一件事情，如果在主线程执行某些耗时操作，例如加载网络图片，下载资源文件等会阻塞主线程（导致界面卡死，无法交互），所以就需要使用多线程技术来避免这类情况。iOS中有三种多线程技术 NSThread，NSOperation，GCD，这三种技术是随着IOS发展引入的，抽象层次由低到高，使用也越来越简单。"];
        CTFramesetterRef frameSetter = CTFramesetterCreateWithAttributedString((CFAttributedStringRef)attrString);
        CGMutablePathRef path = CGPathCreateMutable();
        CGPathAddEllipseInRect(path, NULL, CGRectMake(0, 0, rect.size.width, rect.size.height));
        [[UIColor redColor]set];
        CGContextFillEllipseInRect(context, CGRectMake(0, 0, rect.size.width, rect.size.height));
        CTFrameRef frame = CTFramesetterCreateFrame(frameSetter, CFRangeMake(0, [attrString length]), path, NULL);
        CTFrameDraw(frame, context);
        CFRelease(frame);
        CFRelease(path);
        CFRelease(frameSetter);
CGContextSetTextMatrix(context,CGAffineTransformIdentity);
        CGContextTranslateCTM(context, 0, rect.size.height);
        CGContextScaleCTM(context, 1.0, -1.0);
        UIImage *img = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        
        dispatch_async(dispatch_get_main_queue(), ^{
            _labelImageView.image = img;
        });
    });
}

@end
```

如上就是简单的绘制几行文字而已，但是是进行基本的性能的思路之一。

### YYAsyncLayer源码学习我就不贴了，说下整个工作流畅，想学习的对着去看吧

```
1. UIView 的 backing layer 设置为 YYAsyncLayer
2. 修改UIView对象的text、textColor、textSize...等等，需要对内容进行重新绘制
3. [UIView setNeedsDisplay] or [UIView对象.layer setNeedsDisplay] 触发某一个属性值修改后的重绘
4. 将 {UI对象, text, setNeedsDisplay}、{UI对象, textColor, setNeedsDisplay}、{UI对象, textSize, setNeedsDisplay} 分别打包成Transaction对象
5. -[Transaction commit]
6. 将当前执行commit的Transaction对象，使用一个暂时的Set容器保存
7. 等到runloop即将休息的时候，在将当前Set容器保存的所有的Transaction对象，提交给runloop暂存，runloop进入休眠
8. runloop在下一个轮回唤醒时，执行上一次设置的所有的Transaction对象的操作
9. 被遍历的某一个Transaction对象的制定的SEL的消息被发送，一般是执行 [CALayer setNeedsDisplay]
10. -[YYAsyncLayer setNeedsDisplay] 被调用
11. -[YYAsyncLayer display] 被调用
12. -[YYAsyncLayer _displayAsync:是否异步绘制] 被调用
13. 给当前重绘任务创建一个新的Task：YYAsyncLayerDisplayTask *task = [YYAsyncLayer对象.delegate newAsyncDisplayTask]; 
14. task.willDisplay(); 告诉外界即将开始绘制
15. 构造传递给外界用来判断是否结束取消此次绘制的 isCancelled block，捕获局部counter值和counter对象
16. 开始异步子线程
17. 创建CGContext画布、以及初始化
18. if (task.didDisplay) task.didDisplay(self, 当前绘制是否结束);
19. 完成绘制，UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
20. YYAsyncLayer对象.contents = (__bridge id)(image.CGImage);
```

从中还有一个受益匪浅的东西，就是`YYTransaction`这个类。是YYKit作者参看了ASDK源码，然后从中剥离出来的。我稍微了解了下，是模拟CoreAnimation进行界面重绘的场景。

- (1) 每一个重绘的操作，都会由CoreAnimation打包成Transaction对象，并且执行commit
- (2) 而执行完commit的Transaction对象，并不会立刻开始重绘，而是添加到一个临时的内存缓存中存储
- (3) 等到主线程RunLoop即将休息的时候，再将缓存中已经commit的Transaction对象的重绘操作，全部注册到RunLoop。只是注册到RunLoop，并没有立刻由RunLoop处理。因为，此时RunLoop已经快休息了。
- (4) RunLoop休息完毕之后，开始一个新的轮回时，就会去处理之前注册的所有的Transaction对象，通知进行重绘
- (5) 最终，就回去执行UIView或CALayer的setNeedsDisplay了

有一个`isCancelld`block，让外界得知当前的绘制操作，是否已经被YYAsyncLayer内部取消掉了。当执行了`-[UIView setNeedsDisplay] 或 -[CALayer setNeedsDisplay]`时，就会开启一个`新`的绘制任务。而YYAsyncLayer内部模拟了系统CALayer，会`立刻结束掉当前`正在执行或即将执行的`绘制任务`，从而直接进行`最后一次`的绘制任务。

可以回调执行`YES == isCancelld();`就表示，当前绘制任务已经被`取消`掉了，也就不会去绘制。比如，快速滚过的cell，又会拿来被其他的NSIndexPath重用，也就是被重绘，即调用`setNeedsDisplay`发起一个新的重绘任务，自然就不会执行快速滚过的NSIndexPath对应的数据内容的绘制。

通过这个途径，也解决了之前VVeboTableView快速滚动对于一些不可见的cell的绘制的问题，同时也解决了VVeboTableView会出现`空白`的问题。

因为`VVeboTableView`是当快速滚动`停止`之后，才会去对`visble cells`进行绘制。而YYAsyncLayer，只要调用了`setNeedsDisplay`，就会提交一个`YYTransaction`对象，包含了重绘的操作。只需要等待MainRunLoop下一轮被唤醒就会去执行重绘。而一旦cell快速滚动到不可见看了，需要被拿来重用的时候，就又会调用`setNeedsDisplay`，就会将之前的一次绘制取消掉。

当前进行绘制时，其判断当前是否有最新的绘制，即判断是否又执行了`-[UIView setNeedsDisplay] 或 -[CALayer setNeedsDisplay]`。巧妙的利用了计数器的自增，以及block捕获拷贝值的办法，真的不得不得佩服作者的脑袋...

简单的列举下YYAsyncLayer具体进行绘制的逻辑:

- (1) 获取当前YYAsyncLayer要完成的绘制任务Task对象
- (2) 然后区分绘制任务Task，是同步进行还会异步进行
- (3) 通过计数器值对比，判断是否开启了新的绘制任务
- (4) 如果绘制区域的宽度或高度，任意一个小于1，则直接结束绘制，并且释放掉contents
- (5) 异步到子线程，开始创建context上下文
- (6) 回传创建初始化完毕的context，让外界在context上进行随意的绘制
	- 绘制一段文本
	- 绘制一些图片
	- 或者图文的混排
- (7) 绘制完毕之后，通知context在子线程上完成渲染
- (8) 获取context渲染完毕的image，然后塞给`layer.contents`显示
- (9) 期间几处判断外界是否执行了`-[CALayer setNeedsDisplay]`，开启新的绘制任务。如果有，则是结束掉当前的绘制操作，回调给外界，并释放掉绘制过程中产生的各种对象

基于此，我想这也就是为何YYAsyncLayer继承自CALayer来实现，而不是继承自其他专用的CALayer子类去实现的原因了，是想做成一个可以进行任意内容绘制的CALayer。

YYAsyncLayer只是对单个CALayer的异步绘制渲染，加入有很多个叠起来的CALayer了？我从文章中看到了一种这样的思路，也是ASDK进行优化的办法:

- (1) 对单个CALayer的异步子线程进行绘制渲染得到图像
- (2) 将多个层叠的CALayer渲染得到的图像，再进行图像的合成
- (3) 将最后合成的图像，再塞给最终外面的容器View.layer或容器layer进行显示

恩，待消化完YYAsyncLayer后，再去看看ASDK这座大山...