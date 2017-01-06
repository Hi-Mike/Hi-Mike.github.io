---
layout: post
title: "Notification学习及兼容性解决"
date: 2016-10-14
categories:android
---

Notification作为一个常用的功能，几乎所有的APP都会用上。前段时间做项目的时候写出的代码在Android 5.0以上的效果很差，正好看到郭霖的blog正好讲了一个微技巧，很有用，完美解决了问题。

### 基本Notification

在如今Android版本跨度与兼容性严峻的形式下，使用兼容包是不可避免的了。我使用的是v7包里的 *NotificationCompat.Builder* 来构建 *Notification* 。先来看下最初的效果：

![微技巧前](http://7xnzl2.com1.z0.glb.clouddn.com/origin_notify.PNG)

![微技巧前](http://7xnzl2.com1.z0.glb.clouddn.com/origin_notify_bg.png)

其实效果还行，只是不够完美，但是如果直接用应用图标，就要看造化了。接受了微技巧后的效果：

![微技巧后](http://7xnzl2.com1.z0.glb.clouddn.com/after_notify.PNG)

![微技巧后](http://7xnzl2.com1.z0.glb.clouddn.com/after_notify_bg.PNG)

![画的图标](http://7xnzl2.com1.z0.glb.clouddn.com/notify.png)

逼格还是提升了不少的。

最后简单的代码：

```java
//setContentIntent等自行设置
Notification notification = new NotificationCompat.Builder(this)
                .setTicker("通知来啦")
                .setContentTitle("title")
                .setContentInfo("content info")
                .setWhen(System.currentTimeMillis())
                .setContentText("content text")
                .setSmallIcon(R.mipmap.notify)
                .setLargeIcon(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher))
                .setColor(Color.parseColor("#8ABC52"))
                .build();
        notificationManager.notify(0, notification);
```

### 自定义notification的layout

自定义notification的布局的时候，由于各个ROM的背景定制的不一样，导致自定义的时候难以做适配。经过在下面参考2中的学习，测试了下，主要是设置字体在5.0上下的样式。我只测试了6.0的真机和4.4的模拟器，国内的常见ROM并没有做测试。测试的效果还行：

![微技巧后](http://7xnzl2.com1.z0.glb.clouddn.com/custom4.4.png)

![画的图标](http://7xnzl2.com1.z0.glb.clouddn.com/custom6.0.png)

5.0以下和之上的字体样式不一样，所以得弄两套布局。给一下简单的代码：

```java
RemoteViews remoteViews = new RemoteViews(getPackageName(), R.layout.notification_custom);
        Notification notification = new NotificationCompat.Builder(this)
                .setSmallIcon(R.mipmap.notify)
                .setContent(remoteViews)
                .build();
        notificationManager.notify(1, notification);
```

```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@android:color/transparent"
    android:orientation="vertical">

    <ImageView
        android:id="@+id/img"
        android:layout_width="56dp"
        android:layout_height="56dp"
        android:src="@mipmap/ic_launcher" />

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerVertical="true"
        android:layout_toRightOf="@id/img"
        android:text="哈哈哈哈哈哈"
        android:textAppearance="@android:style/TextAppearance.Material.Notification.Info" />

    <ImageView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentRight="true"
        android:src="@mipmap/notify"
        android:tint="#999999" />

</RelativeLayout>
```

### 前台Service

在一些情况下，应用可以将Service与Notification结合，如一些音乐类的应用，将Service转为前台以此提高优先级。

```java
RemoteViews remoteViews = new RemoteViews(getPackageName(), R.layout.notification_custom);
        Notification notification = new NotificationCompat.Builder(this)
                .setSmallIcon(R.mipmap.notify)
                .setContent(remoteViews)
                .build();
		//注意这里的id不能为0，否则系统不认的，看startForeground的注释
        notificationManager.notify(1, notification);
        startForeground(1, notification);
```

走走停停，以后慢慢补充。

参考学习：

[Android通知栏的微技巧](http://blog.csdn.net/sinyu890807/article/details/50945228)

[Android自定义Notification并没有那么简单](http://www.sixwolf.net/blog/2016/04/18/Android%E8%87%AA%E5%AE%9A%E4%B9%89Notification%E5%B9%B6%E6%B2%A1%E6%9C%89%E9%82%A3%E4%B9%88%E7%AE%80%E5%8D%95/)