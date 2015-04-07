---
layout: post
title: "Android -- ActionBar"
date: 2015-03-29 11:14:43 +0800
comments: true
categories: android
---
ActionBar是在安卓3.0（API11）之后引入Android框架，如果想在3.0之前的版本使用，就得使用支持库去完成这个工作了。支持库支持2.1（API7）级以上版本。所以当我们引入类的时候聚需要主义，如果是在API11及以上版本可以直接引入Android框架中的ActionBar   
```
import android.app.ActionBar
```
如果是在API7及以上版本，就需要引入支持库中的ActionBar
```
import android.support.v7.app.ActionBar
```
在API11以上的版本，当我们创建Activity的时候是默认自带ActionBar的，如果要在API7以上使用ActionBar的时候，Activity需要继承自ActionBarActivity，在代码中我们可以通过` getSupportActionBar()`获取ActionBar,如果需要隐藏ActionBar可以调用它的hide()方法。如果需要再次显示可以调用show()方法
```
ActionBar ab = getSupportActionBar();
ab.hide();
//ab.show();
```
当调用hide()方法的时候，系统会对界面布局进行调整，填充原来ActionBar的位置。   <!--more-->
当我们启动一个Activity的时候，在显示ActionBar的时候，系统是调用`onCreateOptionsMenu()`方法来布局ActionBar的。如下
```java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.menu_main, menu);
    return true;
}
```
其中menu_main就是ActionBar中的item布局，对这个布局进行修改，就可以看到效果。
```xml
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools" tools:context=".MainActivity">
    <item
        android:id="@+id/group_chat"
        android:title="GroupChat"
        android:icon="@drawable/menu_group_chat_icon"
        />

    <item
        android:id="@+id/feedback"
        android:title="FeedBack"
        android:icon="@drawable/menu_feedback_icon"
        />

    <item
        android:id="@+id/scan"
        android:title="Scan"
        android:icon="@drawable/menu_scan_icon"
        />
</menu>
```
效果如下   
{% img  /images/android/actionbar/1.png %}
默认都是在overflow按钮中的，如果想将其显示在ActionBar中，可以为item添加属性`app:showAsAction="ifRoom"`，这个属性表示如果ActionBar的空间够用个话就显示在ActionBar中，如果空间不够就显示overflow按钮中，默认这个只显示了一个图片，如果需要显示文本的话，指定withText就行，`app:showAsAction="ifRoom|withText"`,两种效果如下
{% img  /images/android/actionbar/2.png %}
{% img  /images/android/actionbar/3.png %}

###点击事件
当用户点击ActionBar上的按钮时，系统会调用Activity的`onOptionsItemSelected(MenuItem item)`方法，该方法返回一个MenuItem，我们可以通过它来来判断点击的是哪个Action。
```java
@Override
public boolean onOptionsItemSelected(MenuItem item) {
    int id = item.getItemId();

    switch (item.getItemId()) {
        case R.id.group_chat:
            System.out.println("-----touch group chat");
            break;
        case R.id.feedback:
            System.out.println("-----touch feed back");
            break;
        case R.id.scan:
            System.out.println("-----touch scan");
            break;
        default:
            break;
    }
    return super.onOptionsItemSelected(item);
}
```
###使用应用图标向上导航
在ActionBar作为导航栏用的时候，当我们需要返回上一个调用界面的时候，就可以使用ActionBar的这个特性。首先我们需要在代码中设置ActionBar具有向上导航的功能。

```
ActionBar ab = getSupportActionBar();
ab.setDisplayHomeAsUpEnabled(true);
```

这样在ActionBar的左边就会出现一个向左的箭头，这时点击还没反应，此时需要指定点击后跳转的Activity。
```
<application >
    <activity
        android:name="com.example.myfirstapp.MainActivity">
    </activity>
    <activity
        android:name="com.example.myfirstapp.DisplayMessageActivity"
        android:label="@string/title_activity_display_message"
        android:parentActivityName="com.example.myfirstapp.MainActivity" >
        <meta-data
            android:name="android.support.PARENT_ACTIVITY"
            android:value="com.example.myfirstapp.MainActivity" />
    </activity>
</application>
```   
通过在Application或者Activity指定属性`android:parentActivityName`，将其指向向上跳转的父类。在Android4.1(API16)级以上版本中只需在Activity中指定`parentActivityName`就可以，如果为了支持低版本，需要添加`<meta-data>`部分。
或者重写activity中的 getSupportParentActivityIntent() 和 onCreateSupportNavigateUpTaskStack()。
取决于用户如何到达当前界面的，父级activity也可能不同，那么这么做就很合适了。也就是说，如果用户有许多可以到达当前界面的路径，那么向上按钮应该按照用户实际到达的路劲回退导航。   

当用户按下向上按钮时系统会调用 getSupportParentActivityIntent() 去导航你的应用（在你应用自己的任务栈内）。取决于用户如何到达当前位置的，向上导航上应该打开的activity也会不同，所以你应该重写这个方法以返回可以打开合适父级activity的 Intent。  

当用户按下向上按钮并且你的activity也运行在不属于你应用的任务栈中时，系统会为你的activity调用 onCreateSupportNavigateUpTaskStack()。因此，当用户向上导航时你必须使用 TaskStackBuilder 传递到这个方法去建立那个应该合并的合适的回退栈。    

虽然你重写了 getSupportParentActivityIntent() 来具体指定用户在应用中的具体导航，但是你不想实现 onCreateSupportNavigateUpTaskStack()，那么你可以像上面所示在清单文件中声明“默认”父级activity达到同样的效果。然后 onCreateSupportNavigateUpTaskStack() 的默认实现会基于清单中声明的父级activity去合并回退栈。   
注解：如果你是使用大量的fragment而不是activity去构建你应用层级结构，那么上面的选项都不会生效。要在fragment里向上导航，你应该改为重写 onSupportNavigateUp()，通常通过调用 popBackStack() 方法把当前fragment从回退栈里出栈去来执行合适的fragment事务。    
###操作视图
在app中经常见到一种情况就是在ActionBar中点击搜索按钮，弹出搜索条，这个功能就可以通过只需要在Item中指定`app:actionViewClass`属性，同时在`app:showAsAction`添加collapseActionView值就可完成。collapseActionView是用来声明操作视图应该被收缩到一个按钮中的可选操作。
```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:yourapp="http://schemas.android.com/apk/res-auto" >
    <item android:id="@+id/action_search"
          android:title="@string/action_search"
          android:icon="@drawable/ic_action_search"
          app:showAsAction="ifRoom|collapseActionView"
          app:actionViewClass="android.support.v7.widget.SearchView" />
</menu>   
```    
如果需要配置操作视图，比如添加监听，可以在`onCreateOptionsMenu()`回调方法中，传递相应的 MenuItem 给静态方法 MenuItemCompat.getActionView() 去获取操作视图的对象。可以下边这样获取上面例子的搜索控件    
```java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.main_activity_actions, menu);
    MenuItem searchItem = menu.findItem(R.id.action_search);
    SearchView searchView = (SearchView) MenuItemCompat.getActionView(searchItem);
    // Configure the search info and add any event listeners
    //...
    return super.onCreateOptionsMenu(menu);
}
```
###伸缩操作视图
像上边这样，当我们点击按钮,搜索视图展现，当我们点击向上或者返回按钮的时候，搜索视图又收缩回去了，如果我们想在展开或者收缩的这个过程中同时对Activity中的视图进行一些变动操作的时候，可以使用下边的监听方法去完成这个工作    

```java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.menu_main, menu);

    MenuItem item = menu.findItem(R.id.group_chat);

    MenuItemCompat.setOnActionExpandListener(item, new MenuItemCompat.OnActionExpandListener() {
        @Override
        public boolean onMenuItemActionExpand(MenuItem menuItem) {
            System.out.println("---展开的时候");
            return true; //true 表示可展开， 否则不可展开
        }

        @Override
        public boolean onMenuItemActionCollapse(MenuItem menuItem) {
            System.out.println("---收缩的时候");
            return true;//true 表示可收缩，否则不可收缩
        }
    });
    return super.onCreateOptionsMenu(menu);
}
```
###ActionProvider
和操作视图一样，ActionProvider也是在ActionBar的一个按钮上。不同的是，ActionProvider完全控制操作行为变现和并在按下时显示下拉菜单。我们可以继承ActionProvider类来实现自己的ActionProvider,不过安卓已经提供了一些预构建的类来方便我们调用，比如ShareActionProvider,可以用来显示分享类表。在使用ShareActionProvider的时候，首先需要在item中进行`app:actionProviderClass`属性指定，如下   
```
<item
android:id="@+id/group_chat"
android:title="GroupChat"
android:icon="@drawable/menu_group_chat_icon"
app:showAsAction="ifRoom"
app:actionProviderClass="android.support.v7.widget.ShareActionProvider"
/>
```   

此时运行程序，会在右上角出现一个分享按钮，但是点击后无效果，接着指定分享操作    

```
private ShareActionProvider actionProvider;
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    // Inflate the menu; this adds items to the action bar if it is present.
    getMenuInflater().inflate(R.menu.menu_main, menu);

    MenuItem item = menu.findItem(R.id.group_chat); //获取Action
    actionProvider = (ShareActionProvider) MenuItemCompat.getActionProvider(item); //获取ShareActionProvider
    actionProvider.setShareIntent(getDefaultIntent()); //设置Intent
    return super.onCreateOptionsMenu(menu);
}

private Intent getDefaultIntent() {
    Intent intent = new Intent(Intent.ACTION_SEND);
    intent.setType("image/*");
    return intent;
}
```   

这样在点击share按钮后，就是如图效果    
{% img  /images/android/actionbar/4.png %}    
###添加导航栏选项卡
首先需要实现一个类，实现ActionBar.TabListener接口   

```java
import android.app.Activity;
import android.support.v4.app.Fragment;
import android.support.v7.app.ActionBar;

/**
 * Created by ypengju on 2015/3/29.
 */
public class MyActionBarTab<T extends Fragment> implements ActionBar.TabListener {

    private Fragment fragment;
    private final Activity mActivity;
    private final String mTag;
    private final Class<T> mClass;

    public MyActionBarTab(Activity mActivity, String mTag, Class<T> mClass) {
        this.mActivity = mActivity;
        this.mTag = mTag;
        this.mClass = mClass;
    }

    @Override
    public void onTabSelected(ActionBar.Tab tab, android.support.v4.app.FragmentTransaction fragmentTransaction) {
        if (fragment == null) {
            fragment = Fragment.instantiate(mActivity, mClass.getName());
            fragmentTransaction.add(android.R.id.content,fragment,mTag);
        } else {
            fragmentTransaction.attach(fragment);
        }
    }

    @Override
    public void onTabUnselected(ActionBar.Tab tab, android.support.v4.app.FragmentTransaction fragmentTransaction) {
        if (fragment != null) {
            fragmentTransaction.detach(fragment);
        }
    }

    @Override
    public void onTabReselected(ActionBar.Tab tab, android.support.v4.app.FragmentTransaction fragmentTransaction) {

    }
}
```    

接着实现Fragmenty用来显示选项卡视图    

```
public class OneFragment extends Fragment {
    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        return super.onCreateView(inflater, container, savedInstanceState);
    }
}
```    

最后将其添加到ActionBar中    

```
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    ActionBar ab = getSupportActionBar();
    ab.setDisplayHomeAsUpEnabled(true);
    ab.setNavigationMode(ActionBar.NAVIGATION_MODE_TABS);
    ab.setDisplayShowTitleEnabled(false);
    ab.addTab(ab.newTab().setText("Tab选项卡一").setTabListener(new MyActionBarTab<OneFragment>(this, "one",OneFragment.class)));
    ab.addTab(ab.newTab().setText("Tab选项卡二").setTabListener(new MyActionBarTab<TwoFragment>(this, "two",TwoFragment.class)));
}
```    

效果如图    
{% img  /images/android/actionbar/5.png %}