---
layout: post
title: 2017-04-24-iOS 中 RunLoop 工作原理以及 NSRunloop 的使用
date: 2017-02-26
categories: iOS
---



# Animation 控制进度

Customizing the Timing of an Animation

Timing is an important part of animations, and with Core Animation you specify precise timing information for your animations through the methods and properties of the CAMediaTiming protocol. Two Core Animation classes adopt this protocol. The CAAnimation class adopts it so that you can specify timing information in your animation objects. The CALayer also adopts it so that you can configure some timing-related features for your implicit animations, although the implicit transaction object that wraps those animations usually provides default timing information that takes precedence.

When thinking about timing and animations, it is important to understand how layer objects work with time. Each layer has its own local time that it uses to manage animation timing. Normally, the local time of two different layers is close enough that you could specify the same time values for each and the user might not notice anything. However, the local time of a layer can be modified by its parent layers or by its own timing parameters. For example, changing the layer’s speed property causes the duration of animations on that layer (and its sublayers) to change proportionally.

To assist you in making sure time values are appropriate for a given layer, the CALayer class defines the convertTime:fromLayer: and convertTime:toLayer: methods. You can use these methods to convert a fixed time value to the local time of a layer or to convert time values from one layer to another. The methods take into account the media timing properties that might affect the local time of the layer and return a value that you can use with the other layer. Listing 5-3 shows an example that you should use regularly to get the current local time for a layer. The CACurrentMediaTime function is a convenience function that returns the computer’s current clock time, which the method takes and converts to the layer’s local time.

Listing 5-3  Getting a layer’s current local time
CFTimeInterval localLayerTime = [myLayer convertTime:CACurrentMediaTime() fromLayer:nil];
Once you have a time value in the layer’s local time, you can use that value to update the timing-related properties of an animation object or layer. With these timing properties, you can achieve some interesting animation behaviors, including:

Use the beginTime property to set the start time of an animation. Normally, animations begin during the next update cycle. You can use the beginTime parameter to delay the animation start time by several seconds. The way to chain two animations together is to set the begin time of one animation to match the end time of the other animation.
If you delay the start of an animation, you might also want to set the fillMode property to kCAFillModeBackwards. This fill mode causes the layer to display the animation’s start value, even if the layer object in the layer tree contains a different value. Without this fill mode, you would see a jump to the final value before the animation starts executing. Other fill modes are available too.

The autoreverses property causes an animation to execute for the specified duration and then return to the starting value of the animation. You can combine this property with the repeatCount property to animate back and forth between the start and end values. Setting the repeat count to a whole number (such as 1.0) for an autoreversing animation causes the animation to stop on its starting value. Adding an extra half step (such as a repeat count of 1.5) causes the animation to stop on its end value.
Use the timeOffset property with group animations to start some animations at a later time than others.
Pausing and Resuming Animations

To pause an animation, you can take advantage of the fact that layers adopt the CAMediaTiming protocol and set the speed of the layer’s animations to 0.0. Setting the speed to zero pauses the animation until you change the value back to a nonzero value. Listing 5-4 shows a simple example of how to both pause and resume the animations later.

Listing 5-4  Pausing and resuming a layer’s animations
-(void)pauseLayer:(CALayer*)layer {
   CFTimeInterval pausedTime = [layer convertTime:CACurrentMediaTime() fromLayer:nil];
   layer.speed = 0.0;
   layer.timeOffset = pausedTime;
}
 
-(void)resumeLayer:(CALayer*)layer {
   CFTimeInterval pausedTime = [layer timeOffset];
   layer.speed = 1.0;
   layer.timeOffset = 0.0;
   layer.beginTime = 0.0;
   CFTimeInterval timeSincePause = [layer convertTime:CACurrentMediaTime() fromLayer:nil] - pausedTime;
   layer.beginTime = timeSincePause;
}


//可以对某个 layer 上的动画设置 speed 和 timeOffset
//如果对layer 设置 speed 和 timeOffset, 会影响到该 layer 以及子 layer 的所有动画
//注意,每个 layer 都有相对于父 layer 的 time, 需要计算.(一般转换后都是0)

CAAnimationGroup

Animating Multiple Changes Together

If you want to apply multiple animations to a layer object simultaneously, you can group them together using a CAAnimationGroup object. Using a group object simplifies the management of multiple animation objects by providing a single configuration point. Timing and duration values applied to the group override those same values in the individual animation objects.

Listing 3-4 shows how you would use an animation group to perform two border-related animations at the same time and with the same duration.

Listing 3-4  Animating two animations together
// Animation 1
CAKeyframeAnimation* widthAnim = [CAKeyframeAnimation animationWithKeyPath:@"borderWidth"];
NSArray* widthValues = [NSArray arrayWithObjects:@1.0, @10.0, @5.0, @30.0, @0.5, @15.0, @2.0, @50.0, @0.0, nil];
widthAnim.values = widthValues;
widthAnim.calculationMode = kCAAnimationPaced;
 
// Animation 2
CAKeyframeAnimation* colorAnim = [CAKeyframeAnimation animationWithKeyPath:@"borderColor"];
NSArray* colorValues = [NSArray arrayWithObjects:(id)[UIColor greenColor].CGColor,
            (id)[UIColor redColor].CGColor, (id)[UIColor blueColor].CGColor,  nil];
colorAnim.values = colorValues;
colorAnim.calculationMode = kCAAnimationPaced;
 
// Animation group
CAAnimationGroup* group = [CAAnimationGroup animation];
group.animations = [NSArray arrayWithObjects:colorAnim, widthAnim, nil];
group.duration = 5.0;
 
[myLayer addAnimation:group forKey:@"BorderChanges"];
A more advanced way to group animations together is to use a transaction object. Transactions provide more flexibility by allowing you to create nested sets of animations and assign different animation parameters for each. For information about how to use transaction objects, see Explicit Transactions Let You Change Animation Parameters.

//设置 Group, 似乎只有 duration 可以用, speed 和 timeOffset 都不起作用.

//修改 animation 必须用[displayLabel.layer animationForKey:@"rotationAnimation"];
//不能直接持有引用,因为添加到 layer 后, animation 会被复制



# 转场
http://www.cnblogs.com/breezemist/archive/2013/12/05/3460497.html
https://bestswifter.com/custom-transition-animation/
http://stackoverflow.com/questions/26569488/navigation-controller-custom-transition-animazhua

//自定义转场
https://developer.apple.com/library/content/featuredarticles/ViewControllerPGforiPhoneOS/CustomizingtheTransitionAnimations.html

   [transitionContext completeTransition:success]; 这个传 YES 会调用 push***complete

   //如果转场没完成,不需要自己恢复(barbutton 不知道为什么)
   //似乎是这种情况下 UINavigationController 一直持有旧的 UINavigationItem(设置下 title 就好了,不知道是为什么)

   //注意,这个底层实现可能是修改了主 layer 的 speed, 会导致子 layer 的动画停止

//ViewController 交互式翻页,当手势判断 success 时,动画需要一段时间执行完,这个时候,不应该再手势启动一段动画(会崩溃)

snapshotViewAfterScreenUpdates//iOS7新增 api 可以方别对某个 Viwe 截屏
drawViewHierarchyInRect:afterScreenUpdates:
(似乎截屏后的 frame 也不会变,专为动画设计)


CGContextAddArc(Context, CGFloat x , CGFloat y, CGFloat radius, CGFloat startAngle , CGFloat endAngle, int clockwise);

xy 原点坐标 radius 半径 stargAngle 和 endAngle 传弧度 colckwise(0是逆时针，1是顺时针)

在 CoreGraphics 中 坐标系是 x 向右, y 向上 angle 的0在 x 上,向逆时针方向增长(与数学定义一致)
如果用 UIGraphicsImageContext 获取 context,y 轴的 transform 是翻转的(y 向下)
但是CGContextAddArc 绘制的时候,是不处理 transform(也就是按照 CoreGraphics 坐标系绘图完成后再应用 transform)
绘制带圆弧的 path 需要注意这个!(最好先按照 CoreGraphics坐标系生成一个 CGPath)

repeatCount = repeatDuration/duration(似乎是)

        blueAnim.removedOnCompletion = NO;//如果不设置成 NO, 退到后台动画就没了





        在关键帧动画中还有一个非常重要的参数,那便是calculationMode,计算模式.其主要针对的是每一帧的内容为一个座标点的情况,也就是对anchorPoint 和 position 进行的动画.当在平面座标系中有多个离散的点的时候,可以是离散的,也可以直线相连后进行插值计算,也可以使用圆滑的曲线将他们相连后进行插值计算. calculationMode目前提供如下几种模式：
kCAAnimationLinear calculationMode的默认值,表示当关键帧为座标点的时候,关键帧之间直接直线相连进行插值计算; kCAAnimationDiscrete 离散的,就是不进行插值计算,所有关键帧直接逐个进行显示;  kCAAnimationPaced 使得动画均匀进行,而不是按keyTimes设置的或者按关键帧平分时间,此时keyTimes和timingFunctions无效;  kCAAnimationCubic 对关键帧为座标点的关键帧进行圆滑曲线相连后插值计算，这里的主要目的是使得运行的轨迹变得圆滑；
kCAAnimationCubicPaced 看这个名字就知道和kCAAnimationCubic有一定联系,其实就是在kCAAnimationCubic的基础上使得动画运行变得均匀,就是系统时间内运动的距离相同,此时keyTimes以及timingFunctions也是无效的



1 rotationAnimation.removedOnCompletion = NO;
2 
3 rotationAnimation.fillMode = kCAFillModeForwards;
fillMode的作用就是决定当前对象过了非active时间段的行为. 比如动画开始之前,动画结束之后。如果是一个动画CAAnimation,则需要将其removedOnCompletion设置为NO,要不然fillMode不起作用.

下面来讲各个fillMode的意义 
kCAFillModeRemoved 这个是默认值,也就是说当动画开始前和动画结束后,动画对layer都没有影响,动画结束后,layer会恢复到之前的状态 
kCAFillModeForwards 当动画结束后,layer会一直保持着动画最后的状态 
kCAFillModeBackwards 这个和kCAFillModeForwards是相对的,就是在动画开始前,你只要将动画加入了一个layer,layer便立即进入动画的初始状态并等待动画开始.你可以这样设定测试代码,将一个动画加入一个layer的时候延迟5秒执行.然后就会发现在动画没有开始的时候,只要动画被加入了layer,layer便处于动画初始状态 
kCAFillModeBoth 理解了上面两个,这个就很好理解了,这个其实就是上面两个的合成.动画加入后开始之前,layer便处于动画初始状态,动画结束后layer保持动画最后的状态.