---
layout: post
title: "cocos2d-x -- 多分辨率适配"
date: 2014-10-20 21:06:21 +0800
comments: true
categories: cocos2d-x
---
多分辨率适配一直是cocos2d-x游戏关注的一个重点，所以搞清楚cocos2d-x适配方案尤为重要。   
假设不存在多分辨率适配问题，只有一种分辨率，那我们就为该分辨率设计大小和屏幕一样的图片就OK了，但是因为现在市场手机大小不一，苹果及安卓分辨率各种大小应有尽有。<!-- more -->在这种情况下，我们为每种分辨率适配各自大小的图片很明显是不太可能的，这样就会有应用程序安装包过大，设计任务过重等情况。这种问题在cocos2d-x中尤为突出，所以我们就想尽办法，让一套或者少量几套图就能适配各种分辨率，这样即省时又省力，也可应对未来分辨率增长变化。一般在游戏开发中，都是做一套图，然后来适配不同分辨率来解决此问题。   
了解之前需要理清几个分辨率概念    

* Resource Resoluation: 资源分辨率，也就是我们常用到的图片分辨率
* Screen Resolution: 手机真是屏幕分辨率
* Design Resolution: 设计分辨率 

整个过程都是围绕这几个分辨率展开的，最终的目的是把资源分辨率完美的缩放致手机屏幕分辨率，从而使界面完美展现。而设计分辨率就是在这两种分辨率之间起一个桥梁作用。需要理解就是下边这几个函数   
```
director->getOpenGLView()->setDesignResolutionSize(float width, float height, ResolutionPolicy resolutionPolicy);
Director::getInstance()->setContentScaleFactor(float scaleFactor);
```   
`setDesignResolutionSize`函数设置了设计分辨率及适配方案    
`setContentScaleFactor`函数设置了缩放因子    
所有屏幕大小及显示区域大小都是在`GLViewProtocol`类中完成的，此类位于cocos/platform目录中，可以看到，它是个平台窗口类GLView的基类，在GLViewProtocol中设置大小及适配方案，确保在所有平台都采用一套方案。   
```c++ GLViewProtocol.h
	......
    // real screen size
    Size _screenSize;
    // resolution size, it is the size appropriate for the app resources.
    Size _designResolutionSize;
    // the view port size
    Rect _viewPortRect;
    // the view name
    std::string _viewName;

    float _scaleX;
    float _scaleY;
    ResolutionPolicy _resolutionPolicy;   
    ......
```   
可以看到GLViewProtocol中又几个成员变量   

* _screenSize 手机屏幕真实分辨率大小   
* _designResolutionSize 设计分辨率大小   
* _viewPortRect 显示区域大小   
* _scaleX, _scaleY 宽高比   
* _resulutionPolicy 适配方式     

那么继续看设置设计分辨率大小和适配方式都干了什么   
```
void GLViewProtocol::setDesignResolutionSize(float width, float height, ResolutionPolicy resolutionPolicy)
{
    CCASSERT(resolutionPolicy != ResolutionPolicy::UNKNOWN, "should set resolutionPolicy");
    
    if (width == 0.0f || height == 0.0f)
    {
        return;
    }

    _designResolutionSize.setSize(width, height);
    _resolutionPolicy = resolutionPolicy;
    
    updateDesignResolutionSize();
 }
```   
从源码看我们不能将适配方式设置为UNKNOWN方式，之后对分辨率和适配方式变量赋值，调用updateDesignResolutionSize()函数。继续看updateDesignResolutionSize()函数。   
```c++
void GLViewProtocol::updateDesignResolutionSize()
{
    if (_screenSize.width > 0 && _screenSize.height > 0
        && _designResolutionSize.width > 0 && _designResolutionSize.height > 0)
    {
        _scaleX = (float)_screenSize.width / _designResolutionSize.width;
        _scaleY = (float)_screenSize.height / _designResolutionSize.height;
        
        if (_resolutionPolicy == ResolutionPolicy::NO_BORDER)
        {
            _scaleX = _scaleY = MAX(_scaleX, _scaleY);
        }
        
        else if (_resolutionPolicy == ResolutionPolicy::SHOW_ALL)
        {
            _scaleX = _scaleY = MIN(_scaleX, _scaleY);
        }
        
        else if ( _resolutionPolicy == ResolutionPolicy::FIXED_HEIGHT) {
            _scaleX = _scaleY;
            _designResolutionSize.width = ceilf(_screenSize.width/_scaleX);
        }
        
        else if ( _resolutionPolicy == ResolutionPolicy::FIXED_WIDTH) {
            _scaleY = _scaleX;
            _designResolutionSize.height = ceilf(_screenSize.height/_scaleY);
        }
        
        // calculate the rect of viewport
        float viewPortW = _designResolutionSize.width * _scaleX;
        float viewPortH = _designResolutionSize.height * _scaleY;
        
        //将显示区域在屏幕中居中显示
        _viewPortRect.setRect((_screenSize.width - viewPortW) / 2, (_screenSize.height - viewPortH) / 2, viewPortW, viewPortH);
        
        // reset director's member variables to fit visible rect
        auto director = Director::getInstance();
        director->_winSizeInPoints = getDesignResolutionSize();
        director->createStatsLabel();
        director->setGLDefaultValues();
    }
}
```   
在该方法中，

1. 先计算宽高比，（宽高比就是屏幕实际宽/设计分辨率宽, 实际分辨率高/设计分辨率高，这两个比值就是宽高比，此处用_scaleX和_scaleY两个变量表示）    
2. 然后根据不同适配方式，重新计算宽高比   
3. 再根据设计分辨率和缩放大小计算可视区域大小
4. 之后设置了director中的一些变量。   

我们一个一个分析，来看看适配方式。   
其实在这里就可以清楚的看到设置适配方式的目的就是来按不同方式计算宽高比，及对设计分辨率大小重新定值。根据代码来解析下其中含义   
####ResolutionPolicy::NO_BORDER
次方式计算采用_scaleX和_scaleY中较大的对两者进行赋值，因为是按较大的进行赋值，所以在之后计算可视区域大小的时候，有一边会超出屏幕大小。 在实际应用中就是，有可能显示区域超过手机屏幕，使屏幕边缘图标不能完整显示。  
####ResolutionPolicy::SHOW_ALL
次方式计算采用_scaleX和_scaleY中较小的对两者进行赋值，因为是按较大的进行赋值，所以在之后计算可视区域大小的时候，会有一边小于屏幕大小。在实际手机中看，整个界面都能在屏幕中显示，但是会有黑边情况。
####ResolutionPolicy::FIXED_HEIGHT
此方式以高比为基准，对_scaleX和_scaleY赋值都为_scaleY，用新的宽比重新计算了设计分辨率的宽，这样在计算可视区域的时候，保证高度完全适配，宽度根据高度进行调整。
####ResolutionPolicy::FIXED_HEIGHT
此方式以宽比为基准，对_scaleX和_scaleY赋值都为_scaleX，用新的高比重新计算了设计分辨率的高，这样在计算可视区域的时候，保证宽度完全适配，高度根据高度进行调整。
####ResolutionPolicy::EXACT_FIT
除了代码中看到的四种，其实还有一种EXACT_FIT，因为该方式对宽高比不做任何修改，所以在代码中看不到条件判断，但其实就是直接计算的。可视区域就是屏幕大小，这种方式因为是宽拉宽，高拉高，大多数情况宽高比都不一样，所以会出现图片拉伸的情况。  

其实可以看到设计分辨率在屏幕分辨率和可视区域之间起到了一个桥梁的作用，我们所有的游戏布局都是在设计分辨率上进行的，之后根据适配方式，进行缩放。    
既然所有的布局都是在设计分辨率上进行的，那么图片分辨率和设计分辨率是如何关联，因为图片分辨率可能很大，设计分辨率可能小，这样在设计分辨率上岂不是放不下了，这个问题可以通过设置缩放因子进行解决。
```
Director::getInstance()->setContentScaleFactor(float scaleFactor);
```  
这样我们就可以将图片分辨率调整到设计分辨率，在设计分辨率上进行完整布局。

