# 一、概述  
- AQS是用来构建锁和其他同步组件的基础框架。AQS内部有一个核心变量叫state(volatile类型),代表加锁的状态。另外还有一个变量来记录当前加锁的是哪个线程。此外，AQS内部还有一个FIFO的线程等待队列，被阻塞的线程会依次插入道等待队列的尾部。
![image](https://note.youdao.com/yws/res/3601/735D0FBB035C4A1791CF9F944FE53163)

- state有三种访问方式：getState(),setState(),compareAndSetState()

- 自定义的同步器在实现时只需要实现共享资源state的获取与释放就行了，其他的AQS已经在底层实现好了。
- 自定义同步器主要实现：tryAcquire(int),tryRelease(int),tryAcquireShared(int),tryReleaseShared(int)和isHeldExclusively()这五个函数。
- 以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。
- 再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

# 二、解析
1. 在等待队列中，每一个节点都是对一个等待获取资源的线程的封装，其中包含了需要同步线程本身和它的等待状态（阻塞、是否等待唤醒、是否被取消）。变量waitStatus表示当前节点的等待状态，负值表示结点处于有效等待状态，而正值表示结点已被取消。所以源码中很多地方用>0来判断结点的状态是否正常。
- CANCELLED(1):表示当前节点已经取消调度。当timeout或中断时，会触发变更为此状态，进入该状态后节点将不再变化。
- SIGNAL(-1):表示后继节点在等待当前节点唤醒。后继节点入队时，会将前继节点的状态更新为SIGNAL。
- CONDITION(2):表示节点等待在Condition桑，当其他线程调用了Condition的signal()方法后，CONDITION状态的节点将从等待队列转移到同步队列中，等待获取同步锁。
- PROPAGATE(-3)：共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。
- 0:新节点入队时的默认状态。

2. acquire(int)  
此方法是独占模式下线程获取公共资源的顶层入口。如果获取到资源，线程直接返回，否则进入等待队列，直到获取资源为止，这也是lock()的语义。

```
 public final void acquire(int arg) {
     if (!tryAcquire(arg) &&
         acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
         selfInterrupt();
 }
```
- tryAcquire()尝试直接去获取资源，如果成功则直接返回。
- addWaiter()将该线程加入等待队列的尾部，并标记为独占模式
- acquireQueued()使线程阻塞在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
- 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。


2.1 tryAcquire(int)  
此方法尝试去获取独占资源。如果获取成功，则直接返回true，否则直接返回false。
```
1     protected boolean tryAcquire(int arg) {
2         throw new UnsupportedOperationException();
3     }
```
AQS这里只定义了一个接口，具体资源的获取交由自定义同步器去实现了（通过state的get/set/CAS）！！！

2.2 addWaiter(Node)  
此方法用于将当前线程加入到等待队列的队尾，并返回当前线程所在的结点。
```
private Node addWaiter(Node mode) {
    //以给定模式构造结点。mode有两种：EXCLUSIVE（独占）和SHARED（共享）
    Node node = new Node(Thread.currentThread(), mode);

    //尝试快速方式直接放到队尾。
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }

    //上一步失败则通过enq入队。
    enq(node);
    return node;
}

private Node enq(final Node node) {
    //CAS"自旋"，直到成功加入队尾
    for (;;) {
        Node t = tail;
        if (t == null) { // 队列为空，创建一个空的标志结点作为head结点，并将tail也指向它。
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {//正常流程，放入队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
在enq方法中，同步器通过死循环来保证节点的正确添加，在死循环中只有通过CAS将节点设置为尾节点后，当前线程才能返回，否则当前线程会不断尝试设置。enq方法将并发添加节点的请求通过CAS变得串型化了。  
2.3 acquireQueued(Node, int)  
OK，通过tryAcquire()和addWaiter()，该线程获取资源失败，已经被放入等待队列尾部了。线程进入等待状态休息，直到其他线程彻底释放资源后唤醒自己，自己再拿到资源，然后就可以去干自己想干的事了。
```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;//标记是否成功拿到资源
    try {
        boolean interrupted = false;//标记等待过程中是否被中断过

        //又是一个“自旋”！
        for (;;) {
            final Node p = node.predecessor();//拿到前驱
            //如果前驱是head，即该结点已成老二，那么便有资格去尝试获取资源（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）。
            if (p == head && tryAcquire(arg)) {
                setHead(node);//拿到资源后，将head指向该结点。所以head所指的标杆结点，就是当前获取到资源的那个结点或null。
                p.next = null; // setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了！
                failed = false; // 成功获取资源
                return interrupted;//返回等待过程中是否被中断过
            }

            //如果自己可以休息了，就通过park()进入waiting状态，直到被unpark()。如果不可中断的情况下被中断了，那么会从park()中醒过来，发现拿不到资源，从而继续进入park()等待。
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;//如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
        }
    } finally {
        if (failed) // 如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了），那么取消结点在队列中的等待。
            cancelAcquire(node);
    }
}
```
当前线程在死循环中尝试获取同步状态。而只有前驱节点石头节点才能尝试获取同步状态，因为头接电视成功获取到同步状态的节点，而头节点的线程释放了同步状态以后，将会唤醒其后继节点。后继节点被唤醒后需要检查自己的前驱节点是不是头节点。另外，这样做也符合同步队列的FIFO原则。
2.3.1 shouldParkAfterFailedAcquire(Node, Node)  
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;//拿到前驱的状态
    if (ws == Node.SIGNAL)
        //如果已经告诉前驱拿完号后通知自己一下，那就可以安心休息了
        return true;
    if (ws > 0) {
        /*
         * 如果前驱放弃了，那就一直往前找，直到找到最近一个正常等待的状态，并排在它的后边。
         * 注意：那些放弃的结点，由于被自己“加塞”到它们前边，它们相当于形成一个无引用链，稍后就会被保安大叔赶走了(GC回收)！
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
         //如果前驱正常，那就把前驱的状态设置成SIGNAL，告诉它拿完号后通知自己一下。有可能失败，人家说不定刚刚释放完呢！
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
此方法主要用来检查自己是否真的可以去休息了，如果前驱结点的状态不是SIGNAL，那么自己就不能安心去休息，需要去找个安心的休息点，同时可以再尝试下看有没有机会轮到自己拿号。
2.3.2 parkAndCheckInterrupt()
如果线程找好安全休息点后，就可以去休息了。此方法就是让线程去休息，进入等待状态。

2.4 总结acquireQueued()方法  
结点进入队尾后，检查状态，找到安全休息点；调用park()进入waiting状态，等待unpark()或interrupt()唤醒自己；被唤醒后，看自己是不是有资格能拿到号。如果拿到，head指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程1。

2.5  总结acquire()方法  
调用自定义同步器的tryAcquire()尝试直接去获取资源，如果成功则直接返回；  没成功，则addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；  
acquireQueued()使线程在等待队列中休息，有机会时（轮到自己，会被unpark()）会去尝试获取资源。获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。  
如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。  
![image](https://note.youdao.com/yws/res/3714/BFC41BE1A29A4370BBBF805CA4336DC5)


3.release(int)方法  
它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。这也正是unlock()的语义.
```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;//找到头结点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);//唤醒等待队列里的下一个线程
        return true;
    }
    return false;
}
```
3.1 tryRelease(int)  
正常来说，tryRelease()都会成功的，因为这是独占模式，该线程来释放资源，那么它肯定已经拿到独占资源了，直接减掉相应量的资源即可(state-=arg)，也不需要考虑线程安全的问题。但要注意它的返回值，上面已经提到了，release()是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！所以自义定同步器在实现时，如果已经彻底释放资源(state=0)，要返回true，否则返回false.  
3.2 unparkSuccessor(Node)
```
private void unparkSuccessor(Node node) {
    //这里，node一般为当前线程所在的结点。
    int ws = node.waitStatus;
    if (ws < 0)//置零当前线程所在的结点状态，允许失败。
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;//找到下一个需要唤醒的结点s
    if (s == null || s.waitStatus > 0) {//如果为空或已取消
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev) // 从后向前找。
            if (t.waitStatus <= 0)//从这里可以看出，<=0的结点，都是还有效的结点。
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);//唤醒
}
```
一句话概括：用unpark()唤醒等待队列中最前边的那个未放弃线程.

4.同步器  
同步器拥有首节点和尾节点，没有成功获取同步状态的节点会插入到队列的尾部。由于可能同时有多个线程插入到队列的尾部，因此需要保证线程安全。同步器提供了一个基于CAS操作的方法---compareAndSetTail(Node expect,Node update)来保证线程安全。  
首节点为获取到同步状态的线程。首节点的线程再释放同步状态时，将会唤醒后继节点，而后继节点会在获取同步状态成功时将自己设为首节点。由于只有一个线程能够成功获取到同步状态，因此设置头节点的方法不需要使用CAS来保障。  

5.总结  
在获取同步状态时，同步器维护一个同步队列，获取状态失败的线程会加入到这个同步队列中并在同步队列中自旋；移出队列的条件是前驱节点为头节点且成功获取了同步状态。在释放同步状态时，同步器调用tryRelease(int)方法释放同步状态，然后换行头节点的后继节点。



6.acquireShared(int) 
tryAcquireShared()依然需要自定义同步器去实现。AQS已经把其返回值的语义定义好了：负值代表获取失败；0代表获取成功，但没有剩余资源；正数表示获取成功，还有剩余资源，其他线程还可以去获取。
```
public final void acquireShared(int arg) {
     if (tryAcquireShared(arg) < 0)
         doAcquireShared(arg);
 }

```
tryAcquireShared()尝试获取资源，成功则直接返回；失败则通过doAcquireShared()进入等待队列，直到获取到资源为止才返回。

6.1 doAcquireShared(int)  
此方法用于将当前线程加入等待队列尾部休息，直到其他线程释放资源唤醒自己，自己成功拿到相应量的资源后才返回。
```
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);//加入队列尾部
    boolean failed = true;//是否成功标志
    try {
        boolean interrupted = false;//等待过程中是否被中断过的标志
        for (;;) {
            final Node p = node.predecessor();//前驱
            if (p == head) {//如果到head的下一个，因为head是拿到资源的线程，此时node被唤醒，很可能是head用完资源来唤醒自己的
                int r = tryAcquireShared(arg);//尝试获取资源
                if (r >= 0) {//成功
                    setHeadAndPropagate(node, r);//将head指向自己，还有剩余资源可以再唤醒之后的线程
                    p.next = null; // help GC
                    if (interrupted)//如果等待过程中被打断过，此时将中断补上。
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }

            //判断状态，寻找安全点，进入waiting状态，等着被unpark()或interrupt()
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
共享锁和独占锁很相似。跟独占模式比，还有一点需要注意的是，这里只有线程是head.next时（“老二”），才会去尝试获取资源，有剩余的话还会唤醒之后的队友。那么问题就来了，假如老大用完后释放了5个资源，而老二需要6个，老三需要1个，老四需要2个。老大先唤醒老二，老二一看资源不够，他是把资源让给老三呢，还是不让？答案是否定的！老二会继续park()等待其他线程释放资源，也更不会去唤醒老三和老四了。独占模式，同一时刻只有一个线程去执行，这样做未尝不可；但共享模式下，多个线程是可以同时执行的，现在因为老二的资源需求量大，而把后面量小的老三和老四也都卡住了。

6.2 setHeadAndPropagate(Node, int)  
```
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node);//head指向自己
     //如果还有剩余量，继续唤醒下一个邻居线程
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```
此方法在setHead()的基础上多了一步，就是自己苏醒的同时，如果条件符合（比如还有剩余资源），还会去唤醒后继结点，毕竟是共享模式！

6.3 tryAcquireShared(int)小结  
tryAcquireShared()尝试获取资源，成功则直接返回；失败则通过doAcquireShared()进入等待队列park()，直到被unpark()/interrupt()并成功获取到资源才返回。整个等待过程也是忽略中断的。  
其实跟acquire()的流程大同小异，只不过多了个自己拿到资源后，还会去唤醒后继队友的操作（这才是共享嘛）。  

6.4 releaseShared(int)  
```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {//尝试释放资源
        doReleaseShared();//唤醒后继结点
        return true;
    }
    return false;
}
```
一句话：释放掉资源后，唤醒后继。跟独占模式下的release()相似，但有一点稍微需要注意：独占模式下的tryRelease()在完全释放掉资源（state=0）后，才会返回true去唤醒其他线程，这主要是基于独占下可重入的考量；而共享模式下的releaseShared()则没有这种要求，共享模式实质就是控制一定量的线程并发执行，那么拥有资源的线程在释放掉部分资源时就可以唤醒后继等待结点。例如，资源总量是13，A（5）和B（7）分别获取到资源并发运行，C（4）来时只剩1个资源就需要等待。A在运行过程中释放掉2个资源量，然后tryReleaseShared(2)返回true唤醒C，C一看只有3个仍不够继续等待；随后B又释放2个，tryReleaseShared(2)返回true唤醒C，C一看有5个够自己用了，然后C就可以跟A和B一起运行。而ReentrantReadWriteLock读锁的tryReleaseShared()只有在完全释放掉资源（state=0）才返回true.

6.5 doReleaseShared()  
此方法主要用于唤醒后继。
```
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;
                unparkSuccessor(h);//唤醒后继
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        if (h == head)// head发生变化
            break;
    }
}
```