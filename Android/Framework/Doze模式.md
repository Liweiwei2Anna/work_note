参考：http://blog.csdn.net/gaugamela/article/details/52981984

在Android M中，Google就引入了Doze模式。它定义了一种全新的、低能耗的状态。
 在该状态，后台只有部分任务被允许运行，其它任务都被强制停止。

在之前的博客中分析过Doze模式，就是device idle状态。可能有的地方分析的不是很详细，现在在android7.0上重新分析下。

一、基本原理
Doze模式可以简单概括为： 
 若判断用户在连续的一段时间内没有使用手机，就延缓终端中APP后台的CPU和网络活动，以达到减少电量消耗的目的。





上面这张图比较经典，基本上说明了Doze模式的含义。 
 图中的横轴表示时间，红色部分表示终端处于唤醒的运行状态，绿色部分就是Doze模式定义的休眠状态。

从图中的描述，我们可以看到：如果一个用户停止充电(on battery: 利用电池供电)，关闭屏幕(screen off)，手机处于静止状态(stationary: 位置没有发生相对移动)，保持以上条件一段时间之后，终端就会进入Doze模式。一旦进入Doze模式，系统就减少(延缓)应用对网络的访问、以及对CPU的占用，来节省电池电量。

如图所示，Doze模式还定义了maintenance window。 
 在maintenance window中，系统允许应用完成它们被延缓的动作，即可以使用CPU资源及访问网络。 
 从图中我们可以看出，当进入Doze模式的条件一直满足时，Doze模式会定期的进入到maintenance window，但进入的间隔越来越长。 
 通过这种方式，Doze模式可以使终端处于较长时间的休眠状态。

需要注意的是：一旦Doze模式的条件不再满足，即用户充电、或打开屏幕、或终端的位置发生了移动，终端就恢复到正常模式。 
 因此，当用户频繁使用手机时，Doze模式几乎是没有什么实际用处的。

具体来讲，当终端处于Doze模式时，进行了以下操作： 
1、暂停网络访问。 
2、系统忽略所有的WakeLock。 
3、标准的AlarmManager alarms被延缓到下一个maintenance window。 
 但使用AlarmManager的 setAndAllowWhileIdle、setExactAndAllowWhileIdle和setAlarmClock时，alarms定义事件仍会启动。
 在这些alarms启动前，系统会短暂地退出Doze模式。 
4、系统不再进行WiFi扫描。 
5、系统不允许sync adapters运行。 
6、系统不允许JobScheduler运行。

另外我在另一篇博客中：http://blog.csdn.net/kc58236582/article/details/50554174也详细介绍了Doze模式，可以参考下，上面有一些命令使用等。



二、DeviceIdleController
Android中的Doze模式主要由DeviceIdleController来控制。

[cpp] view plain copy
public class DeviceIdleController extends SystemService  
        implements AnyMotionDetector.DeviceIdleCallback   
可以看出DeviceIdleController继承自SystemService，是一个系统级的服务。 
同时，继承了AnyMotionDetector定义的接口，便于检测到终端位置变化后进行回调。

2.1 DeviceIdleController的初始化
接下来我们看看它的初始化过程。

[cpp] view plain copy
private void startOtherServices() {  
    .........  
    mSystemServiceManager.startService(DeviceIdleController.class);  
    .........  
}  
如上代码所示，SystemServer在startOtherServices中启动了DeviceIdleController，将先后调用DeviceIdleController的构造函数和onStart函数。

构造函数
[cpp] view plain copy
public DeviceIdleController(Context context) {  
    super(context);  
    //deviceidle.xml用于定义idle模式也能正常工作的非系统应用  
    mConfigFile = new AtomicFile(new File(getSystemDir(), "deviceidle.xml"));  
    mHandler = new MyHandler(BackgroundThread.getHandler().getLooper());  
}  
DeviceIdleController的构造函数比较简单，就是在创建data/system/deviceidle.xml对应的file文件，同时创建一个对应于后台线程的handler。这里的deviceidle.xml可以在设置中的电池选项那里。有电池优化，可以将一些应用放到白名单中，调用DeviceIdleController的addPowerSaveWhitelistApp方法，最后会写入deviceidle.xml文件，然后在下次开机的时候DeviceIdleController会重新读取deviceidle.xml文件然后放入白名单mPowerSaveWhitelistUserApps中。

onStart函数
[cpp] view plain copy
public void onStart() {  
    final PackageManager pm = getContext().getPackageManager();  
  
    synchronized (this) {  
        //读取配置文件，判断Doze模式是否允许被开启  
        mLightEnabled = mDeepEnabled = getContext().getResources().getBoolean(  
                com.android.internal.R.bool.config_enableAutoPowerModes);  
  
        //分析PKMS时提到过，PKMS扫描系统目录的xml，将形成SystemConfig  
        SystemConfig sysConfig = SystemConfig.getInstance();  
  
        //获取除了device Idle模式外，都可以运行的系统应用白名单  
        ArraySet<String> allowPowerExceptIdle = sysConfig.getAllowInPowerSaveExceptIdle();  
        for (int i=0; i<allowPowerExceptIdle.size(); i++) {  
            String pkg = allowPowerExceptIdle.valueAt(i);  
            try {  
                ApplicationInfo ai = pm.getApplicationInfo(pkg,  
                        PackageManager.MATCH_SYSTEM_ONLY);  
                int appid = UserHandle.getAppId(ai.uid);  
                mPowerSaveWhitelistAppsExceptIdle.put(ai.packageName, appid);  
                mPowerSaveWhitelistSystemAppIdsExceptIdle.put(appid, true);  
            } catch (PackageManager.NameNotFoundException e) {  
            }  
        }  
  
        //获取device Idle模式下，也可以运行的系统应用白名单  
        ArraySet<String> allowPower = sysConfig.getAllowInPowerSave();  
        for (int i=0; i<allowPower.size(); i++) {  
             String pkg = allowPower.valueAt(i);  
            try {  
                ApplicationInfo ai = pm.getApplicationInfo(pkg,  
                         PackageManager.MATCH_SYSTEM_ONLY);  
                int appid = UserHandle.getAppId(ai.uid);  
                // These apps are on both the whitelist-except-idle as well  
                // as the full whitelist, so they apply in all cases.  
                mPowerSaveWhitelistAppsExceptIdle.put(ai.packageName, appid);  
                mPowerSaveWhitelistSystemAppIdsExceptIdle.put(appid, true);  
                mPowerSaveWhitelistApps.put(ai.packageName, appid);  
                mPowerSaveWhitelistSystemAppIds.put(appid, true);  
            } catch (PackageManager.NameNotFoundException e) {  
            }  
        }  
  
        //Constants为deviceIdleController中的内部类，继承ContentObserver  
        //监控数据库变化，同时得到Doze模式定义的一些时间间隔  
        mConstants = new Constants(mHandler, getContext().getContentResolver());  
  
        //解析deviceidle.xml，并将其中定义的package对应的app，加入到mPowerSaveWhitelistUserApps中  
        readConfigFileLocked();  
  
        //将白名单的内容给AlarmManagerService和PowerMangerService  
        //例如：DeviceIdleController判断开启Doze模式时，会通知PMS  
        //此时除去白名单对应的应用外，PMS会将其它所有的WakeLock设置为Disable状态  
        updateWhitelistAppIdsLocked();  
  
        //以下的初始化，都是假设目前处在进入Doze模式相反的条件上  
        mNetworkConnected = true;  
        mScreenOn = true;  
        // Start out assuming we are charging.  If we aren't, we will at least get  
        // a battery update the next time the level drops.  
        mCharging = true;  
  
        //Doze模式定义终端初始时为ACTIVE状态  
        mState = STATE_ACTIVE;  
        //屏幕状态初始时为ACTIVE状态  
        mLightState = LIGHT_STATE_ACTIVE;  
        mInactiveTimeout = mConstants.INACTIVE_TIMEOUT;  
    }  
  
    //发布服务  
    //BinderService和LocalService均为DeviceIdleController的内部类  
    mBinderService = new BinderService();  
    publishBinderService(Context.DEVICE_IDLE_CONTROLLER, mBinderService);  
    publishLocalService(LocalService.class, new LocalService());  
}  
除去发布服务外，DeviceIdleController在onStart函数中，主要是读取配置文件更新自己的变量，思路比较清晰。

在这里我们仅跟进一下updateWhitelistAppIdsLocked函数：

[cpp] view plain copy
private void updateWhitelistAppIdsLocked() {  
    //构造出除去idle模式外，可运行的app id数组 （可认为是系统和普通应用的集合）  
    //mPowerSaveWhitelistAppsExceptIdle从系统目录下的xml得到  
    //mPowerSaveWhitelistUserApps从deviceidle.xml得到，或调用接口加入；  
    //mPowerSaveWhitelistExceptIdleAppIds并未使用  
    mPowerSaveWhitelistExceptIdleAppIdArray = buildAppIdArray(mPowerSaveWhitelistAppsExceptIdle,  
            mPowerSaveWhitelistUserApps, mPowerSaveWhitelistExceptIdleAppIds);  
  
    //构造不受Doze限制的app id数组 （可认为是系统和普通应用的集合）  
    //mPowerSaveWhitelistApps从系统目录下的xml得到  
    //mPowerSaveWhitelistAllAppIds并未使用  
    mPowerSaveWhitelistAllAppIdArray = buildAppIdArray(mPowerSaveWhitelistApps,  
            mPowerSaveWhitelistUserApps, mPowerSaveWhitelistAllAppIds);  
  
    //构造不受Doze限制的app id数组（仅普通应用的集合）、  
    //mPowerSaveWhitelistUserAppIds并未使用  
    mPowerSaveWhitelistUserAppIdArray = buildAppIdArray(null,  
            mPowerSaveWhitelistUserApps, mPowerSaveWhitelistUserAppIds);  
  
    if (mLocalPowerManager != null) {  
        ...........  
        //PMS拿到的是：系统和普通应用组成的不受Doze限制的app id数组   
        mLocalPowerManager.setDeviceIdleWhitelist(mPowerSaveWhitelistAllAppIdArray);  
    }  
  
    if (mLocalAlarmManager != null) {  
        ..........  
        //AlarmManagerService拿到的是：普通应用组成的不受Doze限制的app id数组   
        mLocalAlarmManager.setDeviceIdleUserWhitelist(mPowerSaveWhitelistUserAppIdArray);  
    }  
}  
updateWhitelistAppIdsLocked主要是将白名单交给PMS和AlarmManagerService。 
注意Android区分了系统应用白名单、普通应用白名单等，因此上面进行了一些合并操作。这里我们有没有发现，systemConfig的app不会加入alarm的白名单，而在Settings中电池那边设置的白名单，会加入Power wakelock的白名单。

onBootPhase函数
与PowerManagerService一样，DeviceIdleController在初始化的最后一个阶段需要调用onBootPhase函数：

[cpp] view plain copy
public void onBootPhase(int phase) {  
    //在系统PHASE_SYSTEM_SERVICES_READY阶段，进一步完成一些初始化  
    if (phase == PHASE_SYSTEM_SERVICES_READY) {  
        synchronized (this) {  
            //初始化一些变量  
            mAlarmManager = (AlarmManager) getContext().getSystemService(Context.ALARM_SERVICE);  
            ..............  
  
            mSensorManager = (SensorManager) getContext().getSystemService(Context.SENSOR_SERVICE);  
            //根据配置文件，利用SensorManager获取对应的传感器，保存到mMotionSensor中  
            ..............  
  
            //如果配置文件表明：终端需要预获取位置信息  
            //则构造LocationRequest  
            if (getContext().getResources().getBoolean(  
                    com.android.internal.R.bool.config_autoPowerModePrefetchLocation)) {  
                mLocationManager = (LocationManager) getContext().getSystemService(  
                        Context.LOCATION_SERVICE);  
                mLocationRequest = new LocationRequest()  
                    .setQuality(LocationRequest.ACCURACY_FINE)  
                    .setInterval(0)  
                    .setFastestInterval(0)  
                    .setNumUpdates(1);  
            }  
  
            //根据配置文件，得到角度变化的门限  
            float angleThreshold = getContext().getResources().getInteger(  
                    com.android.internal.R.integer.config_autoPowerModeThresholdAngle) / 100f;  
            //创建一个AnyMotionDetector，同时将DeviceIdleController注册到其中  
            //当AnyMotionDetector检测到手机变化角度超过门限时，就会回调DeviceIdleController的接口  
            mAnyMotionDetector = new AnyMotionDetector(  
                    (PowerManager) getContext().getSystemService(Context.POWER_SERVICE),  
                    mHandler, mSensorManager, this, angleThreshold);  
  
            //创建两个常用的Intent，用于通知Doze模式的变化  
            mIdleIntent = new Intent(PowerManager.ACTION_DEVICE_IDLE_MODE_CHANGED);  
            mIdleIntent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY  
                    | Intent.FLAG_RECEIVER_FOREGROUND);  
            mLightIdleIntent = new Intent(PowerManager.ACTION_LIGHT_DEVICE_IDLE_MODE_CHANGED);  
            mLightIdleIntent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY  
                    | Intent.FLAG_RECEIVER_FOREGROUND);  
  
            //监听ACTION_BATTERY_CHANGED广播（电池信息发生改变）  
            IntentFilter filter = new IntentFilter();  
            filter.addAction(Intent.ACTION_BATTERY_CHANGED);  
            getContext().registerReceiver(mReceiver, filter);  
  
            //监听ACTION_PACKAGE_REMOVED广播（包被移除）  
            filter = new IntentFilter();  
            filter.addAction(Intent.ACTION_PACKAGE_REMOVED);  
            filter.addDataScheme("package");  
            getContext().registerReceiver(mReceiver, filter);  
  
            //监听CONNECTIVITY_ACTION广播（连接状态发生改变）  
            filter = new IntentFilter();  
            filter.addAction(ConnectivityManager.CONNECTIVITY_ACTION);  
            getContext().registerReceiver(mReceiver, filter);  
  
            //重新将白名单信息交给PowerManagerService和AlarmManagerService  
            //这个工作在onStart函数中，已经调用updateWhitelistAppIdsLocked进行过了  
            //到onBootPhase时，重新进行一次，可能：一是为了保险；二是，其它进程可能调用接口，更改了对应数据，于是进行更新  
            mLocalPowerManager.setDeviceIdleWhitelist(mPowerSaveWhitelistAllAppIdArray);  
            mLocalAlarmManager.setDeviceIdleUserWhitelist(mPowerSaveWhitelistUserAppIdArray);  
  
            //监听屏幕显示相关的变化  
            mDisplayManager.registerDisplayListener(mDisplayListener, null);  
  
            //更新屏幕显示相关的信息  
            updateDisplayLocked();  
        }  
        //更新连接状态相关的信息  
        updateConnectivityState(null);  
    }     
}  
从代码可以看出，onBootPhase方法： 
 主要创建一些本地变量，然后根据配置文件初始化一些传感器，同时注册了一些广播接收器和回到接口， 
 最后更新屏幕显示和连接状态相关的信息。

2.2 DeviceIdleController的状态变化
充电状态的处理
对于充电状态，在onBootPhase函数中已经提到，DeviceIdleController监听了ACTION_BATTERY_CHANGED广播：

[cpp] view plain copy
............  
IntentFilter filter = new IntentFilter();  
filter.addAction(Intent.ACTION_BATTERY_CHANGED);  
getContext().registerReceiver(mReceiver, filter);  
...........  
我们看看receiver中对应的处理：

[cpp] view plain copy
private final BroadcastReceiver mReceiver = new BroadcastReceiver() {  
    @Override public void onReceive(Context context, Intent intent) {  
        switch (intent.getAction()) {  
            .........  
            case Intent.ACTION_BATTERY_CHANGED: {  
                synchronized (DeviceIdleController.this) {  
                    //从广播中得到是否在充电的消息  
                    int plugged = intent.getIntExtra("plugged", 0);  
                    updateChargingLocked(plugged != 0);  
                }  
            } break;  
        }  
    }  
}；  
根据上面的代码，可以看出当收到电池信息改变的广播后，DeviceIdleController将得到电源是否在充电的消息，然后调用updateChargingLocked函数进行处理。
[cpp] view plain copy
void updateChargingLocked(boolean charging) {  
    .........  
    if (!charging && mCharging) {  
        //从充电状态变为不充电状态  
        mCharging = false;  
        //mForceIdle值一般为false，是通过dumpsys命令将mForceIdle改成true的  
        if (!mForceIdle) {  
            //判断是否进入Doze模式  
            becomeInactiveIfAppropriateLocked();  
        }  
    } else if (charging) {  
        //进入充电状态  
        mCharging = charging;  
        if (!mForceIdle) {  
            //手机退出Doze模式  
            becomeActiveLocked("charging", Process.myUid());  
        }  
    }  
}  
becomeInactiveIfAppropriateLocked函数是开始进入Doze模式，而becomeActiveLocked是退出Doze模式。

显示状态处理
DeviceIdleController中注册了显示变化的回调

[cpp] view plain copy
mDisplayManager.registerDisplayListener(mDisplayListener, null);  
回调会调用updateDisplayLocked函数
[cpp] view plain copy
private final DisplayManager.DisplayListener mDisplayListener  
        = new DisplayManager.DisplayListener() {  
    @Override public void onDisplayAdded(int displayId) {  
    }  
  
    @Override public void onDisplayRemoved(int displayId) {  
    }  
  
    @Override public void onDisplayChanged(int displayId) {  
        if (displayId == Display.DEFAULT_DISPLAY) {  
            synchronized (DeviceIdleController.this) {  
                updateDisplayLocked();  
            }  
        }  
    }  
};  
updateDisplayLocked函数和更新充电状态的函数updateChargingLocked类似

[cpp] view plain copy
void updateDisplayLocked() {  
    mCurDisplay = mDisplayManager.getDisplay(Display.DEFAULT_DISPLAY);  
    // We consider any situation where the display is showing something to be it on,  
    // because if there is anything shown we are going to be updating it at some  
    // frequency so can't be allowed to go into deep sleeps.  
    boolean screenOn = mCurDisplay.getState() == Display.STATE_ON;  
    if (DEBUG) Slog.d(TAG, "updateDisplayLocked: screenOn=" + screenOn);  
    if (!screenOn && mScreenOn) {  
        mScreenOn = false;  
        if (!mForceIdle) {//开始进入Doze模式  
            becomeInactiveIfAppropriateLocked();  
        }  
    } else if (screenOn) {//屏幕点亮，退出Doze模式  
        mScreenOn = true;  
        if (!mForceIdle) {  
            becomeActiveLocked("screen", Process.myUid());  
        }  
    }  
}  
becomeActiveLocked函数退出Doze模式
我们先来看看becomeActiveLocked函数

[cpp] view plain copy
//activeReason记录的终端变为active的原因  
void becomeActiveLocked(String activeReason, int activeUid) {  
    ...........  
    if (mState != STATE_ACTIVE || mLightState != STATE_ACTIVE) {  
        ............  
        //1、通知PMS等Doze模式结束  
        scheduleReportActiveLocked(activeReason, activeUid);  
  
        //更新DeviceIdleController本地维护的状态  
        //在DeviceIdleController的onStart函数中，我们已经知道了  
        //初始时，mState和mLightState均为Active状态  
        mState = STATE_ACTIVE;//state是指设备通过传感器判断进入idle  
        mLightState = LIGHT_STATE_ACTIVE;//mLight是背光的状态  
  
        mInactiveTimeout = mConstants.INACTIVE_TIMEOUT;  
        mCurIdleBudget = 0;  
        mMaintenanceStartTime = 0;  
  
        //2、重置一些事件  
        resetIdleManagementLocked();  
        resetLightIdleManagementLocked();  
  
        addEvent(EVENT_NORMAL);  
    }  
}  
scheduleReportActiveLocked函数就是发送MSG_REPORT_ACTIVE消息

[cpp] view plain copy
void scheduleReportActiveLocked(String activeReason, int activeUid) {  
    //发送MSG_REPORT_ACTIVE消息  
    Message msg = mHandler.obtainMessage(MSG_REPORT_ACTIVE, activeUid, 0, activeReason);  
    mHandler.sendMessage(msg);  
}  
我们再看下消息的处理，主要调用了PowerManagerService的setDeviceIdleMode函数来退出Doze状态，然后重新更新wakelock的enable状态， 以及通知NetworkPolicyManagerService不再限制应用上网，还有发送Doze模式改变的广播。

[cpp] view plain copy
.........  
case MSG_REPORT_ACTIVE: {  
    .........  
    //通知PMS Doze模式结束，  
    //于是PMS将一些Doze模式下，disable的WakeLock重新enable  
    //然后调用updatePowerStateLocked函数更新终端的状态  
    final boolean deepChanged = mLocalPowerManager.setDeviceIdleMode(false);  
    final boolean lightChanged = mLocalPowerManager.setLightDeviceIdleMode(false);  
  
    try {  
        //通过NetworkPolicyManagerService更改Ip-Rule，不再限制终端应用上网  
        mNetworkPolicyManager.setDeviceIdleMode(false);  
        //BSS做好对应的记录  
        mBatteryStats.noteDeviceIdleMode(BatteryStats.DEVICE_IDLE_MODE_OFF,  
                activeReason, activeUid);  
    } catch (RemoteException e) {  
    }  
  
    //发送广播  
    if (deepChanged) {  
        getContext().sendBroadcastAsUser(mIdleIntent, UserHandle.ALL);  
    }  
    if (lightChanged) {  
        getContext().sendBroadcastAsUser(mLightIdleIntent, UserHandle.ALL);  
    }  
}  
........  
resetIdleManagementLocked函数就是取消alarm，检测等。

[cpp] view plain copy
void resetIdleManagementLocked() {  
    //复位一些状态变量  
    mNextIdlePendingDelay = 0;  
    mNextIdleDelay = 0;  
    mNextLightIdleDelay = 0;  
  
    //停止一些工作，主要是位置检测相关的  
    cancelAlarmLocked();  
    cancelSensingTimeoutAlarmLocked();  
    cancelLocatingLocked();  
    stopMonitoringMotionLocked();  
    mAnyMotionDetector.stop();  
}  

becomeInactiveIfAppropriateLocked函数开始进入Doze模式
becomeInactiveIfAppropriateLocked函数就是我们开始进入Doze模式的第一个步骤，下面我们就详细分析这个函数

[cpp] view plain copy
void becomeInactiveIfAppropriateLocked() {  
    .................  
    //屏幕熄灭，未充电  
    if ((!mScreenOn && !mCharging) || mForceIdle) {  
        // Screen has turned off; we are now going to become inactive and start  
        // waiting to see if we will ultimately go idle.  
        if (mState == STATE_ACTIVE && mDeepEnabled) {  
            mState = STATE_INACTIVE;  
            ...............  
            //重置事件  
            resetIdleManagementLocked();  
  
            //开始检测是否可以进入Doze模式的Idle状态  
            //若终端没有watch feature, mInactiveTimeout时间为30min  
            scheduleAlarmLocked(mInactiveTimeout, false);  
            ...............  
        }  
  
        if (mLightState == LIGHT_STATE_ACTIVE && mLightEnabled) {  
            mLightState = LIGHT_STATE_INACTIVE;  
            .............  
            resetLightIdleManagementLocked();//重置事件  
            scheduleLightAlarmLocked(mConstants.LIGHT_IDLE_AFTER_INACTIVE_TIMEOUT);  
        }  
    }  
}  
要进入Doze流程，就是调用这个函数，首先要保证屏幕灭屏然后没有充电。这里还有mDeepEnable和mLightEnable前面说过是在配置中定义的，一般默认是关闭（也就是不开Doze模式）。这里mLightEnabled是对应禁止wakelock持锁的，禁止网络。而mDeepEnabled对应是检测设备是否静止，除了禁止wakelock、禁止网络、还会机制alarm。



light idle 和deep idle根据不同条件进入

[java] view plain copy
void becomeInactiveIfAppropriateLocked() {  
        if (DEBUG) Slog.d(TAG, "becomeInactiveIfAppropriateLocked()");  
        if ((!mScreenOn && !mCharging) || mForceIdle) {  
            // Screen has turned off; we are now going to become inactive and start  
            // waiting to see if we will ultimately go idle.  
            if (mState == STATE_ACTIVE && mDeepEnabled) {  
                mState = STATE_INACTIVE;  
                if (DEBUG) Slog.d(TAG, "Moved from STATE_ACTIVE to STATE_INACTIVE");  
                resetIdleManagementLocked();  
                scheduleAlarmLocked(mInactiveTimeout, false);  
                EventLogTags.writeDeviceIdle(mState, "no activity");  
            }  
            if (mLightState == LIGHT_STATE_ACTIVE && mLightEnabled) {  
                mLightState = LIGHT_STATE_INACTIVE;  
                if (DEBUG) Slog.d(TAG, "Moved from LIGHT_STATE_ACTIVE to LIGHT_STATE_INACTIVE");  
                resetLightIdleManagementLocked();  
                scheduleLightAlarmLocked(mConstants.LIGHT_IDLE_AFTER_INACTIVE_TIMEOUT);  
                EventLogTags.writeDeviceIdleLight(mLightState, "no activity");  
            }  
        }  
    }  

light idle模式
我们先看light idle模式，这个模式下、会禁止网络、wakelock，但是不会禁止alarm。

我们先看scheduleLightAlarmLocked函数，这里设置了一个alarm，delay是5分钟。到时间后调用mLightAlarmListener回调。

[cpp] view plain copy
void scheduleLightAlarmLocked(long delay) {  
    if (DEBUG) Slog.d(TAG, "scheduleLightAlarmLocked(" + delay + ")");  
    mNextLightAlarmTime = SystemClock.elapsedRealtime() + delay;  
    mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,  
            mNextLightAlarmTime, "DeviceIdleController.light", mLightAlarmListener, mHandler);  
}  
mLightAlarmListener就是进入lightidle，调用stepLightIdleStateLocked函数

[cpp] view plain copy
private final AlarmManager.OnAlarmListener mLightAlarmListener  
        = new AlarmManager.OnAlarmListener() {  
    @Override  
    public void onAlarm() {  
        synchronized (DeviceIdleController.this) {  
            stepLightIdleStateLocked("s:alarm");  
        }  
    }  
};  
我们来看stepLightIdleStateLocked函数，这个函数会处理mLightState不同状态，会根据不同状态，然后设置alarm，到时间后继续处理下个状态。到LIGHT_STATE_IDLE_MAINTENANCE状态处理时，会发送MSG_REPORT_IDLE_ON_LIGHT。这个消息的处理会禁止网络、禁止wakelock。然后到LIGHT_STATE_WAITING_FOR_NETWORK，会先退出Doze状态（这个时候网络、wakelock恢复）。然后设置alarm，alarm时间到后，还是在LIGHT_STATE_IDLE_MAINTENANCE状态。和之前一样（禁止网络、wakelock）。只是设置的alarm间隔会越来越大，也就是只要屏幕灭屏后，时间越长。设备会隔越来越长的时间才会退出Doze状态，这也符合一个实际情况，但是会有一个上限值。

[cpp] view plain copy
void stepLightIdleStateLocked(String reason) {  
    if (mLightState == LIGHT_STATE_OVERRIDE) {  
        // If we are already in deep device idle mode, then  
        // there is nothing left to do for light mode.  
        return;  
    }  
  
    EventLogTags.writeDeviceIdleLightStep();  
  
    switch (mLightState) {  
        case LIGHT_STATE_INACTIVE:  
            mCurIdleBudget = mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET;  
            // Reset the upcoming idle delays.  
            mNextLightIdleDelay = mConstants.LIGHT_IDLE_TIMEOUT;  
            mMaintenanceStartTime = 0;  
            if (!isOpsInactiveLocked()) {  
                // We have some active ops going on...  give them a chance to finish  
                // before going in to our first idle.  
                mLightState = LIGHT_STATE_PRE_IDLE;  
                EventLogTags.writeDeviceIdleLight(mLightState, reason);  
                scheduleLightAlarmLocked(mConstants.LIGHT_PRE_IDLE_TIMEOUT);//设置alarm，时间到后到下个步骤  
                break;  
            }  
            // Nothing active, fall through to immediately idle.  
        case LIGHT_STATE_PRE_IDLE:  
        case LIGHT_STATE_IDLE_MAINTENANCE:  
            if (mMaintenanceStartTime != 0) {  
                long duration = SystemClock.elapsedRealtime() - mMaintenanceStartTime;  
                if (duration < mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET) {  
                    // We didn't use up all of our minimum budget; add this to the reserve.  
                    mCurIdleBudget += (mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET-duration);  
                } else {  
                    // We used more than our minimum budget; this comes out of the reserve.  
                    mCurIdleBudget -= (duration-mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET);  
                }  
            }  
            mMaintenanceStartTime = 0;  
            scheduleLightAlarmLocked(mNextLightIdleDelay);  
            mNextLightIdleDelay = Math.min(mConstants.LIGHT_MAX_IDLE_TIMEOUT,  
                    (long)(mNextLightIdleDelay * mConstants.LIGHT_IDLE_FACTOR));  
            if (mNextLightIdleDelay < mConstants.LIGHT_IDLE_TIMEOUT) {  
                mNextLightIdleDelay = mConstants.LIGHT_IDLE_TIMEOUT;  
            }  
            if (DEBUG) Slog.d(TAG, "Moved to LIGHT_STATE_IDLE.");  
            mLightState = LIGHT_STATE_IDLE;  
            EventLogTags.writeDeviceIdleLight(mLightState, reason);  
            addEvent(EVENT_LIGHT_IDLE);  
            mHandler.sendEmptyMessage(MSG_REPORT_IDLE_ON_LIGHT);//发送消息，这个消息处理就会关闭网络，禁止wakelock  
            break;  
        case LIGHT_STATE_IDLE:  
        case LIGHT_STATE_WAITING_FOR_NETWORK:  
            if (mNetworkConnected || mLightState == LIGHT_STATE_WAITING_FOR_NETWORK) {  
                // We have been idling long enough, now it is time to do some work.  
                mActiveIdleOpCount = 1;  
                mActiveIdleWakeLock.acquire();  
                mMaintenanceStartTime = SystemClock.elapsedRealtime();  
                if (mCurIdleBudget < mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET) {  
                    mCurIdleBudget = mConstants.LIGHT_IDLE_MAINTENANCE_MIN_BUDGET;  
                } else if (mCurIdleBudget > mConstants.LIGHT_IDLE_MAINTENANCE_MAX_BUDGET) {  
                    mCurIdleBudget = mConstants.LIGHT_IDLE_MAINTENANCE_MAX_BUDGET;  
                }  
                scheduleLightAlarmLocked(mCurIdleBudget);  
                if (DEBUG) Slog.d(TAG,  
                        "Moved from LIGHT_STATE_IDLE to LIGHT_STATE_IDLE_MAINTENANCE.");  
                mLightState = LIGHT_STATE_IDLE_MAINTENANCE;  
                EventLogTags.writeDeviceIdleLight(mLightState, reason);  
                addEvent(EVENT_LIGHT_MAINTENANCE);  
                mHandler.sendEmptyMessage(MSG_REPORT_IDLE_OFF);//醒一下（开启网络、恢复wakelock）  
            } else {  
                // We'd like to do maintenance, but currently don't have network  
                // connectivity...  let's try to wait until the network comes back.  
                // We'll only wait for another full idle period, however, and then give up.  
                scheduleLightAlarmLocked(mNextLightIdleDelay);  
                if (DEBUG) Slog.d(TAG, "Moved to LIGHT_WAITING_FOR_NETWORK.");  
                mLightState = LIGHT_STATE_WAITING_FOR_NETWORK;  
                EventLogTags.writeDeviceIdleLight(mLightState, reason);  
            }  
            break;  
    }  
}  
但是这里只是一个light idle，一旦进入deep idle，light idle设置的alarm会无效的（这个后面细说），也就是说light idle一旦进入deep idle后无效了（因为idle step主要靠alarm驱动，而alarm无效后自然就驱动不了）。


deep idle模式
下面我们再来看deep idle模式，这个模式除了禁止网络、wakelock还会禁止alarm。

我们再来看becomeInactiveIfAppropriateLocked函数中下面代码。是关于deep idle的设置 这里的mInactiveTimeout是半小时

[cpp] view plain copy
void becomeInactiveIfAppropriateLocked() {  
    if (DEBUG) Slog.d(TAG, "becomeInactiveIfAppropriateLocked()");  
    if ((!mScreenOn && !mCharging) || mForceIdle) {  
        // Screen has turned off; we are now going to become inactive and start  
        // waiting to see if we will ultimately go idle.  
        if (mState == STATE_ACTIVE && mDeepEnabled) {  
            mState = STATE_INACTIVE;  
            if (DEBUG) Slog.d(TAG, "Moved from STATE_ACTIVE to STATE_INACTIVE");  
            resetIdleManagementLocked();  
            scheduleAlarmLocked(mInactiveTimeout, false);  
            EventLogTags.writeDeviceIdle(mState, "no activity");  
        }  
我们来看下scheduleAlarmLocked函数，注意如果这里参数idleUntil是true会调用AlarmManager的setIdleUntil函数，调用这个函数后普通应用设置alarm将失效。

[cpp] view plain copy
void scheduleAlarmLocked(long delay, boolean idleUntil) {  
    if (mMotionSensor == null) {  
        //在onBootPhase时，获取过位置检测传感器  
        //如果终端没有配置位置检测传感器，那么终端永远不会进入到真正的Doze ilde状态  
        // If there is no motion sensor on this device, then we won't schedule  
        // alarms, because we can't determine if the device is not moving.  
        return;  
    }  
  
    mNextAlarmTime = SystemClock.elapsedRealtime() + delay;  
    if (idleUntil) {  
        //此时IdleUtil的值为false  
        mAlarmManager.setIdleUntil(AlarmManager.ELAPSED_REALTIME_WAKEUP,  
                mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);  
    } else {  
        //30min后唤醒，调用mDeepAlarmListener的onAlarm函数  
        mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,  
                mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);  
    }  
}  
需要注意的是，DeviceIdleController一直在监控屏幕状态和充电状态，一但不满足Doze模式的条件，前面提到的becomeActiveLocked函数就会被调用。mAlarmManager设置的定时唤醒事件将被取消掉，mDeepAlarmListener的onAlarm函数不会被调用。

因此，我们知道了终端必须保持Doze模式的入口条件长达30min，才会进入mDeepAlarmListener.onAlarm：

[cpp] view plain copy
private final AlarmManager.OnAlarmListener mDeepAlarmListener  
        = new AlarmManager.OnAlarmListener() {  
    @Override  
    public void onAlarm() {  
        synchronized (DeviceIdleController.this) {  
            //进入到stepIdleStateLocked函数  
            stepIdleStateLocked("s:alarm");  
        }  
    }  
};  
下面我们就来看下stepIdleStateLocked函数：

[cpp] view plain copy
void stepIdleStateLocked(String reason) {  
    ..........  
    final long now = SystemClock.elapsedRealtime();  
    //个人觉得，下面这段代码，是针对Idle状态设计的  
    //如果在Idle状态收到Alarm，那么将先唤醒终端，然后重新判断是否需要进入Idle态  
    //在介绍Doze模式原理时提到过，若应用调用AlarmManager的一些指定接口，仍然可以在Idle状态进行工作  
    if ((now+mConstants.MIN_TIME_TO_ALARM) > mAlarmManager.getNextWakeFromIdleTime()) {  
        // Whoops, there is an upcoming alarm.  We don't actually want to go idle.  
        if (mState != STATE_ACTIVE) {  
            becomeActiveLocked("alarm", Process.myUid());  
            becomeInactiveIfAppropriateLocked();  
        }  
        return;  
    }  
  
    //以下是Doze模式的状态转变相关的代码  
    switch (mState) {  
        case STATE_INACTIVE:  
            // We have now been inactive long enough, it is time to start looking  
            // for motion and sleep some more while doing so.  
            //保持屏幕熄灭，同时未充电达到30min，进入此分支  
  
            //注册一个mMotionListener，检测是否移动  
            //如果检测到移动，将重新进入到ACTIVE状态  
            //相应代码比较直观，此处不再深入分析  
            startMonitoringMotionLocked();  
  
            //再次调用scheduleAlarmLocked函数，此次的时间仍为30min  
            //也就说如果不发生退出Doze模式的事件，30min后将再次进入到stepIdleStateLocked函数  
            //不过届时的mState已经变为STATE_IDLE_PENDING  
            scheduleAlarmLocked(mConstants.IDLE_AFTER_INACTIVE_TIMEOUT, false);  
  
            // Reset the upcoming idle delays.  
            //mNextIdlePendingDelay为5min  
            mNextIdlePendingDelay = mConstants.IDLE_PENDING_TIMEOUT;  
            //mNextIdleDelay为60min  
            mNextIdleDelay = mConstants.IDLE_TIMEOUT;  
  
            //状态变为STATE_IDLE_PENDING   
            mState = STATE_IDLE_PENDING;  
            ............  
            break;  
        case STATE_IDLE_PENDING:  
            //保持息屏、未充电、静止状态，经过30min后，进入此分支  
            mState = STATE_SENSING;  
  
            //保持Doze模式条件，4min后再次进入stepIdleStateLocked  
            scheduleSensingTimeoutAlarmLocked(mConstants.SENSING_TIMEOUT);  
  
            //停止定位相关的工作  
            cancelLocatingLocked();  
            mNotMoving = false;  
            mLocated = false;  
            mLastGenericLocation = null;  
            mLastGpsLocation = null;  
  
            //开始检测手机是否发生运动（这里应该是更细致的侧重于角度的变化）  
            //若手机运动过，则重新变为active状态  
            mAnyMotionDetector.checkForAnyMotion();  
            break;  
        case STATE_SENSING:  
            //上面的条件满足后，进入此分支，开始获取定位信息  
            cancelSensingTimeoutAlarmLocked();  
            mState = STATE_LOCATING;  
            ............  
            //保持条件30s，再次调用stepIdleStateLocked  
            scheduleAlarmLocked(mConstants.LOCATING_TIMEOUT, false);  
  
            //网络定位  
            if (mLocationManager != null  
                    && mLocationManager.getProvider(LocationManager.NETWORK_PROVIDER) != null) {  
                mLocationManager.requestLocationUpdates(mLocationRequest,  
                        mGenericLocationListener, mHandler.getLooper());  
                mLocating = true;  
            } else {  
                mHasNetworkLocation = false;  
            }  
  
            //GPS定位  
            if (mLocationManager != null  
                    && mLocationManager.getProvider(LocationManager.GPS_PROVIDER) != null) {  
                mHasGps = true;  
                mLocationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 1000, 5,  
                        mGpsLocationListener, mHandler.getLooper());  
                mLocating = true;  
            } else {  
                mHasGps = false;  
            }  
  
            // If we have a location provider, we're all set, the listeners will move state  
            // forward.  
            if (mLocating) {  
                //无法定位则直接进入下一个case  
                break;  
            }  
        case STATE_LOCATING:  
            //停止定位和运动检测，直接进入到STATE_IDLE_MAINTENANCE  
            cancelAlarmLocked();  
            cancelLocatingLocked();  
            mAnyMotionDetector.stop();  
  
        case STATE_IDLE_MAINTENANCE:  
            //进入到这个case后，终端开始进入Idle状态，也就是真正的Doze模式  
  
            //定义退出Idle的时间此时为60min  
            scheduleAlarmLocked(mNextIdleDelay, true);  
            ............  
            //退出周期逐步递增，每次乘2  
            mNextIdleDelay = (long)(mNextIdleDelay * mConstants.IDLE_FACTOR);  
            ...........  
            //周期有最大值6h  
            mNextIdleDelay = Math.min(mNextIdleDelay, mConstants.MAX_IDLE_TIMEOUT);  
            if (mNextIdleDelay < mConstants.IDLE_TIMEOUT) {  
                mNextIdleDelay = mConstants.IDLE_TIMEOUT;  
            }  
  
            mState = STATE_IDLE;  
            ...........  
            //通知PMS、NetworkPolicyManagerService等Doze模式开启，即进入Idle状态  
            //此时PMS disable一些非白名单WakeLock；NetworkPolicyManagerService开始限制一些应用的网络访问  
            //消息处理的具体流程比较直观，此处不再深入分析  
            mHandler.sendEmptyMessage(MSG_REPORT_IDLE_ON);  
            break;  
  
        case STATE_IDLE:  
            //进入到这个case时，本次的Idle状态暂时结束，开启maintenance window  
  
            // We have been idling long enough, now it is time to do some work.  
            mActiveIdleOpCount = 1;  
            mActiveIdleWakeLock.acquire();  
  
            //定义重新进入Idle的时间为5min （也就是手机可处于Maintenance window的时间）  
            scheduleAlarmLocked(mNextIdlePendingDelay, false);  
  
            mMaintenanceStartTime = SystemClock.elapsedRealtime();  
            //调整mNextIdlePendingDelay，乘2（最大为10min）  
            mNextIdlePendingDelay = Math.min(mConstants.MAX_IDLE_PENDING_TIMEOUT,  
                    (long)(mNextIdlePendingDelay * mConstants.IDLE_PENDING_FACTOR));  
  
            if (mNextIdlePendingDelay < mConstants.IDLE_PENDING_TIMEOUT) {  
                    mNextIdlePendingDelay = mConstants.IDLE_PENDING_TIMEOUT;  
            }  
  
            mState = STATE_IDLE_MAINTENANCE;  
            ...........  
            //通知PMS等暂时退出了Idle状态，可以进行一些工作  
            //此时PMS enable一些非白名单WakeLock；NetworkPolicyManagerService开始允许应用的网络访问  
            mHandler.sendEmptyMessage(MSG_REPORT_IDLE_OFF);  
            break;  
    }  
}  
上面的流程在注释里面已经很明白了，而我们在进入Deep idle时，发送了一个MSG_REPORT_IDLE_ON消息，我们看下面这个消息的处理和之前的MSG_REPORT_IDLE_ON_LIGHT一样的，关闭网络，禁止wakelock。

[cpp] view plain copy
case MSG_REPORT_IDLE_ON:  
case MSG_REPORT_IDLE_ON_LIGHT: {  
    EventLogTags.writeDeviceIdleOnStart();  
    final boolean deepChanged;  
    final boolean lightChanged;  
    if (msg.what == MSG_REPORT_IDLE_ON) {  
        deepChanged = mLocalPowerManager.setDeviceIdleMode(true);  
        lightChanged = mLocalPowerManager.setLightDeviceIdleMode(false);  
    } else {  
        deepChanged = mLocalPowerManager.setDeviceIdleMode(false);  
        lightChanged = mLocalPowerManager.setLightDeviceIdleMode(true);  
    }  
    try {  
        mNetworkPolicyManager.setDeviceIdleMode(true);  
        mBatteryStats.noteDeviceIdleMode(msg.what == MSG_REPORT_IDLE_ON  
                ? BatteryStats.DEVICE_IDLE_MODE_DEEP  
                : BatteryStats.DEVICE_IDLE_MODE_LIGHT, null, Process.myUid());  
    } catch (RemoteException e) {  
    }  
    if (deepChanged) {  
        getContext().sendBroadcastAsUser(mIdleIntent, UserHandle.ALL);  
    }  
    if (lightChanged) {  
        getContext().sendBroadcastAsUser(mLightIdleIntent, UserHandle.ALL);  
    }  
    EventLogTags.writeDeviceIdleOnComplete();  
} break;  
而禁止alarm是通过调用如下函数，注意参数是true。参数是true会调用mAlarmManager.setIdleUntil函数。这样其他的alarm会被滞后（除非在白名单中）

[cpp] view plain copy
scheduleAlarmLocked(mNextIdleDelay, true);  
而每隔一段时间会进入Maintenance window的时间，此时是通过发送MSG_REPORT_IDLE_OFF消息，来恢复网络和wakelock。而这个时候之前设置的mAlarmManager.setIdleUntil的alarm也到期了，因此其他alarm也恢复了。但是这个时间只有5分钟，重新设置了alarm再次进入deep idle状态。


Idle总结
当手机关闭屏幕或者拔掉电源的时候，手机开始判断是否进入Doze模式。

Doze模式分两种，第一种是light idle：

1.light idle

light idle在手机灭屏且没有充电状态下，5分钟开始进入light idle流程。然后第一次进入LIGHT_STATE_INACTIVE流程时，会再定义一个10分钟的alarm。然后系统进入light idle状态。这个状态会使不是白名单的应用禁止访问网络，以及持wakelock锁。

2.deep idle

deep idle除了light idle的状态还会把非白名单中应用的alarm也禁止了。
 此时，系统中非白名单的应用将被禁止访问网络，它们申请的Wakelock也会被disable。 
 从上面的代码可以看出，系统会周期性的退出Idle状态，进入到MAINTENANCE状态，集中处理相关的任务。 
 一段时间后，会重新再次回到IDLE状态。每次进入IDLE状态，停留的时间都会是上次的2倍，最大时间限制为6h。

当手机运动，或者点亮屏幕，插上电源等，系统都会重新返回到ACTIVIE状态。

