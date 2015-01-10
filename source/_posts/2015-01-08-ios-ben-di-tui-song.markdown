---
layout: post
title: "IOS -- 本地通知"
date: 2015-01-08 11:33:25 +0800
comments: true
categories: ios
---
为了提高用户的关注度，我们经常会推送一些新的内容给用户。ios中主要有两种推送，一种是远程通知，一种是本地通知，远程通知是和服务器端配合完成的，这里暂不说明，这篇文章主要说下本地通知。   
本地通知是在ios4.0之后添加的，但是在ios8之后，在设置通知之前，需要先对通知进行注册，注册需要的通知类型，否则收不到响应类型的通知消息。
```objective-c
//ios8需要注册推送
if ([UIApplication instancesRespondToSelector:@selector(registerUserNotificationSettings:)]){
    //通知类型
    UIUserNotificationType types = UIUserNotificationTypeBadge |
    UIUserNotificationTypeSound | UIUserNotificationTypeAlert;
    //设置通知类型和动画
    UIUserNotificationSettings *mySettings =
    [UIUserNotificationSettings settingsForTypes:types categories:nil];
    //注册
    [[UIApplication sharedApplication] registerUserNotificationSettings:mySettings];
}
```
<!--more-->
上边注册了Icon角标，声音，和警告通知，当程序第一次调用`registerUserNotificationSettings`的时候，程序会询问用户是否允许程序发送通知，在用户选择之后(不管是同意与否)，程序会异步调用`- (void)application:(UIApplication *)application didRegisterUserNotificationSettings:(UIUserNotificationSettings *)`函数。所有的注册类型都会可通过`currentUserNotificationSettings`变量获得。
```
UILocalNotification *notification = [[UILocalNotification alloc] init];
if (notification) {
    //设置时区
    notification.timeZone=[NSTimeZone defaultTimeZone];
    //设置推送的时间点
    NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
    [formatter setDateFormat:@"HH:mm:ss"];
    NSDate *date = [formatter dateFromString:@"09:00:00"];
    notification.fireDate=date;
    //通知重复提示的单位，可以是天、周、月
    notification.repeatInterval = kCFCalendarUnitDay;
    //推送的内容
    notification.alertBody = @"this is a notificaiton";
    //推送声音
    notification.soundName = UILocalNotificationDefaultSoundName;
    //应用右上角红色图标数字
    notification.applicationIconBadgeNumber = 1;
    //自定义信息
    NSDictionary *infoDict = [NSDictionary dictionaryWithObject:@"two" forKey:@"one"];
    notification.userInfo = infoDict;
}

UIApplication *app = [UIApplication sharedApplication];
[app scheduleLocalNotification:notification];
```
本地通知是通过`UILocalNotification`类来完成的，首先需要通过`fireDate`设置通知的时间点，还可设置通知的内容，声音，角标数等。除此之外，用户还可通过`userInfo`设置自定义数据。具体可参考[UILocalNotification官方文档](https://developer.apple.com/library/ios/documentation/iPhone/Reference/UILocalNotification_Class/index.html#//apple_ref/occ/cl/UILocalNotification)   

当程序正在运行时，收到通知时，会调用`application:didReceiveLocalNotification`方法。  
```
- (void)application:(UIApplication *)application didReceiveLocalNotification:(UILocalNotification *)notification
{
    NSDictionary *infoDict = notification.userInfo;
    NSString *str = [infoDict objectForKey:@"one"];
    if ([str isEqualToString:@"two"]) {
        NSLog(@"--------yes  equal");
    } else {
        NSLog(@"--------no ");
    }
}
```    

取消通知时，可以使用`cancelLocalNotification`取消具体某个通知或者通过`cancelAllLocalNotifications`取消全部通知。
```
[[UIApplication sharedApplication] cancelAllLocalNotifications];
```
如果我们添加了角标，在通知之后，角标会一直存在，当需要取消角标时，可利用下边语句
```
[[UIApplication sharedApplication] setApplicationIconBadgeNumber:0];
```

从ios8开始，通知添加了通知动作事件，如果有注意到，我们上边的进行注册的时候categories赋值为nil，此变量就是用来添加动作事件的。    
```objective-c
    //ios8需要注册推送
    if ([UIApplication instancesRespondToSelector:@selector(registerUserNotificationSettings:)]){
        UIMutableUserNotificationAction *acceptAction =
        [[UIMutableUserNotificationAction alloc] init];
        acceptAction.identifier = @"accept_action"; //ID
        acceptAction.title = @"Accept";             //按钮内容
        acceptAction.activationMode = UIUserNotificationActivationModeBackground;
        acceptAction.destructive = NO;
        acceptAction.authenticationRequired = NO;
        
        UIMutableUserNotificationAction *cancelAction =
        [[UIMutableUserNotificationAction alloc] init];
        cancelAction.identifier = @"cancel_action";
        cancelAction.title=@"Cancel";
        cancelAction.activationMode = UIUserNotificationActivationModeBackground;
        cancelAction.destructive = NO;
        cancelAction.authenticationRequired = NO;
        
        // First create the category
        UIMutableUserNotificationCategory *inviteCategory =
        [[UIMutableUserNotificationCategory alloc] init];
        inviteCategory.identifier = @"INVITE_CATEGORY";
        [inviteCategory setActions:@[acceptAction, cancelAction]
                        forContext:UIUserNotificationActionContextDefault];
        NSSet *categories = [NSSet setWithObject:inviteCategory];

        //通知类型
        UIUserNotificationType types = UIUserNotificationTypeBadge |
        UIUserNotificationTypeSound | UIUserNotificationTypeAlert;
        UIUserNotificationSettings *mySettings =
        [UIUserNotificationSettings settingsForTypes:types categories:categories];
        //注册
        [[UIApplication sharedApplication] registerUserNotificationSettings:mySettings];
    }
    
    UILocalNotification *notification = [[UILocalNotification alloc] init];
    if (notification) {
        //时区
        notification.timeZone=[NSTimeZone defaultTimeZone];
        NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
        [formatter setDateFormat:@"ss"];
        NSDate *date = [formatter dateFromString:@"10"];
        notification.alertBody = @"haha";
        notification.fireDate=date;
        //通知重复提示的单位，可以是天、周、月
        notification.repeatInterval = kCFCalendarUnitMinute;
        //推送声音
        notification.soundName = UILocalNotificationDefaultSoundName;
        //应用右上角红色图标数字
        notification.applicationIconBadgeNumber = 1;
        notification.category = @"INVITE_CATEGORY";
        
        NSDictionary *infoDict = [NSDictionary dictionaryWithObject:@"two" forKey:@"one"];
        notification.userInfo = infoDict;
        
    }
    
    UIApplication *app = [UIApplication sharedApplication];
    [app scheduleLocalNotification:notification];

```
以上就是一个动作的事件的注册过程，其中用到了`UIMutableUserNotificationAction`和`UIMutableUserNotificationCategory`具体用法可参考官方文档   
[UIMutableUserNotificationAction](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIMutableUserNotificationAction_class/index.html#//apple_ref/occ/cl/UIMutableUserNotificationAction)   
[UIMutableUserNotificationCategory](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIMutableUserNotificationCategory_class/index.html#//apple_ref/occ/cl/UIMutableUserNotificationCategory)   
这两个类，注册完之后，特别需要主意，要在本地通知中进行设置，否则没有效果。值为注册时category指定的ID   
```
notification.category = @"INVITE_CATEGORY";
```
这样当我们收到通知，下拉一下就可看到动作事件，有事件，就有事件的回调函数
```objective-c
- (void)application:(UIApplication *)application handleActionWithIdentifier:(NSString *)identifier forLocalNotification:(UILocalNotification *)notification completionHandler:(void (^)())completionHandler
{
    if ([identifier isEqualToString:@"accept_action"]) {
        NSLog(@"-----accept action");
    } else if ([identifier isEqualToString:@"cancel_action"]) {
        NSLog(@"-----cancel action");
    }
    
    if (completionHandler) {
        completionHandler();
    }
}
```
通过id我们就可以区分不同的动作，然后对其进行相应处理，最后调用`completionHandler();`。

###参考
[官方文档:Local and Remote Notification Programming Guide](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/IPhoneOSClientImp.html#//apple_ref/doc/uid/TP40008194-CH103-SW26)   
[iOS 8创建交互式通知](http://www.cocoachina.com/ios/20141009/9857.html)   
###源码
[https://github.com/ypengju/IOSTest/tree/master/LocalNotification](https://github.com/ypengju/IOSTest/tree/master/LocalNotification)
