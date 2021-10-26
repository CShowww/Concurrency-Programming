# 一、概括
ReentrantLock,即支持可重入的锁，塔表示该锁能够支持一个县城对资源的重复加锁。它是借助CAS和AQS实现的。

# 二、实现
1. 如何实现重进入
- 线程再次获取锁。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再次获得锁。
- 锁的最终释放。线程重复n次获取了锁，随后在第n次释放锁后，其他线程能获取到该锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而锁被释放时，计数自减，当计数等于0时表示已经成功释放。

2. 以非公平锁为例  
该方法增加了再次获取同步状态的处理逻辑：通过判断当前线程是否为获取锁的线程来决定操作是否成功，如果是获取锁的县城再次请求，则将同步状态值进行增加并返回true，表示获取同步状态成功。  
当释放锁时，如果该锁被获取了n次，那么前n-1次tryRelease(int releases)方法必须返回false,而只有同步状态完全释放了，才能返回true。该方法将同步状态是否为0作为最终的释放条件，当同步状态为0时，将占有线程设为null，并返回true，表示释放成功。

3. 公平与非公平  
公平性是针对获取锁而言的。如果一个锁是公平的，那么锁的获取顺序就应该符合请求的绝对时间顺序，即FIFO。
4. 为什么要默认非公平锁呢？
当一个线程请求锁时，只要获取了同步状态即成功获取锁。在这个前提下，刚释放的线程再次获取同步状态的概率更大，因此非公平锁切换次数更少，节省资源。公平锁为了保证锁的获取按照FIFO的原则，进行大量的线程切换，浪费了资源。


# 和Synchronized锁的区别
- 底层实现上来说，synchronized 是JVM层面的锁，是Java关键字，通过monitor对象来完成（monitorenter与monitorexit），对象只有在同步块或同步方法中才能调用wait/notify方法，ReentrantLock是从jdk1.5以来（java.util.concurrent.locks.Lock）提供的API层面的锁。  
synchronized的实现涉及到锁的升级，具体为无锁、偏向锁、自旋锁、向OS申请重量级锁，ReentrantLock实现则是通过利用CAS（CompareAndSwap）自旋机制保证线程操作的原子性和volatile保证数据可见性以实现锁的功能。
- synchronized不需要用户去手动释放锁，synchronized 代码执行完后系统会自动让线程释放对锁的占用； ReentrantLock则需要用户去手动释放锁，如果没有手动释放锁，就可能导致死锁现象。一般通过lock()和unlock()方法配合try/finally语句块来完成，使用释放更加灵活。
- synchronized是不可中断类型的锁，除非加锁的代码中出现异常或正常执行完成；ReentrantLock则可以中断，可通过trylock(longtimeout,TimeUnitunit)设置超时方法或者将lockInterruptibly()放到代码块中，调用interrupt方法进行中断。
```
public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
```

- synchronized为非公平锁,ReentrantLock则即可以选公平锁也可以选非公平锁，通过构造方法new ReentrantLock时传入boolean值进行选择，为空默认false非公平锁，true为公平锁。
- ReentrantLock通过绑定Condition结合await()/singal()方法实现线程的精确唤醒，而不是像synchronized通过Object类的wait()/notify()/notifyAll()方法要么随机唤醒一个线程要么唤醒全部线程。