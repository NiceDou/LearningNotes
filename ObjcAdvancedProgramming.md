
## 最好不要在load方法中写过多耗时的代码

因为App启动时，会等待所有objc类的load方法实现全部执行完，才会走后面的代码逻辑。

## objc中的几种for循环性能比较

http://www.open-open.com/lib/view/open1488680807266.html

主要有如下几种for循环

```c
for (int i = 0; i < 100; i++) {...}
```

```c
for (id obj in objs) {....}
```

```c
[array enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
	....
}];
```

```c
// 通过block回调，在子线程中遍历，对象的回调次序是乱序的,而且调用线程会等待该遍历过程完成
[array enumerateObjectsWithOptions:NSEnumerationConcurrent usingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
	....
}];
```

可以得到如下结论:

- (1) 通常情况下，`for-in`是最快的，因为使用了快速枚举

- (2) `NSEnumerationConcurrent+Block`形式的遍历，会将每一次的遍历任务，都会分配到一个线程上单独执行，只适用于单次遍历任务比较耗时的代码，如果对于普通的轻量级遍历，则反而会显得效率很低

## 主线程上常见比较耗时的代码类型:

- (1) 各种NSObject对象的创建与废弃

- (2) UIKit Obejcts UI对象
	- UI对象的属性值调整
	- UI对象的创建
	- UI对象的销毁（废弃）

- (3) Layout 布局计算
	- 计算文本内容的 宽度计算、高度计算
	- frmae计算、frmae设置、frmae调整

- (4) Rendering 显示数据渲染
	- 文本内容的渲染
	- 图片的解码
	- 图形的绘制

尽量的将如上步骤，全部放到子线程异步执行。

## 创建CALayer的contents的三个途径

```
1. 在创建CALayer时，就设置一个图片显示
2. 实现CALayerDeleate中的绘制函数
3. 重写CALayer子类的绘制函数
```

### 一、直接给UIView或CALayer的contents设置一个显示的图像

```objc
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //1. 设置UIView的背景色
    UIView *view1 = [[UIView alloc] initWithFrame:CGRectMake(10, 10, 300, 200)];
    view1.backgroundColor = [UIColor blackColor];
    [self.view addSubview:view1];
    NSLog(@"view1.layer.contents = %@", view1.layer.contents);
    
    //2. 设置UIView的文本内容
    UILabel *view2 = [[UILabel alloc] initWithFrame:CGRectMake(10, 250, 300, 40)];
    view2.text = @"我是你妈妈咪";
    view2.textColor = [UIColor redColor];
    [self.view addSubview:view2];
    NSLog(@"view2.layer.contents = %@", view2.layer.contents);
    
    //3. 设置UIView的背景图片
    UIImageView *view3 = [[UIImageView alloc] initWithFrame:CGRectMake(10, 300, 300, 200)];
    view3.image = [UIImage imageNamed:@"demo"];
    [self.view addSubview:view3];
    NSLog(@"view3.layer.contents = %@", view3.layer.contents);
}

@end
```

程序运行起来之后，打印分别如下

```
2017-03-25 11:34:19.013 Demo[1215:13225] view1.layer.contents = (null)

2017-03-25 11:34:19.028 Demo[1215:13225] view2.layer.contents = (null)

2017-03-25 11:34:19.034 Demo[1215:13225] view3.layer.contents = <CGImage 0x6000001dc4d0>
	<<CGColorSpace 0x600000037bc0> (kCGColorSpaceICCBased; kCGColorSpaceModelRGB; sRGB IEC61966-2.1)>
		width = 544, height = 184, bpc = 8, bpp = 32, row bytes = 2176 
		kCGImageAlphaPremultipliedLast | 0 (default byte order) 
		is mask? No, has mask? No, has matte? No, should interpolate? Yes
```

前面两种情况时CALayer的contents还不存在，但是设置了图片之后，contents就存在了，且存储的数据类型是`CGImage`。

### 二、在CALayerDelegate回调函数中创建contents

```objc
@implementation ViewController {
	UIView *view1;
}

@implementation BasicViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    [self test2];
}

- (void)test2 {
    
    //1.
    view1 = [[UIView alloc] initWithFrame:CGRectMake(10, 10, 300, 200)];
    [self.view addSubview:view1];
    
    //2.
    view1.layer.delegate = self;
    
    //3. 调用setNeedsDisplay，无法创建contents
//    [view1.layer setNeedsDisplay];
    
    //4. 必须强制调用layer的display，才能创建contents
    [view1.layer display];
    
    //5.
    NSLog(@"log1 >>>> view1.layer.contents = %@", view1.layer.contents);
}

- (void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx {
    
    // 即使不是有如下绘制的代码，contents其实在调用这个方法之前，已经创建完毕
//    CGContextSetLineWidth(ctx, 10.0f);
//    CGContextSetStrokeColorWithColor(ctx, [UIColor redColor].CGColor);
//    CGContextStrokeEllipseInRect(ctx, layer.bounds);
    
    NSLog(@"log2 >>>> view1.layer.contents = %@", view1.layer.contents);
}

@end
```

输出如下

```
2017-03-26 20:06:44.112 AnimationsDemo[1559:17003] log2 >>>> view1.layer.contents = (null)
2017-03-26 20:06:44.113 AnimationsDemo[1559:17003] log1 >>>> view1.layer.contents = <CABackingStore 0x7fb9925c9cd0 (buffer [300 200] BGRA8888)>
```

在调用`-[CALayer drawLayer:inContext:]`之前，CALayer的contents就有数据了。也就是说，我们在`drawLayer:inContext:`什么都不做，contents其实早已经创建好了。

这个情况下的值类型是`CABackingStore`，而上面通过设置图片之后的contents类型是`CGImage`。

注意，一定要使用`-[CALayer display]`才能完成创建CALayer的contents。

### 三、重写CALayer的draw方法创建contents

自定义CALayer子类

```objc
@interface MyLayer : CALayer

@end
@implementation MyLayer

//实现方式一、一个空实现
- (void)drawInContext:(CGContextRef)ctx {
	NSLog(@"log2 >>>> view1.layer.contents = %@", self.contents);
}


/** 
实现方式二、具体做一些绘制
- (void)drawInContext:(CGContextRef)ctx {
    
    //1.
    CGContextSetLineWidth(ctx, 10.0f);
    CGContextSetStrokeColorWithColor(ctx, [UIColor redColor].CGColor);
    CGContextStrokeEllipseInRect(ctx, self.bounds);
    
    //2.
    NSLog(@"log2 >>>> layer1.contents = %@", self.contents);
}
*/

@end
```

```objc
@implementation BasicViewController

- (void)viewDidLoad {
    [super viewDidLoad];
	[self test3];
}

- (void)test3 {

    //1.
    view1 = [[UIView alloc] initWithFrame:CGRectMake(10, 100, 200, 100)];
    [self.view addSubview:view1];
    
    //2.
    layer1 = [MyLayer layer];
    layer1.frame = view1.bounds;
    layer1.backgroundColor = [UIColor blueColor].CGColor;
    [view1.layer addSublayer:layer1];
    
    //3. 必须如下的display，强制进行绘图
//    [layer1 setNeedsDisplay];
    
    //4.
    [layer1 display];
    
    //5.
    NSLog(@"log1 >>>> layer1.contents = %@", layer1.contents);
}

@end
```

输出如下

```
2017-03-26 20:26:50.474 AnimationsDemo[1738:30404] log2 >>>> view1.layer.contents = (null)
2017-03-26 20:26:50.475 AnimationsDemo[1738:30404] log1 >>>> layer1.contents = <CABackingStore 0x7fbba364d260 (buffer [200 100] BGRA8888)>
```

只要实现了`-[CALayer drawInContext:]`，不管有没有进行绘制，都会直接创建CALayer的contents。

### 小结如上三种创建contents的方法

总而言之，在执行了`-[CALayer display]`之后，通过如上三种途径，CALayer的contents就已经存在数据了。只是存在的数据格式不同:

- (1) CGImage
- (2) CABackingStore

后期对CALayer做的各种CoreAnimation动画，其实只是根据CALayer对象缓存的contents进行临时计算绘制，并显示到屏幕上，但其实CALayer的属性值和contents并没有真正的改变。

### 对于如上直接创建CALayer的contents的优化

- (1) 使用 **专用图层** 进行绘制，不要使用如上方法二、方法三，可延迟创建contents

- (2) 提前在子线程将绘制的内容，渲染成为bitmap，然后塞给contents保存

## UIView显示到屏幕上的过程

http://wiki.jikexueyuan.com/project/ios-core-animation/performance-tuning.html

### 大致分为三部分:

- (1) CPU的处理
- (2) GPU的处理
- (3) 绘图硬件的绘制和渲染、最终显示器数据显示

### 一、CPU处理部分

- (1) UIView对象的创建、废弃

- (2) 读取文本数据、图片数据。如果是对于图片，还需要完成:
	- (2.1) 读取图片时，触发图片的压缩文件的`解压缩`，解压后的文件会`比较大`
	- (2.2) 当需要绘制图片时，还需要对压缩图片进行`解码`

- (3) 各种数据计算，然后设置给UIView对象，底层再同步设置给内部的CALayer
	- (3.1) 文本尺寸计算
	- (3.2) 图像尺寸计算
	- (3.3) frame计算
	- (3.4) 背景色、字体、边框、阴影...
	- (3.5) 所有的设置，同步给UIView对象内部的CALayer对象

- (4) CALyer显示 即 `[CALayer display]` 被调用，此时会完成CALayer的contents创建。包括上面提到的三个途径:
	- (4.1) `-[CALayer display] 或 -[CALayer setNeedsDisplay]`方法调用
	- (4.2) 继而调用 `-[CALayer drawInContext:]` 或 `CALayerDelegate`、`重写CALayer drawInContext` 
	- (4.3) 到此为止，CALayer的`contents`已经创建完毕
	- (4.4) 如果预先设置了一个图片给UIView或CALayer，那么此时contents的类型是`CGImage`
		- (4.4.1) 这种情况是在`主线程`完成图片的解压、解码、绘制、渲染
		- (4.4.2) 最后是在`子线程`完成图片的解压、解压、绘制、渲染，得到bitmap，然后设置给CALayer的contents
	- (4.5) 如果没有设置图片，contents类型是`CABackingStore`
	- (4.6) 总之对于 CALayer 初始化时 显示的数据，已经渲染完毕

- (5) `后续` 再对CALayer属性值做出的每一个修改，都打包为`CATransaction`，并临时提交到一个**全局缓存容器**
	- (5.1) 系应该 CALayer 的属性值，然后调用 `setNeedsDisplay`，将此 CALayer 标记为 `待处理`，也就是需要重新进行绘制
	- (5.2) 开始一个事务来包裹CALayer的重绘 `-[CATransaction begin]`
		- (5.2.1) 读取哪一个 CALayer 的 contents
		- (5.2.2) 针对CALayer的哪一个属性值（font、textColor、backgroudColor ... ）进行绘制和渲染
	- (5.3) 如果需要进行图片的绘制，则此时还要进行 `图片的解码`
	- (5.4) 结束并 **提交** 绘制操作的事务  `-[CATransaction commit]`
	- (5.5) 将执行commit消息的 transaction 临时保存到 `全局缓存容器`

- (6) 当 **我们的程序进程中的主线程 RunLoop** 处于 `即将退出、即将休息` 状态回调时，从 **全局缓存容器** 取出全部 `CATransaction` 绘制操作，发送给 **RenderServer单独进程** 进行绘制和渲染
	- (6.1) 将所有的 transaction 发送给 `RenderServer` 去处理
	- (6.2) 发送完毕之后，`RunLoop 就休息` 了
	- (6.3) RenderServer 取出每一个 transaction 进行如下处理
	- (6.4) 取出 CALayer 的 contents
	- (6.5) 取出被修改的 CALayer 属性值
	- (6.6) 根据 `CALayer对象当前的属性值` ，对 contents 做一些图像的合成处理等
	- (6.7) 将处理后的数据 **再次** 显示到屏幕上
	- (6.8) 通过之前添加的 `CFRunLoopSource 1` 主动回调唤醒 RunLoop ，告知已经渲染完毕，可以继续进行其他的绘制操作了
	- (6.9) 我们的App程序进程的主线程RunLoop接收到基于`mach_port`通知被唤醒，继续分配其他的绘制

- (7) 这里需要注意，几个单独的东西
	- (7.1) 我们的App程序进程 和 渲染服务进程，不是同一个进程
	- (7.2) 上面说的 RunLoop 是属于 我们App程序进程中的 主线程的 RunLoop
	- (7.3) 渲染服务进程 完成任务后，通知 我们的App程序进程中主线程的 RunLoop


### 二、GPU处理部分	

- (1) 取出一个个`CATransaction`中的哪一个CALayer、哪一个属性值，进行屏幕绘制

- (2) 如果添加了`CAAnimation`动画，则根据动画执行过程，使用`OpenGL`进行计算、进行渲染成为一帧一帧的`bitmap位图`

- (3) 对设置的 图片、文本、绘制的图形 渲染成为`bitmap位图`

- (4) 对多个`bitmap位图`进行混合处理

- (5) 并对多个层级关系的CALayer各自渲染得到的`bitmap位图`，进行混合处理

- (6) 将最终混合完毕得到的`bitmap位图`，继续渲染为显示器硬件能识别的`纹理`格式，并存入到帧缓冲池中

- (7) 通知显示器硬件去帧缓冲池，读取镇数据，进行屏幕的绘制

### 三、屏幕显示部分 

- (1) 显示器从镇缓存池中，取出当前帧要显示的帧数据
- (2) 将帧数据显示到显示器硬件上

### 能够做的优化

- (1) 将第一部分，除开UIKit对象之外，全部放到子线程异步完成，并且能够缓存就缓存

- (2) 尽量将第二部分的CALayer数据的渲染，提前在子线程异步完成

- (3) 本地高清大图切割加载

- (4) 网络web高清大图分多规格多次拉取，并结合ImageIo渐进式图像加载

## UI性能、使用 `[CALayer renderInContext:]` 将CALayer在子线程完成渲染并得到Image，然后直接塞给`CALayer.contents`显示

eg、把整个屏幕转化为图片

```objc
//1. 
UIImageView* imageV = [[UIImageView alloc]initWithFrame:CGRectMake(0, 0, self.view.frame.size.width, self.view.frame.size.height)];

//2.
UIGraphicsBeginImageContextWithOptions(imageV.frame.size, NO, 0);

//3.
CGContextRef context = UIGraphicsGetCurrentContext();

//4. 把当前的整个画面导入到context中，然后通过context输出UIImage，这样就可以把整个屏幕转化为图片
[self.view.layer renderInContext:context];

//5.
UIImage* image = UIGraphicsGetImageFromCurrentImageContext();

//6.
imageV.image = image;

//7. 
UIGraphicsEndImageContext();
```

可以将上述的代码，全部放到子线程进行优化。

## UI性能、异步子线程进行图像的：解压缩、解码、渲染、圆角 等处理

### 通常读取PNG/JPEG等文件都是压缩图片文件，需要首先进行解压缩，然后进行解码，最终才能作为绘制渲染的图像

- (1) 解码后的图像会比较`大`，通常不会缓存到磁盘文件，只会内存中缓存

- (2) SDWebImage的做法是把`解码`操作从主线程移到`子线程`，让耗时的解码操作不占用主线程的时间


### 触发图片解码的情况

- (1) `+[UIImage imageNamed:]`  加载到原图后立刻进行解码，仍然会保留原始图像

- (2) `+[UIImage imageWithContentsOfFile:]`  图像渲染前进行解码，不会保存原始图像

- (3) 即将对图像进行绘制，立刻也会触发解码

对解码的优化，就是尽量的在子线程完成。并且可以是ImageIO来进行图像解码的优化：

- (1) 开启一个子线程来完成下面所有的任务

- (2) 指定`kCGImageSourceShouldCacheImmediately`参数来读取压缩图片

- (3) `kCGImageSourceShouldCacheImmediately` 决定是否会在加载完后立刻开始解码，并且还会将解码后的图像缓存起来


### 在子线程上，进行图像绘制的时刻，触发图像的解码

```objc
@implementation UIImageHelper

- (void)decompressImageNamed1:(NSString *)name ofType:(NSString *)type completion:(void (^)(UIImage *image))block {
    
//    XZHDispatchQueueAsyncBlockWithQOSBackgroud(^{
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        
        //1. 仅仅只是读取和解压缩图像文件，并没有进行图像的【解码】
        NSString *filepath = [[NSBundle mainBundle] pathForResource:name ofType:type];
        UIImage *image = [UIImage imageWithContentsOfFile:filepath];
        
        //2. 创建绘图上下文
        UIGraphicsBeginImageContextWithOptions(image.size, NO, [UIScreen mainScreen].scale);
        CGContextRef context = UIGraphicsGetCurrentContext();
        
        //3. 绘图区域
        CGRect rect = CGRectMake(0, 0, image.size.width, image.size.height);
        
        //4. 圆角路径切割画布成为圆角的区域画布
        CGFloat cornerWidth = image.size.width * 0.3;
        CGFloat cornerHeight = image.size.height * 0.3;
        CGMutablePathRef path = CGPathCreateMutable();
        CGPathAddRoundedRect(path, NULL, rect, cornerWidth, cornerHeight);
        CGContextAddPath(context, path);
        CGContextClip(context);
        
        //5. 【重要】当绘制图像时，一定会触发图像的【解码】
        [image drawInRect:rect];
        
        //6.
        UIImage *decompressImage = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        
        //7.
        dispatch_async(dispatch_get_main_queue(), ^() {
            block(decompressImage);
        });
    });
}

@end
```

### 在子线程上，使用ImageIO读取图片文件，并使用`kCGImageSourceShouldCacheImmediately`设置项，来让读取图片解压缩之后，并立刻开始图像的`解码`，并由ImageIO自己将解码后的图像在内存中缓存起来

参考自AFImageDownloader的部分代码


```objc
@implementation UIImageHelper

- (void)decompressImageNamed2:(NSString *)name ofType:(NSString *)type completion:(void (^)(UIImage *image))block {
    XZHDispatchQueueAsyncBlockWithQOSBackgroud(^{
    
        //1. 设置使用ImageIo读取图片文件时，直接解压缩、解码图像文件，
        NSDictionary *dict = @{
                               (id)kCGImageSourceShouldCache : @(YES)
                               };
		
		//2. 使用ImageIO读取文件
        NSString *filepath = [[NSBundle mainBundle] pathForResource:name ofType:type];
        NSURL *url = [NSURL fileURLWithPath:filepath];
        CGImageSourceRef source = CGImageSourceCreateWithURL((CFURLRef)url, NULL);
        CGImageRef cgImage = CGImageSourceCreateImageAtIndex(source, 0, (CFDictionaryRef)dict);
        
        //3. Create a bitmap context of a suitable size to draw to, forcing decode
        size_t width = CGImageGetWidth(cgImage);//图片的真实宽度
        size_t height = CGImageGetHeight(cgImage);//图片的真实高度
        size_t bytesPerRow = roundUp(width * 4, 16);
        size_t byteCount = roundUp(height * bytesPerRow, 16);
        if (width == 0 || height == 0) {
            CGImageRelease(cgImage);
            dispatch_async(dispatch_get_main_queue(), ^{
                block(nil);
            });
            return;
        }
        
        //4. Create the colour space and an image buffer
        void *imageBuffer = malloc(byteCount);
        CGColorSpaceRef colourSpace = CGColorSpaceCreateDeviceRGB();
        
        //5. Create the image context and release the colour space.
        CGContextRef imageContext = CGBitmapContextCreate(imageBuffer, width, height, 8, bytesPerRow, colourSpace, kCGImageAlphaPremultipliedLast);
        CGColorSpaceRelease(colourSpace);
        
        //6. Draw the image to the context and release it.
        CGContextDrawImage(imageContext, CGRectMake(0, 0, width, height), cgImage);
        CGImageRelease(cgImage);
        
        //7. Now get an image ref from the context.
        CGImageRef outputImage = CGBitmapContextCreateImage(imageContext);
        
        //8. Clean up memory allocated by the colour space and image buffer.
        CGContextRelease(imageContext);
        free(imageBuffer);
        
        //9. callback outputImage converted UIImage
        dispatch_async(dispatch_get_main_queue(), ^{
            UIImage *image = [UIImage imageWithCGImage:outputImage];
            block(image);
            
            // Release the output image after the callback has been completed.
            CGImageRelease(outputImage);
        });
    });
}

@end
```

### 总结在子线程提前对一些PNG、JPEG等压缩格式的图文文件进行渲染时几个步骤:

- (1) 先让流程在一个后台子线程上完成
- (2) 使用`ImageIO`读取压缩格式图片文件，进行`解压缩`，并`缓存`起来
- (3) 开起一个绘图上下文
- (4) 圆角`CGPathRef`创建，clip剪裁，添加到绘图上下文
- (5) 将解压缩后的图像，在绘图上下文中进行绘制
- (6) 从绘图上下文中，获得最终渲染完毕的`CGImageRef`实例
- (7) 再回到主线程
- (8) 将`CGImageRef`实例，设置给`UIView.layer.contents`属性值进行显示

还可以更近异步，将处理后最终的图像，在内存中缓存起来。

## UI性能、网络超大高清图像的加载优化

### 方案一、分多次从web拉取不同规格的图像进行覆盖显示

- (1) 依次从web上加载不同尺寸的图片，从小到大
- (2) 最开始先拉取一个小尺寸的缩略图做拉伸显示
- (3) 然后拉取中等规格的图，拉取完毕直接覆盖显示
- (4) 最后拉取原图，拉取完成后显示原图

缺点、需要分很多次的网络请求。

### 方案二、直接从web拉取原始图像，以渐进式的方式一点一点的加载

- (1) 直接从web拉取原始的比较大、高清的图像，一次性肯定拉取不完
- (2) 没当接收到web传递过来的一部分图像数据，就显示一部分
- (3) 一个很大、很高清的图像文件，就是一点一点的加载出来

缺点、只有一次请求，但是需要等待很长时间，才能完整的加载出一个图像。

### 方法三、结合使用方案一与方案二

- (1) 先拉取一个小尺寸缩略图做拉伸显示（第一次请求）
- (2) 然后采用方案二，直接拉取原始大图、高清图（第二次请求），并使用渐进式的方式加载部分图像


这种方案结合了前面两种方案各自的优势，既减少了方案一的请求数，又利用了方案二的渐进式图像文件数据的加载。

渐进式的图像文件数据加载的主要步骤:

- (1) 创建一个空的渐进式ImageSource >>> `CGImageSourceCreateIncremental(NULL);`
- (2) 不断的拼接部分图片数据NSData
- (3) 使用当前得到的部分数据NSData，更新渐进式ImageSource
- (4) 从渐进式ImageSource中获取渲染得到CGImageRef，即可用于显示
- (5) 加载完毕，释放废弃掉渐进式ImageSource

### 关于大图的加载，可以使用CATiledLayer图层进行切割分区域显示

..

## YYMemoryCache中在子线程释放废弃对象的`三部曲`

```objc
- (void)removeAll {
    
    //1. 基本变量值清零
    _totalCost = 0;
    _totalCount = 0;
    
    //2. objc单个对象释放与废弃
    _head = nil;
    _tail = nil;
    
    //3. objc Array/Set/Dic 容器类型对象的释放与废弃，固定的三步曲
    if (CFDictionaryGetCount(_nodeMap) > 0) {
        
        //3.1 增加一个【局部】指向，dic.retainCount = 2
        CFMutableDictionaryRef holder = _nodeMap;
        
        //3.2 让之前的指针指向新创建的对象，dic.retainCount = 1，即完释放之前的一个持有关系
        _nodeMap = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        
        //3.3 让此时retainCount==1的dic对象的释放废弃流程异步走到子线程中完成
        if (_releaseAsynchronously) {
            
            //3.3.1 【主线程】或【子线程】，【异步】完成对象的废弃
            if (_releaseOnMainThread && !pthread_main_np()) {
                // 【1.主线程】
                dispatch_async(dispatch_get_main_queue(), ^{
                    CFRelease(holder);//最终在主线程， dic.retainCount = 0
                });
            } else {
                // 【2.子线程】
                dispatch_queue_t queue = _releaseOnMainThread ? dispatch_get_main_queue() : XZHMemoryCacheGetReleaseQueue();
                dispatch_async(queue, ^{
                    CFRelease(holder);//最终在子线程，dic.retainCount = 0
                });
            }
        } else {
            //3.3.2 【当前线程】，【同步】完成对象的废弃
            CFRelease(holder);//最终在执行removeAll所在线程，dic.retainCount = 0
        }
    }
}
```

简写为如下:

```objc
//1. 增加一个【局部】指向，dic.retainCount = 2
CFMutableDictionaryRef holder = _nodeMap;

//2. 让之前的指针指向新创建的对象，dic.retainCount = 1，即完释放之前的一个持有关系
_nodeMap = CFDictionaryCreateMutable(CFAllocatorGetDefault(), 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);

//3. 子线程完成局部指针的释放，并让被执行的对象最终在子线程完成废弃
dispatch_async(dispatch_get_global_queue(0,0), ^{
	CFRelease(holder);//最终在子线程，dic.retainCount = 0
});
```

大致的思路就是让一个`子线程`，去持有一个`retainCount==1`的`局部`对象，最终在`子线程`上完成对`局部`对象的release，从而让局部指针指向的对象`废弃`掉。

> 对象的释放 与 对象的废弃，是两个不同的概念，在不同的时间进行处理。

通常是在子线程创建大体积对象，然后回调主线程使用。或者将主线程上不再使用的对象，异步放到子线程完成释放废弃。

- (1) 主线程创建对象 >>> 子线程异步释放与废弃
- (1) 子线程创建对象（读取文件为对象） >>> 主线程使用完毕 >>> 主线程异步释放与废弃

(1)、(2)的做法都是正确的。因为:

```
一个对象 >>> 一个内存块。
一个线程 >>> 一个代码执行路径。
```

那么任意的代码执行路径，都能够访问到某一块内存块，所以任意的线程是能够操作内存中任意的内存块数据。

但是`(2)`不太合常理，会影响主线程的执行效率，所以不推荐。

## KVO的大致实现总结

- (1) 当一个对象的属性添加了属性观察者之后

- (2) 在程序运行时，系统通过`runtime`提供的函数，创建出一个对象所属类的一个`子类`，名字的格式是`NSKVONotifying_原始类名`

- (3) 此时将被添加属性观察的objc类的`对象->isa`指针，指向上面`(2)`创建出来的objc类，就不再指向之前的objc类了

- (4) 运行时创建的`NSKVONotifying_原始类名`这个类，会重写原来父类中被观察属性property的`setter方法实现`。比如如下:

```objc
- (void)setType:(NSSting *)type {

	//1. 调用父类Cat设置属性值的方法实现
	[super setType:type];
	
	//2. 通知Cat对象的观察者执行回调（从断点效果看是同步执行的）
	[cat对象的观察者 observeValueForKeyPath:@"type"  ofObject:self  change:@{}  context:nil];
}
```

- (5) 当对象的被观察属性值发生改变时（中间类的setter方法实现被调用），就会回调执行观察者的`observeValueForKeyPath: ofObject:change:context:`方法实现，并且是`同步`调用的

- (6) 如下两个方法的返回的`objc_class`结构体实例是`不同`的

```c
object_getClass(被观察者对象) >>> 返回的是替换后的`中间类` >>> 因为读取的是isa指向的Class
```

```c
[被观察对象 class] >>> 仍然然会之前的`原始类`，这个Class应该是备用的
```

- (7) 当对象移除属性观察者之后，该`对象的 isa指针`又会`恢复`指向为`原始类`


##  `objc_msgSend()` 函数类型转换的格式

```c
((void (*)(id, SEL)) (void *) objc_msgSend)(obj, sel1);
```

在`c/c++`下，函数指针强转的格式：

```c
#include <iostream>
using namespace std;

//1. 
void work(void *self, char *sel) {
    
}

//2.
int work(void *self, char *sel ,int x) {
    return 1;
}

//3.
bool work(void *self, char *sel ,int x, int y) {
    return true;
}

//4.
string work(void *self, char *sel ,int x, int y, int z) {
        return string("Hello world~!!");
}
```

如上4个c++重载函数的函数指针类型如下：

```c
返回值类型 (*类型名字)(函数的参数列表)

//1. 
void    (*funcP1)(void *self, char *sel);

//2.
int     (*funcP2)(void *self, char *sel ,int x);

//3.
bool    (*funcP3)(void *self, char *sel ,int x, int y);

//4.
string  (*funcP4)(void *self, char *sel ,int x, int y, int z);
```

通过一个函数的指针变量，调用指向的函数:

```c
void test() {
    
    void *ptr = NULL;
    char sel[] = "sel";
    
    // (返回值类型 (*)(函数的参数列表)函数指针变量)(传递给方法的形参列表)；
    
    //1.
    ((void (*)(void *, char *))work)(ptr, sel);
    
    //2.
    int ret1 = ((int (*)(void *, char *, int))work)(ptr, sel, 19);
    
    //3.
    bool ret2 = ((bool (*)(void *, char *, int, int))work)(ptr, sel, 19, 20);
    
    //4.
    string ret3 = ((string (*)(void *, char *, int, int, int))work)(ptr, sel, 19, 20, 21);
    
}
```

## `IMP Caching`: 使用 `methodForSelector:` 获取objc方法 IMP，然后缓存起来。以后每次调用该oc函数时，直接使用IMP

首先，有一个测试类:

```objc
@interface Person : NSObject
+ (void)logName1:(NSString *)name;
- (void)logName2:(NSString *)name;
@end
@implementation Person
+ (void)logName1:(NSString *)name {
    NSLog(@"log1 name = %@", name);
}
- (void)logName2:(NSString *)name {
    NSLog(@"log2 name = %@", name);
}
@end
```

然后ViewController测试IMP Caching:

```c
#import <objc/runtime.h>

static id PersonClass = nil;
static SEL PersonSEL1;
static SEL PersonSEL2;
static IMP PersonIMP1;
static IMP PersonIMP2;

@implementation ViewController

+ (void)initialize {
    PersonClass = [Person class];
   
    PersonSEL1 = @selector(logName1:);
    PersonSEL2 = @selector(logName2:);
    
    //获取类方法实现
    PersonIMP1 = [PersonClass methodForSelector:PersonSEL1];
    
    //获取对象方法实现
    PersonIMP2 = method_getImplementation(class_getInstanceMethod(PersonClass, PersonSEL2));
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

    //1. 调用类方法实现，需要使用最终编译生产对应的c函数的格式，进行函数类型的强制转换
    ((void (*)(id, SEL, NSString*)) (void *) PersonIMP1)(PersonClass, PersonSEL1, @"我是参数");
    
    //2. 调用对象方法实现
    ((void (*)(id, SEL, NSString*)) (void *) PersonIMP2)([Person new], PersonSEL2, @"我是参数");
    
    NSLog(@"");
}

@end
```

输出结果

```
2017-02-08 22:47:46.586 Test[805:25490] log1 name = 我是参数
2017-02-08 22:47:46.587 Test[805:25490] log2 name = 我是参数
```

## 事件传递 与 事件响应链

### 事件传递

- (1) 当屏幕上产生一个触摸事件之后， `UIApplication` 对象的 `sendEvent:` 方法实现会被调用

- (2) 其中传入的 UIEvent对象 包含了 当前 UIWindow上产生触摸点事件的所有信息

- (3) 然后 UIApplication 将 UIEvent对象 传递给 UIWindow

- (4) UIWindow 将 UIEvent对象 传递给 RootViewController

- (5) RootViewController 将 UIEvent对象 传递给 RootView

- (6) RootView 将 UIEvent对象 传递给 所有的 subviews 

- (7) 继续向 subviews 依次询问 是否能够 处理这个 UIEvent对象
	- 是否 打开交互、大小不为0，不透明，没有隐藏
	- 触摸点是否处于当前view的frame区域内
	- 如果当前view能处理，则继续向自己的subviews继续传递 UIEvent对象
	- 如果当前view的subviews都不能处理，则将返回自己作为hitTest的对象

	
### 事件响应链

- (1) 通过上面的步骤找到了，最终触发UIEvent触摸事件的 UIView 对象
- (2) 再通过 给这个 UIView 设置的 各种 按钮事件、手势、触摸事件..找到对应的target
 
 
### 下面是使用自定义 UIApplication 拦截全局的 UI触摸事件

```objc
@interface MyApplication : UIApplication
@end
@implementation MyApplication

- (BOOL)sendAction:(SEL)action to:(id)target from:(id)sender forEvent:(UIEvent *)event {
    NSLog(@"%s action = %@, target = %@ from = %@ event = %@",  __func__, NSStringFromSelector(action), target, sender, event);
    return [super sendAction:action to:target from:sender forEvent:event];
}

- (void)sendEvent:(UIEvent *)event {
    NSLog(@"sendEvent: event = %@", event);
    [super sendEvent:event];
}

@end
```

main.m 中使用自定义的 Application 

```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, @"MyApplication", NSStringFromClass([AppDelegate class]));
    }
}
```

这样之后，每当屏幕发生触摸点击事件，就会输出类似如下

```
2017-03-29 23:26:15.420 AnimationsDemo[2173:69454] sendEvent: event = <UITouchesEvent: 0x7fec8ac06f50> timestamp: 5121.41 touches: {(
    <UITouch: 0x7fec8d02e110> phase: Ended tap count: 1 window: <UIWindow: 0x7fec8ae0c750; frame = (0 0; 320 568); gestureRecognizers = <NSArray: 0x7fec8ae165f0>; layer = <UIWindowLayer: 0x7fec8ae15180>> view: <UIView: 0x7fec8af0f620; frame = (0 0; 320 568); autoresize = W+H; layer = <CALayer: 0x7fec8af0ba30>> location in window: {114.5, 295.5} previous location in window: {114.5, 295.5} location in view: {114.5, 295.5} previous location in view: {114.5, 295.5}
)}
```

每当屏幕发生触摸事件，就会调用 `sendAction:` 其传入的 UIEvent对象 就是触摸事件的抽象。

## sendAction 发送UIEvent事件

- UIApplication

```c
sendAction:to:from:forEvent:
```

- UIControl

```c
sendAction:to:forEvent:
```

可以通过替换SEL指向的IMP，来拦截这两个负责发送UI事件的方法实现，来做一些额外的处理。

## 当遇到`多种选择条件`时，要尽量使用`查表`法实现

比如 `switch/case`，C Array，如果查表条件是对象，则可以用 NSDictionary 来实现。

比如，如下很多的`if-elseif`的判断语句:

```c
NSString *name = @"XIONG";
    
if ([name isEqualToString:@"XIAOMING"]) {
    NSLog(@"task 1");
} else if ([name isEqualToString:@"LINING"]) {
    NSLog(@"task 2");
} else if ([name isEqualToString:@"MAHANG"]) {
    NSLog(@"task 3");
} else if ([name isEqualToString:@"YHAHA"]) {
    NSLog(@"task 4");
}
```

使用`NSDictionary+Block`来封装`条件对应的key`与`执行代码block`的键值对关系:

```c
NSDictionary *map = @{
	// if条件的key : if条件要执行的代码封装的block
	@"XIONG1" : ^() {NSLog(@"task 1");},
  	@"XIONG2" : ^() {NSLog(@"task 2");},
  	@"XIONG3" : ^() {NSLog(@"task 3");},
  	@"XIONG4" : ^() {NSLog(@"task 4");},
};
```

然后使用某一个条件的`key`值，查询`dic`获取得到要`执行的代码`

```c
//1. 查找if条件key对应的block语句
void (^block)(void) = [map objectForKey:@"XIONG"];

//2. 执行if条件对应的block任务代码
block();
```

查表属于hash算法，在`非常多的if-elseif`判断语句时，效率会提升很多的。

> 如果`key值`都很相似，这个时候就会造成每一次都是从hash表从头到尾遍历冲突，找到下一个，这样效率也很差的。


比如:

```c
NSDictionary *map = @{
                      @"nil" : @1,
                      @"nIl" : @1,
                      @"niL" : @1,
                      @"Nil" : @1,
                      @"NIL" : @1,
                      @"NIl" : @1,
                      };
```

所有的key值都太相似了，那么这种情况下，给一个key值查表时，几乎都是循环挨个遍历。

所以，在给各种`if条件`构造`唯一key`时，尽量要`差别很大`。

## objc对象、objc类、meta类、super 之间`isa`指针与`super_class`指针的指向关系

<img src="./runtime1.jpeg" alt="" title="" width="700"/>

<img src="./runtime2.png" alt="" title="" width="700"/>

### isa指针的指向

- (1) `object->isa` 指向 `objc类`
- (2) `objc类->isa` 指向 `Meta objc类`
- (3) `Meta objc类->isa` 指向 `Meta NSObject类`

### `super_class`指针的指向

- (1) `objc类->super_class` 执行 `父亲 objc类`
- (2) `NSObject类->super_class` 指向 `nil`
- (3) `Meta objc类->super_class` 指向 `Meta NSObject类`
- (4) `Meta NSObject类->super_class` 指向 `Meta NSObject类（自己）`

## 我们编写的NSObject类，在程序运行时加载的过程

```c
//1. 创建一个运行时识别的Class
objc_allocateClassPair(Class superclass, 
					   const char *name, 
					   size_t extraBytes);

//2. 添加实例变量
BOOL class_addIvar(Class cls, 
				   const char *name, 
				   size_t size, 
				   uint8_t alignment,
				    const char *types);

//3. 添加实例方法
BOOL class_addMethod(Class cls, 
					 SEL name, 
					 IMP imp, 
					 const char *types);
					 
//4. 添加实现的协议
BOOL class_addProtocol(Class cls, Protocol *protocol);

//5. 【重要】将处理完毕的Class注册到运行时系统，之后就无法修改【Ivar】
void objc_registerClassPair(Class cls);
```

Ivar在所属`Class`的内存块中，按照偏移量的形式依次排列:

<img src="./Ivar_offset.png" alt="" title="" width="700"/>

### 涉及到一个内存字节对其的问题，其规律就是:

- (1) 每一个Ivar相对于整个存储空间起始地址的`总偏移量`，必须是自身长度的整数倍
- (2) 最终整个存储空间的长度，必须是最大长度Iavr的整数倍
- (3) 满足如上(1)、(2)时，使用空白的字节进行填充，不会做任何使用，只是做对其而已

具体布局规则可查看

```
objc对象的成员变量的内存布局.md
```


> 当执行完最后一步`objc_registerClassPair(Class cls)`，就会进行`Ivar的内存布局`计算了，之后就`无法再改变`了。


下面是一个例子，运行时创建一个类，并添加Ivar、Property、Method，并完成调用:

```c
void PersonLog(id target, SEL sel, NSString *name) {
    NSLog(@"IMP PersonLog: name = %@", name);
}

@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    // 创建一个类，继承自NSObject类的子类
    Class class = Nil;
    class = objc_allocateClassPair([NSObject class], "Person", 0);
    
    // 创建一个属性 >>> @property(nonatomic, copy) NSString *name;
    objc_property_attribute_t type = {"T", "@\"NSString\""};
    objc_property_attribute_t nonatomic = { "N", "" };
    objc_property_attribute_t copy = { "C", "" };
    objc_property_attribute_t ivarName = { "V", "_name"};
    objc_property_attribute_t attrs[] = {type, nonatomic, copy, ivarName};
    bool success = class_addProperty(class, "name", attrs, 4);
    if (success) {
        NSLog(@"add Property success");
    }
    
    // 给类添加一个Ivar
    if (class_addIvar(class, "_name", sizeof(NSString*), log2(sizeof(NSString*)), "@")) {
        NSLog(@"add Ivar success");
    }
    
    // 给Class添加一个Method
    if (class_addMethod(class, @selector(logName), (IMP)&PersonLog, "v@:@")) {
        NSLog(@"add Method success");
    };
    
    // 向运行时系统注册这个类
    // 注意: 添加Ivar、Property、Method..都必须在objc_registerClassPair()之前完成
    if (!objc_lookUpClass("Person")) {
        objc_registerClassPair(class);
    }
    
    
    // 创建类实例
    id p1 = class_createInstance(class, 0);
    
    // 获取类对象的一个Ivar
    Ivar ivar = class_getInstanceVariable(class, "_name");
    
    // 获取Proeprty的type encodings
    objc_property_t proeprty = class_getProperty(class, "name");
    const char *types = property_getAttributes(proeprty);
    NSLog(@"types = %@", [NSString stringWithUTF8String:types]);
    
    // 对Ivar进行设值
    object_setIvar(p1, ivar, @"XiongZenghui");
    
    // 获取Ivar的值
    NSString *name = object_getIvar(p1, ivar);
    NSLog(@"name = %@", name);
    
    //10. 执行Method
    ((void (*)(id,SEL,NSString*)) (void*) objc_msgSend)(p1, @selector(logName), @"hahahahaha");
    
    //11. 废弃掉创建的Class
    objc_destructInstance(class);
}

@end
```

输出

```
2017-03-06 18:52:58.337 Demo[21770:326695] add Property success
2017-03-06 18:52:58.338 Demo[21770:326695] add Ivar success
2017-03-06 18:52:58.338 Demo[21770:326695] add Method success
2017-03-06 18:52:58.338 Demo[21770:326695] types = T@"NSString",N,C,V_name
2017-03-06 18:52:58.338 Demo[21770:326695] name = XiongZenghui
2017-03-06 18:52:58.338 Demo[21770:326695] IMP PersonLog: name = hahahahaha
```

## `读写Ivar > 发送getter/setter消息 > KVC`

Key-Value Coding 使用起来非常方便，但性能上要差于直接调用 Getter/Setter，所以如果能避免 KVC 而用 Getter/Setter 代替，性能会有较大提升。

KVC首先根据`setValue:forKey:`传入的key，查找到对应的`Ivar`，这个过程就已经有一点复杂了。

并且，KVC对于基本数据类型（int、float、double、long、NSInteger..）都会自动封装为NSNumber，而这个对于直接调用getter/setter时，是不必要的步骤。


通过`调用getter/setter`的方式进行Ivar值存取时，虽然是比KVC稍微好一点，但仍然需要走`objc的消息传递`的过程。

但是直接通过获取得到Iavr，直接对Ivar进行值的存取会更快:

```objc
@interface Dog : NSObject
@property (nonatomic, copy) NSString *name;
@end
@implementation Dog
@end
```

```objc
- (void)test {
    
    //1. 获取Ivar。注意：名字默认都是以为下划线开始的。
    // eg： _name、_age ...
    Ivar ivar = class_getInstanceVariable([Dog class], "_name");
    
    //2. 
    Dog *dog = [Dog new];
    
    //3. 直接对Ivar进行写
    object_setIvar(dog, ivar, @"我是你妈");
    
    //4. 直接对Ivar进行读
    NSLog(@"name = %@", object_getIvar(dog, ivar));

}
```

除了第一步查找Ivar之外，后面都是直接绕过了objc的消息传递过程，直接对Ivar进行存取。

但是我在封装一些runtime的工具代码的时候，发现`object_getIvar()、object_setIvar()`会在一些基本数据类型（BOOL、float、int、long ...）时会崩溃，提示`BAD_ACESS`。

解决:

```
http://stackoverflow.com/questions/8356232/object-getivar-fails-to-read-the-value-of-bool-ivar
```

崩溃的原因是由于，`object_getIvar()、object_setIvar()`只适用于`ARC`环境，默认会将Ivar的返回值类型当做是objc对象，就会对返回值对象进行retain/release，而基本类型变量（BOOL、float、int、long ...）是不能retain/release。

但是也可以让`object_getIvar()、object_setIvar()`支持对基本类型的Ivar村取值，使用类似强转`objc_msgSend()`的方式:

```objc
@interface Dog : NSObject <Animal3>
@property (nonatomic, assign) NSInteger uid;
@property (nonatomic, assign) NSInteger age;
@end
@implementation Dog
@end
```

```c
Ivar ivar2 = class_getInstanceVariable([dog class], "_uid");
Ivar ivar3 = class_getInstanceVariable([dog class], "_age");

NSInteger uid = ((NSInteger (*)(id, Ivar))object_getIvar)(dog, ivar2);
NSInteger age = ((NSInteger (*)(id, Ivar))object_getIvar)(dog, ivar3);
```

输出

```
(NSInteger) uid = 1111
(NSInteger) age = 19
```

强制转换`object_getIvar()`的函数返回值类型，来取消对返回值默认当做objc对象。

> 操作Ivar 快于 getter/setter消息传递。这个是有前提的，如果只是一次操作，肯定Ivar会更快。但是如果有很多次的调用，那么直接操作Ivar就会比发送getter/setter消息慢。因为objc的消息发送会有内存缓存的，做了一些优化的。


```objc
- (void)testKVC4 {
    
    Dog *dog = [Dog new];
    dog.name = @"name1111";
    dog.uid = 1111;
    dog.age = 19;
    dog.url = [NSURL URLWithString:@"www.baidu.com"];
    
    int count = 1000000;
    double date_s = CFAbsoluteTimeGetCurrent();
    for (int i = 0; i < count; i++) {
        
        NSString *name = [NSString stringWithFormat:@"name_%d", i];
        
        //1. stters messgae
        dog.name = name;
        
        //2. Ivar
        Ivar ivar = class_getInstanceVariable([dog class], "_name");
        object_setIvar(dog, ivar, name);
    }
    double date_current = CFAbsoluteTimeGetCurrent() - date_s;
    NSLog(@"consumeTime: %f μs",date_current * 11000 * 1000);
}
```

使用发送getter/setter消息，消耗的时间:

```
consumeTime: 10366509.914398 μs
```

使用Ivar，消耗的时间:

```
consumeTime: 12702172.517776 μs
```

明显慢了很多去了，所以如果在多次调用时，还是使用`发送getter/setter消息`是最快的。


## 对 `NSArray/NSSet/NSDictionary` 容器对象进行遍历的时候，转为CoreFoundation容器对象，再进行遍历，效率会更高。这也是struct作为Context的一个应用场景。

通常对于Foundation的写法:

```objc
//1.
NSDictionary *map = @{
                      @"key1" : @"value1",
                      @"key2" : @"value2",
                      @"key3" : @"value3",
                      @"key4" : @"value3",
                      };

//2. 每一次遍历都传入进行使用的公共数据
NSMutableString *info = [[NSMutableString alloc] init];

//3. 遍历容器，取出key、value、保存到公共数据中
[map enumerateKeysAndObjectsUsingBlock:^(NSString*  _Nonnull key, NSString*  _Nonnull obj, BOOL * _Nonnull stop)
{
    [info appendFormat:@"%@=%@", key, obj];
}];

NSLog(@"info = %@", info);
```

转换为CoreFoundation的写法，分为三部曲:

### 第一步、定义每一次CF遍历回调c函数中使用的公共内存数据Context

```c
struct Context {
    void *info;    //注意：c struct中不能定义objc对象类型
};
```

### 第二步、定义每一次CFArray/CFSet/CFDictionary遍历回调的c函数实现

```c
void XZHCFDictionaryApplierFunction(const void *key, const void *value, void *context) {
    //1.
    struct Context *ctx = (struct Context *)context;
    
    //2.
    NSMutableString *info = (__bridge NSMutableString*)(ctx->info);
    
    //3.
    NSString *key_ = (__bridge NSString*)(key);
    NSString *value_ = (__bridge NSString*)(value);
    
    //4.
    [info appendFormat:@"%@=%@", key_, value_];
}
```

### 第三步、将objc格式的NSArray/NSSet/NSDictionary转换为CF容器类型进行遍历

```objc
@implementation ViewController

- (void)test {
    
    //1.
    NSDictionary *map = @{
                          @"key1" : @"value1",
                          @"key2" : @"value2",
                          @"key3" : @"value3",
                          @"key4" : @"value3",
                          };

    //2. 每一次遍历都传入进行使用的公共数据
    NSMutableString *info = [[NSMutableString alloc] init];
    
    
    //3. 创建一个栈Context实例
    struct Context ctx = {0};
    
    //4. Context实例，包装每一次遍历函数中都要操作的objc对象数据
    ctx.info = (__bridge void*)(info);

    //5. 遍历容器，取出key、value、保存到公共数据中
    CFDictionaryApplyFunction((CFDictionaryRef)map,
                          XZHCFDictionaryApplierFunction,
                          &ctx);

    //6.
    NSLog(@"info = %@", info);
}

@end
```

效果与之前Foundation的一致，但是可以看到，使用CF容器遍历时，需要进行编写用于每次传入`CFDictionaryApplyFunction()`中回调c函数的一个`Context`结构体，并且需要对该Context结构体实例进行不断的存取。

但是总体CF容器遍历的效率绝对比Foundation容器遍历高，因为省去了objc消息传递等很多步骤，直接就是c函数调用完成的。

## struct的一些妙用

### struct应用场景2、作为多个参数的打包器Context


分为：方法获取参数集合、方法回传参数集合

```c
typedef struct Context {
    int     arg1;
    char    *arg2;
    float   arg3;
    void    *arg4;
}Context;
```

```c
void func6(Context *ctx) {
    
    //1. 从Context实例获取所有的参数
    int     arg1 = ctx->arg1;
    char    *arg2 = ctx->arg2;
    float   arg3 = ctx->arg3;
    void    *arg4 = ctx->arg4;
    
    //2. 操作所有的参数处理
}

void func7(int x, Context *ctx) {
    char a[] = "hahaha";
    
    // 将处理后的参数，设置到ctx回传出去
    ctx->arg1 = 1;
    ctx->arg2 = a;
    ctx->arg3 = 19.23;
    ctx->arg4 = (void*)"haha";
    
}
```

### struct应用场景3、位段结构体

格式

```c
struct __touchDelegate {
    
    //格式:
    变量类型 成员名 : 分配的长度;
    
    //例子: 通常分配一个二进制位就可以了，表示0与1即可
    unsigned int touchBegin : 1;
};
```

objc中的delegate的定义

```objc
@protocol TouchDelegate : NSObject

- (void)touchBegin;
- (void)touchMoved;
- (void)touchEnd;

@end
```

对应的位段结构体如下

```c
struct __touchDelegate {
    unsigned int touchBegin : 1;
    unsigned int touchMoved : 1;
    unsigned int touchEnd   : 1;
};
```

在设置delegate时，填充这个位段结构体实例


```objc
- (void)setDelegate:(id<TouchDelegate>)delegate {
    _delegate = delegate;
    
    if ([delegate respondsToSelector:@selector(touchBegin)]) {
        __touchDelegate.touchBegin = 1;
    }
    
    if ([delegate respondsToSelector:@selector(touchMoved)]) {
        __touchDelegate.touchMoved = 1;
    }
    
    if ([delegate respondsToSelector:@selector(touchEnd)]) {
        __touchDelegate.touchEnd = 1;
    }
}
```

后后面只需要根据位段结构体实例的对应成员变量值是0还是1，就可以判断是否实现了协议方法。

## `__bridge`、`__bridge_retained`、`__bridge_transfer`

让struct实例，持有`Foundation对象`。当struct实例废弃时，让Foundation对象在子线程上异步释放废弃

### 核心主要牵涉三个用于`c实例`与`objc对象`进行转换的东西

#### `(__bridge_retained CoreFoundation实例)Foundation对象`

```
1. Foundation对象 >>> CoreFoundation实例
2. [Foundation对象 retain]
```

#### `(__bridge_transfer Foundation对象)CoreFoundation实例`

```
1. CoreFoundation实例 >>> Foundation对象 
2. [Foundation对象 release]
```

#### `__bridge` 

```
1. Foundation对象 >>> CoreFoundation实例
2. CoreFoundation实例 >>> Foundation对象 
3. 不会执行任何的`retain/release`效果，仅仅只是类型的转换
```

### 下面demo测试

Foundation 类

```objc
@interface Dog : NSObject
@property (nonatomic, copy) NSString *name;
@end
@implementation Dog
- (void)dealloc {
    NSLog(@"废弃Dog对象，name = %@ on thread = %@", _name, [NSThread currentThread]);
}
@end
```

struct实例 使用 `void*` 万能指针类型持有 Foundation对象，因为struct中不能写objc的NSObejct类型.

```c
typedef struct DogsContext {
    void    *dogs;//持有oc数组对象
}DogsContext;
```

ViewController测试代码

```objc
static DogsContext *_dogsCtx = NULL;

@implementation ViewController

// 测试结构体实例持有oc对象
- (void)testARCBridge1 {
    
    //1. c结构体实例
    _dogsCtx = malloc(sizeof(DogsContext));
    _dogsCtx->dogs = NULL;
    
    //2. 创建测试的oc数组对象
    NSMutableArray *dogs = [NSMutableArray new];
    for (int i = 0; i < 3; i++) {
        Dog *dog = [Dog new];
        dog.name = [NSString stringWithFormat:@"name_%d", (i + 1)];
        NSLog(@"创建Dog对象，name = %@", dog.name);
        [dogs addObject:dog];
    }
    
    //3. struct实例 持有 NSFoundation对象，并对oc对象进行retain，防止oc对象被废弃
    _dogsCtx->dogs = (__bridge_retained void*)dogs;
    //数组对象.retainCount==2（一个是局部指针dogs，另一个是通过 __bridge_retained）

}//数组对象.retainCount==1（局部指针dogs超出作用域被释放）

// 测试从结构体实例中取出oc对象使用，然后不再需要的时候全部一起废弃
- (void)testARCBridge2 {
    
    //1. 先取出c struct实例持有的 NSFoundation对象使用，不进行任何retain/release
    NSMutableArray *array1 = (__bridge NSMutableArray*)_dogsCtx->dogs;
    for (Dog *dog in array1) {
        NSLog(@"使用Dog对象，name = %@", dog.name);
    }
    
    //2. 释放struct实例持有的NSMutableArray数组，继而释放掉了NSMutableArray数组持有的所有的Dogs对象
    
    //2.1 使用 __bridge_transfer 对oc数组对象访问的同时进行release，
    NSMutableArray *holder = (__bridge_transfer NSMutableArray*)_dogsCtx->dogs;
    //数组对象.retainCount==1
    
    //2.2 解决结构体实例指向oc数组对象，并废弃结构体实例
    _dogsCtx->dogs = NULL;
    free(_dogsCtx);
    _dogsCtx = NULL;
    
    //2.3 子线程异步释放废弃oc数组内的其他子对象
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [holder class];//holder.retainCount == 0 标记废弃
    });
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

        [self testARCBridge1];
    [self testARCBridge2];
    
    NSLog(@"");
}

@end
```

输出信息

```
2017-02-08 23:54:34.433 Test[1262:71331] 创建Dog对象，name = name_1
2017-02-08 23:54:34.434 Test[1262:71331] 创建Dog对象，name = name_2
2017-02-08 23:54:34.434 Test[1262:71331] 创建Dog对象，name = name_3

2017-02-08 23:54:34.434 Test[1262:71331] 使用Dog对象，name = name_1
2017-02-08 23:54:34.434 Test[1262:71331] 使用Dog对象，name = name_2
2017-02-08 23:54:34.434 Test[1262:71331] 使用Dog对象，name = name_3

2017-02-08 23:54:34.435 Test[1262:71700] 废弃Dog对象，name = name_1 on thread = <NSThread: 0x7ff0c8e68a50>{number = 2, name = (null)}
2017-02-08 23:54:34.435 Test[1262:71700] 废弃Dog对象，name = name_2 on thread = <NSThread: 0x7ff0c8e68a50>{number = 2, name = (null)}
2017-02-08 23:54:34.435 Test[1262:71700] 废弃Dog对象，name = name_3 on thread = <NSThread: 0x7ff0c8e68a50>{number = 2, name = (null)}
```

## 将一些不太重要的代码放在 `idle（空闲）` 时去执行

```objc
- (void)idleNotificationMethod { 
    // do something here 
} 
 
- (void)registerForIdleNotification  
{ 
	//1. 让一段在空间时刻执行的代码，关注通知 IdleNotification时，再去执行
    [[NSNotificationCenter defaultCenter] addObserver:self 
        selector:@selector(idleNotificationMethod) 
        name:@"自定义通知的key" 
        object:nil]; 
        
    //2. 构建通知NSNotification
    NSNotification *notification = [NSNotification 
        notificationWithName:@"自定义通知的key" object:nil]; 

	//3. 在某个时刻，异步发送通知，并使用空闲模式，让之前的注册的代码执行
    [[NSNotificationQueue defaultQueue] enqueueNotification:notification 
    postingStyle:NSPostWhenIdle]; 
}  
```

## 深拷贝、浅拷贝、可变、不可变

### 深拷贝、浅拷贝 的区别与联系

<img src="./深浅拷贝3.png" alt="" title="" width="700"/>

而一个objc对象的深拷贝，其实建立在浅拷贝的基础之上，继续对objc对象内部所有的Ivar继续执行浅拷贝（递归执行`浅拷贝`）。

### 深拷贝、浅拷贝、可变、不可变，四者之间的关系

<img src="./深浅拷贝1.jpg" alt="" title="" width="700"/>

## 对象关联时所有内存管理策略

| 关联对象指定的内存策略 | 等效的OC对象内存管理修饰符  | 
| :-------------: |:-------------:| 
| `OBJC_ASSOCIATION_ASSIGN` | **atomic + assign** |
| `OBJC_ASSOCIATION_RETAIN_NONATOMIC` | **nonatimic + retain** |
| `OBJC_ASSOCIATION_COPY_NONATOMIC` | **nonatimic + copy** |
| `OBJC_ASSOCIATION_RETAIN` | **atomic + retain** |
| `OBJC_ASSOCIATION_COPY` | **atomic + copy** |

没有指定`nonatomic`，默认就是`atomic`原子属性同步多线程。

## 属性修饰符与对象所有权修饰符的关系

| 属性声明时的修饰符 | 对象所有权修饰符 | 
| :-------------: |:-------------:| 
| assign | `__unsafe_unretained` | 
| copy | `__strong`（首先是拷贝原始对象得到一个新的对象，然后再强引用新的对象，释放老的对象） | 
| retain | `__strong` | 
| strong | `__strong` | 
| `unsafe_unretained` | `__unsafe_unretained` | 
| weak | `__weak` | 


### 对于属性提供的额外几种
	
- copy 

其实和strong/retain很相似的，`只是多一步拷贝`的操作，对于copy修饰的属性setter方法实际上做了如下三件事:

```
- (1) id newObj = [oldObj copy];
- (2) [oldObj release];
- (3) [newObj retain];
```

- assign:
	- 一般使用一些基本数据类型的属性变量（int、float、bool...）
	- 类似`__unsafe_unretaind`

## `__strong` 修饰对象的指针变量

- (1) 会自动添加对象的`retain\release`的消息代码

- (2) 持有与释放的原则
	- 自己生成的对象，自己持有
	- 非自己生成的对象，我也能持有
	- 不再需要自己持有的对象时进行释放
	- 非自己持有的对象无法释放

##  `__unsafe_unretained` 

### 内存管理原则

- (1) `unsafe` 不安全，这点是与`weak`不同点，既不会自动赋值nil
- (2) `unretained` 不会产生`强引用`持有，这是与`weak`的相同点
- (3) **直接使用对象的地址，不管对象是否已经被废弃，都直接访问地址**

所以，如果被访问的地址已经被废弃，可能造成崩溃。

### 使用`__unsafe_unretained`来修饰指向`必定不会被废弃`的对象的指针变量，不会由ARC系统附加做`retain/release`的处理，提高了运行速度

- (1) 使用`__weak`修饰的指针变量指向的对象时，会将被指向的对象，自动注册到自动释放池，防止使用的时候被废弃，但是影响了代码执行效率

- (2) 如果一个对象确定是不会被废弃，或者调用完成之前不会被废弃，就使用`__unsafe_unretained`来修饰指针变量

- (3) `__unsafe_unretained`就是简单的拷贝`地址`，不进行任何的`对象内存管理`，即不修改retainCount

## `__autoreleasing`

分析下`+[NSMutableArray array]`返回值对象处理:

```objc
+ (id)array {

	//1. 生成一个数组对象
	id obj = objc_msgSend(NSMutableArray, @selector(alloc));
	
	//2. 执行对象的初始化init方法
	obj = objc_msgSend(obj, @selector(init));
	
	//3. 返回一个autorelease的返回值
	return objc_autoreleaseReturnValue(obj);
}
```

## 使用`__weak`指针指向的对象时，会自动将对象注册到一个自动释放池，防止提早废弃

那么为什么要这样的了？因为`__weak`变量:

- (1) 支持有对象的一个`弱引用`
- (2) 不能保证访问该对象的整个过程中，对象一定不会被废弃

所以，通过将弱引用的对象，注册到autoreleasePool中，从而保证整个操作过程（autoreleasePool结束之前），弱引用对象都不会被废弃。

但是这样，会造成一些代码执行效率的降低。

如果确认某个对象，在使用期间肯定不会被废弃，那么直接使用`__unsafe_unretained`，直接使用`对象的地址`，而不会对对象进行任何的`retian/release/autorelease`。

## 使用`__strong`强持有方法的返回值对象时，与`__weak`是有区别的

### 有一对重要的函数，用于返回值对象的内存管理优化

第一个、

```c
id objc_autoreleaseReturnValue(id obj);
```

第二个、

```c
id objc_retainAutoreleasedReturnValue(id obj);
```

### 对于`+[NSMutableArray array]`最终的c实现代码为如下:

```objc
@implementation NSMutableArray

+ (id)array {

	//1. 生成一个数组对象
	id obj = objc_msgSend(NSMutableArray, @selector(alloc));
	
	//2. 执行对象的初始化init方法
	obj = objc_msgSend(obj, @selector(init));
	
	//3. 【重点】
	return objc_autoreleaseReturnValue(obj);
}

@end
```

### 对于其他类对象中，使用`__strong`持有该方法的返回值对象的objc代码

```objc
@implementation ViewController 
 
- (void) test  {
	
	id __strong obj = [NSMutableArray array];
}
 
@emd
```

最终被编译成为的c代码大致如下:

```c
void test(id target, SEL sel) {
	
	//1. 调用方法获取返回值对象
	id obj = objc_msgSend(NSMutableArray, @selector(array));

	//2. 【重点】 
	objc_retainAutoreleasedReturnValue(obj);

	//3.
	objc_release(obj);
}
```
	
### `objc_autoreleaseReturnValue(id obj)` 出现在返回一个返回值的函数代码中

```c
id  objc_autoreleaseReturnValue(id obj)
{
	// 1. 如果调用函数的一方，在获取到返回值后，使用了 objc_retainAutoreleasedReturnValue(obj)
	// 就走如下if语句，
	// 首先、对该返回值对象做一个标记
	// 最后、直接返回这个返回值对象
    if (callerAcceptsFastAutorelease(__builtin_return_address(0))) {//判断调用方法，是否使用了 objc_retainAutoreleasedReturnValue(obj)
        tls_set_direct(AUTORELEASE_POOL_RECLAIM_KEY, obj);//对该返回值对象做一个标记
        return obj;
    }

    //2. 相反如果在获取到返回值后，没有使用 objc_retainAutoreleasedReturnValue(obj)
    // 则将对象注册到一个释放池中，然后再返回
    return objc_autorelease(obj);
}
```

### `objc_retainAutoreleasedReturnValue()` 出现在调用某个函数获取一个objc返回值对象的代码的`紧接着的下面一行`

比如:

```c
void test(id target, SEL sel) {
	
	//1. 调用方法获取返回值对象
	id obj = objc_msgSend(NSMutableArray, @selector(array));

	//2. 【重点】 
	objc_retainAutoreleasedReturnValue(obj);

	//3.
	objc_release(obj);
}
```

看`objc_retainAutoreleasedReturnValue(obj);`做了啥:

```c
id objc_retainAutoreleasedReturnValue(id obj)
{

	//1. 判断传入的对象，是否是需要做内存优化。如果需要走如下if语句:
	// 首先、根据标记从缓存中取出返回值对象
	// 然后、取消这个对象的返回值内存优化标记
	// 最后、返回这个对象
    if (obj == tls_get_direct(AUTORELEASE_POOL_RECLAIM_KEY)) {
        tls_set_direct(AUTORELEASE_POOL_RECLAIM_KEY, 0);
        return obj;
    }

    //2. 而如果没有被标记做返回值优化的对象
    // 会被retain一次，增加其retainCount
    return objc_retain(obj);
}
```

基本上就可以看明白`objc_autoreleaseReturnValue(id obj)`与`objc_retainAutoreleasedReturnValue(id obj)`这一对函数，在返回值为objc对象时，做的优化了。

两个函数配合起来，`禁止`返回值objc对象被注册到`autorelease pool`的多余过程。	

## objc消息转发阶段总结

<img src="./forwardinvocation.png" alt="" title="" width="700"/>

##  `_objc_msgForward` iOS系统消息转发c函数指针

还有与之差不多意思的:

```c
_objc_msgForward_stret
```

jspatch恰恰就是利用的这个`_objc_msgForward`c方法实现，达到交换任意Method的SEL指向的IMP。

当一个oc类中，找不到某一个SEL对应的IMP时，会进入到系统的消息转发函数。

下面测试下，`_objc_msgForward`到底是如何转发消息的？

首先，有如下测试类:

```objc
@interface Person : NSObject
+ (void)logName1:(NSString *)name;
- (void)logName2:(NSString *)name;
@end
@implementation Person
+ (void)logName1:(NSString *)name {
    NSLog(@"log1 name = %@", name);
}
- (void)logName2:(NSString *)name {
    NSLog(@"log2 name = %@", name);
}
@end
```

ViewController中随便执行一个Perosn对象不存在实现的SEL消息:

```objc
@implementation ViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

    // 此处打一个断点，然后执行下面的lldb调试命令后，再往下执行代码
    static Person *person;
    person = [Person new];
    [person performSelector:@selector(hahahaaha)];
    
    NSLog(@"");
}

@end
```

断点后，在lldb中输入如下调试命令，会打印出所有运行时发送的消息:

```c
(lldb) call (void)instrumentObjcMessageSends(YES)
```

程序崩溃后，进入Mac电脑系统如下目录:

```c
cd /tmp/
```

找到该目录下类似如下结构的文件，然后打开

```
msgSends-901
```

打开文件后，只看与`Person`相关的信息大概为如下:

```objc
+ Person NSObject initialize
+ Person NSObject new
- Person NSObject init
- Person NSObject performSelector:
+ Person NSObject resolveInstanceMethod:
+ Person NSObject resolveInstanceMethod:
- Person NSObject forwardingTargetForSelector:
- Person NSObject forwardingTargetForSelector:
- Person NSObject methodSignatureForSelector:
- Person NSObject methodSignatureForSelector:
- Person NSObject class
- Person NSObject doesNotRecognizeSelector:
- Person NSObject doesNotRecognizeSelector:
- Person NSObject class
```

从`Person NSObject performSelector:`开始执行一个不存在实现SEL消息后，依次开始执行:

- (1) `resolveInstanceMethod:` or `resolveClassMethod:`
- (2) `forwardingTargetForSelector:`
- (3) `methodSignatureForSelector:`
- (4) `forwardInvocation:`

所以，`_objc_msgForward`这个指针指向的是，负责完成整个objc消息转发的c函数实现，包括`阶段1、阶段2`。

如果最后阶段`forwardInvocation:`仍然无法处理消息，就产生异常让程序退出。

## `串行 dispatch_queue_t` 缓存池 

对于串行与并发队列的区别:

- (1) 一个`dispatch serial queue实例`，就是`一个`底层线程

- (2) 一个`dispatch concurrent queue实例`，就是`n个`底层线程

YYDispatchQueuePool的核心几点:

- (1) iOS8之前使用Priority，iOS8及之后使用QualityOfService，来操作`dispatch_queue_t`

- (2) 根据 Priority或iOS8及之后使用QualityOfService，不同的各种等级，分别创建一个Context内存块

- (3) 每一个Context内存块，保存`[NSProcessInfo processInfo].activeProcessorCount`个 `dispatch serial queue`实例，让CPU的核心充分利用

- (4) 根据对应的等级，从对应的Context中，随机取出一个dispatch serial queue实例，进行任务调度

### 注意如果直接对NSThread进行缓存，一定要做如下几件事

- (1) 获取NSThread对象的RunLoop（完成RunLoop的创建与绑定）
- (2) 给RunLoop添加RunLoopSource监听的事件源（随便加一个NSMachPort）
- (3) 让NSThread的`-[NSRunLoop run]`

这样，这个NSThread对象的状态，才会一直处于执行状态，即使`isExecuting == YES`，这样也才能让这个NSThread对象，永远的随时随刻接收任务执行。

但是如果直接对`dispatch_queue_t`实例，进行缓存的话，是不需要我们手动做上面的一些事情的，我估计是GCD底层线程池已经做了处理。

## `@package`块声明ivar

这一类的iavr，可以再当前`.m`中任意进行访问。
让一些Ivar只想在`静态类库`或`framework`中的类对象中的代码可以访问。


## `hitTest:withEvent:` 与 `pointInside:withEvent:`

### 关于`-[UIView hitTest:withEvent:]`的源码大致实现

```objc
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
{
    // 1. 判断自己是否能够接收触摸事件（是否打开事件交互、是否隐藏、是否透明）
    if (self.userInteractionEnabled == NO || self.hidden == YES || self.alpha <= 0.01) return nil;
    
    // 2. 调用 pointInside:withEvent:， 判断触摸点在不在自己范围内（frame）
    if (![self pointInside:point withEvent:event]) return nil;
    
    // 3. 从`上到下`（最上面开始）遍历自己的所有子控件，看是否有子控件更适合响应此事件
    int count = self.subviews.count;
    for (int i = count - 1; i >= 0; i--) {
        UIView *childView = self.subviews[i];
        
        // 将产生事件的坐标，转换成当前相对subview自己坐标原点的坐标
        CGPoint childPoint = [self convertPoint:point toView:childView];
        
        // 又继续交给每一个subview去hitTest
        UIView *fitView = [childView hitTest:childPoint withEvent:event];
        
        // 如果childView的subviews存在能够处理事件的，就返回当前遍历的childView对象作为事件处理对象
        if (fitView) {
            return fitView;
        }
    }
    
    //4. 没有找到比自己更合适的view
    return self;
}
```

可以看到这个`hitTest:withEvent:`函数实现，主要就是测试这个UIView对象，到底能不能够处理这个UI触摸事件。

结束`hitTest:withEvent:`的条件:

- (1) `self.userInteractionEnabled == NO || self.hidden == YES || self.alpha <= 0.01`
- (2) `![self pointInside:point withEvent:event]`

执行`hitTest:`的层次顺序如下:

```
- UIApplication
	- UIWindow
		- RootView
			- Subviews[n-1]
			- Subviews[n-2] 
			- ....
			- Subviews[0]
```

## 使用Category Associate 扩大UI的事件响应区域

```objc
#import <UIKit/UIKit.h>

@interface UIButton (EnlargeTouchArea)

/**
 *  设置按钮上下左右的扩展响应区域
 */
- (void)setEnlargeEdgeWithTop:(CGFloat)top
                        right:(CGFloat)right
                       bottom:(CGFloat)bottom
                         left:(CGFloat)left;

@end
```

```
#import "UIButton+EnlargeTouchArea.h"
#import <objc/runtime.h>

static void *kButtonUpKey = &kButtonUpKey;
static void *kButtonLeftKey = &kButtonLeftKey;
static void *kButtonDownKey = &kButtonDownKey;
static void *kButtonRightKey = &kButtonRightKey;

@implementation UIButton (EnlargeTouchArea)

- (void)setEnlargeEdgeWithTop:(CGFloat)top right:(CGFloat)right bottom:(CGFloat)bottom left:(CGFloat)left
{
    objc_setAssociatedObject(self, kButtonUpKey, @(top), OBJC_ASSOCIATION_ASSIGN);
    objc_setAssociatedObject(self, kButtonLeftKey, @(left), OBJC_ASSOCIATION_ASSIGN);
    objc_setAssociatedObject(self, kButtonDownKey, @(bottom), OBJC_ASSOCIATION_ASSIGN);
    objc_setAssociatedObject(self, kButtonRightKey, @(right), OBJC_ASSOCIATION_ASSIGN);
}

- (CGRect) enlargedRect
{
    NSNumber* topEdge = objc_getAssociatedObject(self, &kButtonUpKey);
    NSNumber* rightEdge = objc_getAssociatedObject(self, &kButtonRightKey);
    NSNumber* bottomEdge = objc_getAssociatedObject(self, &kButtonDownKey);
    NSNumber* leftEdge = objc_getAssociatedObject(self, &kButtonLeftKey);
    
    if (topEdge && rightEdge && bottomEdge && leftEdge)
    {
        // 上下左右分别扩大响应区域
        return CGRectMake(
                          self.bounds.origin.x - leftEdge.floatValue,
                          self.bounds.origin.y - topEdge.floatValue,
                          self.bounds.size.width + leftEdge.floatValue + rightEdge.floatValue,
                          self.bounds.size.height + topEdge.floatValue + bottomEdge.floatValue
                          );
    } else {
        return self.bounds;
    }
}

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    
    // 扩大后的响应区域
    CGRect rect = [self enlargedRect];
    
    // 如果扩大的响应区域 == 当前自身的响应区域，直接执行父类的事件处理
    if (CGRectEqualToRect(rect, self.bounds))
    {
        return [super hitTest:point withEvent:event];
    }
    
    // 扩大的响应区域 > 当前自身的响应区域
    return CGRectContainsPoint(rect, point) ? self : nil;
}

@end
```

## 位移枚举 + Mask掩码

这种枚举适用于一个统一的枚举类型来定义，具备:

- (1) 多种情况
- (2) 每一种情况，又分为其他的小情况

一个简单的demo


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

如下就是分别获取得到 1~8位、9~16位、17~24位 这三个区段的所谓的Mask掩码

```c
0xFF		>>> 1111,1111 >>> 获取低8位值
0xFF00 		>>> 1111,1111,0000,0000 >>> 获取9-16位值
0xFF0000  	>>> 1111,1111,0000,0000,0000,0000 >>> 获取17-24位值
```

使用当前的枚举混合值，通过与Mask掩码，进行`按位与`获取Mask掩码对应长度的值:

```c
(1) & 上 `FF` 获取低8位的值
(2) & 上 `FF00` 获取第9位到16位的值
(3) & 上 `FF0000` 获取第17位到24位的值
```

示例代码

```c
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

//5. 
NSLog(@"state = %ld", state);
NSLog(@"person state = %ld", state & PersonStateMask);
NSLog(@"house state = %ld", state & HouseStateMask);
NSLog(@"car state = %ld", state & CarStateMask);
```

## `-[NSObject class]`、`+[NSObject class]`、`objc_getClass(<#const char *name#>)`的区别

### `-[NSObject class]`源码实现

```c
- (Class) class
{
  return object_getClass(self);
}
```

### `+[NSObject class]`源码实现

```c
+ (Class) class
{
  return self;
}
```

### `objc_getClass(<#const char *name#>)`源码实现

```c
Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();//读取的是isa指针，所指向的objc_class实例
    else return Nil;
}
```

所以对于如下两种使用`object_getClass(obj)`得到的Class是不同的:

```
1. Class cls1 = object_getClass([Person new]);
2. Class cls1 = object_getClass([Person class]);
```

前者是MetaClass，或者是Class。

## Cache缓存数据、在多线程环境下使用的代码模板

```objc
@interface ClassMapper : NSObject
@end

@implementation ClassMapper
+ (instancetype)mapperWithClass:(Class)cls {
    if (Nil == cls) {return nil;}
    
    /**
     *  1. 单例模板控制缓存正确初始化、信号值为1的信号量初始化
     */
    static CFMutableDictionaryRef       _cache = NULL;
    static dispatch_semaphore_t         _semephore = NULL;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _cache = CFDictionaryCreateMutable(kCFAllocatorDefault, 32, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
        _semephore = dispatch_semaphore_create(1);
    });
    
    /**
     *  2. 先查询缓存
     */
    const void *clsName =  (__bridge const void *)(NSStringFromClass(cls));
    dispatch_semaphore_wait(_semephore, DISPATCH_TIME_FOREVER);
    ClassMapper *clsMapper = CFDictionaryGetValue(_cache, clsName);
    dispatch_semaphore_signal(_semephore);
    
    /**
     *  3. 如果有缓存就直接返回，如果没有缓存则创建新的对象并完成缓存
     */
    if (!clsMapper) {
        clsMapper = [ClassMapper new];
        
        dispatch_semaphore_wait(_semephore, DISPATCH_TIME_FOREVER);
        CFDictionarySetValue(_cache, clsName, (__bridge const void *)(clsMapper));
        dispatch_semaphore_signal(_semephore);

    }

    return clsMapper;;
}
@end
```

## Category 并不会`覆盖`原始方法实现，以及多个Category重写相同的方法实现的调用顺序

主要涉及的问题

```
1. Category是否会覆盖原始Class中的Method？
2. 多个Category添加相同的Method，调用顺序是什么？
```

有2个问题：

```
1. Category是否会覆盖原始Class中的Method？
2. 多个Category添加相同的Method，调用顺序是什么？
```

答案是，都是在xcode编译路径出现的越后面的分类中重写的method会被调用，而之前的都不会调用。

为什么？

- (1) 所有的`objc_method`都是存放到一个链表`method_list`中
- (2) 而对于分类中出现的`objc_method`，runtime环境会使用`头插法`将其插入到链表`method_list`中

所以，只是因为最晚出现的method，头插到了第一个位置。然后从第一个位置找到了method之后，就不会再继续往后找了，也就不会调用原始类和出现在之前的分类method了。

在程序运行时，我们写的每一个objc类，在内存中的表现：

```c
- objc_class
	- ivar_list
	- property_list
	- method_list
	- protocol_list
- meta objc_class
	- method_list
```

具体测试，搜索`Category覆盖原始类中的方法实现存在的问题.md`。

## 触发CPU与GPU的离屏渲染的场景

### CPU触发离屏渲染

- (1) 使用`CoreGraphics`库函数进行绘制图像

- (2) 重写`-[UIView drawRect]`方法实现中写的任何绘制代码
    - 甚至是`空方法实现`也会触发

### GPU触发离屏渲染

- (1) CALayer对象设置 shouldRasterize（光栅化）

- (2) CALayer对象设置 masks（遮罩）

- (3) CALayer对象设置 shadows（阴影）

- (4) CALayer对象设置 group opacity（不透明）

- (5) 所有`文字`的绘制（UILabel、UITextView...），包括`CoreText`绘制文字、`TextKit`绘制文字


尽量避免GPU离屏渲染，但是为了能够异步进行绘制，也有可能操作CPU离屏渲染。

## 最好不要重写`-[UIView drawRect:]`来完成文本、图形的绘制，而是使用`专用图层`来专门完成绘制

### 首先清楚，CPU与GPU的强项与弱势：

- (1) CPU、对数据的计算处理相当快，但是对于图像的渲染很差
- (2) GPU、有很多核心来同时做图像的渲染，所以很快。但是对于数据的计算处理，是很慢的


所以，一定要充分利用GPU与CPU的强项：

```
CPU >>> 大量进行数据计算，少进行图像渲染
GPU >>> 大量进行图像渲染，少进行数据计算
```

- (1) `OpenGL`绘制图像，会交给`GPU`完成渲染
- (2) `CoreGraphics`绘制图像，会交给`CPU`完成渲染

所以，最好让CPU只做一些`数据计算`，而CPU只会`图像渲染`，这样整体性能会提升很多。

### 为什么不要重写`drawRect:` 

- (1) 只要重写 drawRect: ，就会给layer立马创建一个contents

- (2) CoreGraphics 的图像绘制，会触发 CPU的离屏渲染 ，而CPU的 图像渲染 能力是很差的，最好不要在主线程上弄

- (3)  专用图层 把图像渲染代码使用OpenGL操作 GPU 来完成图像的渲染，内存优化
	- CAShapeLayer、CATextLayer、CATiledLayer ... 


### 下面是使用专用图层来绘制自定义路径的代码模板，`代替`使用重写`drawRect:`

```objc
@implementation XingNengVC {
    UIView  *_bottomView;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //1. 创建UIView容器
    _bottomView = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 300, 200)];
    [self.view addSubview:_bottomView];
    
    //2. 创建具体绘制图像的专用图层
    CAShapeLayer *layer = [CAShapeLayer layer];
    layer.frame = _bottomView.bounds;
    
    //3. 设置要绘制的图像路径
    UIBezierPath *path = [[UIBezierPath alloc] init];
    [path moveToPoint:CGPointMake(175, 100)];
    [path addArcWithCenter:CGPointMake(150, 100) radius:25 startAngle:0 endAngle:2*M_PI clockwise:YES];
    [path moveToPoint:CGPointMake(150, 125)];
    [path addLineToPoint:CGPointMake(150, 175)];
    [path addLineToPoint:CGPointMake(125, 225)];
    [path moveToPoint:CGPointMake(150, 175)];
    [path addLineToPoint:CGPointMake(175, 225)];
    [path moveToPoint:CGPointMake(100, 150)];
    [path addLineToPoint:CGPointMake(200, 150)];
    
    //4. 将要绘制的路径设置给layer
    layer.path = path.CGPath;
    
    //3.
    [_bottomView.layer addSublayer:layer];
}

@end
```

### 但是还是无法避免，很多时候还是需要自定义绘制，并且也还是有相应的优化方法

- (1) 如果是绘制文本，使用CoreText在子线程预先渲染得到bitmap
- (2) 如果是绘制图片，同样在子线程并使用ImageIO读取解压并解码，然后使用CoreGraphics渲染得到bitmap

总之，尽量的在子线程完成。

## objc对象的 `释放` 与 `废弃` ，是两个 `不同的阶段`

### 释放

应该是释放对象的`持有`，即对objc对象发送`retain\release\autorelase`等消息，修改objc对象的`retainCount`值，但是对象的内存一直都还存在。

释放持有的操作，是`同步`的。

### 废弃

当某个`空闲`时间，系统才会将内存的数据全部擦除干净，然后将这块内存`合并为系统未使用的内存`中。而此时如果程序继续访问该内存块，就会造成程序崩溃。

内存的彻底`废弃`操作，是`异步`的，也就是说有一定的`延迟`。

### 执行了`-[NSObject dealloc]`，并不是说对象所在内存就被`废弃`了。只是对于常理来说，这个对象已经`标记`为即将废弃，程序中也不要再继续使用了。


```objc
- (void)testMRC {

    _mrc = [[MRCTest alloc] init];
    NSLog(@"[_mrc retainCount] = %lu", [_mrc retainCount]);
    
    MRCTest *tmp1 = [_mrc retain];
    NSLog(@"[_mrc retainCount] = %lu", [_mrc retainCount]);
    
    [_mrc release];
    NSLog(@"[_mrc retainCount] = %lu", [_mrc retainCount]);
    
    [tmp1 release];
    NSLog(@"[_mrc retainCount] = %lu", [_mrc retainCount]);
    
    //【重要】尝试多次输出retainCount
    for (NSInteger i = 0; i < 10; i++) {
        NSLog(@"[_mrc retainCount] = %lu", [_mrc retainCount]);//【重要】循环执行几次之后，崩溃到此行
    }
}
```

运行之后，结果崩溃到for循环中的第二次或第三次循环，`程序崩溃`报错如下:

```
thread 1:EXC_BAD_ACCESS .... 
```

释放掉对象之后，指向该对象的指针，仍然会保留在局部方法块的所在栈中，仍然是可以在短暂的时间内继续通过指针访问到对象。但是超过一定时间后，对象才会被彻底废弃掉，这个时候如果还去使用这个指针就会造成程序崩溃。

那这样是说最终对象的内存废弃过程，是一个`异步`执行的吗？或者说有一定的`延迟时间`吗？

是`延迟`的，因为最终对象内存会被擦除掉，并与系统内存合并到一起，所以这个过程确实是一个异步的。	

## objc对象弱引用实现

### NSValue


```objc
//1. NSValue弱引用方式包装一个objc对象
NSValue *value = [NSValue valueWithNonretainedObject:@"objc对象"];

//2. 从NSValue获取弱引用的objc对象
id weakObj = [value nonretainedObjectValue];
```

### `block` + `__weak`

- (1) 返回值类型是id，参数类型是void，的block类型定义

```c
typedef id (^WeakReferenceBlcok)(void);
```

- (2) 将外界传入的需要弱引用处理的对象，借助block，进行`__weak`处理

```c
WeakReferenceBlcok makeWeakReference(id obj) {
    
    //1. 对外界传入的对象进行弱引用
    id __weak weakObj = obj;
    
    //2. 返回一个Block，执行Block后，让外界拿到 __weak 处理后的弱引用用对象
    return ^() {
        return weakObj;
    };
}
```

- (3) 对NSMutableDictionary稍加封装，添加如上的处理代码

```objc
@interface XZHDic : NSObject {
    NSMutableDictionary *_dic;//TODO: 初始化代码那些就不写了.....
}

- (void)weak_setObject:(id)anObject forKey:(NSString *)aKey;
- (id)weak_getObjectForKey:(NSString *)key;

@end
@implementation XZHDic

- (void)weak_setObject:(id)anObject forKey:(NSString *)aKey {
    //1.
    WeakReferenceBlcok block = makeWeakReference(anObject);
    
    //2.
    [_dic setObject:block forKey:aKey];
}

- (id)weak_getObjectForKey:(NSString *)key {
    //1.
    WeakReferenceBlcok block = [_dic objectForKey:key];
    
    //2.
    return (block ? block() : nil);
}

@end
```

### 利用 NSProxy 或 NSObject 的消息转发转发阶段一 `forwardingTargetForSelector:`


```objc
#import <Foundation/Foundation.h>
#import "Human.h"

@interface XZHProxy : NSProxy <Human>

/**
 *  被代理的弱引用对象
 */
@property (nonatomic, weak, readonly) id<Human> target;

/**
 *  传入要被弱引用的对象
 */
- (instancetype)initWithTarget:(id<Human>)target;

@end
@implementation XZHProxy

- (instancetype)initWithTarget:(id<Human>)target {
    _target = target;
    return self;
}

- (id)forwardingTargetForSelector:(SEL)selector {
    return _target;
}

- (BOOL)respondsToSelector:(SEL)aSelector {
    return [_target respondsToSelector:aSelector];
}

@end
```

具体参考`几种弱引用实现方法.md`.


## FMDatabaseQueue解决`dispatch_sync(queue, ^(){});`可能导致多线程死锁

### 主要是如下两个相关函数的使用:

给queue绑定一个标记值

```c
dispatch_queue_set_specific(dispatch_queue_t queue, 
							const void *key,
							void *context, dispatch_function_t destructor);
```	

取出queue绑定的标记值

```c
void * dispatch_get_specific(const void *key);
```

有点类似runtime中的 `objc_setAssociatedObject()`与`objc_getAssociatedObject()`，动态绑定一个对象。

那么在使用`dispatch_sync(){queue, block}`之前，取出queue绑定的标记值，看是否与当前即将要执行`dispatch_sync`任务的的queue绑定的标记值，是否是一样的。

- (1) 如果一样，说明当前即将执行`dispatch_sync`任务的的queue，就是之前的queue
- (2) 如果不一样，说明是不一样的queue

### 当queue时`串行`队列时，造成`dispatch_sync(){}`线程死锁的模板如下:

```objc
假设下面都是使用同一个queue（串行队列才会有问题）

- (void)test1 {
	dispatch_async(queue, ^() {
		[self test2];//让test2处于queue分配的线程
	});
}

- (void)test2 {
	// 当前方法执行已经处于串行queue分配的唯一线程上了

	dispatch_sync(queue, ^() {
		NSLog(@"hello world!");
	});
}
```

### FMDB的具体写法

唯一key定义

```objc
static const void * const kDispatchQueueSpecificKey = &kDispatchQueueSpecificKey;
```

创建一个串行队列来执行数据库的所有操作

```objc
_queue = dispatch_queue_create([[NSString stringWithFormat:@"fmdb.%@", self] UTF8String], NULL);
```

通过key唯一标记，将`当前FMDBDataBaseQueue对象`，绑定给queue

```objc
dispatch_queue_set_specific(_queue, kDispatchQueueSpecificKey, (__bridge void *)self, NULL);
```

在`inDatabase:`方法中即将执行`dispatch_sync(quue, block);`之前，取出queue绑定的标记值，看是否等于`当前FMDBDataBaseQueue对象`。如果相等说明是同一个`串行queue`，就不能执行`dispatch_sync(quue, block);`，否则会出现线程的死锁。

```objc
- (void)inDatabase:(void (^)(FMDatabase *db))block {

	//1. 取出queue绑定的标记值
    FMDatabaseQueue *currentSyncQueue = (__bridge id)dispatch_get_specific(kDispatchQueueSpecificKey);
    
    //2. 标记值必须不能等于当前的FMDBDataBaseQueue对象。
    // 否则让程序崩溃退出
    assert(currentSyncQueue != self && "inDatabase: was called reentrantly on the same queue, which would lead to a deadlock");
    
    //3. 后面是dispatch_sync()的代码.....
    //.......
}
```

### 下面是一个通过定义`c struct`，来代替前面的`FMDBDataBaseQueue对象`作为标记值的例子

Context结构体定义

```c
typedef struct NetworkMetaDataDispatchQueueContext {
    char reversed;
}NetworkMetaDataDispatchQueueContext;
```

定义一个全局存在的结构体实例，用来判断是否一致

```c
static NetworkMetaDataDispatchQueueContext _context;
```

唯一key定义

```objc
static const void * const kNetworkCacheMetaDispatchQueueSpecificKey = &kNetworkCacheMetaDispatchQueueSpecificKey;
```

使用唯一key，给queue绑定全局静态的Context实例

```objc
dispatch_queue_t queue = queue = dispatch_queue_create("queue.xzhnrtwork.incomplet.data.write", attr);
dispatch_queue_set_specific(queue, kNetworkCacheMetaDispatchQueueSpecificKey, (void*)(&_context), NULL);
```

执行`dispatch_sync(queu, block);`之前，取出queue绑定的context，判断是否全全局context一致，避免处于同一线程，使用dispatch_sync()导致线程发生死锁

```objc
- (void)doWork {

	//1.
	if ((void*)(&_context) == dispatch_get_specific(kNetworkCacheMetaDispatchQueueSpecificKey)) {
		return;
	}
	
	//2. 
	dispatch_sync(queue, ^() {
		...........
	});
}
```

## 使用`消息转发阶段1`机制，来模拟`多继承`

### 有三个抽象接口

```objc
@protocol SayChinese <NSObject>
- (void)sayChinese;
@end
```

```objc
@protocol SayEnglish <NSObject>
- (void)sayEnglish;
@end
```

```objc
@protocol SayFranch <NSObject>
- (void)sayFranch;
@end
```

### 内部拥有三个具体实现类，但是我希望不对外暴露实现，只是我内部知道具体实现

```objc
@interface __SayChineseImpl : NSObject <SayChinese>
@end

@implementation __SayChineseImpl
- (void)sayChinese {
    NSLog(@"说中国话");
}
@end
```

```objc
@interface __SayEnglishImpl : NSObject <SayEnglish>
@end

@implementation __SayEnglishImpl
- (void)sayEnglish {
    NSLog(@"说英语话");
}
@end
```

```objc
@interface __SayFranchImpl : NSObject <SayFranch>
@end

@implementation __SayFranchImpl
- (void)sayFranch {
    NSLog(@"说法国话");
}
@end
```

注意，这些实现类是私有的，不向外暴露的。

### 向外暴露`一个类`，来操作上面三个类所有方法实现，即多继承的效果

```objc
/**
 *  模拟继承多种语言
 */
@interface SpeckManager : NSObject <SayChinese, SayEnglish, SayFranch>

@end

@implementation SpeckManager {
    id<SayChinese> _sayC;
    id<SayEnglish> _sayE;
    id<SayFranch> _sayF;
}

- (instancetype)init
{
    self = [super init];
    if (self) {
        _sayC = [__SayChineseImpl new];
        _sayE = [__SayEnglishImpl new];
        _sayF = [__SayFranchImpl new];
    }
    return self;
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    
    if (aSelector == @selector(sayChinese)) {
        return _sayC;
    }
    
    if (aSelector == @selector(sayEnglish)) {
        return _sayE;
    }
    
    if (aSelector == @selector(sayFranch)) {
        return _sayF;
    }
    
    return [super forwardingTargetForSelector:aSelector];
}

@end
```

## type encodings 数据类型的系统存放的字符值

### `objc_property_attribute_t.name[0]` 的type encodings

一个使用`@property`定义的属性正串编码中的，`第一个字符`:

```c
 static const char XZHPropertyAttributeBegin = 'T';//T@\"NSString\",C,N,V_name，作为属性编码的开始符，不作为属性权限修饰符
static const char XZHPropertyAttributeIvarName = 'V';//V_name，表示Ivar的名字，不作为属性权限修饰符
static const char XZHPropertyAttributeCopy = 'C';
static const char XZHPropertyAttributeCustomGetter = 'G';
static const char XZHPropertyAttributeCustomSetter = 'S';
static const char XZHPropertyAttributeDynamic = 'D';
static const char XZHPropertyAttributeEligibleForGarbageCollection = 'P';
static const char XZHPropertyAttributeNonAtomic = 'N';
static const char XZHPropertyAttributeOldStyle = 't';
static const char XZHPropertyAttributeReadOnly = 'R';
static const char XZHPropertyAttributeRetain = '&';
static const char XZHPropertyAttributeWeak = 'W';
```

### `objc_ivar` 的type encodings

```c
static const char XZHIvarTypeUnKnown = _C_UNDEF;//?
static const char XZHIvarTypeObject = _C_ID;//@
static const char XZHIvarTypeClass = _C_CLASS;//#
static const char XZHIvarTypeSEL = _C_SEL;//:
static const char XZHIvarTypeChar = _C_CHR;//c
static const char XZHIvarTypeUnsignedChar = _C_UCHR;//C
static const char XZHIvarTypeInt = _C_INT;//i
static const char XZHIvarTypeUnsignedInt = _C_UINT;//I
static const char XZHIvarTypeShort = _C_SHT;//s
static const char XZHIvarTypeUnsignedShort = _C_USHT;//S
static const char XZHIvarTypeLong = _C_LNG;//l
static const char XZHIvarTypeUnsignedLong = _C_ULNG;//L
static const char XZHIvarTypeLongLong = 'q';
static const char XZHIvarTypeUnsignedLongLong = 'Q';
static const char XZHIvarTypeFloat = _C_FLT;//f
static const char XZHIvarTypeDouble = _C_DBL;//d
static const char XZHIvarTypeLongDouble = 'D';
static const char XZHIvarTypeBOOL = 'B';
static const char XZHIvarTypeVoid = _C_VOID;//v
static const char XZHIvarTypeCPointer = _C_PTR;//^
static const char XZHIvarTypeCString = _C_CHARPTR;//*
static const char XZHIvarTypeCArray = _C_ARY_B;//[
static const char XZHIvarTypeCArrayEnd = _C_ARY_E;//]
static const char XZHIvarTypeCStruct = _C_STRUCT_B;//{
static const char XZHIvarTypeCStructEnd = _C_STRUCT_E;//}
static const char XZHIvarTypeCUnion = _C_UNION_B;//(
static const char XZHIvarTypeCUnionEnd = _C_UNION_E;//)
static const char XZHIvarTypeCBitFields = _C_BFLD;//b
```

### `runtime.h`中定义出的type encoding字符

```c
#define _C_ID       '@'
#define _C_CLASS    '#'
#define _C_SEL      ':'
#define _C_CHR      'c'
#define _C_UCHR     'C'
#define _C_SHT      's'
#define _C_USHT     'S'
#define _C_INT      'i'
#define _C_UINT     'I'
#define _C_LNG      'l'
#define _C_ULNG     'L'
#define _C_LNG_LNG  'q'
#define _C_ULNG_LNG 'Q'
#define _C_FLT      'f'
#define _C_DBL      'd'
#define _C_BFLD     'b'
#define _C_BOOL     'B'
#define _C_VOID     'v'
#define _C_UNDEF    '?'
#define _C_PTR      '^'
#define _C_CHARPTR  '*'
#define _C_ATOM     '%'
#define _C_ARY_B    '['
#define _C_ARY_E    ']'
#define _C_UNION_B  '('
#define _C_UNION_E  ')'
#define _C_STRUCT_B '{'
#define _C_STRUCT_E '}'
#define _C_VECTOR   '!'
#define _C_CONST    'r'
```

注意，`'A'`是一个字符，`"A"`才是一个字符串。

## NSMethodSignature

每一个objc method最终都会编译成对应的c函数实现，其转换格式如下:

<img src="./NSMethodSignature1.png" alt="" title="" width="700"/>

NSMethodSignature 方法签名，正是用来打包上面的所有的types。

```objc
@interface NSMethodSignature : NSObject {
@private
    void *_private;
    void *_reserved[6];
}

// 传入c函数最终的整体编码，构造一个方法签名
+ (nullable NSMethodSignature *)signatureWithObjCTypes:(const char *)types;

//1. 方法的形参总数
@property (readonly) NSUInteger numberOfArguments;

//2. 获取每一个形参的数据类型
- (const char *)getArgumentTypeAtIndex:(NSUInteger)idx NS_RETURNS_INNER_POINTER;

//3. 暂时不知道是干嘛的
@property (readonly) NSUInteger frameLength;

//4. 暂时不知道是干嘛的
- (BOOL)isOneway;

//5. 方法返回值的数据类型的type encoding字符串
@property (readonly) const char *methodReturnType NS_RETURNS_INNER_POINTER;

//6. 返回值的长度。如果大于0，表示非void类型，否则就是void类型
@property (readonly) NSUInteger methodReturnLength;

@end
```

那么可以看到NSMethodSignature包含如下信息

- (1) 方法有多少个参数、以及每一个参数的类型
- (2) 方法的返回值类型

完全可以描述一个最终编译生成的c函数。

## jspatch 替换方法实现的逻辑

<img src="./jspatch.png" alt="" title="" width="700"/>

## objc对象的等同性判断写法模板

重写NSObject的实例方法

- (1) isEqual:
- (2) hash

```objc
@implementation Person

- (instancetype)initWithPid:(NSString *)pid Age:(NSInteger)age Name:(NSString *)name {
    self = [super init];
    if (self) {
        _pid = [pid copy];
        _name = [name copy];
        _age = age;
    }
    return self;
}

- (BOOL)isEqualToPerson:(Person *)person {
    
    //1. 如果这是两个相同对象，直接就等价了
    if (self == person) return YES;
    
    //2. 依次比较对象内部的子数据项
    if (_age != person.age) return NO;
    if (![_name isEqualToString:person.name]) return NO;
    if (![_pid isEqualToString:person.pid]) return NO;
    
    //3. 默认返回YES，表示对象等价
    return YES;
}

- (BOOL)isEqual:(id)object {
    
    //1. 比较对象的类型是否一致
    if ([self class] == [object class]) {
        //2.2 其他对象指针比较、对象子数据项比较交给isEqualToPerson:
        return [self isEqualToPerson:(Person *)object];
    } else {
        //2.1 不是同一个类型的对象
        return [super isEqual:object];
    }
}

- (NSUInteger)hash {
    NSInteger ageHash = _age;
    NSUInteger nameHash = [_name hash];
    NSUInteger pidHash = [_pid hash];
    return ageHash ^ nameHash ^ pidHash;
}

@end
```

## NSArray 与 NSSet 在存储item对象时的区别

### 先看区别

- (1) NSArray 纯粹是依赖 `index` 来存取 item对象

- (2) NSSet 是根据 `-[NSObject hash] 与 -[NSObject isEqual:]` 来存取

- (3) **NSSet压根就没有获取某个位置的item对象的接口，只能是获取全部的items对象**

### NSSet 是 散列表 无顺序的集合，就是一个哈希表的存储结构。我们无法指定 hash 表 存储对象的 位置，而是必须通过 hash表自己 计算一个元素的存储位置的基本思路如下：

假设有一组待加入到set的序列集合

```
{12，13，25，23，38，34，6，84，91}
```

hash表内部，根据传入的item值，计算得到插入位置的算法:

```
address(key) = key % 11
```

那么最终插入到 hash map 的结构如下:

<img src="./hashmap01.jpg" alt="" title="" width="700"/>

可以看到，当对传入的item值，计算的到相同地址时，就需要挨个往后探测，找到一个空的位置，进行当前item值得插入。

而如果对已经存在于hash表中的item值，进行修改，就会造成插入位置的错误。


### 测试一、不重写 hash、`isEqual:` 情况下

```objc
@interface HashObject : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger uid;
@end
@implementation HashObject
@end
```

```objc
- (void)test1 {
    NSMutableSet *set = [NSMutableSet new];
    
    // obj1
    HashObject *obj1 = [HashObject new];
    obj1.name = @"haha 1";
    obj1.uid = 1111;
    NSLog(@"obj1.hash = %ld", obj1.hash);
    
    // obj2
    HashObject *obj2 = [HashObject new];
    obj2.name = obj1.name;
    obj2.uid = obj1.uid;
    NSLog(@"obj2.hash = %ld", obj2.hash);
    
    // add objets
    [set addObject:obj1];
    [set addObject:obj2];
    
    // log set
    NSLog(@"set = %@", set);
}
```

输出

```
2017-03-28 20:05:13.625 Demos[16469:470717] obj1.hash = 106102872327712
2017-03-28 20:05:13.625 Demos[16469:470717] obj2.hash = 106102872328352
2017-03-28 20:05:13.626 Demos[16469:470717] set = {(
    <HashObject: 0x60800003c620>,
    <HashObject: 0x60800003c8a0>
)}
```

即使两个对象的属性值一样，但是hash值，还是不一样的，这就是系统的`hash+isEqual:`的实现。

```objc
- (void)test2 {
    
    // obj1
    HashObject *obj1 = [HashObject new];
    obj1.name = @"haha 1";
    obj1.uid = 1111;
    NSLog(@"obj1.hash = %ld", obj1.hash);
    
    // obj2
    HashObject *obj2 = [HashObject new];
    obj2.name = @"haha 2";
    obj2.uid = 2222;
    NSLog(@"obj2.hash = %ld", obj2.hash);
    
    // modify obj2
    obj2.name = obj1.name;
    obj2.uid = obj1.uid;
    NSLog(@"obj2.hash = %ld", obj2.hash);
    
}
```

输出

```
2017-03-28 20:08:41.122 Demos[16552:473488] obj1.hash = 105553116519328
2017-03-28 20:08:41.122 Demos[16552:473488] obj2.hash = 105553116519136
2017-03-28 20:08:41.122 Demos[16552:473488] obj2.hash = 105553116519136
```

即使后面修改对象的属性值，该对象的hash值还是不会发生变化。

### 测试二、重写 hash、`isEqual:` 情况下

```objc
@interface HashObject : NSObject
@property (nonatomic, copy) NSString *name;
@property (nonatomic, assign) NSInteger uid;
@end
@implementation HashObject

//////////////////////// 重写 hash、isEqual: ////////////////////////

- (NSUInteger)hash {
    return [_name hash] ^ _uid;
}
- (BOOL)isEqual:(id)object {
    if ([object class] == [HashObject class]) {
        return [self isEqualToObject:object];
    }
    return [super isEqual:object];
}

- (BOOL)isEqualToObject:(HashObject*)object {
    
    //1.
    if (self == object) {return YES;}
    
    //2.
    if (![_name isEqualToString:object.name]) {return NO;}
    if (_uid != object.uid) {return NO;}
    
    //3.
    return YES;
}

@end
```

再执行test2的输出

```
2017-03-28 20:10:44.644 Demos[16596:475247] obj1.hash = 9345447525204606
2017-03-28 20:10:44.645 Demos[16596:475247] obj2.hash = 9345447525207748
2017-03-28 20:10:44.645 Demos[16596:475247] obj2.hash = 9345447525204606
```

可以看到，我们自己重写了之后的情况，对象的hash值已经可以发生变化了。

```objc
- (void)test4 {
    NSMutableSet *set = [NSMutableSet new];
    
    // obj1
    HashObject *obj1 = [HashObject new];
    obj1.name = @"haha 1";
    obj1.uid = 1111;
    NSLog(@"obj1.hash = %ld", obj1.hash);
    
    // obj2
    HashObject *obj2 = [HashObject new];
    obj2.name = obj1.name;
    obj2.uid = obj1.uid;
    NSLog(@"obj2.hash = %ld", obj2.hash);
    
    // add objets
    [set addObject:obj1];
    [set addObject:obj2];
    
    // log set
    NSLog(@"set = %@", set);
}
```

输出

```
2017-03-28 20:13:17.798 Demos[16668:477445] obj1.hash = 9345447525204606
2017-03-28 20:13:17.798 Demos[16668:477445] obj2.hash = 9345447525204606
2017-03-28 20:13:17.799 Demos[16668:477445] set = {(
    <HashObject: 0x60000022f560>
)}
```

可以看到，重写之后，set对于相同的hash值，只能保存一个对象了。

我想之前在没有重写hash、`isEqual:`时的对象，进行加入set时，系统的实现可能还加入了对象的`地址`以及其他的参数的hash运算，所以导致最终的hash值，其实并不一样。

```objc
- (void)test5 {
    NSMutableSet *set = [NSMutableSet new];
    
    // obj1
    HashObject *obj1 = [HashObject new];
    obj1.name = @"haha 1";
    obj1.uid = 1111;
    NSLog(@"obj1.hash = %ld", obj1.hash);
    
    // obj2
    HashObject *obj2 = [HashObject new];
    obj2.name = @"haha 2";
    obj2.uid = 2222;
    NSLog(@"obj2.hash = %ld", obj2.hash);
    
    // add objets
    [set addObject:obj1];
    [set addObject:obj2];
    
    // log set
    NSLog(@"set = %@", set);
    
    // modify obj2
    obj2.name = obj1.name;
    obj2.uid = obj1.uid;
    NSLog(@"obj2.hash = %ld", obj2.hash);
    
    // log set
    NSLog(@"set = %@", set);
    
}
```

输出

```
2017-03-28 20:15:38.804 Demos[16703:478839] obj1.hash = 9345447525204606
2017-03-28 20:15:38.805 Demos[16703:478839] obj2.hash = 9345447525207748
2017-03-28 20:15:38.805 Demos[16703:478839] set = {(
    <HashObject: 0x60800022a160>,
    <HashObject: 0x608000226e00>
)}
2017-03-28 20:15:38.805 Demos[16703:478839] obj2.hash = 9345447525204606
2017-03-28 20:15:38.805 Demos[16703:478839] set = {(
    <HashObject: 0x60800022a160>,
    <HashObject: 0x608000226e00>
)}
```

当修改了obj2的属性值后，obj2对象的hash值，已经变为和obj1的hash值是一样的了。

但是此时set中，却存在两个相同hash值得对象，这个违背了 哈希表 数据结构，理论上来说是有问题的。

但是也可以看到，set并不会 **主动的去过滤掉** 某一个相同的对象。

## NSAarray与NSMuatbleArray在alloc、init时的类型是不同的

> NSArray、NSMutableArray中的类簇应用.md

```objc
- (void)testArrayAllocInit {
    
    id obj1 = [NSArray alloc];
    id obj2 = [NSMutableArray alloc];
    
    id obj3 = [obj1 init];
    //id obj4 = [obj1 initWithCapacity:16];//崩溃，因为obj1不是 __NSArrayM
    id obj5 = [obj1 initWithObjects:@"1", nil];
    
    id obj6 = [obj2 init];
    id obj7 = [obj2 initWithCapacity:16];
    id obj8 = [obj2 initWithObjects:@"1", nil];
    
    NSLog(@"");
}
```

输出如下

```
(__NSPlaceholderArray *) obj1 = 0x00007ff79bc04b20
(__NSPlaceholderArray *) obj2 = 0x00007ff79bc06d00
(__NSArray0 *) obj3 = 0x00007ff79bc02eb0
(__NSArrayI *) obj5 = 0x00007ff79be0fb60 @"1 object"
(__NSArrayM *) obj6 = 0x00007ff79be1e350 @"0 objects"
(__NSArrayM *) obj7 = 0x00007ff79be1e380 @"0 objects"
(__NSArrayM *) obj8 = 0x00007ff79be22db0 @"1 object"
```

可以得到：

- (1) 最终使用的并不是NSArray、NSMuatbleArray，而是内部的一些带有`__`的私有类

- (2) `[NSArray alloc]`与`[NSMuatbleArray alloc]`得到都是`__NSPlaceholderArray`

- (3) `[[NSArray alloc] init]`得到的是`__NSArray0`

- (4) `[[NSArray alloc] initWithObjects:@"1", nil]`得到的是`__NSArrayI`

- (5) `NSMuatbleArray`任何的init函数，得到的都是`__NSArrayM`

可以肯定的是，这些内部的私有类，都是NSArray或NSMuatbleArray的`子类`。

```objc
- (void)testArrayAllocInit2 {
    id obj1 = [NSArray alloc];
    id obj2 = [NSMutableArray alloc];
    id obj3 = @[@"1"];
    id obj4 = [obj3 mutableCopy];
    id obj5 = @[];

    Class cls1 = class_getSuperclass([obj1 class]);
    Class cls2 = class_getSuperclass([obj2 class]);
    Class cls3 = class_getSuperclass([obj3 class]);
    Class cls4 = class_getSuperclass([obj4 class]);
    Class cls5 = class_getSuperclass([obj5 class]);
    
    NSLog(@"");
}
```

输出如下

```
(__NSPlaceholderArray *) obj1 = 0x00007fcf8b506cb0
(__NSPlaceholderArray *) obj2 = 0x00007fcf8b5030f0
(__NSArrayI *) obj3 = 0x00007fcf8b41a940 @"1 object"
(__NSArrayM *) obj4 = 0x00007fcf8b41a980 @"1 object"
(__NSArray0 *) obj5 = 0x00007fcf8b5045e0

(Class) cls1 = NSMutableArray
(Class) cls2 = NSMutableArray
(Class) cls3 = NSArray
(Class) cls4 = NSMutableArray
(Class) cls5 = NSArray
```

根据输出可以得到这几种Array的继承结构：

```
- NSArray
	- NSMutableArray
		- __NSPlaceholderArray
		- __NSArrayM
	- __NSArrayI
	- __NSArray0
```

所以类簇最终实现方式以`继承`的方式可能是比较好的一种设计方法。

###  那`-[__NSPlaceholderArray init]`是如何判断到底是生成NSArray还是NSMutableArray的子类对象了?

看了下`《iOS高级内存管理编程指南》`中关于类对象类簇的部分内容，得到如下的伪代码实现:

- (1) `__NSPlacehodlerArray`全局使用一个`单例对象 A`，来记录`[NSArray alloc]`的操作

```c
static __NSPlacehodlerArray *GetPlaceholderForNSArray() {
    static __NSPlacehodlerArray *instanceForNSArray;
    if (!instanceForNSArray) {
        instanceForNSArray = [__NSPlacehodlerArray alloc];
    }
    return instanceForNSArray;
}
```

- (2) `__NSPlacehodlerArray`全局使用一个`单例对象 B`，来记录`[NSMutableArray alloc]`的操作

```c
static __NSPlacehodlerArray *GetPlaceholderForNSMutableArray() {
    static __NSPlacehodlerArray *instanceForNSMutableArray;
    if (!instanceForNSMutableArray) {
        instanceForNSMutableArray = [__NSPlacehodlerArray alloc];
    }
    return instanceForNSMutableArray;
}
```

- (3) NSArray的alloc方法实现

```objc
@implementation NSArray
+ (id)alloc
{
    if (self == [NSArray class]) {
        return GetPlaceholderForNSArray();//获取单例A
    }
}
@end
```

- (4) NSMutableArray的alloc方法实现

```objc
@implementation NSMutableArray
+ (id)alloc
{
    if (self == [NSMutableArray class]) {
        return GetPlaceholderForNSMutableArray();//获取单例B
    }
}
@end
```

- (5) `-[__NSPlacehodlerArray init]`方法实现

```objc
@implementation __NSPlacehodlerArray
- (id)init
{	
    if (self == GetPlaceholderForNSArray()) {//单例A
    	/**
    	 *	重新创建一个__NSArrayI对象返回
    	 */
        self = [[__NSArrayI alloc] init];
    }
    else if (self == GetPlaceholderForNSMutableArray()) {//单例B
    	/**
    	 *	重新创建一个__NSArrayM对象返回
    	 */
        self = [[__NSArrayM alloc] init];
    }
    return self;
}
@end
```


### 新增内部私有实现类的规则

- (1) 新增的实现类，必须继承自抽象父类
- (2) 子类必须实现所有必须的的方法，其他可以新增自己的方法实现

### 对于类簇类进行比较注意

- (1) 不能使用`[obj1 class] == [obj2 class]`，因为永远都不可能相等
- (2) 必须使用`[obj1 isKindOf:[obj2 class]]`，沿着父类一直比较

```objc
- (void)compare {
    id array = @[@"111", @"222", @"3333", @"4444"];
    
    // 错误比较
    if ([array class] == [NSArray class]) {
       NSLog(@"属于数组");// 其实永远都不会执行
    }
    
    // 正确比较
    if ([array isKindOfClass:[NSArray class]]) {
        NSLog(@"属于数组");
    }
}
```

但是如果是针对我们自己编写的一些基本类，并非是类簇的情况下，可以采用上述两种方法进行比较类型，可以使用如下这几种都可以

```objc
- (void)compare {

    Person *person = [Person new];
    
    /**
     *  因为一个NSObejct类在运行时对应的objc_class结构体实例是一个 全局存在的单例
     */
    if ([person class] == [Person class]) {
        NSLog(@"属于Person类");
    }
    
    if ([person isKindOfClass:[Person class]]) {
        NSLog(@"属于Person类");
    }
    
    if (object_getClass(person) == [Person class]) {
        NSLog(@"属于Person类");
    }
    
    if (object_getClass(person) == objc_getClass("Person")) {
        NSLog(@"属于Person类");
    }
}
```


## 使用`类簇`在不同的iOS系统版本下，对一些不同系统版本的系统函数进行适配

### 暴露给客户的统一入口类.h（类簇类）

```objc
#import <Foundation/Foundation.h>

@interface MyTool : NSObject

// 客户只需要关心调用这个方法即可
- (void)doWork;

@end
```

### 入口类.m中，管理各种iOS系统版本下的实现类，并判断当前系统版本，返回对应的实现类对象


```objc
#import "MyTool.h"

// 其他私有的不同系统版本的实现子类
#import "MyTool_iOS6.h"
#import "MyTool_iOS7.h"
#import "MyTool_iOS8.h"

@implementation MyTool

// 判断当前系统版本，选择对应系统版本下的实现类
+ (instancetype)alloc {
    if (self == [MyTool class]) {
        if (floor(NSFoundationVersionNumber) <= NSFoundationVersionNumber_iOS_6_1) {
            //iOS6
            return [MyTool_iOS6 alloc];
        } else if (floor(NSFoundationVersionNumber) > NSFoundationVersionNumber_iOS_6_1 && floor(NSFoundationVersionNumber) < NSFoundationVersionNumber_iOS_8_0) {
            //iOS7
            return [MyTool_iOS7 alloc];
        } else if (floor(NSFoundationVersionNumber) > NSFoundationVersionNumber_iOS_7_1){
            //iOS8及以上
            return [MyTool_iOS8 alloc];
        }
    }
    return [super alloc];
}

- (void)doWork { /* 空实现，由具体子类去实现 */ }

@end
```

### 入口类内部依赖的不同系统下的实现类

- iOS6下的实现类

```objc
#import "MyTool.h"

@interface MyTool_iOS6 : MyTool

@end
```

```objc
#import "MyTool_iOS6.h"

@implementation MyTool_iOS6

- (void)doWork {
    NSLog(@"iOS6 work");
}

@end
```

- iOS7的实现类

```objc
#import "MyTool.h"

@interface MyTool_iOS7 : MyTool

@end
```

```objc
#import "MyTool_iOS7.h"

@implementation MyTool_iOS7

- (void)doWork {
    NSLog(@"iOS7 work");
}

@end
```

- iOS8及以上的实现类

```objc
#import "MyTool.h"

@interface MyTool_iOS8 : MyTool

@end
```

```objc
#import "MyTool_iOS8.h"

@implementation MyTool_iOS8

- (void)doWork {
    NSLog(@"iOS8 work");
}

@end
```

### 最后是客户只需要找到入口类MyTool即可，而不需要知道如上的具体版本下的某一个实现类.

```objc
- (void)test3 {
	
	//1. 
    MyTool *tool = [[MyTool alloc] init];
    
    //2. 
    [tool doWork];
}
```

### 图示小结

<img src="./类簇类应用1.png" alt="" title="" width="700"/>


## 重写`-[UIView setFrame:]`方法，让系统计算完成的frame再进行二次调整

```objc
- (void)setFrame:(CGRect)frame {
    
    //1. 对系统计算完成传入的frame进行内部调整
    CGRect rect = frame;
    rect.size.width += 100;//对宽度调整增加100
    
    //2. 最后让系统设置我们修改过的frame
    [super setFrame:rect];
}
```

也可以重写`layoutSubviews`完成frame计算、设置。

## `__has_include(库/头文件.h)`判断是否导入某个静态库

```
#import "xxx.h" 是从当前工程的编译路径中查找xxx.h
#import <xxx/xxx.h> 从xxx静态库中查找xxx.h
```

```c
#if __has_include(<sqlite3.h>)
//从库中查找 .h
#import <sqlite3.h>
#else
//从编译路径中查找 .h
#import "sqlite3.h"
#endif
```

YYModel.h中的写法


```objc
//是否有 __has_include 这个宏
#ifdef __has_include 

#if __has_include(<YYModel/YYModel.h>)
	//如果存在YYModel库，从库查找.h
	FOUNDATION_EXPORT double YYModelVersionNumber;
	FOUNDATION_EXPORT const unsigned char YYModelVersionString[];
	#import <YYModel/NSObject+YYModel.h>
#else
	//不存在YYModel库，从项目编译路径查找.h
	#import "NSObject+YYModel.h"
#endif

#endif
```

## UIColor转UIImage并且圆角化处理

使用`UIBezierPath`圆角

```objc
@implementation UIImage (XZHAddtions)

- (UIImage *)makeCircularImageWithSize:(CGSize)size
{
	// 1. 图片占据矩形区域的大小
	CGRect circleRect = (CGRect) {CGPointZero, size};
  
 	//2. 开启一个绘图画布
	UIGraphicsBeginImageContextWithOptions(circleRect.size, NO, 0);
  
	//3. 创建圆形的路径
	UIBezierPath *circle = [UIBezierPath bezierPathWithRoundedRect:circleRect cornerRadius:circleRect.size.width/2];
  
  	//4.  剪裁路径之外的区域
	[circle addClip];
  
	//5. 将Image绘制到路径中
  	[self drawInRect:circleRect];
  
	//6. 路径绘制
	[circle stroke];
  
  	//7. 从画布中获取渲染得到的图像
	UIImage *roundedImage = UIGraphicsGetImageFromCurrentImageContext();
  
	//8. 结束画布
	UIGraphicsEndImageContext();
  
  	//9. 返回渲染得到的Image
	return roundedImage;
}

@end
```

还可以通过使用CoreGraphics定义`CGPathRef`完成，代码稍微复杂点。

## 不同类型的`dispatch_queue_t`、`main thread`、`thread pool`他们之间的联系与区别

<img src="./gcd1.png" alt="" title="" width="700"/>

## 一个`dispatch_queue_t`对应多个个`thread`？

<img src="./gcd2.png" alt="" title="" width="700"/>

<img src="./gcd3.jpeg" alt="" title="" width="700"/>

### 串行队列

- (1) async、创建且只创建`1个`子线程
- (2) sync、不使用子线程，而是使用当前线程执行，并且**同步等待**当前线程**执行完毕**，`可以立马拿到返回值`，才会让其他线程进入执行，注意可能会发生线程死锁


### 并发队列

- (1) async、创建`n个`线程，会复用线程，多少个由GCD底层决定
- (2) sync、同上

### 对二种队列分别sync、async的总结

<img src="./gcd4.jpg" alt="" title="" width="700"/>

### 创建`dispatch_queue_t`，也就等同于创建`thread`，那么线程多的影响

<img src="./gcd6.png" alt="" title="" width="700"/>

## 使用GCD来完成多线程同步的模板

### 主要的三点

- (1) 使用 `concurrent queue` 并发队列
- (2) 对于`读取` >>> `dispatch async`
- (3) 对于`写` >>> `dispatch barrier async`

### 代码模板

```objc
//1. 并发队列
dispatch_queue_t concurrentDiapatchQueue=dispatch_queue_create("com.test.queue", DISPATCH_QUEUE_CONCURRENT);
	
//2. 并发无序的任务
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"1 - thread: %@", [NSThread currentThread]);});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"2 - thread: %@", [NSThread currentThread]);});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"3 - thread: %@", [NSThread currentThread]);});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"4 - thread: %@", [NSThread currentThread]);});
	
//3. 需要等待按照循序执行的任务
dispatch_barrier_async(concurrentDiapatchQueue, ^{
    sleep(5); NSLog(@"停止5秒我是同步执行 - thread: %@", [NSThread currentThread]);
});
	
//4. 并发无序的任务
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"6 - thread: %@", [NSThread currentThread]);});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"7 - thread: %@", [NSThread currentThread]);});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"8 - thread: %@", [NSThread currentThread]);});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"9 - thread: %@", [NSThread currentThread]);});
dispatch_async(concurrentDiapatchQueue, ^{NSLog(@"10 - thread: %@", [NSThread currentThread]);});
```

<img src="./gcd5.jpg" alt="" title="" width="700"/>


## 使用`dispatch_semephore_t`来让一段异步代码强制性的同步执行


信号值为1，可用来同步多线程。如果为n，可控制同时最大并发n个线程。


按照执行顺序1、2、3....标记代码的流程。

```objc
//1. 【注意】信号量初始化为0，并不是1
dispatch_semephore_t semephore = dispatch_semaphore_create(0);
	
NSString *name = nil;

//2. 开始异步任务
dispatch_async(dispatch_get_main_queue(), ^{

	//4. 模拟异步任务
    name  = [self.name copy];
    
    //5. 异步任务完成之后，发出信号通知线程往下走
    dispatch_semaphore_signal(semephore);
});
    
//3. 卡住主线程往下走，等待Block执行完毕再往下走
dispatch_semaphore_wait(semephore, DISPATCH_TIME_FOREVER);

//6. 一直等到信号值为1，才会走到下面的代码
NSLog(@"name = %@", name);
```

## 在`for/while`等循环中`加锁`时，需要使用`tryLock`，不能使用`lock`，否则可能会出现线程死锁

```objc
while (!finish) {

    //TODO: 防止在循环中使用pthread_mutex_lock()出现线程死锁，前面的锁没有解除，又继续加锁
    //pthread_mutex_trylock(): 尝试获取锁，获取成功之后才加锁，否则不执行加锁
    
    if (pthread_mutex_trylock(&_lock) == 0) {//获取锁成功，并执行加锁
    
        // 缓存数据读写
        //.....
        
        //解锁
        pthread_mutex_unlock(&_lock);
        
    } else {
    
        //获取锁失败，让线程休眠10秒后再执行解锁
        usleep(10 * 1000);
    }
    
    //解锁
    pthread_mutex_unlock(&_lock);
}
```


注意是执行`加锁`处理的才需要使用`tryLock`。

## 使用原子属性OSAtomic完成多线的排队等待执行

分为32位于64位，对整形变量，在多线程环境下，使用原子性，完成多线程的排队按照顺序的存取。

```c
OSAtomicAdd32(); 加上一个数
OSAtomicAdd32Barrier(); 加上一个数，并使用一个栅栏来防止多线程
OSAtomicIncrement32(); 变量自增
OSAtomicIncrement32Barrier(); 变量自增，并使用一个栅栏来防止多线程
OSAtomicDecrement32(); 变量自减
OSAtomicDecrement32Barrier(); 变量自减，并使用一个栅栏来防止多线程
```

常用也就这几个了。

## CoreAnimation 添加到 CALayer 的写法模板

```objc
- (void)addCircle3LayerAndRotateAnimation {
    
    //1. 创建图层，设置图层位置、大小，添加图层到图层数
    _cirCle3Layer = [CALayer layer];
    _cirCle3Layer.borderWidth = 1;
    
    //2. 只设置layer的width、height
    _cirCle3Layer.frame = CGRectMake(0, 0, 207, 207);
    
    //3. 设置layer的center坐标，默认出现在anchorPoint{0.5, 0.5}的位置,也就是让layer居中显示在父layer中
    // layer.position相当于UIView.center
    _cirCle3Layer.position = CGPointMake(self.view.bounds.size.width/2+1,
                                         self.view.bounds.size.height/4+1);
    
    //4. 添加sub layer
    [self.view.layer addSublayer:_cirCle3Layer];
    
    //5. 设置动画代理
    _cirCle3Layer.delegate = self;
    
    //6. 设置动画
    [_cirCle3Layer setNeedsDisplay];
    
    //7. 创建核心动画，作用在layer某个属性上
    // （默认：动画效果【不会】影响layer原有的属性值）
    CABasicAnimation * animation = [CABasicAnimation animationWithKeyPath:@"transform.rotation"];
    
    //8. 动画时长
    animation.duration = 1.0;
    
    //9. 动画的初始值
    animation.fromValue = [NSNumber numberWithFloat:0];
    
    //10. 动画的结束时的值，旋转360度，即一群
    // 比如，只需要旋转指定angle角度====> [NSNumber numberWithFloat:((angle * M_PI) / 180.f)]
    animation.toValue = [NSNumber numberWithFloat:((360*M_PI)/180.f)];
    
    //8. 动画执行完毕之后的操作
    //kCAFillModeForwards       保持结束时的状态
    //kCAFillModeBackwards      回到开始时的状态
    //kCAFillModeBoth           兼顾以上的两种效果
    //kCAFillModeRemoved        结束时删除效果
    animation.fillMode = kCAFillModeForwards;
    
    //9. 动画的重复次数（不断的重复）
    animation.repeatCount = MAXFLOAT;
    
    //10. 开始动画
    [_cirCle3Layer addAnimation:animation forKey:@"rotation"];
}

-(void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx{
    
    //1. 开启绘图画布
    CGContextSaveGState(ctx);
     
    //2. 不停的在当前Layer的区域中，绘制图像
    
    //2.1
    if (layer == _cirCle3Layer) {
        UIImage * image = [UIImage imageNamed:@"clock_circle3"];
        CGContextDrawImage(ctx, CGRectMake(0, 0, 207, 207), image.CGImage);
    }
    
    //2.2
    if (layer == _shizhenLayer) {
        UIImage * image = [UIImage imageNamed:@"clock_min"];
        CGContextDrawImage(ctx, CGRectMake(0, 0, 50, 50), image.CGImage);
    }
    
    //2.3
    if (layer == _miaozhenLayer) {
        UIImage * image = [UIImage imageNamed:@"clock_time"];
        CGContextDrawImage(ctx, CGRectMake(0, 0, 10, 30), image.CGImage);
    }
    
    //3.
    CGContextRestoreGState(ctx);
}
```


如果`fillMode=kCAFillModeForwards和removedOnComletion=NO`,那么在动画执行完毕后，图层会保持显示动画执行后的状态。

### 注意： 动画执行完毕之后，`图层的属性值`仍然是动画执行之前的数值。

在屏幕上，我们看到的图层的位置、大小...确实已经发生变化了，但其实图层的属性值并没有发生改变，仍然保持动画之前的数值。

可以理解为，屏幕上看到的其实是**假的**，真正的图层还是在**原来**的位置、**原来**的大小、**原来**的颜色透明的，统统都是**原来**的。

## 获取block的类簇类

```c
static Class GetNSBlock() {
    static Class _cls = Nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        id block = ^() {};
        _cls = [block class];
        while (_cls && (class_getSuperclass(_cls) != [NSObject class])) {
            _cls = class_getSuperclass(_cls);
        }
    });
    return _cls;
}
```

也可以用于获取其他类型的类簇类。

## block 循环引用解决之、不使用 `__weak`

测试实体类

```objc
@interface MyDog : NSObject
@property (nonatomic, strong) NSString *name;
@end
@implementation MyDog
- (void)dealloc {
    NSLog(@"%@ - %@ dealloc", self, _name);
}
@end
```

第一种，会自动释放废弃掉的block，捕获外部的对象。

```objc
@implementation BlockViewController

- (void)test9 {
    
    //1.
    MyDog *dog = [MyDog new];
    dog.name = @"ahahha";
    
    //2.
    void (^block)(void) = ^() {
        NSLog(@"name = %@", dog.name);
    };
    
    //3.
    block();
}

@end
```

运行后输出

```
2017-03-22 13:33:42.665 Demos[8797:111915] name = ahahha
2017-03-22 13:33:42.665 Demos[8797:111915] <MyDog: 0x600000007ca0> - ahahha dealloc
```

是可以正常废弃掉外部被持有的对象的。

第二种，block被另外的对象持有时，再捕获外部的对象。

```objc
@implementation BlockViewController {
    void (^_ivar_block)();
}

- (void)test10 {
    
    //1.
    MyDog *dog = [MyDog new];
    dog.name = @"ahahha";
    
    //2.
    _ivar_block = ^() {
        NSLog(@"name = %@", dog.name);
    };
    
    //3.
    _ivar_block();
}

@end
```

运行后输出

```
2017-03-22 13:37:55.803 Demos[8844:114742] name = ahahha
```

并没输出MyDog对象的dealloc信息。

不使用 `__weak` 修饰外部对象的指针，来解决MyDog对象没有被废弃的问题:

```objc
@implementation BlockViewController {
    void (^_ivar_block)();
}

- (void)test11 {
    
    //1.
    MyDog *dog = [MyDog new];
    dog.name = @"ahahha";
    
    //2.
    _ivar_block = ^() {
        NSLog(@"name = %@", dog.name);
    };
    
    //3.
    _ivar_block();
    
    //4. 一定要在最后执行
    _ivar_block = nil;
    
}

@end
```

先看运行输出

```
2017-03-22 13:39:21.803 Demos[8877:116163] name = ahahha
2017-03-22 13:39:21.803 Demos[8877:116163] <MyDog: 0x600000203480> - ahahha dealloc
```

正如上面所说，block其实也是一种NSObject子类的对象。那既然是objc对象，也就会持有其他所有NSObject子类的对象。

block持有objc对象，就是定义block时，在`^(){ .... }`中使用的指针所指向的对象，我个人觉得`^(){ .... }`其实就可以看做是创建一个objc对象，并且retain持有了传入的指针所指向的对象:

```c
//1.
MyDog *dog = [MyDog new];
dog.name = @"ahahha";

//2. 如下就是创建一个block对应的objc对象，并且retain了一次外部的dog对象
// 那么此时，dog对象的retainCount == 2
_ivar_block = ^() {
	NSLog(@"name = %@", dog.name);
};
```

那么既然retain了，就必须在不再使用的时候进行release，只要保证有retain，就必须有release，保持retainCount的有加就有减，即可让外部被捕获的对象正常废弃。


```c
//1.
MyDog *dog = [MyDog new];
dog.name = @"ahahha";

//2. 如下就是创建一个block对应的objc对象，并且retain了一次外部的dog对象
// 那么此时，dog对象的retainCount == 2
_ivar_block = ^() {
	NSLog(@"name = %@", dog.name);
};

//3. 释放废弃掉block对象，那么被retain的dog对象，就会减少一个strong持有，
// 继而就相当于做一次release，就保持了计数器的平衡
_ivar_block = nil;
```

当block被废弃时，MyDog对象就会失去一个strong指向，相当于进行了一次release，然后就正常废弃了。

## 三种定时器的区别: NSTimer、CADisplayLink、`dispatch_source_set_timer(source,start,interval,leeway)`

### NSTimer 的使用demo


```objc
//1. 创建timer，设置间隔回调时间，指定回调函数
NSTimer *timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(action:) userInfo:nil repeats:NO];

//2. 指定timer注册到哪一个runloop下的哪一个mode下
// 关系到是否实时回调
[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];

//3. 不再使用timer
[timer invalidate];
timer = nil;
```

对于NSTimer，严重受到`RunLoop`的影响，注册到不同的mode下，只会当RunLoop处于这个mode时，才会去处理这个timer事件，就会造成某些mode时，不会进行timer的回调。

NSTimer只能够自己制定间隔处理的时间，这个是与CADisplayLink最大的区别。

### CADisplayLink 的使用demo，并让CADisplayLink`弱引用`持有回调对象

通过block实现弱引用对象

```objc
typedef id (^XZHWeakReferenceBlcok)(void);

// 使用block去持有一个 __weak 修饰的对象
static XZHWeakReferenceBlcok MakeWeakReferenceBlcokForTarget(id target) {
    id __weak weakTarget = target;
    return ^() {
        id __strong strongTarget = weakTarget;
        return strongTarget;
    };
}

// 执行block()获取持有的 __weak 对象
static id GetWeakReferenceTargetFromBlock(XZHWeakReferenceBlcok block) {
    if (!block) {return nil;}
    return block();
}
```

CADisplayLink使用

```objc
//1. 创建
_displayLink = [CADisplayLink displayLinkWithTarget:GetWeakReferenceTargetFromBlock(MakeWeakReferenceBlcokForTarget(self)) selector:@selector(_displayLinkDidCallback:)];

//2. 注册到runloop某一个mode
[_displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];

//3. 开始
_displayLink.paused = NO;

//4. 停止
_displayLink.paused = YES;
[_displayLink removeFromRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
[_displayLink invalidate];
_displayLink = nil;
```

同样是受到RunLoop的运行mode的影响，但是与NSTimer的区别是，`CADisplayLink`能让我们保持与`屏幕刷新率同步的频率`去完成一些周期性的事情。

### 基于`dispatch_source_t`的timer source 事件

```c
@implementation ViewController {
    dispatch_source_t _timer;
}

- (void)addTimerSource {
    
    //1. source 回调执行所在的线程队列
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    //2. 创建基于timer类型的source
    _timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0,queue);
    
    //3. 设置回调间隔时间为1秒
    dispatch_source_set_timer(_timer,dispatch_walltime(NULL, 0),1.0*NSEC_PER_SEC, 0);
    
    //4. 回调执行的代码
    dispatch_source_set_event_handler(_timer, ^{
        NSLog(@" task in thread %@", [NSThread currentThread]);
    });
    
    //5. 开始处理source
    dispatch_resume(_timer);
    
    //6. 移除并停止source
    //dispatch_source_cancel(_timer);
    //_timer = nil;
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {

// (lldb) po [NSRunLoop currentRunLoop]

	[self addTimerSource];

// (lldb) po [NSRunLoop currentRunLoop]
	
}

@end
```

执行`[self addTimerSource];`之前的打印runloop信息

```
.....

common mode items = <CFBasicHash 0x7fcdca5090b0 [0x10ffc37b0]>{type = mutable set, count = 17,

.....
```

执行`[self addTimerSource];`之后的打印runloop信息

```
......

common mode items = <CFBasicHash 0x7fcdca5090b0 [0x10ffc37b0]>{type = mutable set, count = 17,

......
```

执行前后，发现 主线程 runloop 的 common mode items 中 source的总个数 并没有发生变化，那真的不是通过runloop来处理的，所以 `dispatch_source_t` 可以不受runloop轮回的影响。

## `dispatch_source_t` 监听系统底层的事件

### Dispatch Source 一共可以监听 `六大类` 事件，总共分为11个类型

```
1. Timer Dispatch Source：定时调度源。
2. Signal Dispatch Source：监听UNIX信号调度源，比如监听代表挂起指令的 SIGSTOP信号。
3. Descriptor Dispatch Source：监听文件相关操作和Socket相关操作的调度源。
4. Process Dispatch Source：监听进程相关状态的调度源。
5. Mach port Dispatch Source：监听Mach相关事件的调度源。
6. Custom Dispatch Source：监听自定义事件的调度源。
```

总共分为11个类型

```c
DISPATCH_SOURCE_TYPE_DATA_ADD：属于自定义事件，可以通过dispatch_source_get_data函数获取事件变量数据，在我们自定义的方法中可以调用dispatch_source_merge_data函数向Dispatch Source设置数据，下文中会有详细的演示。
DISPATCH_SOURCE_TYPE_DATA_OR：属于自定义事件，用法同上面的类型一样。
DISPATCH_SOURCE_TYPE_MACH_SEND：Mach端口发送事件。
DISPATCH_SOURCE_TYPE_MACH_RECV：Mach端口接收事件。
DISPATCH_SOURCE_TYPE_PROC：与进程相关的事件。
DISPATCH_SOURCE_TYPE_READ：读文件事件。
DISPATCH_SOURCE_TYPE_WRITE：写文件事件。
DISPATCH_SOURCE_TYPE_VNODE：文件属性更改事件。
DISPATCH_SOURCE_TYPE_SIGNAL：接收信号事件。
DISPATCH_SOURCE_TYPE_TIMER：定时器事件。
DISPATCH_SOURCE_TYPE_MEMORYPRESSURE：内存压力事件。
```

### 再来看下 创建 dispatch source 函数的 参数含义:

```c
dispatch_source_t dispatch_source_create(
	dispatch_source_type_t type, // (1)
	uintptr_t handle,	// (2)
	unsigned long mask,	// (3)
 	dispatch_queue_t queue	// (4)
);	
```

type：第一个参数用于标识Dispatch Source要监听的事件类型，共有11个类型。


handle：第二个参数是取决于要监听的事件类型。比如：

- (1) 如果是监听 `Mach端口` 相关的事件，那么该参数就是 `mach_port_t` 类型的 `Mach端口号`

- (2) 其他一般情况下，设置为 `0` 就可以了

mask：第三个参数同样取决于要监听的事件类型。比如如果是监听 `文件属性更改` 的事件，那么该参数就标识 `文件的哪个属性`，比如 `DISPATCH_VNODE_RENAME`。

queue：第四个参数设置回调函数所在的队列。

### `dispatch_source_get_handle(source)` 获取 创建的 disaptch source 的 type 值

```c
uintptr_t
dispatch_source_get_handle(dispatch_source_t source);
```

返回值如下

```
 *  DISPATCH_SOURCE_TYPE_DATA_ADD:        n/a
 *  DISPATCH_SOURCE_TYPE_DATA_OR:         n/a
 *  DISPATCH_SOURCE_TYPE_MACH_SEND:       mach port (mach_port_t)
 *  DISPATCH_SOURCE_TYPE_MACH_RECV:       mach port (mach_port_t)
 *  DISPATCH_SOURCE_TYPE_MEMORYPRESSURE   n/a
 *  DISPATCH_SOURCE_TYPE_PROC:            process identifier (pid_t)
 *  DISPATCH_SOURCE_TYPE_READ:            file descriptor (int)
 *  DISPATCH_SOURCE_TYPE_SIGNAL:          signal number (int)
 *  DISPATCH_SOURCE_TYPE_TIMER:           n/a
 *  DISPATCH_SOURCE_TYPE_VNODE:           file descriptor (int)
 *  DISPATCH_SOURCE_TYPE_WRITE:           file descriptor (int)
```

### 还可以注册 dipatch source 被取消时的 回调

```c
void
dispatch_source_set_cancel_handler(
	dispatch_source_t source,
	dispatch_block_t handler
);
```

## RunLoop、RunLoop的基本组成结构

基本结构

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

<img src="./runloop1.png" alt="" title="" width="350"/>

涉及到的主要api

```
__CFRunLoop
	__CFRunLoopMode 1
		CFMutableSetRef _sources0
		CFMutableSetRef _sources1;
		CFMutableArrayRef _observers;
	    CFMutableArrayRef _timers;
	__CFRunLoopMode 2
		CFMutableSetRef _sources0
		CFMutableSetRef _sources1;
		CFMutableArrayRef _observers;
	    CFMutableArrayRef _timers;
	__CFRunLoopMode 3
		CFMutableSetRef _sources0
		CFMutableSetRef _sources1;
		CFMutableArrayRef _observers;
	    CFMutableArrayRef _timers;
```

## RunLoop、NSThread 成功开启 RunLoop（需要添加 RunLoopSource）后的作用

### 开起 RunLoop 之后，只要 RunLoopSource 一直存在，并且 RunLoop 不退出，则有:

- (1) 即使是局部创建的 NSThread对象， 也`不会`被废弃
- (2) 会卡住 NSThread入口函数中 `[runloop run];` 后面的代码执行
- (3) 但不会卡住当前开起RunLoop的NSThread线程对象的执行（神奇）
- (4) NSThread 对象的 状态 一直处于 `isExecuting == YES` 
- (5) NSThread 对象 可以一直的接受 线程任务执行

### 下面是创建一个局部的NSThread对象，并添加 RunLoopSource 然后开起 RunLoop 的测试

自定义的NSThread，在dealloc的时候log信息

```objc
@interface MyThread : NSThread
@end
@implementation MyThread

- (void)dealloc {
    NSLog(@"%@ dealloc", self);
}

@end
```

ViewController测试

```objc
@implementation ThreadViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1.
    MyThread *thread = [[MyThread alloc] initWithTarget:self
                                               selector:@selector(doInitThread)
                                                 object:nil];
    
    //2.
    [thread start];
}

- (void)doInitThread {
    
    //1.
    NSThread *t = [NSThread currentThread];
    
    //2.
    [t setName:@"MyThread"];
    
    //3. Foundation版本、开启当前线程的runloop
    {
        //3.1
        NSRunLoop *runloop = [NSRunLoop currentRunLoop];
        
        //3.2
        NSLog(@"doInitThread >>>> 添加port事件之前");
        
        //3.3 【重要】 注释下面这句添加事件源的代码
        [runloop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        
        //3.4
        NSLog(@"doInitThread >>>> 添加port事件之后, runloop还未开启");
        
        //3.5
        [runloop run];
        
        //3.6
        NSLog(@"doInitThread >>>> runlop开始执行");
    }
    
    //4.
    NSLog(@"MyThread doInitThread >>>> %@", [NSThread currentThread]);
}

@end
```

程序运行后的输出

```
2017-03-29 13:48:21.283 Demos[7740:128487] doInitThread >>>> 添加port事件之前
2017-03-29 13:48:21.283 Demos[7740:128487] doInitThread >>>> 添加port事件之后, runloop还未开启
```

发现并没有看到 MyThread对象的 dealloc输出信息，说明 MyThread对象并没有被废弃掉，还是处于内存中。

并且，doInitThread实现内的流程会卡在 `3.5句` 代码`[runloop run];`地方，不会再往下执行。

那么将添加 runloopsource 和 开起 runloop 的代码注释掉，再看效果:

```objc
@implementation ThreadViewController

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
    //1.
    MyThread *thread = [[MyThread alloc] initWithTarget:self
                                               selector:@selector(doInitThread)
                                                 object:nil];
    
    //2.
    [thread start];
}

- (void)doInitThread {
    
    //1.
    NSThread *t = [NSThread currentThread];
    
    //2.
    [t setName:@"MyThread"];
        
    //3.
    NSLog(@"MyThread doInitThread >>>> %@", [NSThread currentThread]);
}

@end
```

得到的输出

```
2017-03-29 13:52:31.177 Demos[7824:133122] MyThread doInitThread >>>> <MyThread: 0x600000277380>{number = 3, name = MyThread}
2017-03-29 13:52:31.179 Demos[7824:133122] <MyThread: 0x600000277380>{number = 3, name = MyThread} dealloc
```

这次可以看到 NSThread对象的 dealloc输出信息了，说明此时是废弃了。

### 总结：如果想要将自己创建的NSThread对象，永远保持存活状态（并不只是不废弃，而是随时能够接受线程任务执行），就需要给线程做几件事：

- (1) 创建线程的 Runloop （调用 get runloop 的方法即可）
- (2) 创建 RunLoopSource，并且注册到 Runloop 的某一个 mode 下
- (3) 执行 `[runloop run] 或 CFRunLoopRun()` 开起 当前NSThread对象的 Runloop

这样之后，线程的state才会一直是`isExecuting`，否则就是`isFinished`。

## RunLoop、给自行创建的NSThread对象主动创建 AutoreleasePool 

```objc
+ (NSThread *)networkRequestThread {
	static NSThread *_networkRequestThread = nil;
	static dispatch_once_t oncePredicate;
	dispatch_once(&oncePredicate, ^{
	    
		//1. 创建单例NSThread对象
		_networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
		
		//2. 开启线程对象
		[_networkRequestThread start];
	});

	return _networkRequestThread;
}

///NSThread入口函数
+ (void)networkRequestThreadEntryPoint:(id)__unused object {

	// 使用释放池进行包裹
	@autoreleasepool {
		[[NSThread currentThread] setName:@"AFNetworking"];
		NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
		[runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
		[runLoop run];
	}
}
```

对于自己创建的`NSThread`子线程，一定要手动创建释放池。步骤如下:

```
1. 创建NSThread
2. 在NSThread 线程入口函数中完成如下事情
3. 创建runloop [NSRunLoop currentRunLoop]
4. 添加 runloop observer
5. 监听几个状态（ (1)runloop即将进入、(2)runloop即将休息、(3)runloop即将退出）
6. 在 (1) 时，直接创建 NSAutoreleasePool
7. 在 (2) 时，先释放废弃之前的 NSAutoreleasePool，再创建新的 NSAutoreleasePool
8. 在 (3) 时，直接释放废弃 NSAutoreleasePool
```


## RunLoop、给子线程的runloop添加2个Observer，用来给子线程永远保持一个最新鲜的释放池

提供NSThread分类方法创建释放池的入口

```objc
// 标记是否创建过释放池
static NSString *const kXZHNSThreadAutoleasePoolKey      = @"XZHNSThreadAutoleasePoolKey";

// 标记存储释放池数组
static NSString *const kXZHNSThreadAutoleasePoolStackKey = @"XZHNSThreadAutoleasePoolStackKey";

@implementation NSThread (XZHAddtions)

+ (void)xzh_addAutoreleasePool {

    //1. 主线程thread，会自己创建释放池
    if ([NSThread isMainThread]) {return;}
    
    //2. 线程的字典对象
    NSMutableDictionary *threadDic = [NSThread currentThread].threadDictionary;
    
    //2. 是否已经创建了释放池
    if ([threadDic objectForKey:kXZHNSThreadAutoleasePoolKey]) {return;}
    
    //4. 给当前线程创建释放池
    AutoreleasePoolSetup();
    
    //5. 标记当前线程已经创建了释放池
    [threadDic setObject:kXZHNSThreadAutoleasePoolKey forKey:kXZHNSThreadAutoleasePoolKey];
}

@end
```

`AutoreleasePoolSetup()`完成给线程注册释放池，子线程的runloop添加两个observer.

```c
static void AutoreleasePoolSetup() {

    //1. 给runloop添加第一个observer
    AddRunLoopObserverForAutoreleasePoolPush();
    
    //2. 给runloop添加第二个observer
    AddRunLoopObserverForAutoreleasePoolPop();
}
```

给runloop添加第一个observer: `AddRunLoopObserverForAutoreleasePoolPush();`完成创建释放池，监听`kCFRunLoopEntry`，其oder最大，表示优先级最高

```c
static void AddRunLoopObserverForAutoreleasePoolPush() {
    
    /**
     *  注意: 使用CoreFoundation版本的RunLoop的api，因为是线程安全的
     */

    //1. 获取/创建 当前线程的runloop
    CFRunLoopRef runLoop = CFRunLoopGetCurrent();
    
    //2. 创建 runloop observer，监听 `runloop进入`的状态
    /*
    创建runloop observer函数的参数意义:
    CFRunLoopObserverCreate(CFAllocatorRef allocator,
                            CFOptionFlags activities,//监听runloop的哪一个状态
                            Boolean repeats,//重复性不断监听
                            CFIndex order,//RunLoopObserver的优先级，当在Runloop同一运行阶段中有多个CFRunLoopObserver时，根据这个来先后调用，CFRunLoopObserver，默认值是0
                            CFRunLoopObserverCallBack callout,//observer回调c函数
                            CFRunLoopObserverContext *context);//用于给observer回调c函数中，传递参数的context
     */
    
    /**
     *  observer的优先级最大:
     *  0x7FFFFFFF是一个十六进制数
     *  转成二进制数==>0111,1111,1111,1111,1111,1111,1111,1111==>32位二进制数，第一位0，表示正数
     *  而int类型变量占用32个二进制位，第一位是符号位（表示正负数），后面31位是数值位
     *  所以，0x7FFFFFFF表示最大的int类型正数，也可以使用 INT_MAX 代替
     */
    CFIndex order = 0x7FFFFFFF;
    CFRunLoopObserverRef pushObserver = NULL;
    pushObserver = CFRunLoopObserverCreate(CFAllocatorGetDefault(),
                                           kCFRunLoopEntry,
                                           true,
                                           order,
                                           XZHCFRunLoopObserverCallBack,
                                           NULL);
    
    //3. observer注册给runloop，注意runloop mode >>> commom modes
    CFRunLoopAddObserver(runLoop, pushObserver, kCFRunLoopCommonModes);
    
    //4.
    CFRelease(pushObserver);
}
```

给runloop添加第二个observer: `AddRunLoopObserverForAutoreleasePoolPop();`完成释放释放池，监听`kCFRunLoopBeforeWaiting 与 kCFRunLoopExit`，其oder最`小`，表示优先级最`低`

```c
static void AddRunLoopObserverForAutoreleasePoolPop() {

    //1.
    CFRunLoopRef runLoop = CFRunLoopGetCurrent();
    
    //2. runloop observer 优先级最低:
    CFIndex order = -0x7FFFFFFF;
    
    //3. 监听runloop状态: 休眠、退出执行
    CFRunLoopObserverRef popObserver = NULL;
    popObserver = CFRunLoopObserverCreate(CFAllocatorGetDefault(),
                                          kCFRunLoopBeforeWaiting | kCFRunLoopExit,
                                          true,
                                          order,
                                          XZHCFRunLoopObserverCallBack,
                                          NULL);
    
    //4.
    CFRunLoopAddObserver(runLoop, popObserver, kCFRunLoopCommonModes);
    
    //5.
    CFRelease(popObserver);
}
```

两个observer统一走的回调函数


```c
static void XZHCFRunLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *context)
{
	// 根据当前runloop的状态进行不同的操作
    switch (activity) {
            
        //状态一、runloop即将进入
        case kCFRunLoopEntry: {
            XZHAutoreleasePoolPush();// 直接创建新的释放池
        }
            break;
            
        //状态二、runloop即将休眠
        case kCFRunLoopBeforeWaiting: {
            XZHAutoreleasePoolPop();// 先废弃老的释放池
            XZHAutoreleasePoolPush();// 再创建新的释放池
        }
            break;
            
        //状态三、runloop即将退出
        case kCFRunLoopExit: {
            XZHAutoreleasePoolPop();// 直接释放老的释放池
        }
            break;
            
        default:
            break;
    }
}
```

从上面的逻辑可以看出，主要是三种状态：

- (1) runloop即将进入，直接创建释放池
- (2) runloop即将睡眠，先释放掉老的释放池，再重新创建一个新的释放池
- (3) runloop即将退出，直接释放持有的释放池

`XZHAutoreleasePoolPush()`  直接创建新的释放池

```c
static inline void XZHAutoreleasePoolPush() {
    
    //1. 读取线程对象的字典
    NSMutableDictionary *dic =  [NSThread currentThread].threadDictionary;

    //2. 获取线程对象存储的pool数组对象
    CFMutableArrayRef autoreleasePools = (__bridge CFMutableArrayRef)([dic objectForKey:kXZHNSThreadAutoleasePoolStackKey]);
    
    //3. 如果pool数组不存在，则先创建数组对象，并存入到线程字典对象中
    if (!autoreleasePools) {
        autoreleasePools = CFArrayCreateMutable(kCFAllocatorDefault, 1, NULL);
        [dic setObject:(__bridge id)(autoreleasePools) forKey:kXZHNSThreadAutoleasePoolStackKey];
        CFRelease(autoreleasePools);
    }
    
    //4. 创建新的pool对象，并加入到数组最后一个位置
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
    CFArrayAppendValue(autoreleasePools, (__bridge void*)(pool));
}
```

`XZHAutoreleasePoolPop()`完成释放掉现在的释放池

```c
static inline void XZHAutoreleasePoolPop() {

    //1. 获取线程对象存储的pool数组对象
    CFMutableArrayRef autoreleasePools = (__bridge CFMutableArrayRef)([[NSThread currentThread].threadDictionary objectForKey:kXZHNSThreadAutoleasePoolStackKey]);
    
    //2. 获取最后的一个pool
    NSAutoreleasePool *lastPool = CFArrayGetValueAtIndex(autoreleasePools, CFArrayGetCount(autoreleasePools)-1);
    
    //3. 清除pool中的所有的对象
    [lastPool drain];
    
    //4. 移除pool
    CFArrayRemoveValueAtIndex(autoreleasePools, CFArrayGetCount(autoreleasePools)-1);
}
```

## RunLoop、界面重绘（重新绘制）的过程 

### 大致的过程

- (1) 关注 Main RunLoop 的如下状态改变，是MainRunLoop最清闲的时候
	- `kCFRunLoopBeforeWaiting` 即将休息
	- `kCFRunLoopExit` 即将退出

- (2) 当 UIView/CALayer 做出修改后，就会被标记为`待处理`，打包为一个`CATransaction`事务
	- (2.1) UIView/CALayer 的层级发生变化
	- (2.2) 修改 UIView/CALayer 对象的属性值后
	- (2.3) 发送了 UIView/CALayer 的 `setNeedsLayout/setNeedsDisplay`消息

- (3) 参考YYTransaction的结构，理解`CATransaction`的结构为:
	- (3.1) 是哪一个CALayer被比较为`待处理`，需要重新绘制
	- (3.2) 是根据CALayer的哪一个`属性`的最新值，进行重新的绘制

- (4) CATransaction **提交** 后，是先提交到一个全局缓存区，类似于NSSet集合
	- (4.1) 先提交到缓冲区，因为很可能后续还会频繁的发生对同样属性修改后，避免每一次都去进行绘制
	- (4.2) 如果后续再发生同样属性的绘制，先删除之前的transaction，而保存最新的transaction

- (5) 某个时刻， Main RunLoop 处于`kCFRunLoopBeforeWaiting 或 kCFRunLoopExit`状态了
	- (5.1) render server 所有的操作，都是处于一个 **单独的系统进程** 上
	- (5.2) 从全局缓存区，取出所有的transaction绘制事务
	- (5.3) 将所有的transaction绘制事务，发送给 render server
	- (5.4) render server 挨个对每一个transaction进行绘制
	- (5.5) render server 取出trnsaction中指向的是哪一个CALayer
	- (5.6) render server 继而取出 CALayer 的 contents
	- (5.7) render server 然后取出 CALayer 是哪一个 属性值 修改的重绘
	- (5.8) 最后根据 `contents + 属性值数据 + 以及内部私有的状态数据` 进行绘制渲染，并通知显示器显示

	
真正对 CALayer数据 进行绘制和渲染，发生在单独的渲染服务进程上，完成之后直接存储到帧数据缓冲池，并通知显示器读取显示。

### 整个过程中，对于RunLoop的source变化

#### (1) 在执行关注RunLoop状态改变之前

```
(lldb) po [NSRunLoop mainRunLoop]
<CFRunLoop 0x7febf3e04340 [0x107fed7b0]>{wakeup port = 0x1403, stopped = false, ignoreWakeUps = false, 
current mode = UIInitializationRunLoopMode,
common modes = <CFBasicHash 0x7febf3c05540 [0x107fed7b0]>{type = mutable set, count = 2,
entries =>
	0 : <CFString 0x109a80270 [0x107fed7b0]>{contents = "UITrackingRunLoopMode"}
	2 : <CFString 0x10800db60 [0x107fed7b0]>{contents = "kCFRunLoopDefaultMode"}
}
,
common mode items = <CFBasicHash 0x7febf3c04b30 [0x107fed7b0]>{type = mutable set, count = 16,
..............
```

只有16个source。

#### (2) 在执行关注RunLoop状态改变之后

```
(lldb) po [NSRunLoop mainRunLoop]
<CFRunLoop 0x7febf3e04340 [0x107fed7b0]>{wakeup port = 0x1403, stopped = false, ignoreWakeUps = false, 
current mode = UIInitializationRunLoopMode,
common modes = <CFBasicHash 0x7febf3c05540 [0x107fed7b0]>{type = mutable set, count = 2,
entries =>
	0 : <CFString 0x109a80270 [0x107fed7b0]>{contents = "UITrackingRunLoopMode"}
	2 : <CFString 0x10800db60 [0x107fed7b0]>{contents = "kCFRunLoopDefaultMode"}
}
,
common mode items = <CFBasicHash 0x7febf3c04b30 [0x107fed7b0]>{type = mutable set, count = 17,
......
```

有17个source了，比之前多了一个，就是添加的RunLoopObserver。

```
<CFRunLoopObserver 0x7febf3c71010 [0x107fed7b0]>{
valid = Yes, 
activities = 0xa0, 
repeats = Yes, 
order = 16777215,
callout = XZHMainRunLoopObserverCallback (0x10731c520), 
context = <CFRunLoopObserver context 0x0>}
```

并且这个observer分别存在于如下RunLoop Mode

```
1. UITrackingRunLoopMode
2. kCFRunLoopDefaultMode
```

#### (3) 当执行RunLoop状态改变，挨个发送YYTrnsaction对象消息后

```
(lldb) po [NSRunLoop mainRunLoop]
<CFRunLoop 0x7febf3e04340 [0x107fed7b0]>{wakeup port = 0x1403, stopped = false, ignoreWakeUps = false, 
current mode = kCFRunLoopDefaultMode,
common modes = <CFBasicHash 0x7febf3c05540 [0x107fed7b0]>{type = mutable set, count = 2,
entries =>
	0 : <CFString 0x109a80270 [0x107fed7b0]>{contents = "UITrackingRunLoopMode"}
	2 : <CFString 0x10800db60 [0x107fed7b0]>{contents = "kCFRunLoopDefaultMode"}
}
,
common mode items = <CFBasicHash 0x7febf3c04b30 [0x107fed7b0]>{type = mutable set, count = 18,
```

source变成18了，又比之前多了一个 `CFRunLoopSource`。

```
<CFRunLoopSource 0x7febf3c73af0 [0x107fed7b0]>{
signalled = No, 
valid = Yes, 
order = 0, 
context = <CFRunLoopSource MIG Server> 
{port = 15111, subsystem = 0x10dc95f70, context = 0x7febf3d03ba0}}
```

也分别存在于如下 RunLoop mode

```
1. UITrackingRunLoopMode
2. kCFRunLoopDefaultMode
```

并且这个`CFRunLoopSource`是 **sources1** 的类型，因为包含在`sources1`的数组中，如下:

```
sources1 = [
	......
]
```

`sources1` 类型的 `CFRunLoopSource` 包含了一个 `mach_port` 和 一个回调（函数指针），被用于通过内核和其他线程相互发送消息，这种 Source 能主动唤醒 RunLoop 的线程。

因为最终的CALayer的绘制、渲染是在单独的渲染服务进程中执行，所有当渲染服务进程完成了一个绘制渲染任务之后，就通知我们的App程序进程中的主线程，已经完成了。

## delegate是1对1的，只能覆盖式设置，可以预先将别人的delegate保存起来

```objc
static id<UITableViewDelegate> otherDelegate;

@implementation ViewController {
    UITableView *_tv;
}

- (void)test {
    
    //1.
    otherDelegate = _tv.delegate;
    
    //2.
    _tv.delegate = self;
}

- (void)tableView:(UITableView *)tableView didDeselectRowAtIndexPath:(NSIndexPath *)indexPath {
    
    //1. 我们先做点处理
    //......
    
    //2. 再通知其他的delegate回调
    if ([otherDelegate respondsToSelector:@selector(tableView:didDeselectRowAtIndexPath:)]) {
        [otherDelegate tableView:tableView didSelectRowAtIndexPath:indexPath];
    }
}

@end
```

## 程序崩溃监控

### 分为两种崩溃

- (1) `objc` 代码崩溃
- (2) `c/c++` 代码崩溃

对于`(1)`种代码崩溃，一般情况下都能快速定位。而对于`(2)`代码崩溃，直接蹦到main函数中，很难排查。


### 同时监听这两种代码崩溃

```c
void uncaughtExceptionHandler(NSException *exception)
{
    //1. 异常的堆栈信息
    NSArray *stackArray = [exception callStackSymbols];
    
    //2. 出现异常的原因
    NSString *reason = [exception reason];

    //3. 异常名称
    NSString *name = [exception name];
    
    //4. 调用栈详细信息
    NSString *exceptionInfo = [NSString stringWithFormat:@"Exception reason：%@\nException name：%@\nException stack：%@",name, reason, stackArray];
}
```

```c
void SignalCallbackFunc(int code) {
    switch (code) {
        case SIGSEGV: { /* ... */ }
            break;
        case SIGABRT: { /* ... */ }
            break;
        case SIGILL: { /* ... */ }
            break;
        case SIGFPE: { /* ... */ }
            break;
        case SIGBUS: { /* ... */ }
            break;
        case SIGPIPE: { /* ... */ }
            break;
    }
}

void InstallUncaughtExceptionHandler() {
    
    //1. 无效指针、空指针、指针未初始化、栈溢出
    signal(SIGSEGV, SignalCallbackFunc);
    
    //2. 接收到abort()信号
    signal(SIGABRT, SignalCallbackFunc);
    
    //3. 总线错误（什么意思...）
    signal(SIGILL, SignalCallbackFunc);
    
    //4. 数学计算问题，比如 1/0
    signal(SIGFPE, SignalCallbackFunc);
    
    //5. 非法指令，或者没有权限
    signal(SIGBUS, SignalCallbackFunc);
    
    //6. 管道另一边没有进程接受数据
    signal(SIGPIPE, SignalCallbackFunc);
}
```

```objc
@implementation AppDelegate


- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {

    //1. 注册监听objc代码崩溃（同样和覆盖delegate一样，先保存其他的函数指针）
    NSSetUncaughtExceptionHandler(&uncaughtExceptionHandler);
    
    //2. 注册监听c/c++代码崩溃（同样和覆盖delegate一样，先保存其他的函数指针）
    InstallUncaughtExceptionHandler();
    
    return YES;
}

@end
```

## 运行时内存监控

### 通常情况下内存泄露的情况

- (1) UINavigationController 在 pop 一个ViewController对象后，这个ViewController对象没有从内存中废弃

- (2) block与其他对象之间的循环强引用

- (3) delegate与其他对象之间的循环强引用

- (4) objc对象之间的循环强引用

### 主要有两个开源的监测代码

- (1) MLeaksFinder，只能指针基于`UINavigationController`的pop ViewController的检测

- (2) FBMemoryProfiler，比上面的更强大，增加对循环强引用对象的检测

### MLeaksFinder 实现原理

第一步、替换掉UIViewController的如下几个对象方法实现，来跟踪UIViewController对象的声生命周期

```objc
[self swizzleSEL:@selector(viewDidDisappear:) withSEL:@selector(swizzled_viewDidDisappear:)];
[self swizzleSEL:@selector(viewWillAppear:) withSEL:@selector(swizzled_viewWillAppear:)];
[self swizzleSEL:@selector(dismissViewControllerAnimated:completion:) withSEL:@selector(swizzled_dismissViewControllerAnimated:completion:)];
```

第二步、替换掉UINavigationController的如下几个对象方法实现，来监控对VC的push与pop


```objc
[self swizzleSEL:@selector(pushViewController:animated:) withSEL:@selector(swizzled_pushViewController:animated:)];
[self swizzleSEL:@selector(popViewControllerAnimated:) withSEL:@selector(swizzled_popViewControllerAnimated:)];
[self swizzleSEL:@selector(popToViewController:animated:) withSEL:@selector(swizzled_popToViewController:animated:)];
[self swizzleSEL:@selector(popToRootViewControllerAnimated:) withSEL:@selector(swizzled_popToRootViewControllerAnimated:)];
```

第三步、在拦截替换的popVC方法实现中

```objc
__weak id weakSelf = self;
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    __strong id strongSelf = weakSelf;
    [strongSelf assertNotDealloc];
});
```

延迟2秒执行通过分类添加的`assertNotDealloc`方法实现。

`__weak`指针指向的对象被废弃时，会自动赋值为`nil`。所以，如果ViewController对象被废弃，那么2秒后，assertNotDealloc这个消息不会发送，啥事都没有。

而如果两秒后，ViewController对象没有废弃，那么assertNotDealloc消息将会被发送。

而assertNotDealloc消息，就是让程序崩溃。

### FBMemoryProfiler

待学习。

## FPS实时计算

通过`block` + `__weak`实现objc对象的弱引用。

```c

typedef id (^XZHWeakReferenceBlcok)(void);

static XZHWeakReferenceBlcok MakeWeakReferenceBlcokForTarget(id target) {
    id __weak weakTarget = target;
    return ^() {
        id __strong strongTarget = weakTarget;
        return strongTarget;
    };
}
static id GetWeakReferenceTargetFromBlock(XZHWeakReferenceBlcok block) {
    if (!block) {return nil;}
    return block();
}
```

创建CADisplayLink（屏幕刷新定时器），并注册到`主线程的 RunLoop`

```objc
//1.
_displayLink = [CADisplayLink displayLinkWithTarget:GetWeakReferenceTargetFromBlock(MakeWeakReferenceBlcokForTarget(self))
                                                   selector:@selector(_displayLinkDidCallback:)];
                                                   
//2.
[_displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];

//3. 
_displayLink.paused = NO;
```

CADisplayLink没当屏幕进行绘制时，都会回调的函数实现

```objc
- (void)_displayLinkDidCallback:(CADisplayLink *)displayLink {

    //第一次屏幕绘制，只记录开始时间
    if (_lastTime == 0) {
        _lastTime = displayLink.timestamp;
        return;
    }
    
    // 第二次及以后进行屏幕绘制
    _count++;
    CFTimeInterval duration = displayLink.timestamp - _lastTime;
    if (duration < 1.0) {return;}
    _lastTime = displayLink.timestamp;
    
    // 每一次屏幕绘制需要的单位时间 = 屏幕绘制的总次数 / 总时间
    NSInteger fps = lround(_count/duration);
#if DEBUG
    NSLog(@"fps = %ld", fps);
#endif
    
    // 设置fps label显示内容
    _fpsLabel.text = [NSString stringWithFormat:@"%@ FPS", @(fps)];
    if (fps >= kNormalFPS) {
        _fpsLabel.textColor = [UIColor greenColor];
    } else if (fps >= kLowerFPS) {
        _fpsLabel.textColor = [UIColor colorWithRed:255/255.0 green:215/255.0 blue:0/255.0 alpha:1];
    } else {
        _fpsLabel.textColor = [UIColor redColor];
    }
    
    // 清空数据，等待下一次的FPS计算
    _count = 0;
}
```

不再需要监控屏幕刷新频率就移除CADisplayLink事件

```objc
_displayLink.paused = YES;
[_displayLink removeFromRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
[_displayLink invalidate];
```

## 查找当前屏幕上显示的ViewController

```objc
@implementation UIViewController (nevermore)

+ (UIViewController *)nvm_visiableViewController {
  UIViewController *rootViewController = [[[[UIApplication sharedApplication] delegate] window] rootViewController];
  return [UIViewController nvm_topViewControllerForViewController:rootViewController];
}

+ (UIViewController *)nvm_topViewControllerForViewController:(UIViewController *)viewController {
  if ([viewController isKindOfClass:[UITabBarController class]]) {
  	// 递归调用
    return [self nvm_topViewControllerForViewController:[(UITabBarController *)viewController selectedViewController]];
  } else if ([viewController isKindOfClass:[UINavigationController class]]) {
    return [(UINavigationController *)viewController visibleViewController];
  } else {
    if (viewController.presentedViewController) {
    
    	// 递归调用
      return [self nvm_topViewControllerForViewController:viewController.presentedViewController];
    } else {
      return viewController;
    }
  }
}

@end
```

## 利用一个单例 cell对象 进行 frame 计算

```objc
@interface NVMRetailProductCellFrame : NSObject
@property (nonatomic, assign) CGRect flagImageViewFrame;
@property (nonatomic, assign) CGRect productImageViewFrame;
@property (nonatomic, assign) CGRect productNameLabelFrame;
....
@end
@implementation NVMRetailProductCellFrame
@end
```

```objc
@interface NVMRetailProductCell : UITableViewCell

- (NVMRetailProductCellFrame *)computeCellHeightWithGood:(NVMRetailGoodModel*)model {

	//1. 
	NVMRetailProductCellFrame *frameModel = [NVMRetailProductCellFrame new];
	
	//2. subviews frame 计算
	.........frame 计算...........
	
	//3.
	frameModel.height = ......
	
	//4.
	return frameModel;
}

@end
```

```objc
@implementation NVMRetailProductListView

- (void)computeCellFramesWithModelList:(NSArray*)goods flag:(BOOL)flag {

	//1. 
    static NVMRetailProductCell *cell = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        cell = [[NVMRetailProductCell alloc] initWithStyle:UITableViewCellStyleSubtitle reuseIdentifier:@"frames"];
    });
    
    //2. 计算传入的good list frame
    NSMutableArray *frames = [NSMutableArray new];
    for (int i = 0; i < goods.count; i++) {
        NVMRetailGoodModel *good = [goods objectAtIndex:i];
        NVMRetailProductCellFrame *frame = [cell computeCellHeightWithGood:good];
        [frames addObject:frame];
    }
    
    //3.
    if (flag == YES) {
        _frameList = [NSMutableArray new];
    }
    
    //4. 
    [_frameList addObjectsFromArray:frames];
}

@end
```