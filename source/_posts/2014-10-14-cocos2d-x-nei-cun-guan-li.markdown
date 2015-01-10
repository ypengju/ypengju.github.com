---
layout: post
title: "cocos2d-x -- 内存管理"
date: 2014-10-14 22:13:24 +0800
comments: true
categories: cocos2d-x
---
cocos2d-x采用引用计数来对内存进行管理。在2.X版本中，所有类都继承CCObject类来完成内存的管理，而在3.X版本中，cocos2d-x将管理引用计数的内单独了处理命名为Ref，在这个类中对引用计数进行操作。<!-- more -->
```c++ Ref.h
class CC_DLL Ref
{
public:
    void retain();
    void release();
    Ref* autorelease();
    unsigned int getReferenceCount() const;

protected:
    Ref();
public:
    virtual ~Ref();

protected:
    unsigned int _referenceCount;
    friend class AutoreleasePool;
};
```
去掉跟脚本有关的东西，其实就定义了一个引用计数变量`_referenceCount`，以及操作引用计数的retain()和release()方法和自动管理内存的autorelease()方法。所有引擎的提供的控件都提供了一个静态的create()方法，在内部实现了autorelease()调用，所以我们在调用Sprite::create()方法生成精灵时，并不需要对其内存释放进行过多考虑。引擎内部实现了引用计数加减操作，来完成对内存的释放工作。看着几个方法的内部实现
```c++ 
Ref::Ref()
: _referenceCount(1) // when the Ref is created, the reference count of it is 1
{
#if CC_ENABLE_SCRIPT_BINDING
    static unsigned int uObjectCount = 0;
    _luaID = 0;
    _ID = ++uObjectCount;
    _scriptObject = nullptr;
#endif
    
#if CC_USE_MEM_LEAK_DETECTION
    trackRef(this);
#endif
}

Ref::~Ref()
{
#if CC_ENABLE_SCRIPT_BINDING
    // if the object is referenced by Lua engine, remove it
    if (_luaID)
    {
        ScriptEngineManager::getInstance()->getScriptEngine()->removeScriptObjectByObject(this);
    }
    else
    {
        ScriptEngineProtocol* pEngine = ScriptEngineManager::getInstance()->getScriptEngine();
        if (pEngine != nullptr && pEngine->getScriptType() == kScriptTypeJavascript)
        {
            pEngine->removeScriptObjectByObject(this);
        }
    }
#endif


#if CC_USE_MEM_LEAK_DETECTION
    if (_referenceCount != 0)
        untrackRef(this);
#endif
}

void Ref::retain()
{
    CCASSERT(_referenceCount > 0, "reference count should greater than 0");
    ++_referenceCount;
}

void Ref::release()
{
    CCASSERT(_referenceCount > 0, "reference count should greater than 0");
    --_referenceCount;

    if (_referenceCount == 0)
    {
#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
        auto poolManager = PoolManager::getInstance();
        if (!poolManager->getCurrentPool()->isClearing() && poolManager->isObjectInPools(this))
        {
            CCASSERT(false, "The reference shouldn't be 0 because it is still in autorelease pool.");
        }
#endif

#if CC_USE_MEM_LEAK_DETECTION
        untrackRef(this);
#endif
        delete this;
    }
}

Ref* Ref::autorelease()
{
    PoolManager::getInstance()->getCurrentPool()->addObject(this);
    return this;
}
```
* 特别注意构造函数中，对引用计数赋值为1，所以当我们new一个对象的时候，默认引用计数就是1了。
* retain()方法对引用计数加一   
* release()方法对引用计数减一，当引用计数为0时，delete掉   
* autorelease()将引用添加到了管理池中，用引擎进行管理   

前两个方法比较简单直接，只是对引用计数的加减运算，现在来看下引擎是怎么进行自动管理的   
```c++ PoolManager.h
class CC_DLL PoolManager
{
public:
    static PoolManager* getInstance();
    static void destroyInstance();
    AutoreleasePool *getCurrentPool() const;

    bool isObjectInPools(Ref* obj) const;
    friend class AutoreleasePool;
    
private:
    PoolManager();
    ~PoolManager();
    void push(AutoreleasePool *pool);
    void pop();
    static PoolManager* s_singleInstance;
    std::vector<AutoreleasePool*> _releasePoolStack;
};
```
PoolManager是一个单例，其内部定义了一个保存`AutoreleasePool *`类型的栈，继续看AutoreleasePool
```C++ AutoreleasePool
class CC_DLL AutoreleasePool
{
public:
    /**
     * @warn Don't create an auto release pool in heap, create it in stack.
     * @js NA
     * @lua NA
     */
    AutoreleasePool();
    
    /**
     * Create an autorelease pool with specific name. This name is useful for debugging.
     */
    AutoreleasePool(const std::string &name);
    
    /**
     * @js NA
     * @lua NA
     */
    ~AutoreleasePool();

    /**
     * Add a given object to this pool.
     *
     * The same object may be added several times to the same pool; When the
     * pool is destructed, the object's Ref::release() method will be called
     * for each time it was added.
     *
     * @param object    The object to add to the pool.
     * @js NA
     * @lua NA
     */
    void addObject(Ref *object);

    /**
     * Clear the autorelease pool.
     *
     * Ref::release() will be called for each time the managed object is
     * added to the pool.
     * @js NA
     * @lua NA
     */
    void clear();
    
#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
    /**
     * Whether the pool is doing `clear` operation.
     */
    bool isClearing() const { return _isClearing; };
#endif
    
    /**
     * Checks whether the pool contains the specified object.
     */
    bool contains(Ref* object) const;

    /**
     * Dump the objects that are put into autorelease pool. It is used for debugging.
     *
     * The result will look like:
     * Object pointer address     object id     reference count
     *
     */
    void dump();
    
private:
    /**
     * The underlying array of object managed by the pool.
     *
     * Although Array retains the object once when an object is added, proper
     * Ref::release() is called outside the array to make sure that the pool
     * does not affect the managed object's reference count. So an object can
     * be destructed properly by calling Ref::release() even if the object
     * is in the pool.
     */
    std::vector<Ref*> _managedObjectArray;
    std::string _name;
    
#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
    /**
     *  The flag for checking whether the pool is doing `clear` operation.
     */
    bool _isClearing;
#endif
};
```  
  
也很简单清晰，内部管理着一个存储`Ref *`类型的容器，所以当调用autorelease()方法时，引擎是将调用对象添加到PoolManager中的_releasePoolStack栈最后一个AutoreleasePool的_managedObjectArray容器中。其他几个方法就是对这个容器的操作了。  
 
这里引用官方文档的一段说明，更清晰的说明这个工作过程。  [《官方文档》](http://cn.cocos2d-x.org/article/index?type=cocos2d-x&url=/doc/cocos-docs-master/manual/framework/native/v2/memory/refcount-autoreleasepool/zh.md)  
CCAutoreleasePool不能被用在多线程中，所以假如你游戏需要网络线程，请仅仅在网络线程中接收数据，改变状态标志，不要这个线程里面调用cocos2d接口  

CCAutoreleasePool的逻辑是，当你调用object->autorelease()，object就被放到自动释放池中。自动释放池能够帮助你保持这个object的生命周期，直到当前消息循环的结束。在这个消息循环的最后，假如这个object没有被其他类或容器retain过，那么它将自动释放掉。例如，layer->addChild(sprite)，这个sprite增加到这个layer的子节点列表中，他的声明周期就会持续到这个layer释放的时候，而不会在当前消息循环的最后被释放掉。   
这就是为什么你不能在网络线层中管理CCObject生命周期，因为在每一个UI线程的最后 ，自动释放对象将会被删除，所以当你调用这些被删掉的对象的时候，你就会遇到crash。   
简而言之，这只有两种情况你需要调用release（）方法

1. 你new一个cocos2d::CCObject子类的对象，例如CCSprite，CCLayer等。
2. 你得到cocos2d::CCObject子类对象的指针，然后在你的代码中调用过retain方法。  

上边可以看到如何讲一个对象添加到自动释放池中，下边来看看引擎何时进行释放。
在CCDirector.cpp文件中，可以找到下边方法   
```cpp
void DisplayLinkDirector::mainLoop()
{
    if (_purgeDirectorInNextLoop)
    {
        _purgeDirectorInNextLoop = false;
        purgeDirector();
    }
    else if (! _invalid)
    {
        drawScene();
     
        // release the objects
        PoolManager::getInstance()->getCurrentPool()->clear();
    }
}
```   
可以看到，在绘制后调用了clear()方法。  
```cpp
void AutoreleasePool::clear()
{
#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
    _isClearing = true;
#endif
    for (const auto &obj : _managedObjectArray)
    {
        obj->release();
    }
    _managedObjectArray.clear();
#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
    _isClearing = false;
#endif
}
```   
在clear()方法中会调用release()方法对容器中所有引用对象的引用计数减一，然后清空容器。   
以一个简单的例子来说明下我自己的理解，在HelloWorld::init()方法中生成一个精灵。
```c++ 
bool HelloWorld::init()
{
    //////////////////////////////
    // 1. super init first
    if ( !Layer::init() )
    {
        return false;
    }
 
    Sprite* s = Sprite::create("CloseNormal.png");
    s->setTag(100);
    addChild(s);
    log("---- count before %d", s->getReferenceCount());
    
    this->scheduleUpdate();
    
    return true;
}

void HelloWorld::update(float dt)
{
    Ref * node = this->getChildByTag(100);
    log("-- after count %d", node->getReferenceCount());
}
```   

* 使用create方法创建精灵，此时s引用计数为1，并被添加到`AutoreleasePool`中   
* addChild(s)将s添加到HelloWorld层，此时引用计数加一，为2    
* 然后调到mainLoop()函数，调用drawScene()进行绘制，此时会调到HelloWorld::update()函数，打印引用计数为2      
* 绘制之后，对`AutoreleasePool`中对象的引用计数减一，并清空`AutoreleasePool`    
* 当下一帧调用到update()函数，打印的引用计数就为1了，当HelloWorld层从内从中移除时，s也会被移除。    
可以看到，在操作`AutoreleasePool`时，都是从`PoolManager`进行调用的，那什么时候在`PoolManager`的`_releasePoolStack`队列中添加`AutoreleasePool *`的？
```cpp
AutoreleasePool::AutoreleasePool(const std::string &name)
: _name(name)
#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
, _isClearing(false)
#endif
{
    _managedObjectArray.reserve(150);
    PoolManager::getInstance()->push(this);
}

AutoreleasePool::~AutoreleasePool()
{
    CCLOGINFO("deallocing AutoreleasePool: %p", this);
    clear();
    
    PoolManager::getInstance()->pop();
}
```  
这个可以在`AutoreleasePool`的构造函数中看到，将自身添加到`PoolManager`的队列中，在析构的时候pop掉。

###参考

1. [总结Cocos2d-x内存管理机制](http://cn.cocos2d-x.org/tutorial/show?id=1669)
2. [Cocos2d-x中的引用计数（Reference Count）和自动释放池（AutoReleasePool）](http://cn.cocos2d-x.org/article/index?type=cocos2d-x&url=/doc/cocos-docs-master/manual/framework/native/v2/memory/refcount-autoreleasepool/zh.md)