---
layout: article
title: "Android 系统架构 —— Activity 的启动 之 请求方的暂停"
permalink: android-source/activity-launch1
key: android-source-activity-launch1
tags: AndroidFramework
---
## 前言
在 SystemServer 进程启动的的学习中, 我们知道当所有的服务都准备好了之后, AMS 的 systemReady 会启动 Launcher 这个应用程序
 
不过 Launcher 的启动比起我们日常使用 Activity 的启动要少了一些步骤, 这里我们以最普通的 Activity 跳转进行分析

<!--more-->

## 一. Activity 启动的发起
```
class SourceActivity : AppCompatActivity() {

    ......

    private fun initViews() {
        tvContent.setOnClickListener {
            // 通过 intent 显示跳转
            val intent = Intent(this, TargetActivity::class.java)
            // 添加 Flag 为新启动一个任务栈
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            // 开始执行新 Activity 的启动
            startActivity(intent)
        }
    }
    
}
```
调用了 Activity.startActivity 方法, 我们看看它做了什么
```
   /**
    * Activity.startActivity
    */
    @Override
    public void startActivity(Intent intent) {
        // 调用了 Activity 的重载方法
        this.startActivity(intent, null);
    }

   /**
    * Activity.startActivity
    */
   @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // 从上面一个重载方法中可知, 很明显会走到这里 
            startActivityForResult(intent, -1);
        }
    }
    
   /**
    * Activity.startActivityForResult
    */
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
        // 调用了一个带有 Options 参数的重载方法
        startActivityForResult(intent, requestCode, null);
    }
    
   /**
    * Activity.startActivityForResult
    */
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        // 这个 mParent 是一个 Activity 实例对象, 我们这里主要关注它为 null 的情况
        if (mParent == null) {
            // 创建一个 options 选项, 用于执行 Transition 动画
            options = transferSpringboardActivityOptions(options);
            // 这里调用 Instrumentation 的 execStartActivity 发起了 Activity 的启动
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            ......
        } else {
            ......
        }
    }
```
好的经过了一系列的重载方法, 我们看到启动 Activity 的任务会交由 Instrumentation 对象 mInstrumentation 的 execStartActivity 方法来启动
```
    /**
     * Instrumentation.execStartActivity
     *
     * @param who              是谁发起执行 Activity 启动的, 即我们的 SourceActivity.
     * @param contextThread    发起 Activity 动作所在的进程描述, 是一个 ApplicationThread 的 Binder 对象
     * @param token            是一个 ActivityRecord 的对象, 用于描述
     * @param target           即接收 result 的 Activity, 即我们的 SourceActivity
     * @param intent           需要执行的意图
     * @param requestCode      SourceActivity 发起的请求码
     * @param options          页面切换的 options, 这里不做关注
     *
     */
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        // 获取当前进程的描述
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        ......
        try {
            .......
            // 将相关参数传入, 这里要发起 Activity 的启动了
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            // 检查 result
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
    
    // 获取了一个 IActivityManager 的实例
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }
    
    // 这是一个单例
    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    // 通过 ServiceManager 获取 IActivityManager 代理对象
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    // 将代理对象封装成一个 IActivityManager 给当前进程使用
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
```
好了可以看到, 最终会通过 IActivityManager 这个封装好的 BinderProxy 发起 Activity 启动, 这个 IActivityManager 即 AMS 的接口

接下来我们到系统服务进程中看看 Activity 的启动流程

## 二. AMS 处理启动 Activity 的请求
```
    /**
     * ActivityManagerService.startActivity
     */
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
            
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
    
    /**
     * ActivityManagerService.startActivityAsUser
     */
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        // 通过 ActivityStarter.execute 进行 Activity 的启动, 这里来分析一下比较重要的参数
        return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)                    // Activity 发起进程的描述 ApplicationThread
                .setCallingPackage(callingPackage)    // 发起进程的包名
                .setResolvedType(resolvedType)        
                .setResultTo(resultTo)                // 发起 Activity 的 token, 一个 ActivityRecord 对象
                .setResultWho(resultWho)              // 发起 Activity 的 mEmbeddedID
                .setRequestCode(requestCode)          // 请求码
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();
    }
```
好的下面就进入 ActivityStarter 中看看他的 execute
```
class ActivityStarter {
    /**
     * ActivityStarter.execute
     */
    int execute() {
        try {
            if (mRequest.mayWait) {
                ......
            } else {
                // 可以看到回调了其内部的 startActivity, 这里有海量的参数, 我们只关心上面看到的即可, 这里不做赘述
                return startActivity(......);
            }
        } 
    }
    
    private int startActivity(......) {
        // 记录最后一个 Activity 启动的原因
        mLastStartReason = reason;
        // 记录最后一个 Activity 启动的时间
        mLastStartActivityTimeMs = System.currentTimeMillis();
        // 记录最后一个 Activity 的描述, 是一个 ActivityRecord 对象, 存放在缓存的首位置
        mLastStartActivityRecord[0] = null;// 因为最后一个 Activity 尚未启动, 所以这里先置为 null
        // **Point: 这里又回调了一个 startActivity 的重载方法**
        mLastStartActivityResult = startActivity(......);
        ......
    }
    
    // 开始正式执行 Act 的启动
    private int startActivity(......) {
        ......
        ProcessRecord callerApp = null;
        if (caller != null) {
            // mService 是一个 AMS 对象, 这里通过 SourceActivity 所在进程的描述 ApplicationThread 获取一个 ProcessRecord 对象
            // ProcessRecord 是 AMS 中对一个进程的描述, 它内部保存了进程相关的数据
            callerApp = mService.getRecordForAppLocked(caller);
            if (callerApp != null) {
                callingPid = callerApp.pid;         // 记录 process id.
                callingUid = callerApp.info.uid;    // 记录 user id.
            }
        }
        // 描述请求发起的 Activity
        ActivityRecord sourceRecord = null;
        // 描述新 Activity 启动之后, 用于接收 result 的 Activity
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
            // mSupervisor 是 ActivityStackSupervisor 的对象, 它是一个 Activity 任务栈的监视者
            // resultTo 我们知道, 它是 SourceActivity 的 token, 即一个 ActivityRecord 对象
            // 这里通过 ActivityStackSupervisor 获取其对应的 Binder 实体对象, 用于更好的描述当前请求发起的 Activity
            sourceRecord = mSupervisor.isInAnyStackLocked(resultTo);
            ......
            if (sourceRecord != null) {
                // 若请求码大于 0, 并且 sourceRecord 对应的 Activity 非 finish 状态
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    // 那么就将 sourceRecord 赋给 resultRecord, 表示它就是要接收 result 的 Activity
                    // 显然 SourceActivity 接收不到返回值, 因为我们的 requestCode 为 -1;
                    resultRecord = sourceRecord;
                }
            }
        }
        // 接下来就是验证 Flags 的正确性了
        final int launchFlags = intent.getFlags();
        ......
        // 1. 创建一个新的 ActivityRecord 对象 r 用于描述即将启动的目标 Activity
        ActivityRecord r = new ActivityRecord(......);
        // 现将它缓存到 outActivity 中
        if (outActivity != null) {
            outActivity[0] = r;
        }
        // 这里回调了一个参数较少的 startActivity, 然后会回调到 startActivityUnchecked 中
        return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                true /* doResume */, checkedOptions, inTask, outActivity);
    }
    
    private int startActivityUnchecked(......) {
        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor);
        // 计算启动的与 Task 相关的 Flags, 保存在 mLaunchFlags
        computeLaunchingTaskFlags();
        computeSourceStack();
        // 这个 mIntent 中持有我们当前要启动的 TargetActivity 的数据
        mIntent.setFlags(mLaunchFlags);
        // 判断要启动的 Activity 是否是被重新利用的
        ActivityRecord reusedActivity = getReusableIntentActivity();
        // 显然我们启动的新 Activity
        if (reusedActivity != null) {
            ......
        }
        if (mStartActivity.packageName == null) {
            final ActivityStack sourceStack = mStartActivity.resultTo != null
                    ? mStartActivity.resultTo.getStack() : null;
            if (sourceStack != null) {
                sourceStack.sendActivityResultLocked(-1 /* callingUid */, mStartActivity.resultTo,
                        mStartActivity.resultWho, mStartActivity.requestCode, RESULT_CANCELED,
                        null /* data */);
            }
            ActivityOptions.abort(mOptions);
            return START_CLASS_NOT_FOUND;
        }

        // 检查要启动的 Activity 是否就在我们的栈顶, 若是则判断是否需要再启动一个
        final ActivityStack topStack = mSupervisor.mFocusedStack;
        final ActivityRecord topFocused = topStack.getTopActivity();
        final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
        final boolean dontStart = ......
        ......
        // 用于判断是否需要创建新的任务栈
        boolean newTask = false;
        // 获取 SourceActivity 所在的任务栈
        final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                ? mSourceRecord.getTask() : null;
        // mLaunchFlags & FLAG_ACTIVITY_NEW_TASK 我们设置了这个标记为, 所以需要创建新的任务重
        if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
                && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
            newTask = true;
            // 2. 这里执行创建新的任务栈, 传入 taskToAffiliate, 是为了判断, 我们新的任务栈与发起的 Activity 是否为同一个栈
            // 若为同一个则无需再次创建, 执行之后 mTargetStack 就为我们即将启动的 Activity 的任务栈了
            result = setTaskFromReuseOrCreateNewTask(taskToAffiliate, topStack);
        } else if (mSourceRecord != null) {
           ......
        }
        // mTargetStack 将最后已暂停的 Activity 置为 null
        mTargetStack.mLastPausedActivity = null;
        ......
        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                .......
            } else {
                // 若 mTargetStack 是可以获取焦点的, 并且目前它并未获取焦点
                if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                    // 那么将它移动到前面
                    mTargetStack.moveToFront("startActivityUnchecked");
                }
                // 任务栈监视器 调用 resumeFocusedStackTopActivityLocked 恢复栈顶 Activity 的焦点
                mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
            }
        }
        ......
        return START_SUCCESS;
    }
    
}
   /**
    * ActivityStackSupervisor.resumeFocusedStackTopActivityLocked
    */
    boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
        ......
        if (targetStack != null && isFocusedStack(targetStack)) {
            // 调用了 ActivityStack.resumeTopActivityUncheckedLocked
            // 这里的 target 为 mStartActivity 仍然为 SourceActivity.
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }
        ......
        return false;
    }
```
总结一下 ActivityStarter 中的两个 startActivity 方法分别作了如下操作
- **创建一个新的 ActivityRecord 对象 r 用于描述要启动的目标 Activity**
- **判断是否需要为即将启动的 Activity 创建新的任务栈**
  - **调用 setTaskFromReuseOrCreateNewTask 方法, 创建新的任务栈, 返回 ActivityStack 对象**

### 恢复栈顶 Activity
之后便会调用我们新创建的任务栈对象 ActivityStack.resumeTopActivityUncheckedLocked 执行后续操作
```
class ActivityStack {
    
    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        boolean result = false;
        try {
            // 防止递归调用的保护
            mStackSupervisor.inResumeTopActivity = true;
            // 执行了 resumeTopActivityInnerLocked
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }
    
    // 恢复当前任务栈, 栈顶的 Activity, 此时栈顶 Activity 还未创建
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        // 从任务栈监视器中获取, 当前是否需要离开之前页面, 很显然为 true, 我们要离开 SourceActivity
        boolean userLeaving = mStackSupervisor.mUserLeaving;
        mStackSupervisor.mUserLeaving = false;// 还原其成员变量为初始值
        // 判断是否为正在做暂停操作
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
        // 当前已经激活的 Activity, 即我们的 SourceActivity
        if (mResumedActivity != null) {
            // 所以先执行 startPausingLocked 
            pausing |= startPausingLocked(userLeaving, false, next, false);
        }
        if (pausing && !resumeWhilePausing) {
            ......
            return true;
        } else if (mResumedActivity == next && next.isState(RESUMED)
                && mStackSupervisor.allResumedActivitiesComplete()) {
            ......
            return true;
        }
        ......
        return true;
    }
```
可以看到到若是发起方 Activity 非暂停态, 则调用 startPausingLocked 方法, 先去将发起请求的 Activity 置为 paused 状态

## 二. 处理发起 Activity 的暂停
```
class ActivityStack {

    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean pauseImmediately) {
        ......
        // 当前需要被 pasue 的 Activity 
        ActivityRecord prev = mResumedActivity;

        mPausingActivity = prev;    // 将正在 pausing 的 Activity 置为 prev, 即我们的 SourceActivity
        mLastPausedActivity = prev; // 将最后被 pause 的 Activity 置为 prev, 即我们的 SourceActivity
        mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
                || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
        ......
        if (prev.app != null && prev.app.thread != null) {
            try {
                // 1. 很重要, 调用了 scheduleTransaction 执行 SourceActivity 的 pausing 操作
                // 这行代码分为三步
                // 1.1 调用了 AMS 的 getLifecycleManager, 获取了一个 ClientLifecycleManager 对象, 它用于管控请求发起端 Activity 的生命周期
                // 1.2 调用 PauseActivityItem.obtain, 获取了一个 PauseActivityItem 的对象, 描述了 Activity 生命周期中的 Pause 请求
                // 1.3 调用了 ClientLifecycleManager 的 scheduleTransaction 方法, 执行请求的事务
                mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                        PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
            } catch (Exception e) {
                .......
            }
        } else {
           .......
        }
        if (mPausingActivity != null) {
            // pauseImmediately, 由上面的传参可知, pauseImmediately 为 false
            if (pauseImmediately) {
                .......
                return false;
            } else {
                // 2. 因此, 我们需要等待 SourceActivity 完成才继续往下执行, 这里进行了超时处理
                schedulePauseTimeout(prev);
                return true;
            }

        } else {
            .......
            return false;
        }
    }
    
    private static final int PAUSE_TIMEOUT = 500;
    
    private void schedulePauseTimeout(ActivityRecord r) {
        final Message msg = mHandler.obtainMessage(PAUSE_TIMEOUT_MSG);
        msg.obj = r;
        r.pauseTime = SystemClock.uptimeMillis();
        // 通过 Handler 发送了一个 Delayed 的 msg, 即超过 500mm, 则会认定 pausing 操作超时
        mHandler.sendMessageDelayed(msg, PAUSE_TIMEOUT);
    }

}
```
可以看到 ActivityStack 中的 startPausingLocked 方法
- 首先会通过 ClientLifecycleManager.scheduleTransaction 方法, 执行发起 Activity 的暂停操作
- 在执行 pausing 操作的同时, 会调用 schedulePauseTimeout 延时发送 pausing 超时事件

接下来下来我们分分析一下 ClientLifecycleManager.scheduleTransaction
```
class ClientLifecycleManager {
    // 执行生命周期变更的事务
    void scheduleTransaction(@NonNull IApplicationThread client, @NonNull IBinder activityToken,
            @NonNull ActivityLifecycleItem stateRequest) throws RemoteException {
        // 将参数封装成为一个 ClientTransaction, 描述这个客户端的事务
        final ClientTransaction clientTransaction = transactionWithState(client, activityToken,
                stateRequest);
        // 执行这个事务
        scheduleTransaction(clientTransaction);
    }
    
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        // 获取客户端进程的描述
        final IApplicationThread client = transaction.getClient();
        // 执行这个事务
        transaction.schedule();
        if (!(client instanceof Binder)) {
            // 若不是 Binder 本地对象, 则释放
            transaction.recycle();
        }
    }
    
    // 将参数封装到 ClientTransaction 中
    private static ClientTransaction transactionWithState(@NonNull IApplicationThread client,
            @NonNull IBinder activityToken, @NonNull ActivityLifecycleItem stateRequest) {
        final ClientTransaction clientTransaction = ClientTransaction.obtain(client, activityToken);
        clientTransaction.setLifecycleStateRequest(stateRequest);
        return clientTransaction;
    }
    
}
public class ClientTransaction implements Parcelable, ObjectPoolItem {

    // 这个事务处理对应的客户端进程
    private IApplicationThread mClient;
    // 这个事务要处理的 Activity 的声明周期条目
    private ActivityLifecycleItem mLifecycleStateRequest;
    // 这个事务要处理的 Activity 的描述, 即我们的 SourceActivity 的 ActivityRecord 对象
    private IBinder mActivityToken; 
    
    // 执行这个事务
    public void schedule() throws RemoteException {
        // 调用 IApplicationThread 的 scheduleTransaction
        mClient.scheduleTransaction(this);
    }
    
}
```
可以看到最终会回调 ApplicationThread 的 scheduleTransaction, 至此就回到了发起 Activity 的应用进程去执行暂停操作

### 应用进程执行 Activity 的暂停
```
public final class ActivityThread extends ClientTransactionHandler {
    .......
    
    private class ApplicationThread extends IApplicationThread.Stub {
       
        ......
    
        @Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            // 1. 这个方法在其父类 ClientTransactionHandler 中实现
            ActivityThread.this.scheduleTransaction(transaction);
        }
    }
    
     class H extends Handler {
     
         public void handleMessage(Message msg) {
             .......
             switch (msg.what) {
               case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    // 3. 这里又调用 TransactionExecutor 处理这个事务
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        // 将这个事务释放掉...
                        transaction.recycle();
                    }.
                    break;
            }
            ......
         }
     }
    
}

public abstract class ClientTransactionHandler {

    void scheduleTransaction(ClientTransaction transaction) {
        // 可以看到这里调用 ClientTransaction 的 preExecute 表示为执行前的预处理
        transaction.preExecute(this);
        // 2. 通过 Handle 发送了 EXECUTE_TRANSACTION 处理这个事务
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
    
}

public class TransactionExecutor {

    public void execute(ClientTransaction transaction) {
        // 执行 transaction 中添加的 callback, 我们并没有添加过 callback , 所以只关注下面一个方法
        executeCallbacks(transaction);
        // 4. 执行生命周期变更
        executeLifecycleState(transaction);
        ......
    }
 
    private void executeLifecycleState(ClientTransaction transaction) {
        // 即我们在 AMS 中看到的 PauseActivityItem
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        ......
        // 将 AMS 传递过来的 ActivityRecord 代理对象转为 ActivityClientRecord
        final IBinder token = transaction.getActivityToken();
        final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
        
        // 5. 最终执行这个 Pasue 的请求
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }   
}
```
代码到这里又回到了 AMS 中, 去处理这个事务了, 我们看看 PauseActivityItem.execute/postExecute 的实现

```
public class PauseActivityItem extends ActivityLifecycleItem {

    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        // 1. 调用了客户端传递过来的 Binder 代理对象的 AppThread(ClientTransactionHandler).handlePauseActivity 方法, 又回到了客户端处理 Activity 暂停的操作
        client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions,
                "PAUSE_ACTIVITY_ITEM");
    }
    
    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        ......
        try {
            // 2. 告诉 AMS, token 对应的 Activity 暂停了
            ActivityManager.getService().activityPaused(token);
        } catch (RemoteException ex) {
            ......
        }
    }
}
```
可以看到, 当暂停结束之后, 会通过 AMS.activityPaused 通知 AMS 暂停完成了

## 三. AMS 处理暂停完成
```
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    ......
    @Override
    public final void activityPaused(IBinder token) {
        synchronized(this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                // 告诉它所在的任务栈, 这个 Activity 已经暂停了
                stack.activityPausedLocked(token, false);
            }
        }
    }
    ......
}
```
暂停之后, AMS 找到发起 Activity 的任务栈, 通知它已经暂停了, 下面看看 ActivityStack 做了哪些处理 

```java
class ActivityStack {
    final void activityPausedLocked(IBinder token, boolean timeout) {
      
        final ActivityRecord r = isInStackLocked(token);
        if (r != null) {
            // 1. 移除暂停超时的 MSG
            mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
            if (mPausingActivity == r) {
                mService.mWindowManager.deferSurfaceLayout();
                try {
                    // 2. 调用了暂停完成
                    completePauseLocked(true /* resumeNext */, null /* resumingActivity */);
                } finally {
                    mService.mWindowManager.continueSurfaceLayout();
                }
                return;
            } else {
               ......
            }
        }
        ......
    }
}
```
这里移除了预埋的 PAUSE_TIMEOUT_MSG, 然后调用了 **completePauseLocked**, 继续执行目标 Activity 的启动

```
class ActivityStack {

    private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
        ActivityRecord prev = mPausingActivity;
        if (prev != null) {
            ......
            // 将暂停的 ACT 置为 null 
            mPausingActivity = null;
        }
        if (resumeNext) {
            // 1. 获取当前焦点栈, 即要启动 Activity 的栈
            final ActivityStack topStack = mStackSupervisor.getFocusedStack();
            if (!topStack.shouldSleepOrShutDownActivities()) {
                // 2. 可以看到又回调了 resumeFocusedStackTopActivityLocked 这个方法
                // 它最终会调用 ActivityStack.resumeTopActivityInnerLocked
                // 这次就不是执行 SourceActivity 的暂停了, 而是我们 TargetActivity 的启动
                mStackSupervisor.resumeFocusedStackTopActivityLocked(topStack, prev, null);
            } else {
                ......
            }
        }
        ......
    }
    
}
```
可以看到 completePauseLocked 中首先找到当前焦点栈, 然后继续执行 resumeFocusedStackTopActivityLocked 方法

这个方法我们在上面已经调用过一次了, 只是上次执行的是请求 Activity 的暂停, 现在这次调用就是真正执行 目标 Activity 的启动了

## 总结
这篇文章主要分析了 Activity 启动, 发起方 Activity 的暂停过程, 整个流程如下
- 准备阶段
  - 获取目标 Activity 的 ActivityRecord
    - 不存在则创建一个
  - 获取目标 Activity 所在的任务栈 ActivityStack
    - 不存在则创建一个
- 若发起方 Activity 未暂停, 则通知目标应用进程处理请求 Activity 的暂停
- 暂停成功之后, 通知 AMS, 继续进行目标 Activity 的启动操作

这一阶段的时序图入下

![请求发暂停的流程图](https://i.loli.net/2019/11/20/mI3gMrYOSB5hF2d.png)
