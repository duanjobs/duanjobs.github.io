---
title: 如何判断Android应用程序前后台切换
date: 2017-09-24 21:08:46
categories: Android
keywords: 前后台切换, Android
description: Android前后台切换
tags: [Android, application]
---
#### 现有的成熟方案
[AndroidProcess](https://github.com/wenmingvs/AndroidProcess )

在这个AndroidProcess中提供了六种方法去判断Android前后台切换的逻辑，个人认为最合适的是方法三，但是方法三有局限性：** 如果Activity在不同的进程，那么判断将会不准确 **
#### 重写的过程
根据方案三 重写了一个接口`MonitorLifecycleCallbacks`，用application implements 这个接口，即可
在application中得到回调。

- 重写的第一个方案。本来是在 `onResume` 和 `onPause` 两个生命周期里判断,但是出现了问题：** 发现小米等rom在当前app界面，锁屏后再按侧边键开启屏幕，此时app界面仍未显现，但此时居然会走完一个完整生命周期，所以那时候会先回调onBecameForeground 然后迅速回调onBecameBackground，这样的快速回调两次。**但是原生rom没这个问题。

#### 如何解决？
那就得从生命周期说起，比如从Activity A跳转至Activity B，A和B的生命周期变化经历了如下一个过程：
`A onPause ——> B onCreate ——> B onStart ——> B onResume ——> A onStop`
A的`onPause`在B的`onResume`前面调用。但是，在小米那个情况中，一个完整生命周期 `onPause` 是在 `onResume` 后面的，无法和切换逻辑统一。所以，这里利用`onStop`和 `onResume` 这两个生命周期。我们发现：不管是切换Activity，还是一个Activity的完整生命周期，`onStop`都在 `onResume`后面。这样就做到了逻辑统一。所以，这里采用延迟得做法，如果 有`onStop `则取消 `onResume`中的回调。这样的话，在切换Activity时，逻辑正常。在锁屏开屏时可以屏蔽掉回到前台的回调，但屏蔽不了回到后台的回调。但是已经可以达到需求的要求了。

```
/**
 * Created by duanjobs on 17/9/16.
 * sdk>=14
 */

public class MonitorLifecycleCallbacks implements Application.ActivityLifecycleCallbacks {
    public static final long CHECK_DELAY = 600;

    private int count = 0;
    private int before;

    public interface Listener {
        void onForeground();

        void onBackground();
    }

    private static MonitorLifecycleCallbacks instance;
    private Handler handler = new Handler();
    private Listener listener;
    private Runnable check;

    public static MonitorLifecycleCallbacks init(Application application) {
        if (instance == null) {
            instance = new MonitorLifecycleCallbacks();
            application.registerActivityLifecycleCallbacks(instance);
        }
        return instance;
    }

    public static MonitorLifecycleCallbacks get(Application application) {
        if (instance == null) {
            init(application);
        }
        return instance;
    }

    public static MonitorLifecycleCallbacks get(Context ctx) {
        if (instance == null) {
            Context appCtx = ctx.getApplicationContext();
            if (appCtx instanceof Application) {
                init((Application) appCtx);
            }
            throw new IllegalStateException(
                    "Foreground is not initialised and " +
                            "cannot obtain the Application object");
        }
        return instance;
    }

    public void setListener(Listener listener) {
        this.listener = listener;
    }


    @Override
    public void onActivityCreated(Activity activity, Bundle savedInstanceState) {

    }

    @Override
    public void onActivityStarted(Activity activity) {

    }

    @Override
    public void onActivityResumed(Activity activity) {
        before = count;
        count++;
        handler.postDelayed(check = new Runnable() { //纯粹是为了防止小米等rom 当前app界面
            // 锁屏后再按侧边键开屏，虽然当前app界面没显示，
            // 但此时居然会走完一个完整生命周期，所以那时候会快速回调两次：先onForeground 然后
            //onBackground。但是原生rom没这个问题，这个操作只是屏蔽了onForeground
            //并没有屏蔽onBackground 所以在onBackground回调中操作要谨慎            			@Override
            public void run() {
                if (before <= 0 && count > 0) {
                    listener.onForeground();
                }
            }
        }, CHECK_DELAY);

    }

    @Override
    public void onActivityPaused(Activity activity) {

    }

    @Override
    public void onActivityStopped(Activity activity) {
        before = count;
        count--;
        if (check != null)
            handler.removeCallbacks(check);
        if (before > 0 && count == 0) {
            listener.onBackground();
        } else if (before > 0 && count > 0) {
            LogUtil.log.d("still Foreground");
        }

    }

    @Override
    public void onActivitySaveInstanceState(Activity activity, Bundle outState) {

    }

    @Override
    public void onActivityDestroyed(Activity activity) {

    }
}
```
上面就是完整的代码了~
