
  CopyOnWriteArrayList可以理解为一个线程安全的ArrayList集合。它的继承体系如下：
![图片](https://images-cdn.shimo.im/HVEhK0LXvQEzBL7l/image.image/png!thumbnail)
对比于ArrayList的构造方法，真的很相似！
![图片](https://images-cdn.shimo.im/PCIlgHqqqyAl7faq/image.image/png!thumbnail)

![图片](https://images-cdn.shimo.im/SrReVG9nNHozDGPR/image.image/png!thumbnail)

![图片](https://images-cdn.shimo.im/5YszMBl1cfgzv830/image.image/png!thumbnail)
也就是一个数组的转化过程。
由第一个图很清楚看出来，它和ArrayLsit一样，实现了List接口。它能做到线程安全取决与它的两个特殊的成员变量。下面就详解下这两个成员变量。

![图片](https://images-cdn.shimo.im/M42W13JVlF4vG5YX/image.image/png!thumbnail)

![图片](https://images-cdn.shimo.im/Ekj62njBBqo1KGMU/image.image/png!thumbnail)
翻译：锁保护所有的突变器。（我们对ReentrantLock的内置监视器有轻微的偏好，而这两种方法都可以。）
CopyOnWriteArrayList内部也是一个对象数组，和ArrayList不同的是它还用volatile修饰，除此之外，它还自带了个锁。
首先ReentrantLock排它锁保证每一次只能有一个线程对其内部的数据进行访问，这首先保证了互斥，接着某个线程对其进行了写操作之后，由于在java内存模型中，每个线程都有自己独立的内存空间，也叫线程缓存。而所有线程共享访问的是主存区域的变量、资源。所以当某个线程对某个资源进行了写操作之后，如果没有及时更新到主存中去，那么其他线程访问的这个资源则就是已过期的，无效的。这个时候关键字volatile通过在底层加入内存屏障，防止指令重排序，来保证每个线程在对该数组进行操作时，总能看到其它线程对该volatile变量最后的写入，也就是可见性。
这样最终就做到了线程之间的同步以及资源的安全。
![图片](https://images-cdn.shimo.im/wd62A6c3fjI6tnJ4/image.image/png!thumbnail)

![图片](https://images-cdn.shimo.im/CgItkao9Ab0dmPWo/image.image/png!thumbnail)
通过源码我们可以将这个过程分为以下几个步骤：
前提：给整个写操作上了synchronized锁！！！！
1：先获取原来的集合对象。
2：获取集合的长度。
3：创建一个新的数组，其长度为元素组+1；
4：将新增的元素放在数组末端。
5：将引用指向新数组。
以上的过程用两幅图片来演示就是这样的：
![图片](https://images-cdn.shimo.im/qKzHTfXO6yMj3bsh/image.image/png!thumbnail)

![图片](https://images-cdn.shimo.im/LQHW2GMvvZw5uBdm/image.image/png!thumbnail)
问题：现在我们直到了它的写操作是加锁的了，那么读操作呢？
当有线程并发的读操作的时候，可以分为三种情况：
1：读的时候，新数组还没有写入。此时会直接读取原数组的内容。
2：读的时候，新数组已经写入了，但是引用还是指向的原数组，所以读取的还是原数组。
3：读的时候，新数组已经写入了并且引用也指向了新数组。这个时候读取的是新数组。
综上所述，读操作是不用加锁的。

优点：读操作效率很高。
缺点：写操作效率很低，因为要新建一个数组再复制移动。
适用场景：适合数据量少，读操作多写操作少的场景。

![图片](https://images-cdn.shimo.im/8MMwrwgpk7Uf0MoY/image.image/png!thumbnail)
这是CopyOnArraySet的源码，结果发现它其实就是CopyOnWriteArrayList实现的，所以CopyOnWriteArrayList的特点它都有。要说不同点就是List和Set的一大不同点，就是不能包括重复值。
以下是它如何实现去重复的源码：
![图片](https://images-cdn.shimo.im/7Jciq1fSijYE5QiL/image.image/png!thumbnail)

![图片](https://images-cdn.shimo.im/BxwrbGTy8sYF76mb/image.image/png!thumbnail)
也就说说，如果没有重复的，就添加进集合，如果有就值为false。

![图片](https://images-cdn.shimo.im/wNtddAXjyBIOL7H6/image.image/png!thumbnail)
和hashMap的几个重要参数一样。
![图片](https://images-cdn.shimo.im/pnYk63IzhKInCvXf/image.image/png!thumbnail)
从这个构造方法可以看出来，创建的时候其实也是没进行初始化的，用的时候才会去检查。
这里有一个HashMap没有的变量：concurrencyLevel，并发级别。参考博客[https://blog.csdn.net/hitxueliang/article/details/24734861](https://blog.csdn.net/hitxueliang/article/details/24734861)

![图片](https://images-cdn.shimo.im/t9nLAw6iO0YtfeZh/image.image/png!thumbnail)

![图片](https://images-cdn.shimo.im/yFXA9P5Vbcccr0xG/image.image/png!thumbnail)
这是hashmap的数据结构，数组加链表。在hashmap中，为了实现线程安全，使用了synchnoized锁，而且锁住的是整个table数组。这样的锁太重了，A线程在使用put方法的时候，其他线程只能被阻塞等待，甚至连调用get方法获取元素都不行，效率很地下，现在也几乎淘汰了。
而ConcurrentHashMap和hashmap不一样，它使用的是锁分段的技术。
通俗的说，hashMap是给整体加了个锁，所有的数据都由一把锁锁着，所以在高并发的时候，所有线程争抢一份资源。而ConcurrentHashMap将数据分块存储，每一块加一个锁，当多线程访问的时候，其他的资源仍可以被访问。
ConcurrentHashMap引入了segments数据结构的概念，其实segments中包含的就是多个Entry数组，segments就扮演者hashmap中synchonized的锁的作用。

![图片](https://images-cdn.shimo.im/S2QJ6Zdzz8cRONMC/image.image/png!thumbnail)

![图片](https://images-cdn.shimo.im/OEnnXq9M8cgiSE9b/image.image/png!thumbnail)
可以发现，val是用volatile修饰的，其写操作先于读操作，也就是说每次get所获得值都是可见的。
get的过程就和hashmap是一个道理，无非就是在hashmap的基础上多加了一层对segments数组的查找。（两次哈希）。

put方法因为是写操作，所以一定要加锁了。不同于hashtable，ConcurrentHashMap的锁是加在segment数组上的。多个segment数组之间不受影响，这也是为什么ConcurrentHashMap效率高的原因。
比较之前hashmap的put方法，同样的也是两个步骤，第一步先判断是否需要扩容，第二步添加数据。同样将这两个方法与hashmap的这两步骤相比，也有不一样。
hashmap扩容的时候，是在插入元素之后再判断是否需已经达到容量，如果达到了就扩容。但是着就会出现扩容了之后但是没有插入元素的情况，也就是无效扩容了。看源码：
![图片](https://images-cdn.shimo.im/HL1bYMOhHGQ6HpGw/image.image/png!thumbnail)
而ConcurrentHashMap的扩容是将这一个segment的容量扩大为原来的的两倍。网上的都是JDK1.8的，我看的是JDK1.9，在1.9中，ConcurrentHashMap源代码1085行中，扩容的方法传入了一个叫bincount的参数，往上继续寻找在

