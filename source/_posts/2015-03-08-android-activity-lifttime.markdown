---
layout: post
title: "android -- Activity"
date: 2015-03-08 23:07:17 +0800
comments: true
categories: android
---
Activity是应用程序的一个组件，它提供一个屏幕使用户可以与程序进行交互，像打电话，看微博，发邮件，浏览地图等其实都是在Activity中完成的，我们平时看到Android应用程序的界面，其实就是一个Activity。通常一个应用程序会含有多个Activity，像左右滑动切换界面，不同的功能界面等。其实就是从一个Activity切换的另一个Activity，新的Activity显示了，那么旧的Activity呢，它被压入了手机后台的一个栈中，栈具有后进先出的特性，所以当我们点击Back键的时候，程序就会从后台栈中取出上次压入的Activity。Activity在前台与后台之间切换的时候，会调用不同的回调函数，这就是Activity的生命周期了，我们可以在这些对应的函数中完成我们想要的操作。   

###创建Activity
当我们创建Activity的时候，必须继承自Activity或者它的子类。同时必须实现父类中的`onCreate()`方法。因为系统会调用改方法来创建Activity，在该方法中，我们应该对页面进行显示处理。还有其他一些生命周期方法，比如`onStart()`， `onStop()`， `onResume()`等可根据需要来自己实现。创建好Activity后，我们就需要对页面添加空间，使程序可以和用户进行交互。Android提供了大量的控件供程序使用，比如按钮，复选框，图片等。我们可以通过程序对这些控件进行布局管理，通常每个Activity我们都会配置一个对应的XML文件进行布局管理，然后通过程序进行调用响应事件处理。<!--more-->
###Activity配置
每个Activity我们都需在AndroidManifest.xml文件中进行生命，生命方式：    
```xml Activity声明
<manifest ... >
  <application ... >
      <activity android:name=".ExampleActivity" />
      ...
  </application ... >
  ...
</manifest >
```
activity元素只有一个属性`android:name`，此属性用来指明Activity的名称。如果你想使你的Activity能被其他应用程序使用，你就需要为这个Activity标签指定<inetent-filter>元素,在每个程序的AndroidManifest.xml文件中，都会有类似下边的代码   
```xml
<activity android:name=".ExampleActivity" android:icon="@drawable/app_icon">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```
上边的activity被指定为程序打开时第一个加载的Activity。<action>元素指定这是一个这个应用程序主入口点。<category>元素指定，这个activity应该被列入系统应用程序列表中（为了允许用户启动这个activity）。如果想让你自己的Activity被其他程序使用，你就必须在此<activity>元素中包涵<inetent-filter>元素和<action>元素，可以选择性的包涵<category>元素或者一个<data>元素。这些元素指定你的activity能响应的intent的类型。
###打开一个Activity
当我们在Activity直接切换的时候，我们需要用到Intent类，我们需要在Intent中指定我们跳转到的Activity，然后调用`startActivity()`方法，就像下边调用一样：
```
Intent intent = new Intent(this, SecondActivity.class)
startActivity(intent)
```
除了直接调用外，有时我们还需要在两个Activity之间传输数据，我们可以使用Intent的`putExtra()`方法在Intent中添加数据。这样在SecondActivity中我们可以将传输过来的数据解析使用。除了startActivityForResult()，这个方法有两个参数，第一个为Intent，第二个为自定义的请求码，调用这个函数的时候，我们需要在调用的Acitivity中实现`onActivityResult()`方法。
```
Intent intent = new Intent(LifecycleActivity.this, SecondActivity.class);
startActivityForResult(intent, INTENT_FOR_RESULT);
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    System.out.println("--result --");
}
```
是否调用成功需要根据resultCode进行判断，并且通过requestCode不同的请求。
###生命周期
Activity中最重要的一个知识点，这个我们必须搞清楚并记牢，那就是一个Activity的生命周期，我们在做开发的时候要时刻清楚，Activity在什么时候会干什么事。下边是官网文档给出的一个Activity生命周期图。   

{% img  /images/android/activity/activity_lifecycle.png %}

一个Activity从创建到显示我们可以看到它依次调用了`onCreate()`，`onStart()`，`onResume()`等方法。

* onCreate()方法 在Activity第一次被创建的时候调用
* onStart()方法  在Activity需要显示的时候调用
* onResume()方法 在Activity重新获取屏幕显示的时候调用

在调用玩onResume()方法后，Activity处于用户可见状态。如果此时被另一个Activity替换，当前Activity由可见到不可见，进入后台栈中，依次会调用`onPause()`，`onStop()`方法。

* onPause() 在Activity被另一个Activity替换的时候调用
* onStop()  在Activity完全不可见的时候调用

此时Activity并没有被系统回收而处于后台，如果系统资源不够用，系统会选择性的清楚一些后台程序，所以处于后台的Activity有可能被系统回收，在被系统回收或者我们选择退出程序的时候，`onDestroy()`方法会被调用，用于回收资源。   
如果一个后台的Activity从后台重新回到可见状态，系统会重新调用`onRestart()`，`onStart()`，`onResume()`方法，此处注意，调用了onRestart()方法，而在第一次创建到可见的时候，并不会调用该方法。下边两个Activity之间切换的时候回调函数的调用次序。

图1 第一次打开Activity1的时候
{% img  /images/android/activity/1.png %}   

图2 从Activity1跳转到Activity2的时候   
{% img  /images/android/activity/2.png %}
先调用了Activity1的`onPause()`方法，保存状态，然后创建Activity2，当Activity2完全显示的时候，调用Activity1的`onStop()`方法，停止Activity1的活动。   

图3 点击Back回到Activity1
{% img  /images/android/activity/3.png %}    
注意此时多调用了Activity1的`onRestart()`方法和Activity2的`onDestroy()`方法

###保存状态
当我们的Activity回到后台是，它并没有被系统销毁，所以Activity的状态和信息都被系统所保持，当Activity回到屏幕时，又可以正常显现，但是当我们Activity被系统销毁的时候，我们就会丢失这些信息。为了保存在数据，可以调用`onSaveInstanceState()`方法保存状态，并实现onRestoreInstanceState()，在Activity不可见的时候，系统会调用onSaveInstanceState()保存状态，万一Activity被销毁，当Activity被重新创建的时候，系统会调用onRestoreInstanceState()恢复之前的状态。