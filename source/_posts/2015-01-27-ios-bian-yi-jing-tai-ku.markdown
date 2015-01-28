---
layout: post
title: "IOS -- 编译静态库"
date: 2015-01-27 16:47:25 +0800
comments: true
categories: ios
---
今天项目接第三方静态库被坑了一下，模拟器编译错误，真机编译通过，原来不懂，以为项目路径等问题，查了半天，没发现问题，后来才发现是第三方静态库的问题，被坑也叫坑，有坑赶紧填，填了就是路，上！    
原来ios静态库分为两种，模拟器和真机的，两个静态是不一样的，原因在于模拟器用的是i386框架的，真机使用的arm6,arm7框架的，上边的坑就是这样埋下的。下边来从一个新的静态库工程开始，看看静态库的创建过程。   
####创建静态库工程
如下图开始创建静态库工程
{% img  /images/libstatic/1.png %}    
<!--more-->为了方便，只定义了两个方法`printHelloWorld`和`printABCD`
{% img  /images/libstatic/2.png %}  
编译之前，我们需要选择是真机库还是模拟器库，如下图，IOS Device就是真机，模拟器编译出来的就是模拟器的，但需要注意的是模拟器用4S编译的库在使用的时候，如果用5以上的模拟器，会报错，所以编译模拟器库的时候最好选择5以上版本进行编译。
{% img  /images/libstatic/3.png %}   
此时直接编译会出错
{% img  /images/libstatic/4.png %}
我测的是debug静态库，需要进行如下签名设置，release版本也需要进行相关签名设置
{% img  /images/libstatic/5.png %}
编译通过后，在Products文件夹下回出现.a文件，右键->Show in Finder，Debug-iphoneos文件夹下的就是真机库，Debug-iphonesimulator文件夹下的就是模拟器库。
{% img  /images/libstatic/6.png %}   
如果你将真机的库在模拟器上使用就会报找不到i386,找不到文件，这就是我在项目中遇到的坑了，其实我们可以使用`lipo`命令来查看.a文件的信息
```
Debug-iphoneos$ lipo -info libHelloStatic.a 
Architectures in the fat file: libHelloStatic.a are: armv7 arm64 
Debug-iphonesimulator$ lipo -info libHelloStatic.a 
input file libHelloStatic.a is not a fat file
Non-fat file: libHelloStatic.a is architecture: i386
```   
这样可以可以看出模拟器的是i386的，真机是arm的。但是分两个总是很麻烦，我们可以用lipo命令将两个包合并成一个包，这样两个都就可以使用了。   
```
$ lipo -create ./os/libHelloStatic.a ./similator/libHelloStatic.a -output ./libHelloStatic.a
```
前两个是文件路径，最后一个是输出路径，合并后查看信息
```
$ lipo -info libHelloStatic.a 
Architectures in the fat file: libHelloStatic.a are: armv7 i386 arm64 
```
填坑结束！
