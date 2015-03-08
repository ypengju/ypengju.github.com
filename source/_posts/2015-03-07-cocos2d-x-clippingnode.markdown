---
layout: post
title: "cocos2d-x -- ClippingNode"
date: 2015-03-07 14:24:05 +0800
comments: true
categories: cocos2d-x
---
ClippingNode是一个用于剪切的类，例如在游戏新手引导中会指定特定点击区域就可以使用此类来做，或者在游戏滚动公告中，也可以使用此类来完成。    
如下边这个例子：
```cpp
Size visibleSize = Director::getInstance()->getVisibleSize();
Vec2 origin = Director::getInstance()->getVisibleOrigin();
//文字
auto label = LabelTTF::create("这是一个美好的早上，我坐在窗前敲代码", "Arial", 20);
label->setPosition(Vec2(visibleSize.width/2, visibleSize.height/2));
label->setColor(Color3B(0, 255, 0));
this->addChild(label);
//遮罩形状及大小
auto stencil = LayerColor::create(Color4B(255, 255, 255, 255), 400, 50);
stencil->ignoreAnchorPointForPosition(false);
stencil->setAnchorPoint(Vec2(0.5, 0.5));
//剪切
auto clipping = ClippingNode::create();
clipping->setStencil(stencil);
clipping->setInverted(true);
clipping->set
clipping->setPosition(Vec2(visibleSize.width/2 + origin.x, visibleSize.height/2 + origin.y));
this->addChild(clipping);
//要剪切的内容，此处为一个背景
auto sprite = Sprite::create("HelloWorld.png");
clipping->addChild(sprite);
```
<!--more-->
{% img  /images/clippingnode/1.png %}    
本来屏幕中间有一句话，但是在上边放一张图片会遮住文字的显示，为了能看到图片下边的文字，那就需要在图片上挖个洞，ClippingNode就是用来挖这个洞的，挖洞就得有形状大小啊，这就是stencil遮罩的作用了，此处定义了一个LayerColor,它本身就是一个矩形，然后使用了他的尺寸，所以意思就是此处需要挖一个长400，宽50的矩形洞。（除了矩形，我们还可以设置其他任一形状）规则设置好了，开始挖洞，生成ClippingNode节点，将规则尺寸通过setStencil()方法设置进去，将节点添加到层。好了规则设置好了，现在就需要设置挖的内容了，就是要在什么上挖洞，前边知道要在一张图片上挖洞，那么把这个图片添加进来，此处用Sprite创建了一张图片，注意，这个精灵节点是添加到ClippingNode中的，不是添加到层。到目前为止，完成了在一张图片上挖一个长400，宽50的矩形洞，挖完洞有两个东西，一个就是挖掉的400x50的矩形，另一个就是图片剩下的内容，到底显示那个里，我们可以通过setInverted()方法进行设置，当为true的时候，显示挖掉剩下的内容，当为false时，显示挖掉的内容。上边例子，设置为true，显示挖掉剩下的图片，通过洞，就可以看到图片地下的内容，这就是我们需要的结果。下边图片就是设置为false时，显示的效果。   
 {% img  /images/clippingnode/2.png %} 
 ClippingNode还有一个方法setAlphaThreshold()这个方法设置alpha测试值，值的范围是0.0~1.0，默认是1，当这个值为1时，引擎绘制所有内容，当这个值小于1时，只有剪切的内容的alpha值大于这个值的时候，才会被引擎绘制，否则不进行绘制。