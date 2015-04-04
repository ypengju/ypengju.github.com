---
layout: post
title: "Android IOS 推送"
date: 2015-03-23 21:29:11 +0800
comments: true
categories: ios android
---
之前简单实现IOS本地消息推送的功能，但是这种方式太过死板，不能灵活应用，像游戏这种不定时开启活动，需要推送告知玩家的活动消息，这个变化性就比较大，一般游戏推送服务都需要和服务器端来配合完成，但是自己实现一套推送系统还是比较麻烦的，所以为了快速完成开发功能，好多游戏都是采用第三方推送系统。现在有好多优秀的第三方推送平台，有极光，个推，百度，小米的，每个平台都有自己的一些优势，可根据自己需求进行选择，这篇文章就学习总结接入极光推送平台到Android和IOS应用。   

首先需要在["极光官网"](https://www.jpush.cn/)注册帐号，下载SDK，这些都不用多说。下边开始动手      
###Android平台接入
极光的SDK还是做的很不错的，配置很简单，只需要简单的操作就可以完成，之前我也想试试个推来着，然后我看了下SDK接入文档后，毫不犹豫的选择了极光，哈哈，主要是简单啊。   
这里我使用的SDK版本是：Jpush_Android_SDK_1.7.3    
解压后有几个目录，doc下是文档，example是一个eclipse工程，可以直接跑，直接进行测试，libs下就是我们真正需要的库文件了，除此之外还有一个AndroidManifest.xml文件。因为比较简单，可以按照官方文档迅速进行配置，这里主要说下我在Android Studio中遇到的问题。   <!--more-->
第一个问题，当将jar文件添加到工程中之后，调用极光API的时候，找不到类，
```
JPushInterface.setDebugMode(true);
JPushInterface.init(this);
```
这是由于没有配置build，右键项目->Open Module Settings->Dependencies，然后将添加的jar文件添加进来。然后引入文件就可解决这个问题。   
第二个问题，当我按照官方配置然后跑起来的时候，在Android Studio中是直接报错了，报错内容如下：   
```
03-25 15:48:50.629    1738-1738/com.ypengju.jpushapplication E/AndroidRuntime﹕ FATAL EXCEPTION: main
    java.lang.ExceptionInInitializerError
            at cn.jpush.android.service.ServiceInterface.a(Unknown Source)
            at cn.jpush.android.api.JPushInterface.init(Unknown Source)
            at com.ypengju.jpushapplication.MainActivity.onCreate(MainActivity.java:19)
            at android.app.Activity.performCreate(Activity.java:5008)
            at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1079)
            at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2023)
            at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2084)
            at android.app.ActivityThread.access$600(ActivityThread.java:130)
            at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1195)
            at android.os.Handler.dispatchMessage(Handler.java:99)
            at android.os.Looper.loop(Looper.java:137)
            at android.app.ActivityThread.main(ActivityThread.java:4745)
            at java.lang.reflect.Method.invokeNative(Native Method)
            at java.lang.reflect.Method.invoke(Method.java:511)
            at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:786)
            at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:553)
            at dalvik.system.NativeStart.main(Native Method)
```
然后在官方问答中搜寻下，找到解决方法，JPush SDK 迁移到 Android Studio 需要添加.SO文件打包到APK的lib文件夹中,可以编辑 build.gradle 脚本，自定义 *.so 目录：
```
android {
    // .. android settings ..
    sourceSets.main {
      jniLibs.srcDirs = ['libs']  // <-- Set your folder here!
    }
 }
```
所以将中间sourceSets.main部分代码，添加到Build.gradle中就可以了。至此Android Studio配置Jpush成功，运行程序，然后推送，成功收到。    
Jpush在安卓设备上，不管应用处于打开或者关闭状态，应用都可接受到推送消息，这是由于当我们第一次运行程序后，Jpush以Android Service的形式长期运行在后台。    
在推送界面我们可以看到，除了通送通知外，极光推送还可推送消息和富媒体，消息通送可以使我们设备在收到消息后，调用的注册功能，比如说到一个消息，跳转到指定页面等。这个可根据需求实现。    
