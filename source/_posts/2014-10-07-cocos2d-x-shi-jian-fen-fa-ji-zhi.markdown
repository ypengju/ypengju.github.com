---
layout: post
title: "cocos2d-x -- 事件分发机制"
date: 2014-10-07 23:21:25 +0800
comments: true
categories: cocos2d-x
---
cocos2d-x 3.2中关于事件分发机制的类都位于base目录下，其中主要三大类：

* 以Event为基类的，事件类
* 以EventListener为基类的，事件监听类
* EventDispatcher，事件分发类

一般我们只注册事件监听类，当发生相应事件时，由事件分发器分发事件，调用我们注册的回调函数，完成事件。  <!-- more -->
###事件类型 
在Event.h中我们可以看到有以下七种事件类型：
```c++
    enum class Type
    {
        TOUCH,  				//触摸事件
        KEYBOARD,			//键盘事件
        ACCELERATION,		//加速器事件
        MOUSE,				//鼠标事件
        FOCUS,				//UI控件事件
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID || CC_TARGET_PLATFORM == CC_PLATFORM_IOS)
        GAME_CONTROLLER,	//游戏手柄事件
#endif
        CUSTOM				//自定义事件
    };
```   
对应的也有七种事件监听，都已CCEventListenerXXX命名，分别用于注册对应事件处理代码。   
###使用方法
先来看一个完整的单点事件触摸例子。  
```c++ 单点触摸
auto listener = EventListenerTouchOneByOne::create();   //生成监听器
//添加touchBegan方法，手指接触屏幕是调用
listener->onTouchBegan = [](Touch *touch, Event *unused_event)
{
    log("you touch begin method");
    return true;
};
//添加touchMoved方法，手指在屏幕滑动时调用
listener->onTouchMoved = [](Touch *touch, Event *unused_event)
{
    log("you touch moved method");
};
//添加touchEnded方法，手指离开屏幕时调用
listener->onTouchEnded = [](Touch *touch, Event *unused_event)
{
    log("you touch ended method");
};
//添加touchCancelled，点击事件被打断是调用
listener->onTouchCancelled = [](Touch *touch, Event *unused_event)
{
    log("you touch cancenlled method");
};
//添加监听器
this->getEventDispatcher()->addEventListenerWithSceneGraphPriority(listener, this);
```    

1. 首先生成单点触摸监听器，对应的还有一个多点触摸监听器。
2. 注册事件发生时，不同阶段的回调函数。
3. 将监听器添加到事件分发器中，有系统在事件发生时，完成事件分发

这就是基本的事见监听注册过程，其他事件类型，也都按照这个过程来做。  
###剖析 
上边的例子，注册了一个点击事件，了解了事件监听的使用方法，但是并不知道从手指点击屏幕到成功调用注册函数之间到底发生了哪些事情，接下来主要剖析下整个调用过程，看引擎是怎么完成一个事件分发的。
####事件
事件基类Event类中，主要由三个属性
```c++ Event.h
	...
	Type _type;     ///< Event type
    bool _isStopped;       ///< whether the event has been stopped.
    Node* _currentTarget;  ///< Current target
	...
```
前后不重要代码省略，三个属性主要说明了事件的类型，调用时事件是否以完成，以及调用事件的当前节点。其子类也基本类似，只是添加新的属性来说明新的事件特性及对应的set、get方法。    
####事件监听
```c++ EventListener.h
	 ...
    std::function<void(Event*)> _onEvent;   /// Event callback function

    Type _type;                             /// Event listener type
    ListenerID _listenerID;                 /// Event listener ID
    bool _isRegistered;                     /// Whether the listener has been added to dispatcher.

    int   _fixedPriority;   // The higher the number, the higher the priority, 0 is for scene graph base priority.
    Node* _node;            // scene graph based priority
    bool _paused;           // Whether the listener is paused
    bool _isEnabled;        // Whether the listener is enabled
    ...   
```    
`_onEvent` - 注册的监听回调函数，`_type` - 事件监听类型，如下：
```c++ 
enum class Type
    {
        UNKNOWN,
        TOUCH_ONE_BY_ONE,
        TOUCH_ALL_AT_ONCE,
        KEYBOARD,
        MOUSE,
        ACCELERATION,
        FOCUS,
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID || CC_TARGET_PLATFORM == CC_PLATFORM_IOS)
		GAME_CONTROLLER,
#endif
        CUSTOM
    };
```   
这里多了一个默认类型和对单点触摸和多点触摸的监听事件进行了区分。`_listenerID` - 监听ID，在分发中会用到，`_fixedPriority` - 优先级，优先级高的会先被调用，`_node` - 监听节点，`_paused` - 是否暂停，`_isEnabled` - 是否可用。其他子类，也基本类似，只是针对不同事件类型，添加了对应的回调函数。每个listen类中都有clone方法，当为其他节点添加监听时，可以使用已有监听的clone来为其添加监听，并在原有监听回调中进行逻辑处理。   
####事件分发
了解了事件和监听后，来具体看下cocos2d-x是怎么注册监听的，已经如何将事件传递给监听调用回调。先来看注册监听。
#####注册监听
前面例子的最后一句话，将我们生成的监听类添加到了分发器中，在3.2中有三类方法，添加监听
```c++
this->getEventDispatcher()->addCustomEventListener(const std::string &eventName, const std::function<void (EventCustom *)> &callback);
this->getEventDispatcher()->addEventListenerWithFixedPriority(cocos2d::EventListener *listener, <#int fixedPriority);
this->getEventDispatcher()->addEventListenerWithSceneGraphPriority(cocos2d::EventListener *listener, cocos2d::Node *node);
```    
依次来看看这三种方法的具体实现。   

1. addCustomEventListener(const std::string &eventName, const std::function<void (EventCustom *)> &callback);
它接受两个参数，一个事件名，一个回调函数。看看内部实现：
```c++ 
EventListenerCustom* EventDispatcher::addCustomEventListener(const std::string &eventName, const std::function<void(EventCustom*)>& callback)
{
    EventListenerCustom *listener = EventListenerCustom::create(eventName, callback);
    addEventListenerWithFixedPriority(listener, 1);
    return listener;
}
```
oh，调到addEventListenerWithFixedPriority了，那还是直接看addEventListenerWithFixedPriority吧。   
2. addEventListenerWithFixedPriority(cocos2d::EventListener *listener, <#int fixedPriority);
接两个参数，一个监听器，一个优先级，继续看内部实现
```c++ addEventListenerWithFixedPriority
void EventDispatcher::addEventListenerWithFixedPriority(EventListener* listener, int fixedPriority)
{
    CCASSERT(listener, "Invalid parameters.");
    CCASSERT(!listener->isRegistered(), "The listener has been registered.");
    CCASSERT(fixedPriority != 0, "0 priority is forbidden for fixed priority since it's used for scene graph based priority.");
    
    if (!listener->checkAvailable())
        return;
    
    listener->setAssociatedNode(nullptr);
    listener->setFixedPriority(fixedPriority);
    listener->setRegistered(true);
    listener->setPaused(false);

    addEventListener(listener);
}
```   
首先断言进行验证，判断是否已经注册过，每个监听类只能注册一次。传入的优先级不能为0，因为0是场景的默认处理优先级。然后检测监听器是否可以，针对不同的监听器，有不同的检测条件，比如单点触摸检测是否注册了touchBegan回调函数，键盘事件是否注册了按下抬起事件等。之后关联节点，因为这里没有节点出入，所以为nullptr，然后设置优先级，设置注册、暂停标志，最后调用addEventListener方法，我们继续看源码   
```c++ addEventListener
void EventDispatcher::addEventListener(EventListener* listener)
{
    if (_inDispatch == 0)
    {
        forceAddEventListener(listener);
    }
    else
    {
        _toAddedListeners.push_back(listener);
    }

    listener->retain();
}
```   
可以看到，根据事件是否在分发中，对监听器进行了不同的处理。首先看`_inDispatch == 0`时的调用过程。   
```c++ forceAddEventListener
void EventDispatcher::forceAddEventListener(EventListener* listener)
{
    EventListenerVector* listeners = nullptr;
    EventListener::ListenerID listenerID = listener->getListenerID();
    auto itr = _listenerMap.find(listenerID);
    if (itr == _listenerMap.end())
    {
        
        listeners = new EventListenerVector();
        _listenerMap.insert(std::make_pair(listenerID, listeners));
    }
    else
    {
        listeners = itr->second;
    }
    
    listeners->push_back(listener);
    
    if (listener->getFixedPriority() == 0)
    {
        setDirty(listenerID, DirtyFlag::SCENE_GRAPH_PRIORITY);
        
        auto node = listener->getAssociatedNode();
        CCASSERT(node != nullptr, "Invalid scene graph priority!");
        
        associateNodeAndEventListener(node, listener);
        
        if (node->isRunning())
        {
            resumeEventListenersForTarget(node);
        }
    }
    else
    {
        setDirty(listenerID, DirtyFlag::FIXED_PRIORITY);
    }
}
```
* 生成一个零时数据容器
* 获取监听器ID，在_listenerMap监听器容器中进行查找
* 如果_listenerMap监听器容器中没有对应的ID的监听容器，穿件一个新的添加进去。
* 如果有，将对应ID的注册过的监听器容器赋值给零时容器
* 判断传进参数监听器优先级是否为0，然后进行处理，如果为0，设置为脏，至于什么才是脏下边继续分析，此时获得关联节点，然后将节点与监听类进行关联。不为0时，只设置为了脏，下边继续看怎么设置的
```c++ setDirty   
void EventDispatcher::setDirty(const EventListener::ListenerID& listenerID, DirtyFlag flag)
{    
    auto iter = _priorityDirtyFlagMap.find(listenerID);
    if (iter == _priorityDirtyFlagMap.end())
    {
        _priorityDirtyFlagMap.insert(std::make_pair(listenerID, flag));
    }
    else
    {
        int ret = (int)flag | (int)iter->second;
        iter->second = (DirtyFlag) ret;
    }
}
```    
在_priorityDirtyFlagMap中，对对应事件类型ID添加标志，所有添加到_priorityDirtyFlagMap中的事件类型都为脏数据。接下来看看怎么关联节点和监听类的
```c++ associateNodeAndEventListener
void EventDispatcher::associateNodeAndEventListener(Node* node, EventListener* listener)
{
    std::vector<EventListener*>* listeners = nullptr;
    auto found = _nodeListenersMap.find(node);
    if (found != _nodeListenersMap.end())
    {
        listeners = found->second;
    }
    else
    {
        listeners = new std::vector<EventListener*>();
        _nodeListenersMap.insert(std::make_pair(node, listeners));
    }
    
    listeners->push_back(listener);
}
```   
由代码可知一个节点的对应注册事件都存储在`_nodeListenersMap`容器类中。此时将新的监听类添加到节点的监听类容器中。然后递归调用节点的所有类，使节点处于监听状态即设置暂停标志为false。
```c++ resumeEventListenersForTarget
void EventDispatcher::resumeEventListenersForTarget(Node* target, bool recursive/* = false */)
{
    auto listenerIter = _nodeListenersMap.find(target);
    if (listenerIter != _nodeListenersMap.end())
    {
        auto listeners = listenerIter->second;
        for (auto& l : *listeners)
        {
            l->setPaused(false);
        }
    }
    
    for (auto& listener : _toAddedListeners)
    {
        if (listener->getAssociatedNode() == target)
        {
            listener->setPaused(false);
        }
    }

    setDirtyForNode(target);
    
    if (recursive)
    {
        const auto& children = target->getChildren();
        for (const auto& child : children)
        {
            resumeEventListenersForTarget(child, true);
        }
    }
}
```
如果`_inDispatch == 0`不成立是，调用`_toAddedListeners.push_back(listener);`说明此时分发器正在工作，则直接将监听器添加到分发表中。  
 
再看addEventListenerWithSceneGraphPriority的注册过程    
也接受两个参数，一个监听器，一个监听节点。  
```c++
void EventDispatcher::addEventListenerWithSceneGraphPriority(EventListener* listener, Node* node)
{
    CCASSERT(listener && node, "Invalid parameters.");
    CCASSERT(!listener->isRegistered(), "The listener has been registered.");
    if (!listener->checkAvailable())
        return;
    listener->setAssociatedNode(node);
    listener->setFixedPriority(0);
    listener->setRegistered(true);
    addEventListener(listener);
}
``` 
和addEventListenerWithFixedPriority最大不同就在于设置关联节点和设置优先级上，其他都是调到addEventListener方法进行处理，上边以进行了调用分析。   
#####响应事件
上边已注册了监听，接下来看看一个事件发生时的处里过程，我以一个单点触摸为例，进行说明。    
cocos2d-x点击事件并不是直接从硬件获得的，而是通过不同平台原生点击事件，当点击事件发生时，调用原生点击事件处理器，之后再将点击事件传递给引擎，引擎再对点击事件进行分发处理。如下简单分析下android和ios的点击调用过程。  
andoird：   
android的原生点击事件注册在platform/android/java/src/org/cocos2dx/lib/Cocos2dxGLSurfaceView类中的onTouchEvent()方法中。在此方法中对不同点击事件进行处理，然后传递给Cocos2dxRenderer.java类中的对应的handleActionXXX()函数中，在此方法中，通过jni调用注册的c++函数，这些函数具体实现都在android/jni/TouchesJni.cpp这个类中，然后统一调到platform/GLViewProtocol类中的handleTouchesXXX()类中进行事件处理，最后进行事件分发。   
ios：   
是从ios/CCEAGLView.mm类中的ios的touchBegin等方法，调到GLViewProtocol类中，因为oc和c++可以直接调用，所以这里没有android那么复杂。  
可以看到所有的点击事件都是统一到GLViewProtocol类来进行事件处理的。    

```c++ GLViewProtocol::handleTouchesBegin
void GLViewProtocol::handleTouchesBegin(int num, intptr_t ids[], float xs[], float ys[])
{
    intptr_t id = 0;
    float x = 0.0f;
    float y = 0.0f;
    int unusedIndex = 0;
    EventTouch touchEvent;
    for (int i = 0; i < num; ++i)
    {
        id = ids[i];
        x = xs[i];
        y = ys[i];
        auto iter = g_touchIdReorderMap.find(id);
        // it is a new touch
        if (iter == g_touchIdReorderMap.end())
        {
            unusedIndex = getUnUsedIndex();
            // The touches is more than MAX_TOUCHES ?
            if (unusedIndex == -1) {
                CCLOG("The touches is more than MAX_TOUCHES, unusedIndex = %d", unusedIndex);
                continue;
            }
            Touch* touch = g_touches[unusedIndex] = new Touch();
			touch->setTouchInfo(unusedIndex, (x - _viewPortRect.origin.x) / _scaleX,
                                     (y - _viewPortRect.origin.y) / _scaleY);
            
            CCLOGINFO("x = %f y = %f", touch->getLocationInView().x, touch->getLocationInView().y);    
            g_touchIdReorderMap.insert(std::make_pair(id, unusedIndex));
            touchEvent._touches.push_back(touch);
        }
    }
    if (touchEvent._touches.size() == 0)
    {
        CCLOG("touchesBegan: size = 0");
        return;
    }
    touchEvent._eventCode = EventTouch::EventCode::BEGAN;
    auto dispatcher = Director::getInstance()->getEventDispatcher();
    dispatcher->dispatchEvent(&touchEvent);
}
```     
以GLViewProtocol::handleTouchesBegin为例，可以看到，所有的触摸点都包装在类Touch中，然后再将Touch点全部放置到EventTouch事件中，然后由引擎进行分发调用。   
接着具体看看引擎是如何分发的，从dispatchEvent()方法开始。   

```c++
void EventDispatcher::dispatchEvent(Event* event)
{
    if (!_isEnabled)
        return;
    
    updateDirtyFlagForSceneGraph();
    
    
    DispatchGuard guard(_inDispatch);
    
    if (event->getType() == Event::Type::TOUCH)
    {
        dispatchTouchEvent(static_cast<EventTouch*>(event));
        return;
    }
    
    auto listenerID = __getListenerID(event);
    
    sortEventListeners(listenerID);
    
    auto iter = _listenerMap.find(listenerID);
    if (iter != _listenerMap.end())
    {
        auto listeners = iter->second;
        
        auto onEvent = [&event](EventListener* listener) -> bool{
            event->setCurrentTarget(listener->getAssociatedNode());
            listener->_onEvent(event);
            return event->isStopped();
        };
        
        dispatchEventToListeners(listeners, onEvent);
    }
    
    updateListeners(event);
}
```   
* 首先更新了脏数据标志
* 然后对_inDispatch标志进行了操作
* 对触摸事件进行了单独处理
* 获取事件监听器ID，根据ID排序后对事件进行处理，然后调用注册的回调函数  

dispatchTouchEvent中分别对单点触摸和多点触摸分别进行了处理，然后调用不同的注册回调函数。完成事件的触摸。   
这就是点击事件的整个处理流程，由于对EventDispatcher中的处理，有些地方还是理解的不投透彻，所以后续还会根据理解进行更新。  

其他类型的事件与点击事件的处理也都大同小异，比如键盘事件，它只在桌面程序及安卓程序上有效，基本情况也类似，都是在不同平台调用，然后传递给引擎处理。

