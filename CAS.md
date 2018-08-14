CAS
CAS是英文单词Compare and Swap的缩写，翻译过来就是比较并替换。
CAS机制中使用了3个基本操作数：内存地址V，旧的预期值A，要修改的新值B。
更新一个变量的时候，只有当变量的预期值A和内存地址V当中的实际值相同时，才会将内存地址V对应的值修改为B。
我们看一个例子：
1. 在内存地址V当中，存储着值为10的变量。

![图片](https://images-cdn.shimo.im/sn1xH9c9Ot0lDFfF/image.image/png!thumbnail)
2. 此时线程1想把变量的值增加1.对线程1来说，旧的预期值A=10，要修改的新值B=11

![图片](https://images-cdn.shimo.im/2jZCZRNO6JUSyuEn/image.image/png!thumbnail)
3. 在线程1要提交更新之前，另一个线程2抢先一步，把内存地址V中的变量值率先更新成了11。

![图片](https://images-cdn.shimo.im/DJxEwSQqq4gbLyzx/image.image/png!thumbnail)
4. 线程1开始提交更新，首先进行A和地址V的实际值比较，发现A不等于V的实际值，提交失败。

![图片](https://images-cdn.shimo.im/clcE7Ysm6MEehuUc/image.image/png!thumbnail)
5. 线程1 重新获取内存地址V的当前值，并重新计算想要修改的值。此时对线程1来说，A=11，B=12。这个重新尝试的过程被称为自旋。

![图片](https://images-cdn.shimo.im/yynwrx62oTIcTOz3/image.image/png!thumbnail)
6. 这一次比较幸运，没有其他线程改变地址V的值。线程1进行比较，发现A和地址V的实际值是相等的。

![图片](https://images-cdn.shimo.im/NX0WiDFcnMMrUaux/image.image/png!thumbnail)
7. 线程1进行交换，把地址V的值替换为B，也就是12.

![图片](https://images-cdn.shimo.im/wygp1W3Gl1UZH5bN/image.image/png!thumbnail)
从思想上来说，CAS属于乐观锁，乐观地认为程序中的并发情况不那么严重，所以让线程不断去重试更新。
在java中除了上面提到的Atomic系列类，以及Lock系列类夺得底层实现，甚至在JAVA1.6以上版本，synchronized转变为重量级锁之前，也会采用CAS机制。
**CAS的缺点：**
1） CPU开销过大
在并发量比较高的情况下，如果许多线程反复尝试更新某一个变量，却又一直更新不成功，循环往复，会给CPU带来很到的压力。
2） 不能保证代码块的原子性
CAS机制所保证的知识一个变量的原子性操作，而不能保证整个代码块的原子性。比如需要保证3个变量共同进行原子性的更新，就不得不使用synchronized了。
3） ABA问题
假设内存中有一个值为A的变量，存储在地址V中。
![图片](https://images-cdn.shimo.im/tC4cv4F31BQVCnLy/image.image/png!thumbnail)

此时有三个线程想使用CAS的方式更新这个变量的值，每个线程的执行时间有略微偏差。线程1和线程2已经获取当前值，线程3还未获取当前值。
![图片](https://images-cdn.shimo.im/IBf1qOH2jVgXzmVq/image.image/png!thumbnail)

接下来，线程1先一步执行成功，把当前值成功从A更新为B；同时线程2因为某种原因被阻塞住，没有做更新操作；线程3在线程1更新之后，获取了当前值B。
![图片](https://images-cdn.shimo.im/QUiBHKaHYzYWZE20/image.image/png!thumbnail)
在之后，线程2仍然处于阻塞状态，线程3继续执行，成功把当前值从B更新成了A。
![图片](https://images-cdn.shimo.im/2IVhfR3pmxwBeqgR/image.image/png!thumbnail)
最后，线程2终于恢复了运行状态，由于阻塞之前已经获得了“当前值A”，并且经过compare检测，内存地址V中的实际值也是A，所以成功把变量值A更新成了B。
![图片](https://images-cdn.shimo.im/8fPV9LZZLooBk3wI/image.image/png!thumbnail)

