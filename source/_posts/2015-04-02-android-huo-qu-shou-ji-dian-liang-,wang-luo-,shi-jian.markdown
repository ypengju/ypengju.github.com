---
layout: post
title: "Android -- 获取手机电量，网络，时间"
date: 2015-04-02 14:34:38 +0800
comments: true
categories: android
---
为了在游戏中达到更好的用户体验，现在好多游戏都会将当前电量，网络连接，手机时间等显示在游戏界面中方便玩家更好的安排游戏时间，虽然这是个小小的提示，但却是一个很方便贴心的小功能，获得越来越多游戏采用，但是在cocos2d-x中并不支持这些功能，需要利用原生系统去获取这些信息，并反馈给cocos2d-x中，这篇文章主要学习总结了在安卓手机中如何实现获取这些信息。    
电量，这个在游戏中可以用一个进度条来实现这个功能，所以只需要知道电量百分比然后传给cocos2d-x就可以实习这个功能。网络，其实主要获取使用wifi和数据流量就可以了，至于是用2G还是3G个人感觉没有必要弄的这么细，如果是wifi的话可以提示wifi信号强弱。时间的话，其实可以在cocos2d-x中来获取，这里也列出安卓中获取时间的方法。除了获取这些信息外，还需要注册一个BroadcastReceiver，当这些量变换时，来接受系统的一些通知，用来改变这些值来通知cocos2d-x改变界面显示。<!--more-->
###获取电量
```java
private BroadcastReceiver mReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        if (Intent.ACTION_BATTERY_CHANGED.equals(action)) {
            changeBattary(intent);     
        }
    }
};

@Override
protected void onResume() {
    super.onResume();
    //注册监听电池电量变化
    registerReceiver(mReceiver, new IntentFilter(Intent.ACTION_BATTERY_CHANGED));
}

/**
 * 改变电池电量
 */
private void changeBattary(Intent intent) {
    int level = intent.getIntExtra("level", 0);
    int scale = intent.getIntExtra("scale", 100);
    mTextView.setText("Battery level: "
                    + String.valueOf(level * 100 / scale) + "%");
}
```    
首先需要生成一个BroadcastReceiver，然后在onResume()方法中对电量变换系统广播进行注册，当手机电量进行变换的时候，系统发出Intent.ACTION_BATTERY_CHANGED广播，此时我们注册的mReceiver收到，并调用onReveive方法，然后我们对收到的电量改变广播进行处理，获取当前电量，此处将它显示在了一个TextView中，游戏中我们可以将这个值传递给cocos2d-x，然后用进度条表示当前电量。可以考虑在不同电量时，采用不同进度条颜色提示以达到最好的效果。   
###获取网络状态
```java
public static final int TYPE_NET_WORK_DISABLED = 0;  //网络不可用
public static final int TYPE_WIFI = 1;               //wifi网络
public static final int TYPE_DATA = 2;               //数据流量网络

/**
 * 检查当前网络
 * @param context
 */
private void checkCurrentNetwork(final Context context) {
    long start = System.currentTimeMillis();
    int checkNetworkType = checkNetType(this);
    Log.i("NetType", "===========time:" + (System.currentTimeMillis() - start));
    switch (checkNetworkType) {
        case TYPE_WIFI:
            mNetWork.setText("network: WIFI");
            break;
        case TYPE_NET_WORK_DISABLED:
            mNetWork.setText("network: NO");
            break;
        case TYPE_DATA:
            mNetWork.setText("network: DATA");
            break;
        default:
            break;
    }
}

/**
 * 检测网络类型
 * @param context
 * @return
 */
private int checkNetType(Context context) {
    final ConnectivityManager connectivityManager = (ConnectivityManager)context.getSystemService(Context.CONNECTIVITY_SERVICE);
    final NetworkInfo networkInfo = (NetworkInfo)connectivityManager.getActiveNetworkInfo();
    //判断网络是否可用
    if (networkInfo == null || !networkInfo.isAvailable()) {
        //网路不可用
        return TYPE_NET_WORK_DISABLED;
    } else {
        // NetworkInfo不为null开始判断是网络类型
        int netType = networkInfo.getType();
        if (netType == ConnectivityManager.TYPE_WIFI) {
            return TYPE_WIFI;
        } else if (netType == ConnectivityManager.TYPE_MOBILE) {
            return TYPE_DATA;
        }
    }
    return TYPE_NET_WORK_DISABLED;
}
```
使用网络是，需要先在AndroidManifest.xml文件中进行权限声明
```
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```    
定义了表示网络状态的三个常量，用来表示无网络，wifi网络，流量网络。通过Context.CONNECTIVITY_SERVICE获取设备网络连接管理类，然后获取网络信息，然后进行网络状态判断，如果对流量连接还需要判断具体运营商网络情况，可参考这篇博文["Android端如何获取手机当前的网络状态,比如wifi还是3G, 还是2G, 电信还是联通,还是移动"](http://blog.csdn.net/yinkai1205/article/details/8983861)。当网络切换的时候，应该在网络中变换显示，所以这里也要加一个监听在onResume()方法中
```
//注册监听网络连接变化
registerReceiver(mReceiver, new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION));
```
###获取wifi信号强度
```java
    /**
     * 改变wifi信号强弱
     * @param context
     */
    private void changeWifiRssi(Context context) {
        int networkType = checkNetType(context);
        if (networkType == TYPE_WIFI) {
            WifiManager wifiManager = (WifiManager)context.getSystemService(WIFI_SERVICE);
            WifiInfo wifiInfo = wifiManager.getConnectionInfo();
            int signalLevel = WifiManager.calculateSignalLevel(wifiInfo.getRssi(), 4); // 0 -- 3
            mWifiRssi.setText("" + signalLevel);
        }
    }
```
这里判断先判断当前网络状态是不是wifi，如果是wifi的话，获取wifi信息，getRssi()方法返回wifi信号的强弱，将其传入calculateSignalLevel方法中，第二参数为分为多少个等级，函数返回当前wifi信号等级，取值范围是从0--所分等级-1，像上边的返回值就是0--3。最强为3。当wifi信号强弱变换时，也需要实时返回给游戏界面，所以这里也需要监听wifi信号的变换。   
```
//注册监听网络wifi型号强弱变化
registerReceiver(mReceiver, new IntentFilter(WifiManager.WIFI_STATE_CHANGED_ACTION));
```
同时需要在AndroidManifest.xml文件中进行权限声明
```
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
```
###获取本地时间
```
private String getCurrentTime() {
    String timeformat;
    ContentResolver cv = this.getContentResolver();
    String strTimeFormat = android.provider.Settings.System.getString(cv,
                                       android.provider.Settings.System.TIME_12_24);
    if(strTimeFormat.equals("24"))
    {
        timeformat = "HH:mm:ss";
    } else {
        timeformat = "hh:mm:ss";
    }
    
    SimpleDateFormat formatter = new SimpleDateFormat(timeformat, Locale.getDefault());
    Date date = new Date(System.currentTimeMillis());
    String curTimeString = formatter.format(date);
    return curTimeString;
}
```
这里为达到更好的用户体验，游戏中的时间制应该和用户系统设置时间制保持一致，所以这里先判断系统使用的是12小时制还是24小时制，然后用SimpleDateFormat对时间进行格式化，然后获取当前系统时间。此时获取的时间是不变的，因为这里显示的时间是详细到秒的，所以需要每秒对时间进行更新，一般游戏中，我们显示到分钟就可以了，这样可以运行更少，更省电一点。这里做一个定时器，对时间进行跟新
```
Handler mHandler = new Handler() {
    public void handleMessage(Message msg) {
        mCurrentTime.setText(getCurrentTime().toString());
        super.handleMessage(msg);
    };
};

Timer time = new Timer();
TimerTask tasker = new TimerTask() {
    @Override
    public void run() {
        mHandler.sendEmptyMessage(1);
    }
};
time.schedule(tasker, 1000, 1000);
```


上边所有注册的监听器，应该在onStop()方法中进行销毁。
```
@Override
protected void onStop() {
    super.onStop();
    unregisterReceiver(mReceiver);
}
```
这样基本我们在游戏中显示电量，网络状态，时间的功能就可以实现了。
