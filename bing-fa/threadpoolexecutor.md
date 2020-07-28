##### 

深入理解java线程池—ThreadPoolExecutor[https://www.jianshu.com/p/ade771d2c9c0](https://www.jianshu.com/p/ade771d2c9c0 "深入理解java线程池—ThreadPoolExecutor")

[https://blog.csdn.net/rebirth\_love/article/details/51954836](https://blog.csdn.net/rebirth_love/article/details/51954836)

ThreadPoolExecutor基于JDK8源码解析

* ## 继承关系

## ![](/assets/ThreadPoolExecutor.jpg)

第一部分：第一段从第3行到第26行，是双层无限循环，尝试增加线程数到ctl变量，并且做一些比较判断，如果超出线程数限定或者ThreadPoolExecutor的状态不符合要求，则直接返回false，增加worker失败。第二部分：从第28行开始到结尾，把firstTask这个Runnable对象传给Worker构造方法，赋值给Worker对象的task属性。Worker对象把自身（也是一个Runnable）封装成一个Thread对象赋予Worker对象的thread属性。**锁住整个线程池并实际增加worker到workers的HashSet对象当中（！！！HashSet底层实现为HashMap非线程安全）**。成功增加后开始执行t.start\(\)，就是worker的thread属性开始运行，实际上就是运行Worker对象的run方法。Worker的run\(\)方法实际上调用了ThreadPoolExecutor的runWorker\(\)方法。在看runWorker\(\)之前先看一下Worker对象。

