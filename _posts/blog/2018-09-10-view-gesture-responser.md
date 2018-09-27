---
layout: post
title: 实现 iOS UIView 及其 Subview 透明区域的事件穿透
description: UIView 透明区域的事件穿透的一种方案。
category: blog
tag: iOS, UIView
---

在应对日常的需求时，有一种场景是这样的：在当前屏幕上堆叠着一堆的 view，其中某个 view 有些部分是不透明的，有些部分是透明或半透明的，这时候我们希望这个 view 不透明的部分能够响应点击事件，透明或半透明的部分则不要响应点击事件并把点击事件透出给它底下的其他 view 来响应。


解决这个需求，需要先了解 UIView 的两个 API：

```
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event;
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event;
```

然后来了解一下当用户触摸一下屏幕时，整个处理流程是怎样的：


- 1、当用户点击屏幕时，会产生一个触摸事件，系统会将该事件加入到一个由 UIApplication 管理的事件队列中。
- 2、UIApplication 会从事件队列中取出最前面的事件进行分发以便处理，通常，先发送事件给应用程序的主窗口(UIWindow)。
- 3、主窗口会调用 `hitTest:withEvent:` 方法在视图(UIView)层次结构中找到一个最合适的 UIView 来处理触摸事件。`hitTest:withEvent:` 方法大致处理流程是这样的：
	- 3.1、首先调用当前视图的 `pointInside:withEvent:` 方法判断触摸点是否在当前视图内：
		- 若 `pointInside:withEvent:` 方法返回 NO，说明触摸点不在当前视图内，则当前视图的 `hitTest:withEvent:` 返回 nil。
		- 若 `pointInside:withEvent:` 方法返回 YES，说明触摸点在当前视图内，则遍历当前视图的所有子视图(subviews)，调用子视图的 `hitTest:withEvent:` 方法重复前面的步骤，子视图的遍历顺序是从 top 到 bottom，即从 subviews 数组的末尾向前遍历，直到有子视图的 `hitTest:withEvent:` 方法返回非空对象或者全部子视图遍历完毕。
		- 若第一次有子视图的 `hitTest:withEvent:` 方法返回非空对象,则当前视图的 `hitTest:withEvent:` 方法就返回此对象，处理结束。
		- 若所有子视图的 `hitTest:withEvent:` 方法都返回 nil，则当前视图的 `hitTest:withEvent:` 方法返回当前视图自身(self)。
- 4、最终，这个触摸事件交给主窗口的 `hitTest:withEvent:` 方法返回的视图对象去处理。


了解了这个流程后，我们从 `pointInside:withEvent:` 着手来解决这个需求，主要思路判断当前 view 及 sub view 的点击位置颜色的 alpha 值是否大于阈值，来决定事件是否由其处理。具体实现如下：



```
#import "MyGestureContainerView.h"

#define MyAlphaThreshold 0.5

@implementation MyGestureContainerView

- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    CGFloat alpha = [self alphaOfPoint:point];
    // 处于当前 view 及 sub view 的点击位置颜色的 alpha 值大于阈值，则事件不透传，否则就透传。
    if (alpha > MyAlphaThreshold) {
        return YES;
    } else {
        return NO;
    }
}

// 一种方案：渲染 layer 来获取颜色。
- (CGFloat)alphaOfPoint:(CGPoint)point {
    return [self alphaOfPointFromLayer:point];
}

- (CGFloat)alphaOfPointFromLayer:(CGPoint)point {
    unsigned char pixel[4] = {0};
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    CGContextRef context = CGBitmapContextCreate(pixel, 1, 1, 8, 4, colorSpace, kCGBitmapAlphaInfoMask & kCGImageAlphaPremultipliedLast);
    CGContextTranslateCTM(context, -point.x, -point.y);
    [self.layer renderInContext:context];
    
    CGContextRelease(context);
    CGColorSpaceRelease(colorSpace);
    
    //NSLog(@"pixel: %d %d %d %d", pixel[0], pixel[1], pixel[2], pixel[3]);
    return pixel[3]/255.0;
}

/*
// 另一种方案：截图，然后通过获取截图中的点击出的颜色来取得其 alpha 值。
- (CGFloat)alphaOfPointFromViewScreenShot:(CGPoint)point {
    UIImage *image = [self viewScreenShot];
    CGFloat alpha = [self alphaFromImage:image atX:point.x andY:point.y];
    return alpha;
}

- (UIImage *)viewScreenShot {
    UIGraphicsBeginImageContext(self.bounds.size);
    [self drawViewHierarchyInRect:self.bounds afterScreenUpdates:NO];
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return image;
}

- (CGFloat)alphaFromImage:(UIImage*)image atX:(int)xx andY:(int)yy {
    // First get the image into your data buffer
    CGImageRef imageRef = [image CGImage];
    NSUInteger width = CGImageGetWidth(imageRef);
    NSUInteger height = CGImageGetHeight(imageRef);
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    
    unsigned char *rawData = (unsigned char *)calloc(height * width * 4, sizeof(unsigned char));
    NSUInteger bytesPerPixel = 4;
    NSUInteger bytesPerRow = bytesPerPixel * width;
    NSUInteger bitsPerComponent = 8;
    CGContextRef context = CGBitmapContextCreate(rawData, width, height,
                                                 bitsPerComponent, bytesPerRow, colorSpace,
                                                 kCGImageAlphaPremultipliedLast | kCGBitmapByteOrder32Big);
    
    CGColorSpaceRelease(colorSpace);
    
    CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);
    CGContextRelease(context);
    
    // Now your rawData contains the image data in the RGBA8888 pixel format.
    unsigned long byteIndex = (bytesPerRow * yy) + xx * bytesPerPixel;
    
    CGFloat alpha = (rawData[byteIndex + 3] * 1.0) / 255.0;
    free(rawData);
    return alpha;
}
*/
@end
```

上面列出来两种取点击位置的颜色 alpha 值的方法，一种是通过渲染 layer 来获取颜色，一种是截图来获取颜色。


[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/view-gesture-responser