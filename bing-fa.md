## **重入锁**

Java中的重入锁（即ReentrantLock）   与JVM内置锁\(即synchronized\)一样，是一种排它锁。

ReentrantLock提供了多样化的同步，比如有时间限制的同步（**定时锁**），可以被Interrupt的同步，即**中断锁**（synchronized的同步是不能Interrupt的）等。



在资源竞争不是很激烈的情况下，Synchronized的性能要优于ReetrantLock，

但是在**资源竞争很激烈**的情况下，Synchronized的性能会**下降几十倍**，但是ReetrantLock的性能能**维持常态**；

Atomic原子操作，不激烈情况下，性能比synchronized略逊，而激烈的时候，也能维持常态。

**激烈的时候，Atomic的性能会优于ReentrantLock一倍左右。**

但是其有一个缺点，就是只能同步一个值，一段代码中只能出现一个Atomic的变量，多于一个同步无效。因为他不能在多个Atomic之间同步。  
所以，我们写同步的时候，**优先考虑synchronized**，如果有特殊需要，再进一步优化。ReentrantLock和Atomic如果用的不好，不仅不能提高性能，还可能带来灾难。



synchronized是在JVM层面上实现的，在代码执行时出现异常，JVM会自动释放锁定。

但是使用Lock则不行，lock是通过代码实现的，要保证锁定一定会被释放，就必须将unLock\(\)放到finally{}中

```
try{
  renentrantLock.lock();
  // 用户操作
} finally {
  renentrantLock.unlock();
}
```

ReentrantLock获取锁定与三种方式：  
    a\)  lock\(\), 如果获取了锁立即返回，如果别的线程持有锁，当前线程则一直处于休眠状态，直到获取锁

    b\) tryLock\(\), 如果获取了锁立即返回true，如果别的线程正持有锁，立即返回false；

    c\) tryLock\(long timeout,[TimeUnit](http://houlinyan.iteye.com/java/util/concurrent/TimeUnit.html) unit\)，   如果获取了锁定立即返回true，如果别的线程正持有锁，会等待参数给定的时间，在等待的过程中，如果获取了锁定，就返回true，如果等待超时，返回false；

    d\) lockInterruptibly:如果获取了锁定立即返回，如果没有获取锁定，当前线程处于休眠状态，直到获得锁定，或者当前线程被别的线程中断





重入锁可定义为公平锁或非公平锁，默认实现为**非公平锁**。

**公平锁**是指多个线程获取锁被阻塞的情况下，锁变为可用时，**最先申请锁的线程获得锁**。

在并发环境中，每个线程在获取锁时会先查看此锁维护的**等待队列**，如果为空，或者当前线程线程是等待队列的第一个，就占有锁，否则就会加入到等待队列中，以后会**按照FIFO的规则**从队列中取到自己

可通过在重入锁（RenentrantLock）的构造方法中传入true构建公平锁，如Lock lock = new RenentrantLock\(**true**\)

  
**非公平锁**是指多个线程等待锁的情况下，锁变为可用状态时，**哪个线程获得锁是随机的**。

synchonized相当于非公平锁。可通过在重入锁的构造方法中传入false或者使用无参构造方法构建非公平锁。



**读写锁**  
锁可以保证原子性和可见性。

而原子性更多是针对**写操作**而言。对于读多写少的场景，一个读操作无须阻塞其它读操作，只需要保证读和写  或者 写与写  不同时发生即可。

此时，如果使用重入锁（即排它锁），对性能影响较大。Java中的读写锁（ReadWriteLock）就是为这种读多写少的场景而创造的。  
  
实际上，ReadWriteLock接口并非继承自Lock接口，ReentrantReadWriteLock也只实现了ReadWriteLock接口而未实现Lock接口。

ReadLock和WriteLock，是ReentrantReadWriteLock类的静态内部类，它们实现了Lock接口。  
  
  
一个ReentrantReadWriteLock实例包含一个ReentrantReadWriteLock.ReadLock实例和一个ReentrantReadWriteLock.WriteLock实例。

通过readLock\(\)和writeLock\(\)方法可分别获得读锁实例和写锁实例，并通过Lock接口提供的获取锁方法获得对应的锁。  
  
  
读写锁的锁定规则如下：  
获得读锁后，其它线程可获得读锁而不能获取写锁  
获得写锁后，其它线程**既不能获得读锁也不能获得写锁**

```
public class ReadWriteLockDemo {


  public static void main(String[] args) {
ReadWriteLock readWriteLock = new ReentrantReadWriteLock();


    new Thread(() -> {
readWriteLock.readLock().lock();
      try {
        System.out.println(new Date() + "\tThread 1 started with read lock");
        try {
          Thread.sleep(2000);
        } catch (Exception ex) {
        }
        System.out.println(new Date() + "\tThread 1 ended");
      } finally {
readWriteLock.readLock().unlock();
      }
    }).start();


    new Thread(() -> {
readWriteLock.readLock().lock();
      try {
        System.out.println(new Date() + "\tThread 2 started with read lock");
        try {
          Thread.sleep(2000);
        } catch (Exception ex) {
        }
        System.out.println(new Date() + "\tThread 2 ended");
      } finally {
readWriteLock.readLock().unlock();
      }
    }).start();


    new Thread(() -> {
Lock lock = readWriteLock.writeLock();
      lock.lock();
      try {
        System.out.println(new Date() + "\tThread 3 started with write lock");
        try {
          Thread.sleep(2000);
        } catch (Exception ex) {
          ex.printStackTrace();
        }
        System.out.println(new Date() + "\tThread 3 ended");
      } finally {
lock.unlock();
      }
    }).start();
  }
}
```

  
执行结果如下

```
Sat Jun 18 21:33:46 CST 2016  Thread 1 started with read lock
Sat Jun 18 21:33:46 CST 2016  Thread 2 started with read lock
Sat Jun 18 21:33:48 CST 2016  Thread 2 ended
Sat Jun 18 21:33:48 CST 2016  Thread 1 ended
Sat Jun 18 21:33:48 CST 2016  Thread 3 started with write lock
Sat Jun 18 21:33:50 CST 2016  Thread 3 ended
```

从上面的执行结果可见，thread 1和thread 2都只需获得读锁，因此它们可以并行执行。

而thread 3因为需要获取写锁，必须等到thread 1和thread 2释放锁后才能获得锁。

**条件锁**  
条件锁只是一个帮助用户理解的概念，实际上并没有条件锁这种锁。

对于每个重入锁，都可以通过newCondition\(\)方法绑定若干个条件对象。

重入锁可以创建若干个条件对象，signal\(\)和signalAll\(\)方法只能唤醒相同条件对象的等待。  
一个重入锁上可以生成多个条件变量，不同线程可以等待不同的条件，从而实现更加细粒度的的线程间通信。

Condition是个接口，基本的方法就是await\(\)和signal\(\)方法；

Condition依赖于Lock接口，生成一个Condition的基本代码是lock.newCondition\(\)  
调用Condition的await\(\)和signal\(\)方法，都必须在lock保护之内，就是说必须在lock.lock\(\)和lock.unlock之间才可以使用  


Conditon中的await\(\)对应Object的wait\(\)；  
Condition中的signal\(\)对应Object的notify\(\)；  
Condition中的signalAll\(\)对应Object的notifyAll\(\)。

# 原理

ReentrantLock实现的前提就是AbstractQueuedSynchronizer，抽象队列同步器，简称AQS，是java.util.concurrent的核心。

ReentrantLock中有一个抽象类Sync继承了AQS。



我们知道Lock的本质是AQS，AQS自己维护的队列是当前等待资源的队列，AQS会在被释放后，依次唤醒队列中从前到后的所有节点，使他们对应的线程恢复执行，直到队列为空。

而Condition自己也维护了一个队列，该队列的作用是维护一个等待signal信号的队列。

但是，两个队列的作用不同的，事实上，每个线程也仅仅会同时存在以上两个队列中的一个，流程是这样的：  
  
1. 线程1调用reentrantLock.lock时，尝试获取锁。如果成功，则返回，从AQS的队列中移除线程；否则阻塞，保持在AQS的等待队列中。  
2. 线程1调用await方法被调用时，对应操作是被加入到Condition的等待队列中，等待signal信号；同时释放锁。  
3. 锁被释放后，会唤醒AQS队列中的头结点，所以线程2会获取到锁。  
4. 线程2调用signal方法，这个时候Condition的等待队列中只有线程1一个节点，于是它被取出来，并被加入到AQS的等待队列中。注意，这个时候，线程1 并没有被唤醒，只是被加入AQS等待队列。  
5. signal方法执行完毕，线程2调用unLock\(\)方法，释放锁。这个时候因为AQS中只有线程1，于是，线程1被唤醒，线程1恢复执行。

  
所以：  
发送signal信号只是将Condition队列中的线程加到AQS的等待队列中。

只有到发送signal信号的线程调用reentrantLock.unlock\(\)释放锁后，这些线程才会被唤醒。  
  
可以看到，整个协作过程是靠结点在AQS的等待队列和Condition的等待队列中来回移动实现的，

Condition作为一个条件类，很好的自己维护了一个等待信号的队列，并在适时的时候将结点加入到AQS的等待队列中来实现的唤醒操作。



signal就是唤醒Condition队列中的第一个非CANCELLED节点线程，

而signalAll就是唤醒所有非CANCELLED节点线程，本质是将节点从Condition队列中取出来一个还是所有节点放到AQS的等待队列。

尽管所有Node可能都被唤醒，但是要知道的是仍然只有一个线程能够拿到锁，其它没有拿到锁的线程仍然需要自旋等待，就上上面提到的第4步\(acquireQueued\)。



**阻塞当前的线程:调用JNI的 unsafe.park\(false, 0L\)**

**操作队列时，调用JNI的unsafe.compareAndSwapInt（），循环CAS。**

| compareAndSetHead\(Node update\) | 利用CAS设置头Node |
| :--- | :--- |
| compareAndSetTail\(Node expect, Node update\) | 利用CAS设置尾Node |
| compareAndSetWaitStatus\(Node node, int expect, int update\) | 利用CAS设置某个Node中的等待状态 |



