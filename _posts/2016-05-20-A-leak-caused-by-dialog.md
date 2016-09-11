---
layout: post
title: Dialog引发的内存泄漏
category: 技术
tags: Java
catalog:  true
description: 读书笔记
---




原文链接: [A small leak will sink a great ship][1]
原文作者 : Pierre-Yves Ricau

本文是本人对于 LeakCanary 团队的一篇分析内存泄漏的文章的意译。水平有限，如有不够准确之处，敬请包涵。

主旨：在Lollipop之前的版本，Dialog可能导致内存泄漏。

### 引言

LeakCanary 提示存在内存泄漏：

```
GC ROOT thread com.squareup.picasso.Dispatcher.DispatcherThread.
references android.os.Message.obj
references com.example.MyActivity$MyDialogClickListener.this$0
leaks com.example.MyActivity.MainActivity instance `
```

这段报告是说：一个 [Picasso][2] 线程正持有一个位于栈中的 Message 实例的局部变量，而 Message 持有 DialogInterface.OnClickListener 的引用，而 DialogInterface.OnClickListener 又持有一个被销毁 Activity 的引用。

局部变量由于仅存在于栈内，通常存活时间较短。当线程调用某个方法，系统就会为其分配栈帧。当方法返回，栈帧也会随之被销毁，栈内所有局部变量都会被回收。如果局部变量导致了内存泄漏，一般意味着线程死循环或者阻塞了，而且线程在这种状态时持有着 Message 实例的引用。

于是 Dimitris 和我都去 Picasso 源码中一探究竟：

Dispatcher.DispatcherThread 是一个简单的 HandlerThread：

```
static class DispatcherThread extends HandlerThread {
  DispatcherThread() {
    super(Utils.THREAD_PREFIX + DISPATCHER_THREAD_NAME, THREAD_PRIORITY_BACKGROUND);
  }
}
```

这个线程通过 Handler 接收 Message,很标准的实现方式：

```
private static class DispatcherHandler extends Handler {
  private final Dispatcher dispatcher;

  public DispatcherHandler(Looper looper, Dispatcher dispatcher) {
    super(looper);
    this.dispatcher = dispatcher;
  }

  @Override public void handleMessage(final Message msg) {
    switch (msg.what) {
      case REQUEST_SUBMIT: {
        Action action = (Action) msg.obj;
        dispatcher.performSubmit(action);
        break;
      }
      // ... handles other types of messages
    }
  }
}
```

显然 Dispatcher.DispatcherHandler.handleMessage() 里面没有明显会让本地变量持有 Message 引用的 Bug。

后来出现了越来越多内存泄漏的报告，这些报告不仅来自 Picasso，各种各样线程中的局部变量都存在内存泄漏，而且这些内存泄漏往往和 Dialog 的 click listener 有关。发生内存泄漏的线程有一个共同的特性：它们都是worker thread，而且通过某种阻塞队列接收各自的工作。

看来问题来自于Handler 和 Thread 的工作机制中。

### Handler+Thread 工作原理

HandlerThread也是内部封装了Handler的Thread，让我们看看它的工作原理：

```java
for (;;) {
  Message msg = queue.next(); //从消息队列中取出消息，可能阻塞
  if (msg == null) {
    return;
  }
  msg.target.dispatchMessage(msg);//对应handler处理消息
  msg.recycleUnchecked();//清空msg内容、放回消息池中
}
```
确实有一个本地变量持有 Message 的引用，但它的生命周期本应很短，而且在循环结束时被清除。

我们尝试通过利用阻塞队列实现一个简单的工作者线程来重现这个 Bug，它只发送一个 Message：


```java
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;

    //添加到全局的消息池最前端，消息池长度加一
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```


### 模拟消息队列

```java
static class MyMessage {
  final String message;
  MyMessage(String message) {
    this.message = message;
  }
}
```

```java
static void startThread() {
  final BlockingQueue<MyMessage> queue = new LinkedBlockingQueue<>();
  MyMessage message = new MyMessage("Hello Leaking World");
  queue.offer(message);
  new Thread() {
    @Override public void run() {
      try {
        loop(queue);
      } catch (InterruptedException e) {
        throw new RuntimeException(e);
      }
    }
  }.start();
}
```



```java
static void loop(BlockingQueue<MyMessage> queue) throws InterruptedException {
  while (true) {
    MyMessage message = queue.take();
    System.out.println("Received: " + message);
  }
}
```

在 Message 被打印到 Log 中后，MyMessage 实例就应该被回收了，然而LeakCanary还是报出发生了内存泄漏：

```
* GC ROOT thread com.example.MyActivity$2.<Java Local> (named 'Thread-110')
* leaks com.example.MyActivity$MyMessage instance
```


只要我们向消息队列发送新消息，先前的消息就会被回收，新消息开始泄漏。

在虚拟机中，每个栈帧中都有局部变量。垃圾回收器是保守的：如果局部变量的引用可能还存活，就不会回收它。

换句话说，局部变量msg位于栈上，每次循环都会被重写，重写后才不指向上一次循环时指向的堆上的消息对象。



在进入下一次循环后，虽然上次循环的局部变量已经不可达，但它仍然持有对消息对象的引用。
JIT本可以在引用不可达后就立即将引用手动置空，但它并没有这么做，仅仅是保持引用仍是“活的”。

换句话说，当queue.take()处堵塞时，msg引用不会被重写，此时它还指向先前的堆上的消息对象。虽然对应的消息对象已经经历了被处理，recycleUnchecked()清空内容，接着重新放回消息池中的过程，但不会回收指向它的局部变量。也就是说，我们每次发消息都会泄漏一个空的消息对象。

为验证上述理论，我们手动把引用置为null，并打印message以防编译器将置为null的操作优化去掉：

```java
static void loop(BlockingQueue<MyMessage> queue) throws InterruptedException {
  while (true) {
    MyMessage message = queue.take();
    System.out.println("Received: " + message);
    message = null;
    System.out.println("Now null: " + message);
  }
}
```

这次测试中，`MyMessage`指向的对象在`message`置空后立即被回收了。

这种内存泄漏可以在各种 Thread 和 queue 的实现上复现，因此问题出在虚拟机的bug上，且该问题只能在 Dalvik VM 上复现，无法在 ART VM 或 JVM 上复现。

### AlertDialog

我们一般是这样创建alert dialog：

```java
new AlertDialog.Builder(this)
    .setPositiveButton("Baguette", new DialogInterface.OnClickListener() {
      @Override public void onClick(DialogInterface dialog, int which) {
        MyActivity.this.makeBread();
      }
    })
    .show();
```

OnClickListener类持有activity的引用。匿名内部类会被翻译成以下代码，从而持有了activity的引用：

```java
// MyActivity中的第一个匿名内部类
class MyActivity$0 implements DialogInterface.OnClickListener {
  final MyActivity this$0;
  MyActivity$0(MyActivity this$0) {
    this.this$0 = this$0;
  }
  @Override public void onClick(DialogInterface dialog, int which) {
    this$0.makeBread();
  }
}

new AlertDialog.Builder(this)
    .setPositiveButton("Baguette", new MyActivity$0(this));
    .show();
```

通过建造者模式，设置Builder的样式以应用给dialog

```java
        public Builder setPositiveButton(int textId, final OnClickListener listener) {
            P.mPositiveButtonText = P.mContext.getText(textId);
            P.mPositiveButtonListener = listener;
            return this;
        }
```

创建AlertDialog实例，create()中的`P.apply(dialog.mAlert)`将Builder的样式应用于AlertController

```java
        public AlertDialog create() {
            final AlertDialog dialog = new AlertDialog(P.mContext, mTheme, false);//创建AlertDialog实例
            P.apply(dialog.mAlert);//将Builder的样式应用于AlertController
            dialog.setCancelable(P.mCancelable);
            if (P.mCancelable) {
                dialog.setCanceledOnTouchOutside(true);
            }
            dialog.setOnCancelListener(P.mOnCancelListener);//XXXListener内部类也会作为msg的obj属性被发送
            dialog.setOnDismissListener(P.mOnDismissListener);
            if (P.mOnKeyListener != null) {
                dialog.setOnKeyListener(P.mOnKeyListener);
            }
            return dialog;
        }
```
AlertDialog将工作都委托给了AlertController。
```java
        public void apply(AlertController dialog) {
			......
            if (mPositiveButtonText != null) {
                dialog.setButton(DialogInterface.BUTTON_POSITIVE, mPositiveButtonText,
                        mPositiveButtonListener, null);
            }
            if (mNegativeButtonText != null) {
                dialog.setButton(DialogInterface.BUTTON_NEGATIVE, mNegativeButtonText,
                        mNegativeButtonListener, null);
            }
            if (mNeutralButtonText != null) {
                dialog.setButton(DialogInterface.BUTTON_NEUTRAL, mNeutralButtonText,
                        mNeutralButtonListener, null);
            }
			......
        }
```
我们设置确认键、返回键以及取消逻辑时，其实是在设置Handler的消息处理逻辑：
```java
/**
 * Sets a click listener or a message to be sent when the button is clicked.
 * You only need to pass one of {@code listener} or {@code msg}.
 */
   public void setButton(int whichButton, CharSequence text,
            DialogInterface.OnClickListener listener, Message msg) {

        if (msg == null && listener != null) {
            msg = mHandler.obtainMessage(whichButton, listener);//从消息池中取出消息对象，并将msg.obj指向listener，也就指向了activity
        }
        
        switch (whichButton) {

            case DialogInterface.BUTTON_POSITIVE:
                mButtonPositiveText = text;
                mButtonPositiveMessage = msg;//使用成员变量持有了消息对象的引用
                break;
                
            case DialogInterface.BUTTON_NEGATIVE:
                mButtonNegativeText = text;
                mButtonNegativeMessage = msg;
                break;
                
            case DialogInterface.BUTTON_NEUTRAL:
                mButtonNeutralText = text;
                mButtonNeutralMessage = msg;
                break;
                
            default:
                throw new IllegalArgumentException("Button does not exist");
        }
    }
```

此时**从消息池中取出消息对象**，OnClickListener被包装进了Message对象，而且 AlertController.mButtonPositiveMessage也持有了消息对象的引用。mButtonPositiveMessage一直未被发送，发送的是它的复制。

```java
private final View.OnClickListener mButtonHandler = new View.OnClickListener() {
    @Override public void onClick(View v) {
        final Message m;
        if (v == mButtonPositive && mButtonPositiveMessage != null) {
            m = Message.obtain(mButtonPositiveMessage);//原型模式，m为mButtonPositiveMessage的复制
        } else if (v == mButtonNegative && mButtonNegativeMessage != null) {
            m = Message.obtain(mButtonNegativeMessage);
        } else if (v == mButtonNeutral && mButtonNeutralMessage != null) {
            m = Message.obtain(mButtonNeutralMessage);
        } else {
            m = null;
        }
        if (m != null) {
            m.sendToTarget();//发送的是复制品，而不是mButtonPositiveMessage
        }
        // Post a message so we dismiss after the above handlers are executed.
        mHandler.obtainMessage(ButtonHandler.MSG_DISMISS_DIALOG, mDialogInterface)
                .sendToTarget();
    }
};
```

### Handler+Thread遇上AlertDialog时

首先发送一个消息给 HandlerThread，让消息被消费和回收到消息池中，并且不再向该线程发送消息以确保上一条消息被泄漏。
然后，我们在主线程中展示一个带按钮的dialog，我们很有可能获取到同一个消息对象。因为，消息一旦被回收，就会被放到消息池的最前面。因此，HandlerThread 中的局部变量msg就辗转获得了Activity的引用。


```java
HandlerThread background = new HandlerThread("BackgroundThread");
background.start();
Handler backgroundhandler = new Handler(background.getLooper());
final DialogInterface.OnClickListener clickListener = new DialogInterface.OnClickListener() {
  @Override public void onClick(DialogInterface dialog, int which) {
    MyActivity.this.makeCroissants();
  }
};
backgroundhandler.post(new Runnable() {//1，获取一个消息并向HandlerThread发送
  @Override public void run() {
    runOnUiThread(new Runnable() {//2，获取一个消息并向UiThread发送
      @Override public void run() {
        new AlertDialog.Builder(MyActivity.this) //匿名内部类持有activity的引用
            .setPositiveButton("Baguette", clickListener) //将activity的引用与msg关联上了
            .show();
      }
    });
  }
});
```

运行上面的代码并且旋转屏幕以销毁activity，很有可能发生内存泄漏。
大致过程如下：

1. 一个消息对象被某线程A处理后回到消息池的头部，接着从消息池中取出消息就会取到它，如果消息队列中没有新消息则局部变量msg会一直持有消息的引用,
2. 主线程中用成员变量mButtonPositiveMessage指向这个消息（以下称为原型实例），
3. 主线程中拷贝这个原型实例并将拷贝发出去，引用链MainThread--Handler--MainLooper--拷贝的消息实例--匿名内部类DialogInterface.OnClickListener--Activity；
拷贝消息被处理、被清空内容;
4. 在Activity发生GC时Dialog和原型实例会GC，但是局部变量msg所在的线程A可能并未结束，存在引用链Worker Thread--Looper--局部变量msg--匿名内部类DialogInterface.OnClickListener--Activity一直存在，所以这个activity会泄漏。


### 修复措施

- Dalvik VM 才有该问题，Android 5.0以上使用的是 ART VM，ART VM 和 JVM 不存在此问题。
- 给Dialog设置 OnShowListener、OnDismissListener、OnCancelListener、OnClickListener等时，都要注意此问题。
- 使用LeakCanary检测内存泄漏，默认不检测本泄漏，要想检测可以看[这里][3]。

#### App级的修复

1. static修饰listener的实现类，静态内部类不持有外部类的引用。
2. 在dialog的窗口被移除时清除指向listener的引用。这种方案适合在编码阶段进行修改。

```java
public final class DetachableClickListener implements DialogInterface.OnClickListener {

  public static DetachableClickListener wrap(DialogInterface.OnClickListener delegate) {
    return new DetachableClickListener(delegate);
  }

  private DialogInterface.OnClickListener delegateOrNull;

  private DetachableClickListener(DialogInterface.OnClickListener delegate) {
    this.delegateOrNull = delegate;
  }

  @Override public void onClick(DialogInterface dialog, int which) {
    if (delegateOrNull != null) {
      delegateOrNull.onClick(dialog, which);
    }
  }

  public void clearOnDetach(Dialog dialog) {
    dialog.getWindow()
        .getDecorView()
        .getViewTreeObserver()
        .addOnWindowAttachListener(new OnWindowAttachListener() {
          @Override public void onWindowAttached() { }
          @Override public void onWindowDetached() {
            delegateOrNull = null;
          }
        });
  }
}
```
在DialogInterface.OnClickListener外包裹了一层wrapper
```java
DetachableClickListener clickListener = wrap(new DialogInterface.OnClickListener() {
  @Override public void onClick(DialogInterface dialog, int which) {
    MyActivity.this.makeCroissants();
  }
});

AlertDialog dialog = new AlertDialog.Builder(this) //
    .setPositiveButton("Baguette", clickListener) //
    .create();
clickListener.clearOnDetach(dialog);//监听窗口解除事件，手动释放引用
dialog.show();
```

#### 适合框架或维护性质代码的修复

向消息队列注册MessageQueue.IdleHandler接口，该回调接口会在线程即将阻塞、等待新消息之前被调用（详见android.os.MessageQueue.next()）。我们可以在HandlerThread空闲时发送空消息，确保没有泄漏很久的Message对象，就算泄漏很久了，Message对象也不会持有其他的大对象。

```java
static void flushStackLocalLeaks(Looper looper) {
  final Handler handler = new Handler(looper);
  handler.post(new Runnable() {
    @Override public void run() {
      Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
        @Override public boolean queueIdle() {
          handler.sendMessageDelayed(handler.obtainMessage(), 1000);
          return true;
        }
      });
    }
  });
}
```

参考资料：
[A small leak will sink a great ship][4]
[Android 内存泄漏总结][5]
[Android中导致内存泄漏的竟然是它----Dialog][6]


  [1]: https://corner.squareup.com/2015/08/a-small-leak.html
  [2]: https://github.com/square/picasso
  [3]: https://github.com/square/leakcanary/blob/v1.3.1/library/leakcanary-android/src/main/java/com/squareup/leakcanary/AndroidExcludedRefs.java#L112
  [4]: https://corner.squareup.com/2015/08/a-small-leak.html
  [5]: http://gold.xitu.io/entry/569dd0537db2a20052107544
  [6]: http://bugly.qq.com/bbs/forum.php?mod=viewthread&tid=516