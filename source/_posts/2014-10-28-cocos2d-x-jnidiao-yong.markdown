---
layout: post
title: "cocos2d-x -- jni调用"
date: 2014-10-28 21:26:14 +0800
comments: true
categories: cocos2d-x
---
cocos2d-x要在android手机上运行，就需要在java和c++两种语言之间进行调用，在cocos2d-x中完成这个任务的就是jni，它可以使两者之间互相调用，从而让用c++开发的cocos2d-x游戏，在android上完美运行。   
cocos2d-x中封装了JniHelper类，方便通过c++来调用java方法，此类位于platform/android/jni/目录下，先来看看这个类都提供了哪些方法。<!--more-->   
```c++ JniHelper.h
typedef struct JniMethodInfo_
{
    JNIEnv *    env;
    jclass      classID;
    jmethodID   methodID;
} JniMethodInfo;
class CC_DLL JniHelper
{
public:
    static void setJavaVM(JavaVM *javaVM);
    static JavaVM* getJavaVM();
    static JNIEnv* getEnv();

    static bool setClassLoaderFrom(jobject activityInstance);
    static bool getStaticMethodInfo(JniMethodInfo &methodinfo,
                                    const char *className,
                                    const char *methodName,
                                    const char *paramCode);
    static bool getMethodInfo(JniMethodInfo &methodinfo,
                              const char *className,
                              const char *methodName,
                              const char *paramCode);

    static std::string jstring2string(jstring str);

    static jmethodID loadclassMethod_methodID;
    static jobject classloader;

private:
    static JNIEnv* cacheEnv(JavaVM* jvm);

    static bool getMethodInfo_DefaultClassLoader(JniMethodInfo &methodinfo,
                                                 const char *className,
                                                 const char *methodName,
                                                 const char *paramCode);

    static JavaVM* _psJavaVM;
};
```   
结构体JniMethodInfo中封装了Jni环境，java类，类中的方法。
JniHelper提供了获取Java虚拟机的接口，JNI环境的接口，以及获取Java静态方法，和普通方法的接口，我们最常使用的就是这两个方法了，以及提供了一个将java字符串与c++字符串转化的函数。

```c++ getStaticMethodInfo
    bool JniHelper::getStaticMethodInfo(JniMethodInfo &methodinfo,
                                        const char *className, 
                                        const char *methodName,
                                        const char *paramCode) {
        if ((nullptr == className) ||
            (nullptr == methodName) ||
            (nullptr == paramCode)) {
            return false;
        }

        JNIEnv *env = JniHelper::getEnv();
        if (!env) {
            LOGE("Failed to get JNIEnv");
            return false;
        }
            
        jclass classID = _getClassID(className);
        if (! classID) {
            LOGE("Failed to find class %s", className);
            env->ExceptionClear();
            return false;
        }

        jmethodID methodID = env->GetStaticMethodID(classID, methodName, paramCode);
        if (! methodID) {
            LOGE("Failed to find static method id of %s", methodName);
            env->ExceptionClear();
            return false;
        }
            
        methodinfo.classID = classID;
        methodinfo.env = env;
        methodinfo.methodID = methodID;
        return true;
    }
```     
getStaticMethodInfo方法中，通过JNIEnv检查传进来的java类，方法等都是否存在，如果存在，最后将其分装在JniMethodInfo中，接下来看看具体的调用。最后一个参数paramCode是用来说明Java函数的参数类型个数及返回值。
###c++调用java
我以cocos2d-x 3.2生成的例子为例，来看看c++和java的调用。   
####调用静态函数   
c++端代码：   
```
    JniMethodInfo info;
    bool isHave = JniHelper::getStaticMethodInfo(info, "org/cocos2dx/cpp/AppActivity", "staticMethod1", "()V");
    if (isHave) {
        log(" static void method if");
        info.env->CallStaticVoidMethod(info.classID, info.methodID);
    } else {
        log(" static void method else");
    }
```   
java端代码：   
```
public class AppActivity extends Cocos2dxActivity {
    public static void staticMethod1() {
        System.out.println("this is java static method");
    }
}
```   
运行编译cocos2d-x代码，在android上运行，就可以看到c++调用java打印`this is java static method`。   
此处需要注意，在c++端需要先引进JniHelper的类，并判断是在android环境下运行，否则会报错。  

```c++
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
#include "../cocos2d/cocos/platform/android/jni/JniHelper.h"
#endif
```   
c++端首先判断java端的函数是否存在，然后进行调用，主要就是getXXX方法的四个参数   

* 第一个参数 - JniMethodInfo对象，用来封装java类，方法等   
* 第二个参数 - Java类的路径，以报名加类名的形式   
* 第三个参数 - 方法名   
* 第四个参数 - 函数的参数类型，个数及返回值类型   

这里第四个参数比较特殊，具体说明： 
参数的格式为  `(参数)返回值` 括号内是参数类型和参数个数，括号外是返回值类型。具体与java类型对照图如下：   
{% img  /images/jni/javapar.png %}   
如果函数有多个参数，直接把简写并列即可，但是注意Object与Array型参数简写结尾的分号，示例：  
IIII //4个int型参数的函数   
ILjava/lang/String;I //整形，string类型，整形组合 (int x, String a, int y)  
所以上边例子第四个参数说明调用的是一个无参无返回值类型的函数。   
####调用非静态函数
c++端代码：  
```c++
JniMethodInfo info;
bool isHave = JniHelper::getMethodInfo(info, "org/cocos2dx/cpp/AppActivity", "voidMethod1", "(II)I");
if (isHave) {
    log(" void method if");
    jobject jobj = info.env->NewObject(info.classID, info.methodID);
    jint a = 10;
    jint b = 20;
    jint result = info.env->CallIntMethod(jobj, info.methodID, a, b);
    log("------return result %ld",result);
} else {
    log(" void method else");
}   
```   
java端代码：   
```java
public int voidMethod1(int a, int b) {
    System.out.println("this is java void method a + b = " + (a + b));
    return a + b;
}
```   
编译之后在android上运行，就可以看到打印`this is java void method a + b = 30`和`------return result 30`，分别是c++端和java端打印。此处`info.env->NewObject(info.classID, info.methodID)`方法也会调用一次voidMethod1函数，此处暂未弄清楚原因。   
首先调用JniHelper的getMethodInfo函数确定是否存在调用的函数，如果存在，首先生成java类的对象，然后调用方法，并获得返回值。jni提供了一系列`CallXXXMethod()`的方法，用来调用不同返回值类型的函数。另外可看到，传入的参数和返回值类型都是用`jint`类型，这是jni提供的对应的java数据类型，对应关系如下：   
{% img  /images/jni/javareturn.png %}   
通过JniHelper提供的方法，我们就可以方便的调用java中的函数，来完成android的操作。   
###java调用c++   
java端代码：   
```java
public static void staticMethod2() {
    int add = nativeMethod(1, 8);
    System.out.println("--- add = " + add);
}

public native static int nativeMethod(int a, int b);
```  
c++端代码：   
```c++
#ifdef __cplusplus
extern "C" {
#endif
    jint Java_org_cocos2dx_cpp_AppActivity_nativeMethod(JNIEnv *env, jobject thiz, jint a, jint b) {
        return a + b;
    }
#ifdef __cplusplus
}
#endif

//......
JniMethodInfo info;
bool isHave = JniHelper::getStaticMethodInfo(info, "org/cocos2dx/cpp/AppActivity", "staticMethod2", "()V");
if (isHave) {
    log(" static void method if");
    info.env->CallStaticVoidMethod(info.classID, info.methodID);
} else {
    log(" static void method else");
}  
```   

此处先从c++端调到java，然后再用java调到c++来实现两端互相调用。看上边代码可知，首先调用从c++调用java端的`staticMethod2`方法，在java端`staticMethod2`方法中，再调用`nativeMethod`方法，此方法具体在c++端实现，运行android程序后，可看到在控制台打印`--- add = 9`。说明成功调用。   
`nativeMethod`是在c++端进行实现的，java端相当于只进行了声明，但特别注意，在声明中要添加`native`关键字，以说明次函数为c++端函数。在c++端实现时，方法名具有一定的规则，首先是Java字段，然后是包名，类名，方法名，之间用下划线分开，在参数列表：第一个为Jni运行环境，第二个参数为调用对象，之后的为调用函数时的参数。   

本例代码: [https://github.com/ypengju/cocos2d-xTest/tree/master/JniTest](https://github.com/ypengju/cocos2d-xTest/tree/master/JniTest)


