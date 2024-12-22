---
title: 排查开机进 rescueParty
date: '2024-12-22 12:37:14'
updated: '2024-12-22 12:42:07'
tags:
  - Android
  - 无法开机
permalink: /post/check-the-booting-into-rescueparty-z1l4gnn.html
comments: true
toc: true
---



今天拿到了一台工程机，无法开机。现象是一开机就直接进 fastboot。机器拿到我这里的时候就已经无法进 fastboot 了，但是之前跟我说问题的同事说的是无法开机，会进 rescueParty，并且给我了日志截图。让我排查，所以需要先复现出那个时候这个机器的问题。

第一步是让它能够离开 fastboot。串口日志显示有 `boot_fail`​，但是底层同事说这个没什么问题，而是需要设置一下。

```Bash
[00003619] Loading Image recovery_a Done : 1 ms, Image size : 4096 Bytes
[00003620] Loading Image init_boot_a Done : 1 ms, Image size : 4096 Bytes
[00003620] Loading Image pvmfw_a Done : 0 ms, Image size : 4096 Bytes
[00003621] Slot _a is unbootable, trying alternate slot
[00003621] Err: line:1868 FindBootableSlot() status: Load Error
[00003621] Err: line:2253 LoadImageAndAuth() status: Load Error
[00003621] LoadImageAndAuth failed: Load Error
[00003621] [boot_fail]WriteErrorType : error_type is 11
[00003623] [boot_fail] LogTableHeader : magic=56775AF41BCDE0F0 version=1000 size=400 boot_content_offset=9A00000
[00003625] [bootfail]error_type_header: magic=627759541BCDE0F0 version=1000 size=1000 uefi_log_offset=100000 index=278 error_num=29 head[00003625] [boot_fail]WriteErrorType : last_current_log_offset is 29000
[00003626] [boot_fail]WriteErrorType: temp 901005101
[00003626] [boot_fail]WriteErrorType: DstStr error num:901005101
[00003626] [boot_fail] error_header  :  magic=E99BB480DD25E1F0 version=1000 size=1000 index=278 error_type=901005101 current_error=32 err[00003627] [boot_fail]WriteUefiLog :RAM Log Buffer Length is 65536!
[00003628] [boot_fail] LogTableHeader : magic=56775AF41BCDE0F0 version=1000 size=400 boot_content_offset=9A00000
[00003629] [bootfail]error_type_header: magic=627759541BCDE0F0 version=1000 size=1000 uefi_log_offset=100000 index=278 error_num=29 head[00003630] [boot_fail] WriteUefiLog :Offset is 9B34000
[00003632] [boot_fail] WriteUefiLog :record_uefi_header is not boot_fail.
[00003632] [boot_fail] uefi_header  :   magic=768B480DD48A6592 version=1000 header_size=1000 boot_index=278
[00003634] [boot_fail]uefi_log_header : magic=94682550DD25E1F0 version=1000 size=1000 index=278 counts=15 currents=15 flag=4
[00003635] Launching fastboot
```

所以设置这两个命令之后，就可以看到启动顺利过了 fastboot 阶段，进入后面阶段了。这两条命令

```Bash
fastboot erase misc
fastboot set_active a
```

> ​`fastboot erase misc`​
>
> * **作用**：用于擦除 Android 设备中的 misc 分区的数据.
> * **使用场景**：当设备出现因 misc 分区数据异常导致的故障，如开机卡 recovery、无法正常启动等问题时，可执行此命令来尝试修复.
> * **注意事项**：该命令执行后，misc 分区的数据将被永久删除，操作需谨慎，若执行错误可能导致设备变砖.
>
> ​`fastboot set_active a`​
>
> * **作用**：用于将 Android 设备的活动分区设置为 a 分区.
> * **使用场景**：在具有 a/b 分区的 Android 设备中，当需要切换活动分区以使用不同的系统版本或进行系统更新等操作时，可使用此命令.
> * **注意事项**：此命令仅适用于支持 a/b 分区的设备，在执行前需确保设备已进入 fastboot 模式，且确认设备的分区情况，以免设置错误导致系统无法正常启动.

现象上看可以看到白米，也可以看到上层 Android 徽标，说明启动可以进入上层，也说明了底层没什么问题。下面就继续查上层问题。

现在的现象变成反复重启，在徽标处卡住一段时间，然后就重启，也就是当时同事跟我说的那个现象。从串口的内核日志里可以看到

```Bash
[   39.847029][    T1] reboot: Restarting system with command 'RescueParty'
```

这一般说明上层有什么东西在不断启动失败，最后才会进入 rescueParty 里面。幸运的是，进入上层之后能有十几秒 adb 有效，所以趁这段时间赶紧抓一个 logcat。日志中有很多次 `launcher3`​ 启动失败的记录，显示需要解锁 user 0 才可以使用 widgets。这个报错同时会关闭虚拟机。

```Bash
12-18 07:24:30.610  5405  5405 D AndroidRuntime: Shutting down VM
12-18 07:24:30.610  2337  3206 D qcrilNrd: SUB[0][xiaomi_ecclist.cpp:472] find_matched_ecc_number_based_on_mcc_mnc: xiaomi_ecc find matched ecc list=0
12-18 07:24:30.610  2337  3206 D qcrilNrd: SUB[0][xiaomi_qcril_pbm.cpp:96] xiaomi_qmi_ril_send_ecc_list_indication: function exit <-------------
12-18 07:24:30.611  5405  5405 E AndroidRuntime: FATAL EXCEPTION: main
12-18 07:24:30.611  5405  5405 E AndroidRuntime: Process: com.android.launcher3, PID: 5405
12-18 07:24:30.611  5405  5405 E AndroidRuntime: java.lang.RuntimeException: Unable to start activity ComponentInfo{com.android.launcher3/com.android.launcher3.uioverrides.QuickstepLauncher}: java.lang.IllegalStateException: User 0 must be unlocked for widgets to be available
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:4440)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:4656)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:222)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.servertransaction.TransactionExecutor.executeNonLifecycleItem(TransactionExecutor.java:136)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.servertransaction.TransactionExecutor.executeTransactionItems(TransactionExecutor.java:106)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:83)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2898)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.os.Handler.dispatchMessage(Handler.java:109)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.os.Looper.loopOnce(Looper.java:249)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.os.Looper.loop(Looper.java:337)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.ActivityThread.main(ActivityThread.java:9486)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at java.lang.reflect.Method.invoke(Native Method)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:604)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:904)
12-18 07:24:30.611  5405  5405 E AndroidRuntime: Caused by: java.lang.IllegalStateException: User 0 must be unlocked for widgets to be available
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.os.Parcel.createExceptionOrNull(Parcel.java:3253)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.os.Parcel.createException(Parcel.java:3229)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.os.Parcel.readException(Parcel.java:3212)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.os.Parcel.readException(Parcel.java:3154)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at com.android.internal.appwidget.IAppWidgetService$Stub$Proxy.startListening(IAppWidgetService.java:774)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.appwidget.AppWidgetHost.startListening(AppWidgetHost.java:251)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at com.android.launcher3.uioverrides.QuickstepWidgetHolder.createHost(QuickstepWidgetHolder.java:104)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at com.android.launcher3.widget.LauncherWidgetHolder.<init>(LauncherWidgetHolder.java:94)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at com.android.launcher3.uioverrides.QuickstepWidgetHolder.<init>(QuickstepWidgetHolder.java:80)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at com.android.launcher3.uioverrides.QuickstepWidgetHolder.<init>(QuickstepWidgetHolder.java:0)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at com.android.launcher3.uioverrides.QuickstepWidgetHolder$QuickstepHolderFactory.newInstance(QuickstepWidgetHolder.java:361)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at com.android.launcher3.uioverrides.QuickstepLauncher.createAppWidgetHolder(QuickstepLauncher.java:671)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at com.android.launcher3.Launcher.onCreate(Launcher.java:531)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at com.android.launcher3.uioverrides.QuickstepLauncher.onCreate(QuickstepLauncher.java:678)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.Activity.performCreate(Activity.java:9235)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.Activity.performCreate(Activity.java:9194)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1542)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:4422)
12-18 07:24:30.611  5405  5405 E AndroidRuntime:         ... 13 more
12-18 07:24:30.611  5405  5405 D CompatChangeReporter: Compat change id reported: 312399441; UID 10125; state: ENABLED
12-18 07:24:30.611  5405  5405 W ScoutUtils: Failed to mkdir /data/miuilog/stability/memleak/heapdump/
12-18 07:24:30.612  5405  5405 E MQSEventManagerDelegate: failed to get MQSService.
12-18 07:24:30.613  3383  5825 I DropBoxManagerService: add tag=system_app_crash isTagEnabled=true flags=0x2
12-18 07:24:30.613  3383  3425 W ActivityTaskManager:   Not force finishing home activity com.android.launcher3/.uioverrides.QuickstepLauncher
12-18 07:24:30.615  2338  2338 W qcrilNrd: AIBinder_linkToDeath is being called with a non-null cookie and no onUnlink callback set. This might not be intended. AIBinder_DeathRecipient_setOnUnlinked should be called first.
12-18 07:24:30.618  2338  2338 W qcrilNrd: AIBinder_linkToDeath is being called with a non-null cookie and no onUnlink callback set. This might not be intended. AIBinder_DeathRecipient_setOnUnlinked should be called first.
12-18 07:24:30.618  3383  3518 D ActivityManagerTiming: FinishBooting
12-18 07:24:30.618  5405  5405 I Process : Process is going to kill itself!
12-18 07:24:30.618  5405  5405 I Process : java.lang.Exception
12-18 07:24:30.618  5405  5405 I Process :         at android.os.Process.killProcess(Process.java:1453)
12-18 07:24:30.618  5405  5405 I Process :         at com.android.internal.os.RuntimeInit$KillApplicationHandler.uncaughtException(RuntimeInit.java:193)
12-18 07:24:30.618  5405  5405 I Process :         at java.lang.ThreadGroup.uncaughtException(ThreadGroup.java:1071)
12-18 07:24:30.618  5405  5405 I Process :         at java.lang.ThreadGroup.uncaughtException(ThreadGroup.java:1066)
12-18 07:24:30.618  5405  5405 I Process :         at java.lang.Thread.dispatchUncaughtException(Thread.java:2424)
12-18 07:24:30.618  5405  5405 I Process : Sending signal. PID: 5405 SIG: 9
12-18 07:24:30.619  2338  2338 W qcrilNrd: AIBinder_linkToDeath is being called with a non-null cookie and no onUnlink callback set. This might not be intended. AIBinder_DeathRecipient_setOnUnlinked should be called first.
12-18 07:24:30.620  3383  3518 I WindowManager: Started waking up... (groupId=0 why=ON_BECAUSE_OF_UNKNOWN)
12-18 07:24:30.620  2338  2338 W qcrilNrd: AIBinder_linkToDeath is being called with a non-null cookie and no onUnlink callback set. This might not be intended. AIBinder_DeathRecipient_setOnUnlinked should be called first.
12-18 07:24:30.621  3383  3518 D MiuiPocketModeManager: set inertial and light sensor
12-18 07:24:30.621  3383  3518 I WindowManager: Finished setting display state ... (groupId=0)
12-18 07:24:30.621  3383  3518 I WindowManager: Finished waking up... (groupId=0 why=ON_BECAUSE_OF_UNKNOWN)
12-18 07:24:30.621  3383  3518 I WindowManager: Display 0 turning on...
```

这提示系统中 user 0 解锁失败，有可能是访问了上锁的数据。所以按照这个思路先查。

下面这条日志可能真的代表系统解锁有什么问题，记录下来以后参考用。`strongbox`​ 和安全有关，但可惜这次它不是原因。

```Bash
01-27 09:00:32.349  1151  1182 W keystore2: system/security/keystore2/src/shared_secret_negotiation.rs:248 - ParameterRetrieval { e: Binder(SERVICE_SPECIFIC, -1000), p: Aidl("strongbox") }
```

​`keymint`​ 也可能有问题，这个日志代表还没有完成启动

```Bash
12-18 07:24:19.233  1153  1153 E javacard.strongbox.keymint.operation-impl: Device boot not completed.
```

上面的日志最后发现没什么问题。

同事提供了另一个思路，看看因为日志看着没什么问题，所以尝试跳过 launcher3 看能不能顺利启动   

```Bash
adb wait-for-device root ; adb wait-for-device ; adb shell pm disable com.android.launcher3
```

这个时候 `dumpsys user`​ 显示的是 RUNNING\_UNLOCKED，说明后面系统解锁成功了，因此可以说明解锁没问题，但是和 launcher3 启动的顺序可能错了。等一会再启用 launcher3

```Bash
adb shell pm enable com.android.launcher3
```

这次居然就进到系统里了，而且功能也正常。再启动一个

```Bash
adb shell am start -a android.intent.action.MAIN -c android.intent.category.HOME
```

也没问题。因此自然而然开始怀疑起 launcher3 了。把它的 apk 文件导出来看一下

```Bash
adb shell pm list packages -f | grep com.android.launcher3 # 找到具体文件位置
adb pull /system_ext/priv-app/Launcher3QuickStep/Launcher3QuickStep.apk # 下载到本地
```

用 aapt 看一下它的 manifest

```Bash
.\aapt.exe dump xmltree C:\Users\Administrator\Launcher3QuickStep.apk AndroidManifest.xml
```

这个输出可能比较长，可以用 vim 或者 less 接一下输出，便于阅读。

application 标签里多了一个 `directBootAware`​，而它可以让被标记的对象在系统启动早期即启动。原版只有下面 service 里面有一个。

```Bash
    E: application (line=104)
      A: android:theme(0x01010000)=@0x7f12000f
      A: android:label(0x01010001)=@0x7f1100a1
      A: android:icon(0x01010002)=@0x7f08029f
      A: android:name(0x01010003)="com.android.launcher3.LauncherApplication" (Raw: "com.android.launcher3.LauncherApplication")
      A: android:backupAgent(0x0101027f)="com.android.launcher3.LauncherBackupAgent" (Raw: "com.android.launcher3.LauncherBackupAgent")
      A: android:restoreAnyVersion(0x010102ba)=(type 0x12)0xffffffff
      A: android:hardwareAccelerated(0x010102d3)=(type 0x12)0xffffffff
      A: android:largeHeap(0x0101035a)=@0x7f050003
      A: android:supportsRtl(0x010103af)=(type 0x12)0xffffffff
      A: android:fullBackupOnly(0x01010473)=(type 0x12)0xffffffff
      A: android:extractNativeLibs(0x010104ea)=(type 0x12)0xffffffff
      A: android:fullBackupContent(0x010104eb)=@0x7f140000
      A: android:defaultToDeviceProtectedStorage(0x01010504)=(type 0x12)0xffffffff
      A: android:directBootAware(0x01010505)=(type 0x12)0xffffffff
      A: android:backupInForeground(0x0101051a)=(type 0x12)0xffffffff
      A: android:appComponentFactory(0x0101057a)="androidx.core.app.CoreComponentFactory" (Raw: "androidx.core.app.CoreComponentFactory")
      A: android:usesNonSdkApi(0x0101058e)=(type 0x12)0xffffffff
      A: android:enableOnBackInvokedCallback(0x0101066c)=(type 0x12)0xffffffff
      E: service (line=123)
        A: android:name(0x01010003)="com.android.quickstep.TouchInteractionService" (Raw: "com.android.quickstep.TouchInteractionService")
        A: android:permission(0x01010006)="android.permission.STATUS_BAR_SERVICE" (Raw: "android.permission.STATUS_BAR_SERVICE")
        A: android:exported(0x01010010)=(type 0x12)0xffffffff
        A: android:directBootAware(0x01010505)=(type 0x12)0xffffffff
        E: intent-filter (line=128)
          E: action (line=129)
            A: android:name(0x01010003)="android.intent.action.QUICKSTEP_SERVICE" (Raw: "android.intent.action.QUICKSTEP_SERVICE")
      E: activity (line=133)
        A: android:theme(0x01010000)=@0x7f120158
        A: android:name(0x01010003)="com.android.quickstep.RecentsActivity" (Raw: "com.android.quickstep.RecentsActivity")
```

由于 application 里面多了一个标签，它会让 launcher3 的所有组件都在系统早期启动，很可能把启动时间提前到了解锁之前。这样，正常解锁流程之前访问文件就会出现非法异常，也就是前面的那个异常。

‍
