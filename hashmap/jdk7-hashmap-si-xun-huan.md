#### 一.引言 {#一-引言}

我们在平时的Java代码中经常会用到HashMap这个结构，JDK的官方文档上描述这个结构是非线程安全的，就是说在并发的情况下，HashMap的操作可能不是我们想要的。为了避免并发造成的影响，我们推荐在代码使用ConcurrentMap。一个经常被讨论的问题就是HashMap在高并发的情况下面，使用get方法就会出现死循环，导致应用的load彪高，此时重启应用，一切都会正常。记得在应用出现问题的时候保留现场，出现load彪高，一般需要把线程堆栈使用jstack保留下来，方便后续问题的排查。那么为什么在高并发的情况下，HashMap中会出现死循环呢？下面我们就慢慢分析一下。



#### 二.Hash表数据结构介绍 {#二-Hash表数据结构介绍}

Hash表是一种经典的数据结构，这种数据结构的两个核心元素是Hash数组以及Hash函数。每次往Hash数组中增加一个元素的时候，先通过Hash函数计算出一个数组下表i,然后就在Hash数组中下标为i的位置把这个元素保存下来。获取Hash数组中元素的时候也是按照同样的方法先通过Hash函数计算出元素在Hash数组中的位置，然后再获取。当多个元素通过Hash函数计算出来的值都是i的时候，我们称发生了Hash冲突，此时要么再Hash一次，要么在Hash数组的第i个位置处建立一个链表，把发生冲突的元素依次插入到链表的头部，这样要是Hash函数设计的不够散列化，就会出现很多Hash冲突，这会导致原本O\(1\)的时间复杂度变成了O\(n\)。我们这里所说的HashMap解决冲突是利用链表而不是再Hash的方法。

我们的Hash数组有大小的限制，当Hash表中的元素超过一定数量后，经常会发生Hash冲突，这样会导致Hash数组中元素i处的链表长度不断增加，最终影响我们的查询效率，因此当元素数目超过一定数量后，就会触发rehash操作，就是新建立一个Hash数组，这Hash数组的大小是原来的2倍，然后把老的Hash数组中的元素在新的Hash数组中重新映射一遍，这个操作的代价很高的。

下面我们通过一个简单的示意图来演示一下HashMap的数据结构  
[![](http://bolinyoung.qiniudn.com/hashmap.png "HashMap数据结构图示")](http://bolinyoung.qiniudn.com/hashmap.png)HashMap数据结构图示  
这里Hash数组的大小为5，每个Hash数组中的元素其实就是一个链表的第一个节点，这个链表是一个不带头指针的链表。

#### 三.HashMap死循环分析 {#三-HashMap死循环分析}

我们在上面也提到过，当Hash表中的元素超过一定数量后，会触发rehash的过程，所谓rehash就是把原来Hash表中的元素基于一个新大小的Hash数组再重新映射一遍，以保证查询效率在O\(1\)时间范围内，所谓HashMap的死循环就是在并发rehash的过程中造成的。下面我们先看看HashMap并发rehash的过程，如下图所示:

[![](http://bolinyoung.qiniudn.com/htransfer.png "HashMap的rehash过程")](http://bolinyoung.qiniudn.com/htransfer.png)HashMap的rehash过程

rehash的过程很简单，遍历老的Hash数组table,把Hash数组中的每个元素基于新的Hash数组重新计算一下Hash函数的值，在新的Hash数组中找到相应的位置，把这个值插入到对应链表的第一个节点处，3,4,5处的代码就是做这个事情的，把新来的节点插入到新表i位置对应链表的头部。

下面假设有两个线程同时进入transfer方法中，目前Hash数组大小为2,假设使用的Hash函数是key mod Hash数组的长度，数组中元素的分布为：  
[![](http://bolinyoung.qiniudn.com/1.png "Hash数组中元素的分布")](http://bolinyoung.qiniudn.com/1.png)Hash数组中元素的分布

假设此时Hash表中的元素数目超过了规定的值，因此会触发rehash的过程，此时Hash数组大小会翻倍。我们先看看单个线程rehash过程是怎么发生的，如下图所示:

* 进入while循环后，我们先看看e和e.next的指向  
  [![](http://bolinyoung.qiniudn.com/hashInit.png "e和e.next的指向")](http://bolinyoung.qiniudn.com/hashInit.png)e和e.next的指向

* while循环一次后，观察老数组和新数组中的元素分布，以及e和e.next的指向  
  [![](http://bolinyoung.qiniudn.com/whilefirst.png "第一次while循环")](http://bolinyoung.qiniudn.com/whilefirst.png)第一次while循环

while循环发生一次后，key=3的节点从老表中摘除下来，同时老表的第1个元素值被设置为null,然后重新在新表中根据Hash函数寻找新的位置，把这个元素插入，同时e和e.next继续向下移动，这个是传统的链表移动方式，很好理解。

* while循环第二次后  
  [![](http://bolinyoung.qiniudn.com/whilesecond.png "第二次while循环")](http://bolinyoung.qiniudn.com/whilesecond.png)第二次while循环  
  此时key=7的节点从老表中移动到新表。

* while循环第三次后  
  [![](http://bolinyoung.qiniudn.com/whilelast.png "第三次while循环")](http://bolinyoung.qiniudn.com/whilelast.png)第三次while循环

经过三次while循环后，正常的rehash过程就完成了，这里也不会出现任何问题。

接下来我们看看两个线程并发执行transfer方法的后果：  
我们假设线程一在执行

| 1 | Entry&lt;K,V&gt; next = e.next; |
| :--- | :--- |


时由于某种原因挂起了。此时线程一的上下文如下图所示:  
[![](http://bolinyoung.qiniudn.com/hashInit.png "线程一的上下文")](http://bolinyoung.qiniudn.com/hashInit.png)线程一的上下文  
线程二进入后，正常执行，并且执行结束，结束后Hash表的中数据分布如下：  
[![](http://bolinyoung.qiniudn.com/hashmapthread2.png "线程二执行完后的数据分布")](http://bolinyoung.qiniudn.com/hashmapthread2.png)线程二执行完后的数据分布  
线程一在挂起的时候，e指向了key=3的数据节点,e.next指向了key=7的数据节点，但是线程二执行结束后，key=3的数据节点下一个数据节点不再是key=7这个数据节点了。  
此时线程一开始执行，经过一次while循环后如下图所示:  
[![](http://bolinyoung.qiniudn.com/thread1while1.png "线程一执行第一次while循环后")](http://bolinyoung.qiniudn.com/thread1while1.png)线程一执行第一次while循环后

按照链表的遍历规则，当前的e.next就是下一轮循环的e,因此在本次循环结束后出现了e指向key=7的节点，e.next指向了key=3的节点。

此时再进行第二次循环，把e所指向的节点key=7插入到thread1中Hash表中,如下图所示:  
[![](http://bolinyoung.qiniudn.com/thread1while2.png "线程一执行第二次while循环后")](http://bolinyoung.qiniudn.com/thread1while2.png)线程一执行第二次while循环后

注意理解这里e和e.next的移动

此时再进行第三次循环，把e所指向的key=3的节点插入到线程一中的Hash表中，同时修改key=3节点的next指向，如下图所示：  
[![](http://bolinyoung.qiniudn.com/thread1while3.png "线程一执行第三次while循环后")](http://bolinyoung.qiniudn.com/thread1while3.png)线程一执行第三次while循环后

此时就可以看到出现了一个环，这就是后续死循环出现的根本原因，此时要是get\(15\)立马出现死循环,CPU的load不断彪高,此时重启应用后load就会正常，记得在有应用有问题的保留线程，包括线程堆栈和内存堆栈。

#### 四.总结 {#四-总结}

关于HashMap死循环的问题，表面上看起来很复杂，其实只要自己动手画画图，还是很好理解的，这里涉及不带头节点链表的遍历操作。JVM中的线程共享主内存，同时又有各自私有的本地内存，当线程自己的操作完成后，就会把本地内存中的值刷回到主内存，这样就会导致在并发情况下面，各个线程从主内存中读取到值不一样，因此就需要同步操作来保证个各个线程从主内存中读取到的值都是一样的。如下图所示:  
[![](http://bolinyoung.qiniudn.com/thread.png "线程内存示意图")](http://bolinyoung.qiniudn.com/thread.png)线程内存示意图

