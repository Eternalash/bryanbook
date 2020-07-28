可达性分析（Reachability Analysis）：从GC Roots开始向下搜索，搜索所走过的路径称为引用链。当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可用的。不可达对象。

在Java语言中，GC Roots包括：

> 虚拟机栈中引用的对象。
>
> 方法区中类静态属性实体引用的对象。
>
> 方法区中常量引用的对象。
>
> 本地方法栈中JNI引用的对象。

![](/assets/可达性分析.png)

![](/assets/标记（marking）对象.png)



[https://www.jianshu.com/p/5261a62e4d29](https://www.jianshu.com/p/5261a62e4d29)

