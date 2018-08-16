# 一、核心原理
AbstractQueuedSynchronizer**同步器面向的是锁的实现者**，即其内部已经封装了一些关于锁的操作。这也是上文中提到的两句话：
	（1）同步器的主要使用方式是**继承**，子类通过继承**同步器**并重写指定方法，随后将同步器组合在自定义组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法；
	（2）**子类推荐被定义为自定义同步组件的静态内部类**，同步器自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用。


---
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLjQa8p8Pj9ftd8JibH27zzpmw89icfD4gvtOiaKuo6vhyict2ObicfJbl9Eg/0?wx_fmt=png)
﻿上述五个方法称之为：**同步器可重写的方法**，究其原因，可以根据上述分为四个种类的方法修饰符进行理解。
（2）public final类别
除了上述protected类别的方法，还有一个关键的类别就是public final类别，这是因为，这是我们可以直接使用的方法，称之为“**模板方法**”，当我们实现自定义的同步组件的时候，我们可以调用这些模板方法获取我们需要的东西。主要有如下方法：
常用的模板方法方法含义如下：﻿
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlL8DTsrhD54vbQKjybI7ibU8jrfzcOEf3KKpniajibibY2SKUDquDI0xdicDQ/0?wx_fmt=jpeg)


---
# **二、CountDownLatch的使用**
CountDownLatch是同步工具类之一，可以指定一个计数值，在并发环境下由线程进行减1操作，当计数值变为0之后，被await方法阻塞的线程将会唤醒，实现线程间的同步。

```
	﻿public void startTestCountDownLatch() {
   int threadNum = 10;
   final CountDownLatch countDownLatch = new CountDownLatch(threadNum);

   for (int i = 0; i < threadNum; i++) {
       final int finalI = i + 1;
       new Thread(() -> {
           System.out.println("thread " + finalI + " start");
           Random random = new Random();
           try {
               Thread.sleep(random.nextInt(10000) + 1000);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
           System.out.println("thread " + finalI + " finish");

           countDownLatch.countDown();
       }).start();
   }

   try {
       countDownLatch.await();
   } catch (InterruptedException e) {
       e.printStackTrace();
   }
   System.out.println(threadNum + " thread finish");
}
```

主线程启动10个子线程后阻塞在await方法，需要等子线程都执行完毕，主线程才能唤醒继续执行。



---
# **三、CountDownLatch的源码分析**
## 一、重写核心方法tryAcquireShared(int acquires)以实现自己特定功能await()：

```
protected int tryAcquireShared(int acquires) {

return (getState() == 0) ? 1 : -1;

}
```
二、调用await()进入等待：
```
public void await() throws InterruptedException {

sync.acquireSharedInterruptibly(1);

}

await()方法直接调用AQS提供的模板方法去
首先尝试获取共享锁，实现方式和独占锁类似，由CountDownLatch实现判断逻辑。
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
而模板方法有直接调用使用者重写的核心业务方法：
tryAcquireShared(arg)
这里面定义了使用者业务的实现
protected int tryAcquireShared(int acquires) {


return (getState() == 0) ? 1 : -1;


}
```
返回1代表获取成功，返回-1代表获取失败。如果获取失败，需要调用doAcquireSharedInterruptibly：之后继续调用AQS提供的方法：
```
/**
 * Acquires in shared interruptible mode.
 * @param arg the acquire argument
 */
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

```
doAcquireSharedInterruptibly的逻辑和独占功能的acquireQueued基本相同，阻塞线程的过程是一样的。不同之处：
 
1. 创建的Node是定义成共享的（Node.SHARED）；
2. 被唤醒后重新尝试获取锁，不只设置自己为head，还需要通知其他等待的线程。（重点看后文释放操作里的setHeadAndPropagate）


```
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);

```
方法：
```
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}

```
## 二、重写核心方法tryReleaseShared(int releases)以实现自己特定功能
```
countDown() 对操作变量进行减一操作
释放操作
public void countDown() {
    sync.releaseShared(1);
}
```

```
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```
调用CountDown()
```
public void countDown() {
    sync.releaseShared(1);
}
```
countDown操作实际就是释放锁的操作，每调用一次，计数值减少1：

```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```
同样是首先尝试释放锁，具体实现在CountDownLatch中：（重写的核心业务功能）
```
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```
死循环加上cas的方式保证state的减1操作，当计数值等于0，代表所有子线程都执行完毕，被await阻塞的线程可以唤醒了，下一步调用doReleaseShared：


```
private void doReleaseShared() {
   for (;;) {
       Node h = head;
       if (h != null && h != tail) {
           int ws = h.waitStatus;
           if (ws == Node.SIGNAL) {
             //1
               if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                   continue;            // loop to recheck cases
              unparkSuccessor(h);
           }
           //2
           else if (ws == 0 &&
                    !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
               continue;                // loop on failed CAS
       }
       if (h == head)                   // loop if head changed
           break;
   }
```
}


标记1里，头节点状态如果SIGNAL，则状态重置为0，并调用unparkSuccessor唤醒下个节点。
标记2里，被唤醒的节点状态会重置成0，在下一次循环中被设置成PROPAGATE状态，代表状态要向后传播。
```
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0); 
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
               s = t;
    }
   if (s != null)
        LockSupport.unpark(s.thread);
}
```
在唤醒线程的操作里，分成三步：
 
* 处理当前节点：非CANCELLED状态重置为0；
* 寻找下个节点：如果是CANCELLED状态，说明节点中途溜了，从队列尾开始寻找排在最前还在等着的节点
* 唤醒：利用LockSupport.unpark唤醒下个节点里的线程。

 
线程是在doAcquireSharedInterruptibly里被阻塞的，唤醒后调用到setHeadAndPropagate。
>
```
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node);  
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
                }

}
```


setHead设置头节点后，再判断一堆条件，取出下一个节点，如果也是共享类型，进行doReleaseShared释放操作。下个节点被唤醒后，重复上面的步骤，达到共享状态向后传播。
 
### 要注意，await操作看着好像是独占操作，但它可以在多个线程中调用。当计数值等于0的时候，调用await的线程都需要知道，所以使用共享锁。
###  
### 



