---
layout: post
title: "设计模式 -- 单例"
date: 2015-01-04 16:54:35 +0800
comments: true
categories: designpattern
---
单例可以说是我们最常使用的一种设计模式了，它使得我们在调用过程中始终保持一个实例，用来进行全局调用。
```cpp
//Singleton.h
#ifndef __Singleton__Singleton__
#define __Singleton__Singleton__

#include <stdio.h>
class Singleton{
private:
    Singleton();

public:
    static Singleton* getInstance(); 
};

#endif /* defined(__Singleton__Singleton__) */

//Singleton.cpp
#include "Singleton.h"

static Singleton *instance = nullptr;
Singleton::Singleton()
{
}
Singleton* Singleton::getInstance()
{
    if (instance == nullptr) {
        instance = new Singleton();
    }
    return instance;
}
```   <!--more-->
* 私有构造，确保类不能被实例化，避免多个实例存在。
* 因为这个类其他地方不能实例化，所以需要类自己在静态方法中进行实例，这个方法是在new自己，因为其可以访问私有的构造函数，所以他是可以保证实例被创建出来的。
* 使用一个静态方法，对其进行实例化，先判断是否已经实例化，如果没有，创建实例，否则返回实例。
* 实例化保存在私有静态成员中。
* 然后在其他类中我们就可以使用Singleton::getInstance()来访问实例。   

这是一个非线程安全的，没有考虑内存释放的粗糙版本，为什么，请看这篇文章[深入浅出单实例Singleton设计模式](http://blog.csdn.net/haoel/article/details/4028232)
