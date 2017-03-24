## 显示动画，就是一些主动触发的动画效果

大致分为几种:

- (1) CABasicAnimation 基础动画
- (2) CAKeyframeAnimation 帧动画
- (3) CAAnimationGroup 动画组，组合各种复杂的动画

## CAAnimation 的一些方法参数

==duration==：动画的持续时间;

==repeatCount==：重复次数，无限循环可以设置HUGE_VALF或者MAXFLOAT;

==repeatDuration==：重复时间;
removedOnCompletion：默认为YES，代表动画执行完毕后就从图层上移除，图形会恢复到动画执行前的状态。如果想让图层保持显示动画执行后的状态，那就设置为NO，不过还要设置fillMode为kCAFillModeForwards;

==fillMode==：决定当前对象在非active时间段的行为。比如动画开始之前或者动画结束之后;

==beginTime==：可以用来设置动画延迟执行时间，若想延迟2s，就设置为CACurrentMediaTime()+2，CACurrentMediaTime()为图层的当前时间;
timingFunction：速度控制函数，控制动画运行的节奏;
delegate：动画代理;

==fillMode==属性值（要想fillMode有效，最好设置removedOnCompletion = NO）

```
kCAFillModeRemoved 这个是默认值，也就是说当动画开始前和动画结束后，动画对layer都没有影响，动画结束后，layer会恢复到之前的状态;
kCAFillModeForwards 当动画结束后，layer会一直保持着动画最后的状态 ;
kCAFillModeBackwards 在动画开始前，只需要将动画加入了一个layer，layer便立即进入动画的初始状态并等待动画开始;
kCAFillModeBoth 这个其实就是上面两个的合成.动画加入后开始之前，layer便处于动画初始状态，动画结束后layer保持动画最后的状态;
速度控制函数(CAMediaTimingFunction)
```

kCAMediaTimingFunctionLinear（线性）：匀速，给你一个相对静态的感觉

kCAMediaTimingFunctionEaseIn（渐进）：动画缓慢进入，然后加速离开

kCAMediaTimingFunctionEaseOut（渐出）：动画全速进入，然后减速的到达目的地

kCAMediaTimingFunctionEaseInEaseOut（渐进渐出）：动画缓慢的进入，中间加速，然后减速的到达目的地。默认的动画行为。

## CAPropertyAnimation 的一些方法参数

是CAAnimation的子类，也是个抽象类，要想创建动画对象，应该使用它的两个子类：CABasicAnimation, CAKeyframeAnimation。


===keyPath=== 通过指定CALayer的一个属性名称为keyPath（NSString类型），并且对CALayer的这个属性的值进行修改，达到相应的动画效果。比如，指定@“position”为keyPath，就修改CALayer的position属性的值，以达到平移的动画效果。

## CABasicAnimation 的一些方法参数

```
fromValue：keyPath相应属性的初始值
toValue：keyPath相应属性的结束值
动画过程说明：
随着动画的进行，在长度为duration的持续时间内，keyPath相应属性的值从fromValue渐渐地变为toValue；
keyPath内容是CALayer的可动画Animatable属性，如果
```

```
fillMode=kCAFillModeForwards
removedOnComletion=NO
```
那么在动画执行完毕后，图层会保持显示动画执行后的状态。但在实质上，**CALayer的属性值还是动画执行前的初始值，并没有真正被改变**。

## CAKeyframeAnimation 的一些方法参数

### CAKeyframeAnimation 与 CABasicAnimation的区别:

CABasicAnimation只能从一个数值（fromValue）变到另一个数值（toValue），而CAKeyframeAnimation会使用一个`NSArray`保存这些数值。

### 具体参数的含义

values：上述的NSArray对象。里面的元素称为“关键帧”(keyframe)。动画对象会在指定的时间（duration）内，依次显示values数组中的每一个关键帧；

path：可以设置一个CGPathRef、CGMutablePathRef，让图层按照路径轨迹移动。path只对CALayer的anchorPoint和position起作用。如果==设置了path，那么values将被忽略==；

keyTimes：可以为对应的关键帧指定对应的时间点，其取值范围为0到1.0，keyTimes中的每一个时间值都对应values中的每一帧。如果没有设置keyTimes，各个关键帧的时间是平分的;

## CAAnimationGroup

动画组，可以保存一组动画对象，将CAAnimationGroup对象加入层后，组中所有动画对象可以同时并发运行。


animations：用来保存一组动画对象的NSArray；


默认情况下，一组动画对象是同时运行的，也可以通过设置动画对象的beginTime属性来更改动画的开始时间。

## CATransition 转场动画

用于做转场动画，能够为层提供移出屏幕和移入屏幕的动画效果。

```
type：动画过渡类型
subtype：动画过渡方向
startProgress：动画起点(在整体动画的百分比)
endProgress：动画终点(在整体动画的百分比)
```

## Core Animation的基本使用步骤

- (1) 首先得有CALayer
- (2) 初始化一个CAAnimation对象，并设置一些动画相关属性
- (3) 通过调用`-[CALayer addAnimation:forKey:]`方法，增加CAAnimation对象到CALayer中，这样就能开始执行动画了
- (4) 通过调用`-[CALayer removeAnimationForKey:]`方法可以停止CALayer中的动画

### 特别注意的是:

CoreAnimation动画执行完毕之后，并不会真正修改UIView、CALayer对象的属性值（position、backgroudColor....）。我们看到的只是屏幕上显示出来的一种假象而已，但其实CALayer的属性值一直都**没发生改变**过。

