---
layout: post
title: 使用 CGPDFDocument、CATiledLayer 和 UIPageViewController 做简单的 PDF 阅读器（二）
date: 2017-02-15
categories: iOS
---
# 0
上文已经讲述了如何使用 CGPDFDocument 和 CATiledLayer 绘制 PDF 文件,现在我们添加手势来缩放 PDF, 然后再使用 UIPageViewController 制作一个翻页效果.
# 单页 PDF 的缩放

```
self.pdfContentView = [[UIView alloc]init];
[self.pdfContentContainerView addSubview:self.pdfContentView];
self.pdfContentLayer = [[PDFContentTiledLayer alloc]init];
self.pdfContentLayer.delegate = self;
[self.pdfContentView.layer addSublayer:self.pdfContentLayer];
```
我们将单页 PDF 显示在 PDFContentTiledLayer 然后将 PDFContentTiledLayer 放在 pdfContentView 内显示出来.
因为我们的 pdfContentView 是使用 autolayout 进行布局的,并且支持横屏和竖屏显示,所以在设备方向发生变化时, pdfContentView 的会触发 layout, 我们要在 layout 之后,重置 PDF 缩放状态.

```
- (void)viewDidLayoutSubviews{
    [self adjustPdfContentFrame];
    [self centerScrollViewContent];
    [self drawThumb];
    [super viewDidLayoutSubviews];
}
```

重新计算 PDF 页面的缩放状态

```
//调整 pdfContentView 将 PDF 缩放到 1x
- (void)adjustPdfContentFrame{
    CGSize screenSize = [UIScreen mainScreen].bounds.size;
    CGRect maxContentRect = self.pdfContentContainerView.bounds;
    if (_pdfPage != nil) {
        CGRect pdfPageOriginalPageRect = CGPDFPageGetBoxRect(_pdfPage, PDFContentDefaultPDFBox);
        CGSize adjustPageSize = maxContentRect.size;
        double contentRate = maxContentRect.size.width/(maxContentRect.size.height > 0?maxContentRect.size.height:screenSize.height);
        double pdfOriginalRate = pdfPageOriginalPageRect.size.width/(pdfPageOriginalPageRect.size.height > 0? pdfPageOriginalPageRect.size.height:screenSize.height);
        //判断宽高比
        //pdf 原始宽度大于显示区域宽度,需要将宽度缩放到显示区域宽度,并且相应的缩放高度
        if (pdfOriginalRate > contentRate) {
            double scaleRate = pdfPageOriginalPageRect.size.width/maxContentRect.size.width;
            double scaleWidth = maxContentRect.size.width;
            double scaleHeight = pdfPageOriginalPageRect.size.height/(scaleRate > 0?scaleRate:1);
            adjustPageSize = CGSizeMake(scaleWidth, scaleHeight);
        }else{
            double scaleRate = pdfPageOriginalPageRect.size.height/maxContentRect.size.height;
            double scaleWidth = pdfPageOriginalPageRect.size.width/(scaleRate > 0?scaleRate:1);
            double scaleHeight = maxContentRect.size.height;
            adjustPageSize = CGSizeMake(scaleWidth, scaleHeight);
        }
        
        self.pdfContentView.frame = CGRectMake(0, 0, adjustPageSize.width, adjustPageSize.height);
        self.pdfContentContainerView.contentSize = self.pdfContentView.bounds.size;
        if (!CGRectEqualToRect(self.pdfContentLayer.bounds, self.pdfContentView.layer.bounds)) {
            self.pdfContentLayer.frame = self.pdfContentView.layer.bounds;
            //重新计算之后, CATiledLayer 需要重新绘图,这里重新生成缩略图避免闪屏
            [self drawThumb];
            [self.pdfContentLayer setNeedsDisplay];
        }
    }else{
        self.pdfContentView.frame = CGRectZero;
        self.pdfContentLayer.frame = self.pdfContentView.layer.bounds;
        self.pdfContentContainerView.contentSize = self.pdfContentView.bounds.size;
    }
}
```

```
//保证 pdf 在 ScrollView 中间位置
- (void)centerScrollViewContent
{
    CGFloat iw = 0.0f; CGFloat ih = 0.0f; // Content width and height insets
    
    CGSize boundsSize = self.pdfContentContainerView.bounds.size;
    CGSize contentSize = self.pdfContentContainerView.contentSize; // Sizes
    
    if (contentSize.width < boundsSize.width) iw = ((boundsSize.width - contentSize.width) * 0.5f);
    
    if (contentSize.height < boundsSize.height) ih = ((boundsSize.height - contentSize.height) * 0.5f);
    
    UIEdgeInsets insets = UIEdgeInsetsMake(ih, iw, ih, iw); // Create (possibly updated) content insets
    
    if (UIEdgeInsetsEqualToEdgeInsets(self.pdfContentContainerView.contentInset, insets) == false){
        self.pdfContentContainerView.contentInset = insets;
    }
}
```

将 PDF 页重置后,就开始进行缩放了.
为了将左滑右滑翻页结合起来,我们不把手势放在 PDFContentViewController 类里面,而是把这些手势定义在 PDFViewController 里面.
这个 PDFViewController 的功能主要有如下几条:
 1. 读取 PDF 文件
 2. 从 PDF 文件中读取一个页面
 3. 添加滑动手势,控制翻页.添加双击手势进行放大
 4. 实现翻页效果.
 总之这个 PDFViewController 作为 PDFContentViewController 的容器,完成 PDF 显示之外的控制工作.
 
 当然, PDFContentViewController 本身要有缩放功能,并把这些功能的接口暴露出来给 PDFViewController 使用.
 由于 PDFContentViewController 和是使用 UIScrollView 来进行缩放的,这里和 PDFViewController 的翻页功能产生了一定的冲突,这里先不提,我们先把缩放效果做完.

 翻页很简单,只要实现 UIScrollview 的 delegate 就可以了,其实,不过不需要兼顾翻页效果,也不用处理 UIScrollview 的 delegate.

 ```
 #pragma UIScrollViewDelegate

- (void)scrollViewDidZoom:(UIScrollView *)scrollView{
    //NSLog(@"%@",scrollView);
}

- (void)scrollViewDidEndZooming:(UIScrollView *)scrollView withView:(UIView *)view atScale:(CGFloat)scale{
    NSLog(@"scrollViewDidEndZooming %@   %@   %lf",scrollView,view,scale);
    [self centerScrollViewContent];
}

- (UIView *)viewForZoomingInScrollView:(UIScrollView *)scrollView{
    return self.pdfContentView;
}
```

这里只有一个要注意的就是,每次缩放后,把 PDF 页面中心放到屏幕中间,否则用起来会怪怪的.

```
    [self centerScrollViewContent];
```

缩放就完成了.

# 翻页效果,以及处理翻页动画与 UIScrollView 缩放之间产生的冲突
## 翻页效果的制作
iOS 上实现翻页效果，可以使用系统带的 UIPageViewController，方便快捷，效果非常好。我的直接领导，公司移动端的负责人，现在负责 Android 客户端开发，为了实现相同的效果，花了好几天时间用 openGL 画了一个。哎，确实比我牛好多。
使用 UIPageViewController 需要在它的代理方法(UIPageViewControllerDataSource UIPageViewControllerDelegate)中提供一系列 UIViewController 作为 childViewController 供显示内容用。
1. 直接翻页
```
//直接切换页面
- (void)switchToPage:(NSInteger)pageNumber animated:(BOOL)animated{
    self.pdfCurrentViewController = [self showPage:pageNumber];
    if (self.pdfCurrentViewController != nil) {
        _currentPageNum = pageNumber;
        [self.pdfReaderViewController setViewControllers:@[self.pdfCurrentViewController] direction:UIPageViewControllerNavigationDirectionForward animated:animated completion:nil];
    }
    [self preparePageBuffer];
}
```
`[self.pdfReaderViewController setViewControllers:@[self.pdfCurrentViewController] direction:UIPageViewControllerNavigationDirectionForward animated:animated completion:nil];`
第一个参数是 UIViewController 的 Array，作为 UIPageViewController 数据的 buffer，第二个和第三个参数当然是供做动画使用了。
这里作为 data 的 array 我只传了一个 ViewController，是因为 UIPageViewController 有 UIPageViewControllerDataSource 这个代理可以动态的提供 buffer 数据。当然了，buffer 的具体内容还是要自己生成的，我的代码中有 [self preparePageBuffer]; 这个方法根据成员变量 _currentPageNum 来生成显示前一页和后一页的 UIViewController，这里翻页之后，当然要调用一下啦。
2. 更新 buffer
```
- (nullable UIViewController *)pageViewController:(UIPageViewController *)pageViewController viewControllerBeforeViewController:(UIViewController *)viewController{    
    if (self.pdfPreviousViewController != nil) {
        PDFContentViewController* tmpPage = self.pdfPreviousViewController;
        return tmpPage;
    }
    return nil;
}

- (nullable UIViewController *)pageViewController:(UIPageViewController *)pageViewController viewControllerAfterViewController:(UIViewController *)viewController{
    if (self.pdfNextViewController != nil) {
        PDFContentViewController* tmpPage = self.pdfNextViewController;
        return tmpPage;
    }
    return nil;
}
```
这两个方法就是 delegate 里面动态提供页面 buffer 的方法了。
这里需要注意的是：
**返回的页面应该是当前显示页面下一页的下一页**
解释一下：如果当前显示的页面是 “3”，那么- (nullable UIViewController *)pageViewController:(UIPageViewController *)pageViewController viewControllerBeforeViewController:(UIViewController *)viewController 应该返回第 “5” 页。
这样做的原因就是，手势翻动第三页的时候，第四页内容已经露出来了，要做动画效果么，那么 buffer 里面当然应该放第五页喽，这样才能实现连续的翻页效果。
那么，
3. 生成新的 buffer
这里跟上面是一起的，上面只是用生成好的页面刷新 UIPageViewController 的 buffer，这里才是生成页面的代码。
```
- (void)pageViewController:(UIPageViewController *)pageViewController willTransitionToViewControllers:(NSArray<UIViewController *> *)pendingViewControllers{
    for (UIViewController* vc in pageViewController.viewControllers) {
        if ([vc isKindOfClass:[PDFContentViewController class]]) {
            PDFContentViewController* tmpvc = (PDFContentViewController*)vc;
            [tmpvc resumeScale];
        }
    }
    if (pendingViewControllers.count > 0) {
        PDFContentViewController* currentPage = (PDFContentViewController*)[pendingViewControllers lastObject];
        _currentPageNum = currentPage.pageNumber;
        [self preparePageBuffer];
    }
}
```
每次翻页的时候，都会回调这个方法。这里面 pendingViewControllers 参数传入的是即将显示的页面。比如说：当前阅读的是第三页，然后向后翻页，pendingViewControllers 这个参数传来的是第四页。这个时候，需要刷新 buffer 提前将第五页生成。
之所以 pendingViewControllers 这里是 NSArray，是因为 UIPageViewController 在上下翻页的时候，可能一次显示多个页面。（我的项目实现的是左右翻页，这个 NSArray 里面每次只包含一个页面）。
如果一次翻好几页，这个方法也会调用几次，只要每次都生成 buffer 提供给 UIPageViewController 就能实现连续翻页的效果。

4. 处理一下放弃翻页的情况
有时候，用户翻页到一半放弃了，这时候就需要特殊处理一下。
```
- (void)pageViewController:(UIPageViewController *)pageViewController didFinishAnimating:(BOOL)finished previousViewControllers:(NSArray<UIViewController *> *)previousViewControllers transitionCompleted:(BOOL)completed{    
    if (finished == YES && completed == NO) {
        if (previousViewControllers.count > 0) {
            PDFContentViewController* currentPage = (PDFContentViewController*)[previousViewControllers lastObject];
            _currentPageNum = currentPage.pageNumber;
            [currentPage resumeScale];
            [self preparePageBuffer];
        }
    }
}
```
当翻页动画完成后，会回调这个方法。completed 参数就是表示用户是否完成了翻页(completed == NO 其实就是 Gesture 变成了 Cancel 状态)。
如果没有完成翻页，需要刷新一下 buffer。
## 解决 UIScrollView 与翻页之间的冲突
这里需要在翻页之前把 UIScrollview 的 scale 恢复成1，否则看起来怪怪的。
有两种情况，一个是 UIScrollview 在放大状态点击翻页，另一个是横屏的时候，UIScrollview 与横向充满屏幕，这个时候也处于放大状态，但是可以出发滑动翻页。
这个暂时的解决方案就是在- (void)pageViewController:(UIPageViewController *)pageViewController willTransitionToViewControllers:(NSArray<UIViewController *> *)pendingViewControllers 里面，把 UIScrollview 的 scale 设置成1
```
- (void)resumeScale{
    [self.pdfContentContainerView setZoomScale:1 animated:NO];
}
```
但是在横屏状态下，偶尔出现 UIScrollview 的 anchorpoint 错误的情况，留待以后解决吧。

# 下载 PDF 文件的页面
这里就是另一个话题了这篇文章就先不讲了:-D

[http://stackoverflow.com/questions/13248282/uipageviewcontroller-transition-unbalanced-calls-to-begin-end-appearance-transi](http://stackoverflow.com/questions/13248282/uipageviewcontroller-transition-unbalanced-calls-to-begin-end-appearance-transi)
[http://blog.csdn.net/lqq200912408/article/details/50412893](http://blog.csdn.net/lqq200912408/article/details/50412893)

