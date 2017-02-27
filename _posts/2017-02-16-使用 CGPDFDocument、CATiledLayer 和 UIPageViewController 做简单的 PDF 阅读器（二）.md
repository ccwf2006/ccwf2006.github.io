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


## 解决 UIScrollView 与翻页之间的冲突


# 下载 PDF 文件的页面
这里就是另一个话题了这篇文章就先不讲了:-D



