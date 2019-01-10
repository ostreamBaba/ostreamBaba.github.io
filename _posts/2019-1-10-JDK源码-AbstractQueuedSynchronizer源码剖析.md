---
layout:     post
title:      AbstractQueuedSynchronizer源码分析
subtitle:   关于AQS的源码分析
date:       2019-1-10
author:     ostreamBaba
header-img: img/190110.jpg
catalog: true
tags:
    - JDK源码
---
## AbstractQueuedSynchronizer源码分析

***
**AbstractQueuedSynchronizer，简称AQS，我们着重关注AQS的两个函数：**
- acquire
- release
### 1.acquire()
##### 1.1 addWaiter函数
```java
private AbstractQueuedSynchronizer.Node addWaiter(AbstractQueuedSynchronizer.Node mode) {
    /*
      创建一个Node结点
      Node的Thread为Thread.currentThread
      Node的nextWaiter为独占模式的结点(Node.EXCLUSIVE)
   	*/
    AbstractQueuedSynchronizer.Node node = new AbstractQueuedSynchronizer.Node(Thread.currentThread(), mode );
    //将尾节点赋值给pred
    AbstractQueuedSynchronizer.Node pred = tail;
    //若尾节点不为null 说明Sync队列已经初始化 
    if (pred != null) {
    	//新建节点的prev指针指向pred节点(tail)
        node.prev = pred;
        //通过cas设置node节点为新的尾节点
        if (compareAndSetTail( pred, node )) {
            //cas设置成功 将旧的尾节点指向node(新的tail)
            pred.next = node;
            return node;
        }
    }
    //若失败，则进入enq函数
    //失败原因:有其他线程抢占到了尾节点或者Sync队列还未初始化成功
    enq( node );
    return node;
}
```
图为addWaiter将当前线程封装成Node节点加入Sync队列成功后的Sync队列。
![成功加入sync队列](https://img-blog.csdnimg.cn/20190110011444413.png)
若加入失败，会进入enq(node)函数，我们来看一下该函数的源码:
```java
private AbstractQueuedSynchronizer.Node enq(final AbstractQueuedSynchronizer.Node node) {
    for (;;) {
        AbstractQueuedSynchronizer.Node t = tail;
        //还是先判断Sync队列是否初始化
        if (t == null) { // Must initialize
        	//通过cas设置头节点成功
            if (compareAndSetHead(new AbstractQueuedSynchronizer.Node()))
            	//若成功 则将tail指针也指向头结点
                tail = head;
        } else {
        	//否则 通过cas自旋的方式设置尾结点，直至成功返回
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
enq的函数的作用: 若Sync还未初始化，则用cas方式尝试初始化，否则，通过CAS自旋的方式尝试设置该结点为Sync队列的尾结点。
这里我们用图来表示:
- 首先: 若Sync队列尚未初始化，那么先进行一次初始化，如图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190110173104604.png#pic_center =400x200)
- 初始化完成，CAS自旋的方式尝试设置该结点为Sync队列的尾结点。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190110174602771.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1Zpc2N1,size_16,color_FFFFFF,t_70#pic_center)

<font color="red">注意：我们的所有插入Sync操作都要用cas的方式进行，因为有可能有多个线程在互相竞争。</font>

分析完上面两个函数，addWaiter的作用就是**将当前线程封装成Node节点加入Sync队列**。
整个流程我们梳理一下:
>使用当前线程构造Node,对于一个节点我们需要做的就是将当前节点前驱节点指向尾节点(cur.pre=tail), 尾节点指向它(tail=cur),原来的尾节点的后继节点指向Node.

具体过程:
- 先尝试在队尾快速添加(双向链表保证该操作是O(1)复杂度):
    - 假设Sync queue已经初始化(尾节点不为空)
    - 分配引用pred指向tail节点.
    - 将新建节点的前驱指向尾节点.(cur.pre=pred(old tail))
    - 假设cas操作成功, 那么新建节点就变成了新的尾结点.(tail=cur) 此操作一定要是原子操作.因为有可能有多个线程要更新尾节点.
    - 将旧的尾节点的后继指针指向新的尾巴节.(pred.next=cur(new tail))
- 若尾节点添加失败(cas失败,说明被其他线程更新了)或者该节点是sync队列的第一个入队节点,那么就进行enq函数.这个过程是无限循环的,所以确保了新建的Node一定可以加入Sync Queue中.
##### 1.2.acquireQueued函数
```java
//Sync队列的结点在独占模式且忽略中断的模式下尝试获取资源
final boolean acquireQueued(final AbstractQueuedSynchronizer.Node node, int arg) {
	//标记是否成功获取资源
    boolean failed = true;
    try {
    	//标记当前线程是否被中断
        boolean interrupted = false;
        for (;;) {
        	//获取当前结点的前驱结点
            final AbstractQueuedSynchronizer.Node p = node.predecessor();
            //如果前驱结点是头结点，那么当前结点就是老二结点，那便有资格尝试获取资源
            if (p == head && tryAcquire(arg)) {
            	//若获取资源成功 说明head结点已经释放资源
            	//将当前结点设置为新的头结点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                //返回等待获取资源的过程中是否被中断
                return interrupted;
            }
            //若失败，那么说明自己可以休息了，就进入waiting状态，直到被unpark()
            if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                //如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
                interrupted = true;
        }
    } finally {
    	//若在获取资源的过程中发生异常 将进入这里
    	//若发生异常间并没有获取资源成功 那么将取消获取资源
        if (failed)
            cancelAcquire(node);
    }
}
```
acquireQueued函数的作用:首先会获取当前节点的前驱节点，如果**前驱节点是头结点并且能够成功获取资源**， 设置当前结点为头结点，返回。否则，调用shouldParkAfterFailedAcquire和parkAndCheckInterrupt函数。
> 逻辑流程:

 1. 获取当前结点的前驱结点
 2. 判断前驱结点是否是头结点: 头结点的含义就是当前占有锁且正在运行。若前驱结点是头结点且尝试获取资源成功，设置当前结点为头结点。
 3. 否则的话，进入等待状态。如果没有轮到当前节点运行，那么将当前线程从调度器上摘下，也就是进入等待状态。
如图:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190110175734723.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1Zpc2N1,size_16,color_FFFFFF,t_70#pic_center =800x600)

若前驱结点不是头结点或者获取资源失败，那么会进入等待状态，会调用如下函数:
**shouldParkAfterFailedAcquire函数**
```java
private static boolean shouldParkAfterFailedAcquire(AbstractQueuedSynchronizer.Node pred, AbstractQueuedSynchronizer.Node node) {
	//获取前驱结点的状态
    int ws = pred.waitStatus;
    //如果前驱结点的状态为signal的话 说明前驱结点正在等待被唤醒 那么当前结点就可以安心的进入waiting状态
    if (ws == AbstractQueuedSynchronizer.Node.SIGNAL)
        return true;
    if (ws > 0) {
      	//如果大于0 说明前驱结点为cancelled被取消，那么就一直往前找，直至找到一个正常状态的结点，并且排在它后面.
        do {
            node.prev = pred = pred.prev;
        //直至找到前驱结点的状态不为cancelled为止
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
    	//为PROPAGATE-3或者是0表示无状态,(为CONDITION-2时，表示此节点在condition queue中)
        //如果前驱正常,那么就把前驱节点设置为signal,告诉他释放锁的时候通知一下当前节点。
        //设置前驱结点为SINGAL状态
        compareAndSetWaitStatus(pred, ws, AbstractQueuedSynchronizer.Node.SIGNAL);
    }
    return false;
}
```
shouldParkAfterFailedAcquire函数作用: 只有当前节点的前驱节点为SINGAL时，才可以对该节点所封装的线程进行park操作，否则，不能进行park操作(这里的park操作就是将线程挂起，底层实现是Unsafe类的park操作)。
***
这里我们延续上面的步骤:
假设node的前驱结点为head结点，可是获取资源失败:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190110180752608.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1Zpc2N1,size_16,color_FFFFFF,t_70#pic_center)
此时会进入shouldParkAfterFailedAcquire函数: 由于head结点的waitStatus为0，设置头结点的状态为signal，即
-1，然后返回false，再次进入acquireQueued的循环。
假设再次获取资源失败，那么由于head的状态为-1，那么直接返回true，node结点的线程被park，则进入waiting状态。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190110181515548.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1Zpc2N1,size_16,color_FFFFFF,t_70#pic_center)
以上图示就是整个acquireQueued的流程，直至获取资源成功，或者发生异常。
***
**parkAndCheckInterrupt函数:**
```java
private final boolean parkAndCheckInterrupt() {
    //将当前线程进行park操作
    LockSupport.park(this);
    //返回线程是否被中断过 并重置状态
    return Thread.interrupted();
}
```
parkAndCheckInterrupt函数作用: 首先执行park操作，即禁用当前线程，然后返回该线程是否已经被中断。
park()操作会让当前线程进入waiting状态，只有两种途径可以唤醒该线程：
- 被unpark 
- 被interrupt().

如果在获取资源的过程中发生异常，且没有成功获取到资源的话，会进入cancalAcquire()方法。
cancalAcquire方法会执行:

 1. 将node关联的线程断开
 2. 将node的waitStatus设置为CANCELLED

我们来看一下cancelAcquire这个函数的具体实现:
```java
//取消继续获取资源
private void cancelAcquire(AbstractQueuedSynchronizer.Node node) {
    if (node == null)
        return;
   	//将当前结点的thread设置为null
   	node.thread = null;
    AbstractQueuedSynchronizer.Node pred = node.prev;
    //找到node前驱节点中第一个状态不为cancelled的结点
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    AbstractQueuedSynchronizer.Node predNext = pred.next;
	//设置node结点的ws为cancelled
    node.waitStatus = AbstractQueuedSynchronizer.Node.CANCELLED;
    //如果node结点为尾结点并且cas设置新的尾结点为pred
    if (node == tail && compareAndSetTail(node, pred)) {
        //若成功，cas设置pred的的下一个结点为null
        compareAndSetNext(pred, predNext, null);
    } else {
     	//若失败
        int ws;
        //如果node结点不是tail结点，也不是head的后继结点
        //则将node的前继结点的ws设置为signal
        //并将pred的next结点指向node的下一个结点
        if (pred != head &&
                ((ws = pred.waitStatus) == AbstractQueuedSynchronizer.Node.SIGNAL ||
                        (ws <= 0 && compareAndSetWaitStatus(pred, ws, AbstractQueuedSynchronizer.Node.SIGNAL))) &&
                pred.thread != null) {
            //获取node的下一个结点
            AbstractQueuedSynchronizer.Node next = node.next;
            //设置pred的下一个结点为next
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
        	//否则唤醒该节点的后置结点
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
```
我们分析一下cancelAcquire的三种情况:
- 当node是tail结点的时候，若node出队成功，如下。由于没有任何引用指向node，则node将会被GC回收。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190110152246782.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1Zpc2N1,size_16,color_FFFFFF,t_70#pic_center =500x400)
- 当node节点不是tail结点，且node的前驱结点不是head的结点的时候，如下: 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190110154341937.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1Zpc2N1,size_16,color_FFFFFF,t_70#pic_center =500x400)
<font color="red">注意:此时node结点并没有真正的出队被gc回收，它还有node.next的前驱指针指着，导致它还处于GC roots可达状态。</font>
那么node结点什么时候处于真正意义上的出队呢，其实我们想想我们什么时候才会清除这些状态为CANCELLED的结点，其实shouldParkAfterFailedAcquire和cancelAcquire都有处理这些waitStatus为CANCELLED的结点的步骤。例如:
```java
//shouldParkAfterFailedAcquire
do {
	node.prev = pred = pred.prev;
} while (pred.waitStatus > 0);
//cancelAcquire
while (pred.waitStatus > 0)
    node.prev = pred = pred.prev;
```
这两个函数处理waitStatus为CANCELLED的地方，仅仅需要断开与前继结点状态为cancalled的联系，那么相关waitStatus为cancelled的结点就真正意义上的出队(已没有任何引用指向它)了。
- 当node的前驱结点是头结点的话，其实这里还有一大堆判断。
```java
pred != head && ((ws = pred.waitStatus) == AbstractQueuedSynchronizer.Node.SIGNAL 
|| (ws <= 0 && compareAndSetWaitStatus(pred, ws, AbstractQueuedSynchronizer.Node.SIGNAL))
 pred.thread != null)那些
```
我觉得这一堆判断不成立的话，要么就是node的前驱结点是头结点，要么就是node的前驱结点也处于cancelled状态，那既然处于cancelled状态，自然有必要把资源让给后继的结点，因为cancelled意味着我已经不会参与资源的争抢了，那么我就唤醒后面状态正常的结点来争夺资源。这里node的前驱结点其实也是如此，既然pred现在已经是node结点了，那么说明它持有锁且正在运行，那么就有必要唤醒后面的结点。

我们来看一下unparkSuccessor函数:
```java
private void unparkSuccessor(AbstractQueuedSynchronizer.Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
    	//将状态设置为0
        compareAndSetWaitStatus(node, ws, 0);
    AbstractQueuedSynchronizer.Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        //从后往前查找第一个最前一个状态正常的结点
        for (AbstractQueuedSynchronizer.Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //对其进行唤醒
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
其实unparkSuccessor函数作用就是唤醒node之后第一个状态正常的结点。

以上就是cancelAcquire的全部内容，我们来整理一下流程:
 1. 断开node与其内部Thread的联系
 2. 如果node结点是尾结点，那么直接设置让node出队。
 3. 如果node的前驱结点不是头节点且node不是尾结点，那么就将node的前驱结点的后继指针指向node的后继结点。
 4. 如果node的前驱结点是头结点或者处于cancelled状态，那么就唤醒后面的结点(该结点并不是node结点，而是node之后第一个状态正常的结点)。
***
至此，acquireQueued的整个流程已经讲完了。我们来梳理一下整个aquireQueued的流程:
 1. 判断结点的前驱是否为head并且是否成功获取资源。
 2. 若1均满足，则设置节点为头结点，之后会判断finally块并返回。
 3. 若1不满足，则判断是否需要park当前线程，是否需要park当前线程的逻辑是判断结点的前驱结点是状态是否为 signal。若是，则park当前结点。否则，不进行park操作。
 4. 若park当前线程，之后某个线程对本线程unpark后，并且本线程也获得运行机会。那么，将会继续进行步骤1判断。
***
### 2.release
```java
public final boolean release(int arg) {
	//若尝试是否资源成功
    if (tryRelease(arg)) {
        AbstractQueuedSynchronizer.Node h = head;
        //若头结点不为空并且头结点的状态不为0
        if (h != null && h.waitStatus != 0)
        	//唤醒head后面的结点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
release做的事很简单，就是唤醒head结点后面的结点，而unparkSuccessor上面已经讲过了，这里就不再分析了。
这里我们画一个图吧，其实还是和上面的图有联系的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190110183222910.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1Zpc2N1,size_16,color_FFFFFF,t_70# =x900)
以上，就是线程获取锁的整个流程。
