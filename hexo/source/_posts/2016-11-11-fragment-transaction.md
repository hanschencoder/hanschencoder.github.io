---
layout: post
title: Fragment事务管理源码分析
tags: [Fragment,事务管理,transaction,源码分析]
categories: 
- Android
---

# 概述
在Fragment使用中，有时候需要对Fragment进行`add`、`remove`、`show`、`hide`、`replace`等操作来进行Fragment的显示隐藏等管理，这些管理是通过`FragmentTransaction`进行事务管理的。事务管理是对于一系列操作进行管理，一个事务包含一个或多个操作命令，是逻辑管理的工作单元。一个事务开始于第一次执行操作语句，结束于Commit。通俗地将，就是把多个操作缓存起来，等调用commit的时候，统一批处理。下面会对Fragmeng的事务管理做一个代码分析

<!-- more -->

# 分析入口
```java
    /**
     * 显示Fragment，如果Fragment已添加过，则直接show，否则构造一个Fragment
     *
     * @param containerViewId 容器控件id
     * @param clz             Fragment类
     */
    protected void showFragment(@IdRes int containerViewId, Class<? extends Fragment> clz) {
        FragmentManager fm = getFragmentManager();
        FragmentTransaction ft = fm.beginTransaction();//开始事务管理
        Fragment f;
        if ((f = fm.findFragmentByTag(clz.getName())) == null) {
            try {
                f = clz.newInstance();
                ft.add(containerViewId, f, clz.getName());//添加操作
            } catch (Exception e) {
                e.printStackTrace();
            }
        } else {
            ft.show(f);//添加操作
        }
        ft.commit();//提交事务
    }
```

上面是一个简单的显示Fragment的栗子，简单判断一下Fragment是否已添加过，添加过就直接show，否则构造一个Fragment，最后提交事务。

# 代码分析
## FragmentManager
![2019-9-2-11-19-20.png](http://image.hanschen.site/master/2019-9-2-11-19-20.png)
上图是获取FragmentManager的大体过程

要管理Fragment事务，首先是需要拿到FragmentManager，在Activity中可以通过`getFragmentManager()`方法获取(使用兼容包的话，通过`FragmentActivity#getSupportFragmentManager()`)，在这里我们就不对兼容包进行分析了

```java
    final FragmentController mFragments = FragmentController.createController(new HostCallbacks());
    
    /**
     * Return the FragmentManager for interacting with fragments associated
     * with this activity.
     */
    public FragmentManager getFragmentManager() {
        return mFragments.getFragmentManager();
    }
```
FragmentManager是一个抽象类，它是通过mFragments.getFragmentManager()来获取的，mFragments是FragmentController对象，它通过`FragmentController.createController(new HostCallbacks())`生成，这是一个静态工厂方法：
```java
    public static final FragmentController createController(FragmentHostCallback<?> callbacks) {
        return new FragmentController(callbacks);
    }
```
在这里面直接new了一个FragmentController对象，注意FragmentController的构造方法需要传入一个`FragmentHostCallback`

## FragmentController构造方法
```java
    private final FragmentHostCallback<?> mHost;
    private FragmentController(FragmentHostCallback<?> callbacks) {
        mHost = callbacks;
    }
```
构造方法很简单，传入了一个FragmentHostCallback实例

## FragmentController#getFragmentManager
```java
    public FragmentManager getFragmentManager() {
        return mHost.getFragmentManagerImpl();
    }
```
这里又调用了mHost的getFragmentManagerImpl方法，希望童鞋们没有被绕晕，mHost是一个FragmentHostCallback实例，那我们回过头来看看它传进来的地方

## FragmentHostCallback
这个FragmentHostCallback是一个抽象类，我们可以看到，在Activity中是传入了 `Activity#HostCallbacks`内部类，这个就是FragmentHostCallback的实现类

## FragmentHostCallback#getFragmentManagerImpl
```java
    final FragmentManagerImpl mFragmentManager = new FragmentManagerImpl();
    FragmentManagerImpl getFragmentManagerImpl() {
        return mFragmentManager;
    }
```
终于找到FragmentManager的真身`FragmentManagerImpl`了

## FragmentManagerImpl#beginTransaction
```java
    @Override
    public FragmentTransaction beginTransaction() {
        return new BackStackRecord(this);
    }
```
可以看到，所谓的FragmentTransaction其实就是一个BackStackRecord。到现在，FragmentManager和FragmentTransaction我们都找到了。下图就是各个类之间的关系：
![2019-9-2-11-20-42.png](http://image.hanschen.site/master/2019-9-2-11-20-42.png)


下面开始真正的事务管理分析，我们先选择一个事务add来进行分析

## FragmentTransaction#add
```java
    public FragmentTransaction add(int containerViewId, Fragment fragment, String tag) {
        doAddOp(containerViewId, fragment, tag, OP_ADD);
        return this;
    }
    
    private void doAddOp(int containerViewId, Fragment fragment, String tag, int opcmd) {
    
        //设置fragment的FragmentManagerImpl，mManager其实就是Activity#HostCallbacks中的成员变量
        fragment.mFragmentManager = mManager;

        //设置fragment的tag
        if (tag != null) {
            if (fragment.mTag != null && !tag.equals(fragment.mTag)) {
                throw new IllegalStateException("...");
            }
            fragment.mTag = tag;
        }

        if (containerViewId != 0) {
            if (containerViewId == View.NO_ID) {
                throw new IllegalArgumentException("...");
            }
            if (fragment.mFragmentId != 0 && fragment.mFragmentId != containerViewId) {
                throw new IllegalStateException("");
            }
            //设置fragment的mContainerId以及mFragmentId
            fragment.mContainerId = fragment.mFragmentId = containerViewId;
        }
        
        //新增一个操作
        Op op = new Op();
        op.cmd = opcmd;
        op.fragment = fragment;
        //添加操作
        addOp(op);
    }
    
    //插入到链表的最后
    void addOp(Op op) {
        if (mHead == null) {
            mHead = mTail = op;
        } else {
            op.prev = mTail;
            mTail.next = op;
            mTail = op;
        }
        op.enterAnim = mEnterAnim;
        op.exitAnim = mExitAnim;
        op.popEnterAnim = mPopEnterAnim;
        op.popExitAnim = mPopExitAnim;
        mNumOp++;
    }
```
add的操作步骤为：

 1. 设置fragment的FragmentManagerImpl
 2. 设置fragment的tag
 3. 设置fragment的mContainerId以及mFragmentId
 4. 插入一个类型为OP_ADD的操作到链表最后

这里用到了一个类：
```java
    static final class Op {
        Op next;//下一操作节点
        Op prev;//上一操作节点
        int cmd;//操作类型，可选有：OP_NULL|OP_ADD|OP_REPLACE|OP_REMOVE|OP_HIDE|OP_SHOW|OP_DETACH|OP_ATTACH
        Fragment fragment;//操作的Fragment对象
        int enterAnim;//入场动画
        int exitAnim;//出场动画
        int popEnterAnim;//弹入动画
        int popExitAnim;//弹出动画
        ArrayList<Fragment> removed;
    }
```
这是一个操作链表节点。所有add、remove、hide等事物最终会形成一个操作链

## FragmentTransaction#commit
等所有操作都插入后，最后我们需要调用FragmentTransaction的commit方法，操作才会真正地执行。
```java
    public int commit() {
        return commitInternal(false);
    }
    
    int commitInternal(boolean allowStateLoss) {
        //防止重复commit
        if (mCommitted) {
            throw new IllegalStateException("commit already called");
        }
        
        //DEBUG代码统统不管
        if (FragmentManagerImpl.DEBUG) {
            Log.v(TAG, "Commit: " + this);
            LogWriter logw = new LogWriter(Log.VERBOSE, TAG);
            PrintWriter pw = new FastPrintWriter(logw, false, 1024);
            dump("  ", null, pw, null);
            pw.flush();
        }
        
        mCommitted = true;
        
        //只有调用了addToBackStack方法之后，这个标记才会为true
        if (mAddToBackStack) {
            mIndex = mManager.allocBackStackIndex(this);
        } else {
            mIndex = -1;
        }
        //插入事物队列
        mManager.enqueueAction(this, allowStateLoss);
        return mIndex;
    }
```

## FragmentManagerImpl#enqueueAction
```java
    /**
     * Adds an action to the queue of pending actions.
     *
     * @param action the action to add
     * @param allowStateLoss whether to allow loss of state information
     * @throws IllegalStateException if the activity has been destroyed
     */
    public void enqueueAction(Runnable action, boolean allowStateLoss) {
        if (!allowStateLoss) {
            checkStateLoss();
        }
        synchronized (this) {
            if (mDestroyed || mHost == null) {
                throw new IllegalStateException("Activity has been destroyed");
            }
            if (mPendingActions == null) {
                mPendingActions = new ArrayList<Runnable>();
            }
            mPendingActions.add(action);
            if (mPendingActions.size() == 1) {
                mHost.getHandler().removeCallbacks(mExecCommit);
                mHost.getHandler().post(mExecCommit);
            }
        }
    }
```
这里把操作添加到`mPendingActions`列表里去。并通过mHost.getHandler()获取Handler发送执行请求。从上面的分析知道，mHost就是Activity的HostCallbacks，构造方法中把Activity的mHandler传进去了，这里执行的`mHost.getHandler()`获取到的也就是Activity中的mHandler，这样做是因为需要在主线程中执行
```java
final Handler mHandler = new Handler();
```

再看看mExecCommit中做了什么操作：
```java
    Runnable mExecCommit = new Runnable() {
        @Override
        public void run() {
            execPendingActions();
        }
    };
    
    /**
     * Only call from main thread!
     */
    public boolean execPendingActions() {
        if (mExecutingActions) {
            throw new IllegalStateException("Recursive entry to executePendingTransactions");
        }
        
        //再次检测是否主线程
        if (Looper.myLooper() != mHost.getHandler().getLooper()) {
            throw new IllegalStateException("Must be called from main thread of process");
        }

        boolean didSomething = false;

        while (true) {
            int numActions;
            
            synchronized (this) {
            
                //参数检测
                if (mPendingActions == null || mPendingActions.size() == 0) {
                    break;
                }
                
                numActions = mPendingActions.size();
                if (mTmpActions == null || mTmpActions.length < numActions) {
                    mTmpActions = new Runnable[numActions];
                }
                
                mPendingActions.toArray(mTmpActions);
                mPendingActions.clear();
                mHost.getHandler().removeCallbacks(mExecCommit);
            }
            
            mExecutingActions = true;
            //遍历执行待处理的事务操作
            for (int i=0; i<numActions; i++) {
                mTmpActions[i].run();
                mTmpActions[i] = null;
            }
            mExecutingActions = false;
            didSomething = true;
        }

        doPendingDeferredStart();

        return didSomething;
    }
```
插入了事物之后，就是在主线程中把需要处理的事务统一处理，处理事务是通过执行`mTmpActions[i].run()`进行的，这个mTmpActions[i]就是前面我们通过enqueueAction方法插入的BackStackRecord，童鞋们可能没注意到，它可是一个Runnable，我们来看看它的定义
```java
final class BackStackRecord extends FragmentTransaction implements
        FragmentManager.BackStackEntry, Runnable {
    static final String TAG = FragmentManagerImpl.TAG;
    
    ... ...
}
```
兜兜转转，我们又回到了BackStackRecord

## BackStackRecord#run
```java
    public void run() {
    
        ......
        
        if (mManager.mCurState >= Fragment.CREATED) {
            SparseArray<Fragment> firstOutFragments = new SparseArray<Fragment>();
            SparseArray<Fragment> lastInFragments = new SparseArray<Fragment>();
            calculateFragments(firstOutFragments, lastInFragments);
            beginTransition(firstOutFragments, lastInFragments, false);
        }
        //遍历链表，根据cmd事务类型依次处理事务
        Op op = mHead;
        while (op != null) {
            switch (op.cmd) {
                case OP_ADD: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.enterAnim;
                    mManager.addFragment(f, false);
                }
                break;
                case OP_REPLACE: {
                    Fragment f = op.fragment;
                    int containerId = f.mContainerId;
                    if (mManager.mAdded != null) {
                        for (int i = mManager.mAdded.size() - 1; i >= 0; i--) {
                            Fragment old = mManager.mAdded.get(i);
                            if (old.mContainerId == containerId) {
                                if (old == f) {
                                    op.fragment = f = null;
                                } else {
                                    if (op.removed == null) {
                                        op.removed = new ArrayList<Fragment>();
                                    }
                                    op.removed.add(old);
                                    old.mNextAnim = op.exitAnim;
                                    if (mAddToBackStack) {
                                        old.mBackStackNesting += 1;
                                    }
                                    mManager.removeFragment(old, mTransition, mTransitionStyle);
                                }
                            }
                        }
                    }
                    if (f != null) {
                        f.mNextAnim = op.enterAnim;
                        mManager.addFragment(f, false);
                    }
                }
                break;
                case OP_REMOVE: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.exitAnim;
                    mManager.removeFragment(f, mTransition, mTransitionStyle);
                }
                break;
                case OP_HIDE: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.exitAnim;
                    mManager.hideFragment(f, mTransition, mTransitionStyle);
                }
                break;
                case OP_SHOW: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.enterAnim;
                    mManager.showFragment(f, mTransition, mTransitionStyle);
                }
                break;
                case OP_DETACH: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.exitAnim;
                    mManager.detachFragment(f, mTransition, mTransitionStyle);
                }
                break;
                case OP_ATTACH: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.enterAnim;
                    mManager.attachFragment(f, mTransition, mTransitionStyle);
                }
                break;
                default: {
                    throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
                }
            }

            op = op.next;
        }

        mManager.moveToState(mManager.mCurState, mTransition,
                mTransitionStyle, true);

        if (mAddToBackStack) {
            mManager.addBackStackState(this);
        }
    }
```
到这一步，提交的事务就被真正执行了，我们知道，即使commit了事务之后，也不是同步执行的，是通过Handler发送到主线程执行的。

所有事务的处理都是在run方法里面执行，但是我们留意到，想要搞清楚add、remove等事务背后真正做了什么，还需要深入了解FragmentManagerImpl。

本文主要讲解Fragment事务的流程，FragmentManagerImpl的分析准备放到下一篇分析文章[Fragment源码分析](http://blog.csdn.net/shensky711/article/details/53171248)中，相信通过分析之后，就可以对Fragment的生命周期也有一个很好的认识了