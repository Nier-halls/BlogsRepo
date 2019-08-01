## StartActivity

#### 调用ActivityManagerService.startActivity
参数:
```
ActivityManagerService.startActivity

*IApplicationThread     caller:             ActivityThread.mAppThread //暂时不明确是什么
 String                 callingPackage:     sourceActivity.getBasePackageName() //应用包名
 Intent                 intent:             Intent(sourceActivity, targetActivity.class) //调用startActivity传入的intent，
 String                 resolveType:        Intent.resolveTypeIfNeeded(activity.getContentResolver()) //暂时不明白有什么用
*IBinder                resultTo:           sourceActivity.mToken //暂时不明确是什么
*String                 resultWho:          sourceActivity.mEmbeddedID //暂时不明确是什么 唯一id？
 int                    requestCode:        -1 //是原样返回的吗
 int                    startFlags:         0
 ProfilerInfo           profilerInfo:       null //暂时不明白是什么
 Bundle                 bOptions:           null //需要传递的参数

("*"标记参数后续从源码明确作用)
```
逻辑:Instrumentation.execStartActivity调用ActivityManagerService.startActivity，并且传递相应的参数
作用:


#### ActivityStarter.startActivityMayWait
参数:
```
ActivityStarter.startActivityMayWait

*IApplicationThread         caller                  ActivityThread.mAppThread //暂时不明确是什么
 String                     callingUid              -1
 String                     callingPackage          sourceActivity.getBasePackageName() //应用包名
 Intent                     intent                  Intent(sourceActivity, targetActivity.class) //调用startActivity传入的intent，
 String                     resolveType             Intent.resolveTypeIfNeeded(activity.getContentResolver()) //暂时不明白有什么用
 IVoiceInteractionSession   voiceSession            null
 IVoiceInteractor           voiceInteractor         null
*IBinder                    resultTo:               sourceActivity.mToken //暂时不明确是什么
*String                     resultWho:              sourceActivity.mEmbeddedID //暂时不明确是什么 唯一id？
 int                        requestCode:            -1 //是原样返回的吗
 int                        startFlags:             0
 ProfilerInfo               profilerInfo:           null //暂时不明白是什么
 WaitResult                 outResult               null
 Configuration              globalConfig            null
 Bundle                     bOptions:               null //需要传递的参数
 boolean                    ignoreTargetSecurity    false
 int                        userId                  UserHandle.getCallingUserId() //暂时不确定是什么，可能是进程uid
 IActivityContainer         iContainer              null
 TaskRecord                 inTask                  null
 String                     reason                  "startActivityAsUser"
 
("*"标记参数后续从源码明确作用)
```
* 解析了intent中对应的待启动Component(Activity)组件的配置信息，生成ResolveInfo，ActivityInfo;
* ResolveInfo针对Manifest文件中对应Component(Activity)中的intentFilter标签的信息解析，包含了IntentFilter、resolvePackageName、activityInfo等信息
* ActivityInfo对Manifest文件中Activity标签的信息解析，包含了launchMode、permission、taskAffinity、targetActivity等信息
* 用ServiceManager**加锁**继续调用startActivityLocked()方法转而调用ActivityStarter.startActivity()方法


#### ActivityStarter.startActivity
方法逻辑:

1. 检查调用startActivity的pid是否合法
2. 检查intent中解析的数据是否合法，Activity是否存在
3. 找到sourceActivity(调用startActivity的Activity)在AMS中对应的ActivityRecord
4. 检查调用者的权限是否合法
5. 创建一个ActivityRecord对象来描述当前需要启动的Activity

ActivityRecord构造参数:
```
ActivityRecord

ActivityManagerService      _service                    AMS对应的实例
ProcessRecord               _caller                     根据Activity.startActivity传入的IApplicationThread映射找到的
int                         _launchedFromPid            调用方sourceActivity所在的进程id
int                         _launchedFromUid            调用方sourceActivity所在进程的uid
Intent                      _intent                     intent
String                      _resolveType                Intent.resolveTypeIfNeeded(activity.getContentResolver()) //暂时不明白有什么用
ActivityInfo                aInfo                       解析Manifest获得的信息描述对象
Configuration               _configuration              AMS.getGlobalConfiguration()
ActivityRecord              _resultTo                   null //有什么意义吗？和resultWho有什么区别
String                      _resultWho                  sourceActivity.mEmbeddedID //暂时不明确是什么 唯一id？
int                         _reqCode                    requestCode -1
boolean                     _componentSpecified         intent.getComponent() != null, true
boolean                     _rootVoiceInteraction       false
ActivityStackSupervisor     supervisor                  调用基础方法的执行对象
ActivityContainer           container                   null
ActivityOption              options                     ActivityOptions.fromBundle(bOptions)
ActivityRecord              sourceRecord                发起startActivity的Activity对应的ActivityRecord
```


#### ActivityStarter.startActivityUnchecked

##### computeLaunchingTaskFlags
适配LaunchFlags
* 在sourceActivity启动模式是SingleInstance的情况下 mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK，这里**确保SingleInstance中永远都只有一个Activity**
* 待启动Activity启动模式是 SingleInstance或SingleTask 的情况下 mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK
```
注意点：
mInTask = null
mStartActivity.isResolverActivity() || mStartActivity.noDisplay >>> false
```

##### computeSourceStack
确定待启动Activity的stack，sourceActivity不为null且没有finish，那么待启动Activity就和sourceActivity同stack

##### getReusableIntentActivity
作用:暂时还不明白，是为了确定startActivity的task吗？

逻辑:在Activity的launchMode是 NEW_TASK 或 singleTask 或 singleInstance 的前提下
方法寻找了3类ActivityRecord并且返回，否则返回null

1. launchMode = singleInstance, 找到包名类名和待启动Activity完全一致的ActivityRecord(全局范围ActivityRecord.intent.component == startActivity.intent.component，这里查找非常严格，因此singleInstance必定单独在一个Task中，如果不存在这个Task就会在后面创建)
2. 在分屏的情况下(暂时忽略)
3. launchMode = NEW_TASK 或 singleTask, 返回符合以下条件的Task.topActivity:
    * 寻找taskIntent.component和startActivity完全一致的task
    * 寻找affinityIntent.component和startActivity完全一致的task(不太理解affinityIntent的含义)
    * 寻找rootAffinity(Task底部Activity的affinity)和startActivity.affinity一致的Task并且是位于最底部的Task 

##### 确定Task栈

如果获取的reusedActivity不为null，意味着已经为确定了待启动Activity的Task栈了(mStartActivity.setTask(reusedActivity.getTask()))

##### ClearTop

LaunchFlags如果是FLAG_ACTIVITY_CLEAR_TOP 或者 mLaunchSingleInstance 或者 mLaunchSingleTask（在当前情况因为reusedActivity不为null，因此LaunchFlags必定有NEW_TASK或SingleInstance、SingleTask）则寻找是否在Task中存在待启动Activity的instance，如果有则destory上面的Activity，并且如果存在待启动Acitivity会deliverNewIntentLocked回调onNewIntent

##### setTargetStackAndMoveToFrontIfNeeded
负责找到目标stack，并且把对应的task从stack的中间一道stack栈的顶端

#### setTaskFromIntentActivity
确定是否要新添加一个Activity实例到Task中，这里也就是ReuseActivity所在的Activity（ReuseActivity的具体作用是否就是用来保存一下Task），这个方法控制的关键变量mAddingToTask和mSourceRecord


1. FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK的情况，ActivityStater会尝试清除task中的所有Activity
2. (mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0 || mLaunchSingleInstance || mLaunchSingleTask，感觉这步重复了，之前已经判断过，为什么这里还要进行一次clearTop的操作?这里是为了确定Activity在不存在的情况下可以添加新的Activity到Task中
3. mStartActivity.realActivity.equals(intentActivity.getTask().realActivity) 顶部Acitivty和待启动的一致 && (mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0 || mLaunchSingleTop 启动模式为SingleTop，此时直接回调onNewIntent

#### 数据结构:
##### ActivityRecord
##### ProcessRecord
##### TaskRecord
```
    intent.component:包含包名和启动项的类名
    
    task.rootAffinity:创建task的Activity的affinity，不会改变
    task.intent:创建task的Activity对应的intent
    task.rootAffinity:启动Task的activity的affinity,一般不指定的情况下默认是包名
    task.mTaskHistory:task中保存的所有ActivityRecord，新启Activity排在后面
    task.realActivity:启动task的activity.cmp

    activityRecord.launchMode.LAUNCH_MULTIPLE:就是Manifest中的普通启动模式standard
    
```

    
####TODO
1. 后续是怎么利用返回的Activity来处理的，总结此方法的作用
2. 整理之前的启动逻辑，归纳之前做了什么工作(ActivityRecord的创建, IntentResolve, ActivityInfo的创建，fix flags)
3. 整理启动模式的细节，launchMode
4. 整理整个ActivityThread和ActivityManagerService直接的交互逻辑
5. 思考如何整理一个文档，能够高效的回忆起startActivity的所有逻辑，以什么样的形式来整理归纳
6. ActivityTask和ActivityStack的含义以及区别
7. Activity.task是什么时候被置为null的，onDestroy整个流程AMS是否是同步执行的
8. 并不是singleTask或NEW_TASK就一定会新启动栈，singleInstance一定会新启动栈