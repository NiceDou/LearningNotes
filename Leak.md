## iOS运行时，内存检测

...

## MLeaksFinder

### 实现原理

- (1) swizzle交换了NavigationController的Push和Pop相关函数实现

- (2) 当执行`popViewController`时，附加执行如下这段延时3秒的代码

```c
__weak id weakSelf = self;
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    [weakSelf assertNotDealloc];
});
```

如果，3秒后，对象已经废弃，那么`__weak`修饰的指针自动赋值nil，否则就调用`assertNotDealloc`方法实现，让程序崩溃。

- (3) 能够检测内存泄漏的局限性

只能基于`UINavigationConreoller`push和pop的UIViewController对象以及被持有的UIView对象的废弃检查，并不能够对全局任意的对象进行废弃检查。

而且对一些`单例对象`、`内存缓存起来的对象` ，会发生错误检查。