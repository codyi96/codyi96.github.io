---
title: Android前台服务 - Foreground Service
catalog: true
date: 2020-08-02 01:27:48
subtitle:
header-img: header.jpg
tags:
- Android
- Android Service
---

# 概述
本文将回顾`服务`的相关知识和使用案例，以及如何隐藏由前台服务创建的通知栏
<div align="center">
    <img src="summary.jpg" width="80%" />
</div>

# 服务
`Service`是一种可在后台`长时间运行`的`无界面`应用组件。
<div align="center">
    <img src="service_lifecycle.png" width="80%" title="Service生命周期" />
</div>

## 服务的种类
按照运行现象，可以将服务分为前台服务和后台服务；按照启动方式，又可以将服务分为启动服务和绑定服务。值得一提的是，一个服务类既可以支持启动服务，同时也可以支持绑定，只要实现相应的回调方法即可：`onStartCommand()`（启动服务）和`onBind()`（绑定服务）。

## 服务的特征
1. 服务在应用进程的`主线程执行`，启动或绑定服务并不会创建自己的线程，也不会在单独的进程中执行（除非另行指定）。因此，耗时操作应当在服务中创建新的线程来完成，否则容易ANR
2. 启动服务一旦启动，就会`无限期运行`，直到其调用`stopSelf()`自行停止或其他组件调用`stopServicec()`将其停止
3. 在系统资源不足时，系统会根据优先级主动停止服务。其中`前台服务`拥有较高优先级，一般不会被停止，而`后台服务`的优先级则与运行时间有关，长时间运行的服务被停止的概率更高。同时，如果服务是由于资源不足而被系统停止，那么在系统资源满足的情况下，服务将被系统重启
4. `stopSelf()`和`stopServicec()`并不是将服务立即结束，仅是通知系统尽快销毁而已

## 服务基类
自定义服务可以继承以下两个基类
### Service
适用于所有服务。继承该基类时，服务中的业务操作必须创建新线程执行，因为服务默认使用应用的主线程，这会降低应用正在运行的Activity的性能甚至导致ANR
### IntentService（👎自API 30已废弃）
继承该基类时，服务会自动创建线程来处理每一个到来的Intent，且会在处理完所有Intent后自动停止服务，而无需使用者手动调用`stopSelf()`。
值得注意的是，自定义服务时不应重写`onStartCommand()`方法，否则可能会导致`onHandleIntent()`无法正常响应

## 前台服务
自Android 8.0（API 26）开始，Android系统开始执行严格的[后台执行限制](https://developer.android.com/about/versions/oreo/background)，后台应用不允许默默启动后台服务，只能启动前台服务，而前台应用则可以自由创建前台服务和后台服务。应用进入后台时，有一个持续数分钟的窗口期，在窗口期内仍然可以创建和使用前台/后台Service。窗口期结束后，应用被视为处于空闲状态，系统将停止应用的后台Service，就像服务调用了自己的`stopSelf()`方法一样。
### 后台应用的定义
何为后台应用？🤔
>如果满足以下任意条件，应用将被视为处于前台：
>- 具有可见 Activity（不管该 Activity 已启动还是已暂停）
>- 具有前台 Service
>- 另一个前台应用已关联到该应用（不管是通过绑定到其中一个 Service，还是通过使用其中一个内容提供程序）。例如，如果另一个应用绑定到该应用的 Service，那么该应用处于前台：
>   - IME
>   - 壁纸 Service
>   - 通知侦听器
>   - 语音或文本 Service
>
>如果以上条件均不满足，应用将被视为处于后台。
### 前台服务的启动
前台服务一般需要先通过`startForegroundService()`启动一个后台服务，同时该方法向系统发送信号，表明服务将会自行提升前台。启动服务后，该服务需要在`五秒内`调用自己的`startForeground()`方法显式提升服务至前台。而正是这个`startForeground()`方法，唤起了本文开头的那个烦人的系统通知。

PS. 通过`startService()`启动的服务，如果调用了`startForeground()`方法，也将升级为前台服务。但是这种用法容易给人造成误解，极不推荐使用

### 前台服务与通知栏
当应用在后台时，前台服务中的`startForeground()`方法将唤起一个通知栏。
我们知道，在高版本中系统通知必须寄宿于某个渠道之下，如果此时（指`startForeground()`调用时刻）该渠道未被授予通知权限甚至渠道还未创建完毕，系统会先采用默认的通知提示，即显示`xx应用正在后台运行`的系统通知栏，这个通知栏拥有一个特殊的notifyID，与`startForeground()`方法中传入的notifyID不同。如果渠道已获得授权，将展示用户传给`startForeground()`方法的通知信息。
顺便一提，在渠道未被授权或未创建完毕的情况下，使用`NotificationManager.notify()`将不会有任何响应。

# 隐藏前台服务通知栏
在介绍具体方案之前，不妨先思考一个问题：如果前台服务中调用了`NotificationManager.notify()`方法，会怎么样？很简单，如果`notify()`传入的`notifyID`和`startForeground()`传入的一致，那么将合并显示；反之，将会分别展示两个通知栏。
至此，隐藏前台服务通知栏的方案应该比较清晰了：通过一个前台服务类作为跳板，用其启动真实的服务，即其他组件启动跳板前台服务，跳板服务启动真实服务。
```java
// 跳板服务
public class RouteForegroundService extends Service {
    ...
    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        createNotificationChannel(this, CHANNELID, CHANNELNAME);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            mBuilder = new Notification.Builder(this, CHANNELID)
                    .setSmallIcon(R.drawable.ic_launcher_background)
                    .setContentText("我是跳板服务");
            startForeground(NOTIFY_ID, mBuilder.build());
        }
        MyService.start(this); // 启动真实服务
        stopSelf(); // 自杀
        return super.onStartCommand(intent, flags, startId);
    }
}
```
上述代码以最简单的方式展示了跳板服务的逻辑。在启动真实服务之后，跳板服务完成使命，调用`stopSelf()`关闭前台通知栏，实现前台服务通知栏的“隐藏”（由于碎片化，部分机型会短暂展示）。
如果在真实服务中需要`notify()`，可以设定与跳板服务一致的notifyID，进一步降低用户对前台服务通知栏的感知。

> 如果有多个服务需要采用此类方案，那么应该将跳转服务抽象为一个工具类，将需要启动的真实服务类及启动参数通过Intent传入跳板服务，并在跳板服务中解析参数以启动真实服务，而不是简单地调用`MyService.start()`，这里不再展开。

# 碎片化
相同固件版本下，不同手机对通知渠道的授权行为可能会不一致，一些手机（如OnePlus 6）在创建渠道时会默认开启，而一些手机则不会。同时，一些手机（还是OnePlus 6）即使关闭渠道，也不会出现`xx应用正在后台运行`通知。

# 参考资料
1. [服务概览](https://developer.android.com/guide/components/services)
2. [后台执行限制](https://developer.android.com/about/versions/oreo/background)
3. [IntentService](https://developer.android.com/reference/android/app/IntentService)
