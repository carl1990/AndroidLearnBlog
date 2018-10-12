# Android 使用RemoteViews自定义通知栏

### 
最近接到一个需求，应用需要使用常驻通知栏显示一些自定义的view,并伴随这个一系列操作和视图更新，因此在这里记录一下。

#### 定义RemoteViews

1. 首先定义一个视图的XML文件，此处略去不表 <br>
 **PS:RemotesView只支持一些系统特定的view,并不支持一些自定义View.如下：**
 
	![MacDown logo](./1.png)

2. 通过该XMl生成RemotesView，并且添加到notification中去



		mRemoteViews = new RemoteViews(ContextUtil.get().getPackageName(), R.layout.notification_normal_layout);


		NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
        builder.setSmallIcon(R.mipmap.icon_notification);
        builder.setContent(mRemoteViews);
        builder.setWhen(System.currentTimeMillis());
        
        
#### 更新视图

RemotesView并不能直接获取到相应的View去设置相关属性它提供如下一些API去更新视图：
[传送门](https://developer.android.com/reference/android/widget/RemoteViews#public-methods_3)

之后需要使用之前构建的notification去通知刷线

	NotificationManager notificationManager = (NotificationManager) ContextUtil.get().getSystemService(Context.NOTIFICATION_SERVICE);
            notificationManager.notify(FOREGROUND_ID, notification);
            
### 设置点击事件---跳转
因为RemoteView无法获取view，也就无法想传统的View一样设置点击事件
他是通过
	
	public void setOnClickPendingIntent (int viewId, 
                PendingIntent pendingIntent)
去设置点击事件的，这里请注意一下`PendingIntent`，官方文档看着会比较懵逼，而网上会有一些误导性的文档介绍它 [比较好的一个解释](https://stackoverflow.com/questions/2808796/what-is-an-android-pendingintent) 和两张理解他：<br>
![MacDown logo](./3.jpg)

![MacDown logo](./2.png)

ps ：PendingIntent 只能调用起三种组件：

Activity<br>
Service<br>
Broadcast<br>

所以，我们先构建出一个打开Activity的 PanddingItent

	PendingIntent pendingIntent = PendingIntent.getActivity(context, 0, intent, PendingIntent.FLAG_CANCEL_CURRENT);
	
将其设置到相应的view上，此时点击view就会执行相应的跳转。

### 坑
后面由于在点击RemoteViews上的视图的时候系统跳转的同时，更新视图，所以之前构建的打开的Activity的方法就无法做到了，此时只好通过构建广播的方式来处理跳转和视图更新

	Intent intent = null;
        intent = new Intent("XXXXX_NOTIFY_ACTION");
	PendingIntent pendingIntent = PendingIntent.getBroadcast(context, 0, intent, PendingIntent.FLAG_CANCEL_CURRENT);
	
然后通过RecieVer来处理
	
	
    public class NotificationforReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            if (action.equals("COM_YMM_DRIVER_NOTIFY_ACTION")) {
                switch (intent.getStringExtra("id")) {
                    case NotificationData.NOTIFICATION_TYPE_CARGO:
                        context.startActivity(Router.route(context, Uri.parse("xxxx")));
                        NotificationData notificationCargo = NotificationViewHelper.get().getNotificationCargo();
                        notificationCargo.setMessageCount(0);
                        NotificationViewHelper.get().newsIncoming(notificationCargo);
                        break;
                    case NotificationData.NOTIFICATION_TYPE_CHAT:
                        context.startActivity(Router.route(context, Uri.parse("xxxx")));
                        break;
                    case NotificationData.NOTIFICATION_TYPE_ORDER:
                        context.startActivity(Router.route(context, Uri.parse("xxxxx")));
                        NotificationData notificationOrder = NotificationViewHelper.get().getNotificationOrder();
                        notificationOrder.setMessageCount(0);
                        NotificationViewHelper.get().newsIncoming(notificationOrder);
                        break;
                    default:
                        break;

                }
            }
        }
    }
    
    
 但是此时出现了一个坑就是：
 **点击了通知之后Notification的Statusbar不会自动收起来了**,无法及时看到页面的跳转，用户体验不是很好，所以只好强制收起**StatusBar**
 
 	public static void collapseStatusBar(Context context) {
        try {
            Object statusBarManager = context.getSystemService("statusbar");
            Method collapse;

            if (Build.VERSION.SDK_INT <= 16) {
                collapse = statusBarManager.getClass().getMethod("collapse");
            } else {
                collapse = statusBarManager.getClass().getMethod("collapsePanels");
            }
            collapse.invoke(statusBarManager);
        } catch (Exception localException) {
            localException.printStackTrace();
        }

    }
    
 并且在manifeast中添加如下权限：
 
 	<uses-permission android:name="android.permission.EXPAND_STATUS_BAR" />
 	
 	
 这样就会在点击的时候响应处理的同时自动收起通知栏
 
 

	