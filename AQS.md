使用Lock对象实现同步以及线程间通信 介绍了如何使用Lock实现和synchronized关键字类似的同步功能，只是Lock在使用时需要显式地获取和释放锁，synchronized实现的隐式的获取所和释放锁。
虽然Lock它缺少了（通过synchronized块或者方法所提供的）隐式获取释放锁的便捷性，但是却拥有了**锁获取与释放的可操作性、可中断的获取锁以及超时获取锁**等多种synchronized关键字所不具备的同步特性，何以见得，举个简单的实例：
假设我们需要先获得锁A，然后在获取锁B，当锁B获得后，释放锁A同时获取锁C，当锁C获得后，在释放B同时获得锁D。。。是不是已经被绕晕了，很显然如果使用synchronized实现的话，不但其过程复杂难以控制，并且稍微出错可以说是一种灾难性的后果。
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLIGibZSj0Mn4MyicfPW471OckqC08RfpI4hC55UKzlibpINUQSiczD2NEFg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1)
而关于Lock接口的使用，也在上一篇的内容中详细的介绍了关系Lock接口的使用案例。下边几张图显示了Lock相关类在Java 8 concurrent并发包下的大致位置和关系。
**1、Java 8中locks包下的类：**
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLgqlyBq32jMZWn1NszoHhUXZsB6gdWcyhYmtOosr107kmjbaE4mxjqQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1)
**2、他们之间大致的继承和实现关系如下：**
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlL1oCmnkjAF8ejhazZJOpD6Nyt9Nicp17NHicibxgre6rcd5ybrkzRbTiaicA/0?wx_fmt=jpeg)
从上述截图中可以看到Lock接口的实现主要有：ReentrantLock，其中ReentrantLock中使用到了AbstractQueuedSynchronizer（队列同步器），下边会一起探讨一下AbstractQueuedSynchronizer的设计与实现。
**3、Lock接口的定义：**
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlL4Tqq5QO7TXNia620BoHnpf2FGaLhEvcjqJtiaQaPgfgUYlybyMQibSKdw/0?wx_fmt=png)
**4、Lock各接口的含义：**
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLaia5mylP9IFvBuJ4KoG2egJea0fOapicw2PRdqy4mK36icU7H2S0t8uWw/0?wx_fmt=jpeg)
Lock接口定义了实现一个锁应该具有的方法，下边看一下AQS。
**二、队列同步器AQS**
队列同步器（简称：同步器）AbstractQueuedSynchronizer（英文简称：**AQS**，也是面试官常问的什么是AQS的AQS），是用来**构建锁**或者其他**同步组件**的基础框架，它使用了一个**int成员变量表示同步状态**，通过**内置的FIFO队列**来完成资源获取线程的排队工作。
**这里暴露出了两个含义：**
（1）第一个就是我们知道如果我们使用锁同步共享变量的时候，我们首先应该要知道这个共享变量的状态（是否已经被其他线程锁住等），这也是这个int成员变量的作用；
（2）第二个就是既然是同步访问共享资源，肯定会有一些线程无法获取到共享资源等待获取锁而进入一个容器中进行保存而这容器就是这个内置的FIFO队列。
同步器的主要使用方式是**继承**，子类通过继承**同步器**并实现它的**抽象方法**（这里说抽象方法并不准确，因为他虽然是一个抽象类，但是并没有abstract修饰的抽象方法）来管理同步状态，在抽象方法的实现过程中免不了要对**同步状态（上文中说的int成员变量）**进行更改，这时就需要使用同步器提供的3个方法（getState()、setState()、compareAndSetState()）来进行操作，因为它们能够保证状态的改变是安全的。
**子类推荐被定义为自定义同步组件的静态内部类**，同步器自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持**独占式**地获取同步状态，也可以支持**共享式**地获取同步状态，这样就可以方便实现不同类型的同步组（ReentrantLock、ReentrantReadWriteLock、CountDownLatch等）。
同步器是实现锁（也可以是任意同步组件）的关键，在锁的实现中聚合同步器，利用同步器实现锁的语义。可以这样理解二者之间的关系：
（1）**锁是面向使用者的**，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；
（2）** 同步器面向的是锁的实现者**，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。或者可以把AQS认为是锁的实现者的一个父类。
锁和同步器很好地隔离了**使用者**和**实现者**所需关注的领域。
**1、首先看一下AbstractQueuedSynchronizer的主要方法：**
在看具体的AbstractQueuedSynchronizer方法之前，我们可以大致将AbstractQueuedSynchronizer的方法分为如下几种：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLWW8tujLiaDCWahWsbyN6YGXL1iaAF3RVGXhD7cV5TQPfViaQCklTAQuibg/0?wx_fmt=png)
AbstractQueuedSynchronizer是一个抽象类，但是却没有一个抽象方法，但是主要的方法可以分为上述的四种，我们知道final修饰的方式是不可以被子类重写的，protected修饰的方法是可以被子类重载的，下边展示一下大致分的四类方法。
（1）protected类别
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLS2Yrc2xrsYW27M844PyibNQA5kiaC6BI8jiczVibIWMWqPOFfsJz2XCiaOg/0?wx_fmt=png)
具体代码如下：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLQ3n3Zt3Umr5NW4oRmDQ1gsPrzzFibtzPpM8KF00jFviawvr0AwK1C5iag/0?wx_fmt=png)
AbstractQueuedSynchronizer虽然没有抽象方法，但是提供了五个方法可以让我们在子类中重载，并且这五个方法都是空实现直接抛出异常，也就是说我们要使用这五个方法提供的功能，我们必须要自己在子类中进行实现，这也是“模板方法模式”的一种体现和使用。这五个方法的具体含义如下：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLjQa8p8Pj9ftd8JibH27zzpmw89icfD4gvtOiaKuo6vhyict2ObicfJbl9Eg/0?wx_fmt=png)
上述五个方法称之为：**同步器可重写的方法**，究其原因，可以根据上述分为四个种类的方法修饰符进行理解。
（2）public final类别
除了上述protected类别的方法，还有一个关键的类别就是public final类别，这是因为，这是我们可以直接使用的方法，称之为“**模板方法**”，当我们实现自定义的同步组件的时候，我们可以调用这些模板方法获取我们需要的东西。主要有如下方法：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLPmMqd3QPcNsd38iccUkln4ib1E7BAbUYBloJqW7CicrRoiaByS7gH7UwwA/0?wx_fmt=jpeg)
常用的模板方法方法含义如下：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlL8DTsrhD54vbQKjybI7ibU8jrfzcOEf3KKpniajibibY2SKUDquDI0xdicDQ/0?wx_fmt=jpeg)
同步器提供的上述模板方法基本上分为3类：**独占式获取与释放同步状态**、**共享式获取与释放同步状态**、**查询同步队列中的等待线程情况**。
自定义同步组件将使用同步器提供的模板方法来实现自己的同步语义。只有掌握了同步器的工作原理才能更加深入地理解并发包中其他的并发组件。
（3）protected final类别
上文中，我们至少应该知道了我们要对int类型的同步状态进行修改，下边的三个方法提供了可以修改：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLBsm2F9XT0N94dZpdPMkGNGz2IMTAIGr3NV2uxsEEricavuoFfX4QxmA/0?wx_fmt=png)
另外还有三个：hasWaiters、getWaitQueueLength、getWaitingThreads三个方法。
**2、再看一下AbstractQueuedSynchronizer的内部类：**
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLDXYDiaa03cgK5CQ3j5rq7VXYZRyELOLgqQEN6JqA73iaBoPoGDWmk9ibQ/0?wx_fmt=png)
从上图中可以看到AbstractQueuedSynchronizer有两个内部类：一个是ConditionObject，另一个是Node。
**3、ConditionObject内部类：**
（1）ConditionObject
这个我们知道在使用synchronized的时候是使用wait和notify进行线程间通信，使用ReentrantLock的时候是使用Condition实现的线程间通信，而这正是AbstractQueuedSynchronizer帮我们进一步封装的Condition接口：
（2）Condition接口如下：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLtLziaLZN58qFKEznoB6dWO6arAh0qcicMhHicIYYdu9CiaJibYCc2VBNvaQ/0?wx_fmt=png)
（3）ConditionObject实现了Condition接口：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLHzQVoNKawYXzO2xw9TW74MiaLqhvO3aOWDBAsBoBFR0sp2yQJictMsvg/0?wx_fmt=png)
（4）调用ReentrantLock的newCondition方法正是返回的ConditionObject对象：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLbg5NdKWu6VFYehjgianmhfUEC1ImeX7AZ259cqicxqUZzfUtWOocX9Pw/0?wx_fmt=png)
**4、Node内部类：**
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLE3VJtxyFc8hIhhejEeO0eLOAPdTw6TIcn8Ta7LI0SRWUJmSTmIkYTw/0?wx_fmt=jpeg)
同步器依赖内部的**同步队列**（一个FIFO双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。
（1）同步队列的基本结构
同步队列中的节点（Node）用来保存获取同步状态失败的线程引用、等待状态以及前驱和后继节点，节点的属性类型与名称以及描述如下：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLLuNvibvvAdRMr38icbnYHIx8XcbnbbaqQcCRdW5ltrM5Z9JygibARHIrw/0?wx_fmt=jpeg)
节点是构成同步队列（等待队列，在5.6节中将会介绍）的基础，同步器拥有**首节点**（head）和**尾节点**（tail），没有成功获取同步状态的线程将会成为节点加入该队列的**尾部**，同步队列的基本结构如下图：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLSPOrZxxwQXykgVXIV5xAUXvnd3eUNPVd8rojRTZ9W2icRR9TibJ0cc8g/0?wx_fmt=jpeg)
（2）由于同一时刻只有一个线程能够获取到同步锁，但可以有多个线程进入阻塞，也就是说将需要等待的线程Node插入到尾部是需要进行同步操作的，使用的方法是：compareAndSetTail(Node expect, Node update) ，只有设置成功后，当前节点才正式与之前的节点建立关联。
关于Node节点的细节还有很多，最重要的是我们理解他就是实现的是队列同步的存储功能就行，这个存储功能在尾部存放的是需要排队等待的线程，在头部获取的是获取到锁的线程信息，其他的内容不再进行学习，有兴趣的可以参考其他文章或书籍研究。
**三、ReentrantLock的设计与实现**
ReentrantLock的类图结构如下：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLfkswAvse3xkO5zLBXun4r3ic3TQrzurXngP6n4dNU8BibyNXCMoWEOYA/0?wx_fmt=png)
可以看出ReentrantLock的内部类包含：Sync、NonfairSync（非公平锁）、FairSync（公平锁）。而Sync正是继承了AbstractQueuedSynchronizer这个抽象类，而NonfairSync和FairSync又是继承了Sync的两个静态内部类。
因为我们在上述的学习中已经知道了AbstractQueuedSynchronizer**同步器面向的是锁的实现者**，即其内部已经封装了一些关于锁的操作。这也是上文中提到的两句话：（1）同步器的主要使用方式是**继承**，子类通过继承**同步器**并重写指定方法，随后将同步器组合在自定义组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法；（2）**子类推荐被定义为自定义同步组件的静态内部类**，同步器自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用。
**1、Sync内部类**
在这里Sync是AQS的子类，这个时候我们应该想到上述提到的使用protected 修饰的5个方法，这也是Sync这个子类需要重写的，Sync内部类图如下：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_jpg/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlL2meT7f2u3o0WibQ1z2IQQAgsib4u3EuGmqlJDTibJR2Qus5nYaLr8OGqQ/0?wx_fmt=jpeg)
tryRelease（）方法的作用已经在上边解释了，这里不再赘述！
可以看出对于我们上述说的那5个方法，Sync只重写了一个：tryRelease（），那么其他的几个方法那？
这里需要注意的是：Sync也是一个abstract类，并且这5个方法并不是一定要在子类中进行重写的，ReentrantLock的几个内部类只重写了tryRelease和tryAcquire方法，其他的使用是在ReentrantReadWriteLock中用到的，这也是根据具体的ReentrantLock的实现的实际需求，而其他的方法具体（其实在ReentrantLock就是指tryAcquire）的重写这就需要：NonfairSync和FairSync上场了！
**2、NonfairSync和FairSync内部类**
NonfairSync和FairSync实现差不多，这里只学习FairSync。
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLseicsibyqrVwIduDEjdr87ATIyWXcs4ibEBgxREeGO31qZzlJict7NXrsg/0?wx_fmt=png)
FairSync 实现了Sync的抽象方法lock（），而具体的tryAcquire（）方法即是重写AQS中的tryAcquire（）方法，这里的lock（）方法调用了AQS提供的acquire（） 方法，AQS中acquire方法如下：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLaI5OnMdFfg5J8qc5wBfyJsJK45MRNI4TreX5eywHusd26XhPtl3qQw/0?wx_fmt=png)
AQS中的acquire（）方法调用了AQS中的tryAcquire（）方法，但tryAcquire（）上述说的他是一个空实现，直接抛出的异常，而最终是由FairSync 重写了，所以此时执行的时候，真正调用的就是FairSync 中的tryAcquire（）方法。而我们在使用ReentrantLock的lock或者unlock方法的时候，实际上调用的就是ReentrantLock实现的Lock的接口，而这个接口的实现内部又是调用的Sync里的抽象方法lock（）。
至此，整个的结构大致理了一遍，虽然还有很多细节没有探讨过。
如果，我们对上述的继承关系什么的还不是很懂的话，以及对AQS是如何实现锁的还不了解的话，我们倒不如使用AQS自己设计一个锁，类似ReentrantLock，或者说是ReentrantLock的精简版。
**四、使用AQS自己实现一个锁**
在上边的学习中，我们知道要是实现一个自定义的Lock实现类，首先要实现Lock接口，并且定义一个内部类继承AQS类，重写他的方法，示例如下：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLAUwUiasPHgncibricY4dKZTgJUicCQmEGdib8BTg51nVZspOTEsedhtIcRQ/0?wx_fmt=png)
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLNC0mpOynhZJQhvtFoSXyGMnBIRJBHll4RicialIicIz6QRSyibDicTkd5Pw/0?wx_fmt=png)
测试用例：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLypXriaVn5NuKS4NIqPp15EPpZ8dWKT4VVxicroJ7UvmwfWCEbBQmEp8w/0?wx_fmt=png)
执行结果：
![图片](https://ss.csdn.net/p?http://mmbiz.qpic.cn/mmbiz_png/UtWdDgynLdbWamXHCvfLQrJFlRiatYGlLVkQxnowGT7B0OY3bxiaDXG3TwGZ12SBQc65Kyrib7hxJibZCfEDibocv7A/0?wx_fmt=png)
上述代码，一个重要的却别就是没有ReentrantLock中的NonfairSync和FairSync，那么假设我们添加一个公平锁的话，想起来还是很简答的，直接参考ReentrantLock即可，这里不再赘述。

