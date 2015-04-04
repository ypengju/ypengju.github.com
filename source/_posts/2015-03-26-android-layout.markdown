---
layout: post
title: "Android -- Layout"
date: 2015-03-26 22:18:31 +0800
comments: true
categories: android
---    
###LinearLayout
顾名思义，线性布局，其中的控件要么被竖向排列，要么被横向排列，排列方式是通过android:orientation属性来完成，这是一般应用中最常使用的一种布局文件，比如下边的例子   
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingLeft="16dp"
    android:paddingRight="16dp"
    android:orientation="vertical" >
    <EditText
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="@string/to" />
    <EditText
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="@string/subject" />
    <EditText
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:gravity="top"
        android:hint="@string/message" />
    <Button
        android:layout_width="100dp"
        android:layout_height="wrap_content"
        android:layout_gravity="right"
        android:text="@string/send" />
</LinearLayout>
```
从上到下，一次排列了三个EditText和一个Button，具体使用可参考[LinearLayout](http://developer.android.com/reference/android/widget/LinearLayout.html)和参数[LinearLayout.LayoutParams](http://developer.android.com/reference/android/widget/LinearLayout.LayoutParams.html)    <!--more-->
###RelativeLayout
RelativeLayout是采用相对布局的，每个控件的位置都可指定依赖于其他控件或者父控件的相对位置。比如一个Button在一个EditText控件的下边，又在另一个Button的左边，这样设定后，我们就可确定此Button的位置了。有事后当我们在一个界面布局，有列也有行的时候，如果LinearLayout布局时，就需要好几个LinearLayout嵌套来完成工作，整个结构就比较复杂，但是我们可以只用一个RelativeLayout就可以替代上边所有的LinearLayout。使得整个结构变得简单。RelativeLayout定义了很多属性来完成相对布局的任务，比如`android:layout_below`，`android:layout_toRightOf`等属性，这些属性的值我们指定另一个空间的ID，这样就可完成相对于另一个控件的布局，比如下边的例子:
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingLeft="16dp"
    android:paddingRight="16dp" >
    <EditText
        android:id="@+id/name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="@string/reminder" />
    <Spinner
        android:id="@+id/dates"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_below="@id/name"
        android:layout_alignParentLeft="true"
        android:layout_toLeftOf="@+id/times" />
    <Spinner
        android:id="@id/times"
        android:layout_width="96dp"
        android:layout_height="wrap_content"
        android:layout_below="@id/name"
        android:layout_alignParentRight="true" />
    <Button
        android:layout_width="96dp"
        android:layout_height="wrap_content"
        android:layout_below="@id/times"
        android:layout_alignParentRight="true"
        android:text="@string/done" />
</RelativeLayout>
```
比如第一个Spinner在父控件RelativeLayout的左边，在第一个name控件的下边，在times控件的左边，效果就是下边这个样了。   
{% img  /images/android/layout/RelativeLayout.png %}   
RelativeLayout还有其他的一些参数可参考[RelativeLayout.LayoutParams](http://developer.android.com/reference/android/widget/RelativeLayout.LayoutParams.html)，[RelativeLayout](http://developer.android.com/reference/android/widget/RelativeLayout.html)   
###ListLayout
ListLayout是一个列表布局，列表中的每个item都是通过Adapter将其插入列表中的，这些数据我们通过代码中的数组或数据库查询将其结果展现在视图中。
《暂未完全理解》
###GridLayout
GridLayout有点像ListLayout，只不过ListLayout是单列的，GridLayout是多列的而已。《暂未完全理解》