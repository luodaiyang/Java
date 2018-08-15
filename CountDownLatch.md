# **一、概述CountDownLatch介绍**
CountDownLatch是一个同步工具类，它允许一个或多个线程一直等待，直到其他线程执行完后再执行。例如，应用程序的主线程希望在负责启动框架服务的线程已经启动所有框架服务之后执行。
**CountDownLatch原理**CountDownLatch是通过一个计数器来实现的，计数器的初始化值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就相应得减1。当计数器到达0时，表示所有的线程都已完成任务，然后在闭锁上等待的线程就可以恢复执行任务。
原理示意图：
![图片](https://images-cdn.shimo.im/9oxnNPiRQvsmQKlh/image.image/png!thumbnail)





源码分析：
      　　1、CountDownLatch:A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.
　　　　大致意思：也就是说主线程在等待所有其它的子线程完成后再往下执行
　　　　
　　　　2、构造函数：CountDownLatch(int count)//初始化count数目的同步计数器，只有当同步计数器为0，主线程才会向下执行
　　　　　 主要方法：void await()//当前线程等待计数器为0 
      　　　　　　　　 boolean await(long timeout, TimeUnit unit)//与上面的方法不同，它加了一个时间限制。
     　　　　　　　　 void countDown()//计数器减1
      　　　　　　　　long getCount()//获取计数器的值
      　　3.它的内部有一个辅助的内部类：sync.
它的实现如下：
　　　　　
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31|/**      * Synchronization control For CountDownLatch.      * Uses AQS state to represent count.      */     private static final class Sync extends AbstractQueuedSynchronizer {         private static final long serialVersionUID = 4982264981922014374L;           Sync(int count) {             setState(count);         }           int getCount() {             return getState();         }           protected int tryAcquireShared(int acquires) {             return (getState() == 0) ? 1 : -1;         }           protected boolean tryReleaseShared(int releases) {             // Decrement count; signal when transition to zero             for (;;) {                 int c = getState();                 if (c == 0)                     return false;                 int nextc = c-1;                 if (compareAndSetState(c, nextc))                     return nextc == 0;             }         }     }|

　　4.await()方法的实现
  　　sync.acquireSharedInterruptibly(1);
     -->if (tryAcquireShared(arg) < 0)//调用3中的tryAcquireShared()方法
            doAcquireSharedInterruptibly(arg);//加入到等待队列中

　　5.countDown（）方法的实现
  　　sync.releaseShared(1);
     --> if (tryReleaseShared(arg))//调用3中的tryReleaseShared（）方法
               doReleaseShared();//解锁

# 二、我们来看一个**应用场景1：**

**假设一条流水线上有三个工作者：worker0，worker1，worker2。有一个任务的完成需要他们三者协作完成，worker2可以开始这个任务的前提是worker0和worker1完成了他们的工作，而worker0和worker1是可以并行他们各自的工作的。**
如果我们要编码模拟上面的场景的话，我们大概很容易就会想到可以用join来做。当在当前线程中调用某个线程 thread 的 join() 方法时，当前线程就会阻塞，直到thread 执行完成，当前线程才可以继续往下执行。补充下：join的工作原理是，不停检查thread是否存活，如果存活则让当前线程永远wait，直到thread线程终止，线程的this.notifyAll 就会被调用。
我们首先用join来模拟这个场景：
**Worker类如下：**
```
package com.concurrent.test3;
 
/**
 * 工作者类
 * @author ThinkPad
 *
 */
public class Worker extends Thread {
 
	//工作者名
	private String name;
	//工作时间
	private long time;
	
	public Worker(String name, long time) {
		this.name = name;
		this.time = time;
	}
	
	@Override
	public void run() {
		// TODO 自动生成的方法存根
		try {
			System.out.println(name+"开始工作");
			Thread.sleep(time);
			System.out.println(name+"工作完成，耗费时间="+time);
		} catch (InterruptedException e) {
			// TODO 自动生成的 catch 块
			e.printStackTrace();
		}	
	}
}
```

```
Test类如下：package com.concurrent.test3;
 
 
public class Test {
 
	public static void main(String[] args) throws InterruptedException {
		// TODO 自动生成的方法存根
 
		Worker worker0 = new Worker("worker0", (long) (Math.random()*2000+3000));
		Worker worker1 = new Worker("worker1", (long) (Math.random()*2000+3000));
		Worker worker2 = new Worker("worker2", (long) (Math.random()*2000+3000));
		
		worker0.start();
		worker1.start();
		
		worker0.join();
		worker1.join();
		System.out.println("准备工作就绪");
		
		worker2.start();		
	}
}
```

运行test，观察控制台输出的顺序，我们发现这样可以满足需求，worker2确实是等worker0和worker1完成之后才开始工作的：
worker1开始工作
worker0开始工作
worker1工作完成，耗费时间=3947
worker0工作完成，耗费时间=4738
准备工作就绪
worker2开始工作
worker2工作完成，耗费时间=4513

除了用join外，用CountDownLatch 也可以完成这个需求。需要对worker做一点修改，我把它放在另一个包下：
**Worker:**
```
package com.concurrent.test4;
 
import java.util.concurrent.CountDownLatch;
 
/**
 * 工作者类
 * @author ThinkPad
 *
 */
public class Worker extends Thread {
 
	//工作者名
        private String name;
	//工作时间
	private long time;
	
	private CountDownLatch countDownLatch;
	
	public Worker(String name, long time, CountDownLatch countDownLatch) {
		this.name = name;
		this.time = time;
		this.countDownLatch = countDownLatch;
	}
	
	@Override
	public void run() {
		// TODO 自动生成的方法存根
		try {
			System.out.println(name+"开始工作");
			Thread.sleep(time);
			System.out.println(name+"工作完成，耗费时间="+time);
			countDownLatch.countDown();
			System.out.println("countDownLatch.getCount()="+countDownLatch.getCount());
		} catch (InterruptedException e) {
			// TODO 自动生成的 catch 块
			e.printStackTrace();
		}	
	}
}
```

```
Test:package com.concurrent.test4;
 
import java.util.concurrent.CountDownLatch;
 
 
public class Test {
 
	public static void main(String[] args) throws InterruptedException {
		// TODO 自动生成的方法存根
 
		CountDownLatch countDownLatch = new CountDownLatch(2);
		Worker worker0 = new Worker("worker0", (long) (Math.random()*2000+3000), countDownLatch);
		Worker worker1 = new Worker("worker1", (long) (Math.random()*2000+3000), countDownLatch);
		Worker worker2 = new Worker("worker2", (long) (Math.random()*2000+3000), countDownLatch);
		
		worker0.start();
		worker1.start();
		
		countDownLatch.await();
		System.out.println("准备工作就绪");
		worker2.start();		
	}
}
```

我们创建了一个计数器为2的 CountDownLatch ，让Worker持有这个CountDownLatch 实例，当完成自己的工作后，调用countDownLatch.countDown() 方法将计数器减1。countDownLatch.await() 方法会一直阻塞直到计数器为0，主线程才会继续往下执行。观察运行结果，发现这样也是可以的：
worker1开始工作
worker0开始工作
worker0工作完成，耗费时间=3174
countDownLatch.getCount()=1
worker1工作完成，耗费时间=3870
countDownLatch.getCount()=0
准备工作就绪
worker2开始工作
worker2工作完成，耗费时间=3992
countDownLatch.getCount()=0

那么既然如此，CountDownLatch与join的区别在哪里呢？事实上在这里我们只要考虑另一种场景，就可以很清楚地看到它们的不同了。
# **三、应用场景2：**
**假设worker的工作可以分为两个阶段，work2 只需要等待work0和work1完成他们各自工作的第一个阶段之后就可以开始自己的工作了，而不是场景1中的必须等待work0和work1把他们的工作全部完成之后才能开始。**
试想下，在这种情况下，join是没办法实现这个场景的，而CountDownLatch却可以，因为它持有一个计数器，只要计数器为0，那么主线程就可以结束阻塞往下执行。我们可以在worker0和worker1完成第一阶段工作之后就把计数器减1即可，这样worker0和worker1在完成第一阶段工作之后，worker2就可以开始工作了。
**worker:**
```
package com.concurrent.test5;
 
import java.util.concurrent.CountDownLatch;
 
/**
 * 工作者类
 * @author ThinkPad
 *
 */
public class Worker extends Thread {
 
	//工作者名
    private String name;
	//第一阶段工作时间
	private long time;
	
	private CountDownLatch countDownLatch;
	
	public Worker(String name, long time, CountDownLatch countDownLatch) {
		this.name = name;
		this.time = time;
		this.countDownLatch = countDownLatch;
	}
	
	@Override
	public void run() {
		// TODO 自动生成的方法存根
		try {
			System.out.println(name+"开始工作");
			Thread.sleep(time);
			System.out.println(name+"第一阶段工作完成");
			
			countDownLatch.countDown();
			
			Thread.sleep(2000); //这里就姑且假设第二阶段工作都是要2秒完成
			System.out.println(name+"第二阶段工作完成");
			System.out.println(name+"工作完成，耗费时间="+(time+2000));
			
		} catch (InterruptedException e) {
			// TODO 自动生成的 catch 块
			e.printStackTrace();
		}	
	}
}
```

```
Test：package com.concurrent.test5;
 
import java.util.concurrent.CountDownLatch;
 
 
public class Test {
 
	public static void main(String[] args) throws InterruptedException {
		// TODO 自动生成的方法存根
 
		CountDownLatch countDownLatch = new CountDownLatch(2);
		Worker worker0 = new Worker("worker0", (long) (Math.random()*2000+3000), countDownLatch);
		Worker worker1 = new Worker("worker1", (long) (Math.random()*2000+3000), countDownLatch);
		Worker worker2 = new Worker("worker2", (long) (Math.random()*2000+3000), countDownLatch);
		
		worker0.start();
		worker1.start();	
		countDownLatch.await();
		
		System.out.println("准备工作就绪");
		worker2.start();
		
	}
 
}
```

观察控制台打印顺序，可以发现这种方法是可以模拟场景2的：
worker0开始工作
worker1开始工作
worker1第一阶段工作完成
worker0第一阶段工作完成
准备工作就绪
worker2开始工作
worker1第二阶段工作完成
worker1工作完成，耗费时间=5521
worker0第二阶段工作完成
worker0工作完成，耗费时间=6147
worker2第一阶段工作完成
worker2第二阶段工作完成
worker2工作完成，耗费时间=5384

最后，总结下CountDownLatch与join的区别：调用thread.join() 方法必须等thread 执行完毕，当前线程才能继续往下执行，而CountDownLatch通过计数器提供了更灵活的控制，只要检测到计数器为0当前线程就可以往下执行而不用管相应的thread是否执行完毕。



# 四、CountDownLatch的源码
 

 
### 构造器
 
CountDownLatch和ReentrantLock一样，内部使用Sync继承AQS。构造函数很简单地传递计数值给Sync，并且设置了state。
 
```
Sync(int count) {
    setState(count);
}
```
 
上文已经介绍过AQS的state，这是一个由子类决定含义的“状态”。对于ReentrantLock来说，state是线程获取锁的次数；对于CountDownLatch来说，则表示计数值的大小。
 
### 阻塞线程
 
接着来看await方法，直接调用了AQS的acquireSharedInterruptibly。

 
```
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```
 
```
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```
 
首先尝试获取共享锁，实现方式和独占锁类似，由CountDownLatch实现判断逻辑。

 
```
protected int tryAcquireShared(int acquires) {
   return (getState() == 0) ? 1 : -1;
}
```
 
返回1代表获取成功，返回-1代表获取失败。如果获取失败，需要调用doAcquireSharedInterruptibly：
 
```
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

 
### 释放操作
 
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
 
同样是首先尝试释放锁，具体实现在CountDownLatch中：

 
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
                   continue;            // loop to recheck cases
               unparkSuccessor(h);
           }
           //2
           else if (ws == 0 &&
                    !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
               continue;                // loop on failed CAS
       }
       if (h == head)                   // loop if head changed
           break;
   }
}
```
 
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
 
要注意，await操作看着好像是独占操作，但它可以在多个线程中调用。当计数值等于0的时候，调用await的线程都需要知道，所以使用共享锁。
 
### 限定时间的await
 
CountDownLatch的await方法还有个限定阻塞时间的版本.

 
```
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```
 
跟踪代码，最后来看doAcquireSharedNanos方法，和上文介绍的doAcquireShared逻辑基本一样，不同之处是加了time字眼的处理。
 
```
private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
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
                    return true;
                }
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
 
进入方法时，算出能够执行多久的deadline，然后在循环中判断时间。注意到代码中间有句：

 
```
nanosTimeout > spinForTimeoutThreshold
```
 
```
static final long spinForTimeoutThreshold = 1000L;
```
 
spinForTimeoutThreshold写死了1000ns，这就是所谓的自旋操作。当超时在1000ns内，让线程在循环中自旋，否则阻塞线程。



