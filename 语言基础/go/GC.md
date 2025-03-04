## GC（Garbage Collection）

> 推荐阅读：
>
> [粗线条话GC（一）](https://mp.weixin.qq.com/s?__biz=Mzg5NjIwNzIxNQ==&mid=2247484496&idx=1&sn=c7555433941ff4698f1cc7fcb2a5fedd&chksm=c005d450f7725d462dcf05d8bad834485703f1da34ee174fd5aaedc09da4c7bf821e06bdaf7c&scene=21#wechat_redirect)
>
> [粗线条话GC（二）](https://mp.weixin.qq.com/s?__biz=Mzg5NjIwNzIxNQ==&mid=2247484514&idx=1&sn=7d00b115b321351d17578effccbd2313&chksm=c005d462f7725d7435202ca5accc86c05818d119150ca685bfcb7a19270916a56cb02af38604&scene=21#wechat_redirect)

## 为什么会有GC存在

在我们之前就了解到，程序定义的全局变量，常量等都会分配到数据段中，而函数的局部变量，参数，返回值都会分配到函数调用栈上。那些生命周期超过当前函数的数据，例如闭包的可变变量，还有一些编译阶段不能确定大小的数据，例如反射，都不适合分配到栈上，都会被分配到堆上，在栈上使用其堆上的地址。

随着程序的运行，慢慢的有些数据不会被再次使用，那么为了减少内存，选择回收。

那么回收哪一部分的数据呢？

分配到栈上的数据，他会随着函数调用栈的销毁也会释放自己的内存，所以不用回收这一部分的。

分配到堆上的数据，他们好像没有人对他处理，如果存在时间长了，对于程序就是一种垃圾了，需要回收这一部分的数据。

## 回收方式

### 手动回收

有些编程语言需要程序员在编写程序时候，手动释放那些不需要的，分配到堆上的数据。例如（C/C++）

缺点：

> 手动垃圾回收不仅增加编程负担，而且风险还比较高。一旦释放的早了，后续对该数据的访问就会出错。因为被释放的内存可能已经被清空，或重新分配，甚至已经还给操作系统了，这就是所谓的“悬挂指针”问题；而如果忘了释放，它又会一直占用内存，出现“内存泄漏”。

### 自动回收

越来越多的编程语言已经支持“自动垃圾回收”，包括 `Go` 语言。

会自动解决由运行时候识别不再有用的数据并释放，存何时被释放，被释放的内存如何处理等问题。

我们今天就来看看，自动垃圾回收到底是怎么样的。

## 什么是垃圾?&怎么回收

> 你是垃圾吗？

怎么去区分这个数据是**有用的数据**还是**垃圾**呢？

我们可以确定，程序中用的到的数据，一定是在栈，数据段上存储的数据。也就是说，可以以这些地方的数据作为根节点，可以追踪的范围一定都包含了全部有用的数据。

那么既然追踪不到的的数据，就一定用不到，就是垃圾。

这是**数据的可达性**！

目前主流的垃圾回收算法都是**使用数据“可达性”近似等价于数据有用性的**。

但是能够追踪到的数据不一定是有用！这也为什么是**近似**的原因。这个数据很少，所以可以进行忽略。

### 标记+清扫

我们通过标记方法，去区分数据的有用性。

**三色抽象**可以清晰的展现追踪过程中的数据标记的变化：

1. 垃圾回收开始会把所有的数据都标记为白色
2. 能够直接追踪到的 `root` 节点标记成灰色。灰色代表当前节点展开的追踪还未完成。
3. 当当前的节点的追踪任务完成后，会将该节点标记成黑色。代表此数据是有用的数据，无需再次进行追踪。
4. 当没有灰色节点时候，代表标记工作已完成。
5. 现在有用的数据都是黑色了，垃圾数据则是白色，那么接下来肯定是清除白色的数据。

三色标记法的三色：

* 黑色（Black）：表示对象是 `GC Root` 可达的，即使用中的对象，黑色是“已经被扫描的对象”。
* 灰色（Grey）：表示黑色对像可追踪到的，但还没对它进行扫描。
* 白色（White）：白色是对象的初始颜色，如果扫描完成后，对象依然还是白色的，说明此对象是垃圾对象。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/ibjI8pEWI9L7ryNA4xyYh7cDaEAFdDHibXVlsU7ic2IyvfAWpd01OlqMdaict6qL954HBMVNv2SJIxuhe0OReOWF7Q/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

优点：

* 实现简单
* 三色标记法的标记过程可以增量式（Incremental）地运行（异步执行）

缺点：

* 容易造成内存碎片化，内存碎片化会影响内存分配与程序执行的效率

### 标记+压缩

这里压缩目的与清扫一直都是回收垃圾，但是这里的行为不一样，移动那些有用数据块，达到压缩的效果。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L7ryNA4xyYh7cDaEAFdDHibXqdCmiaZ8icQclsN0BUSXcDWmCTWMoG3kahdOibagzib5lOksZKsPtOdf0A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



优点：

* 解决了内存碎片化的问题

缺点：

* 多次扫描与移动数据块的开销巨大。

> 推荐文章：
>
> [垃圾回收(GC)算法介绍(4)——GC标记-压缩算法](https://nullcc.github.io/2017/11/30/%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6(GC)%E7%AE%97%E6%B3%95%E4%BB%8B%E7%BB%8D(4)%E2%80%94%E2%80%94GC%E6%A0%87%E8%AE%B0-%E5%8E%8B%E7%BC%A9%E7%AE%97%E6%B3%95/)

### 标记+复制式回收

回收过程：

1. 首先将堆内存划分成两个相等的空间，`FROM&TO`。程序执行时候使用 `FROM` 空间
2. 当要进行垃圾回收时候，扫描 `FROM` 空间，将可以追踪到数据复制到 `TO` 空间。
3. 当所有可追踪数据都复制到 `TO` 空间时候，就可以将 `FROM` 和 `TO` 空间进行交换。简单来讲就是负责的功能进行了交换。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/ibjI8pEWI9L7ryNA4xyYh7cDaEAFdDHibXia1g9GeTticqwuaZG7SwMibyZ6XhTv3l6d4Dthota5qMk1kfNEDZohO3Q/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

优点：

* 解决了碎片化问题
* 使用连续的内存块，可以实现高速的内存分配。

缺点：

* 只有一半的堆内存可以使用，降低了堆内存的使用率。

### 标记+分代回收

新生代对象：新创建的对象

老年代对象：经受住特定次数的垃圾回收而依然存活的对象

基于弱分代假说，新生代对象成为垃圾的概率高于老年代对象，所以可以把数据划分为新生代和老年代，降低老年代执行垃圾回收的频率。

优点：

* 不用每次都扫描所有数据，将明显提升垃圾回收执行的效率，
* 新生代和老年代还可以分别采用不同的回收策略，进一步提升回收效益并减少开销。

缺点：

* 写入屏障会对指针更新操作带来额外的负担。
* 另外如果一个程序中大部分对象存活时间都很长的话，会增加新生代 `GC` 的压力，并且导致老年代 `GC` 频繁地运行。

### 标记+引用计数

一个数据对象被引用的次数，程序执行过程中会更新对象及子对象的引用次数。当引用次数更新到 0 时候，就代表这个对象是垃圾了，可以进行回收了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L7ryNA4xyYh7cDaEAFdDHibXWzGlRLywETdC0ickFJqSZlIMpJNkiaOuiaqlUAjlIMgoZEZSkoh3FSD5g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

优点：

* 不用专门去执行扫描任务，垃圾识别的任务已经分摊到每次数据对象的操作

缺点：

* 实现困难
* 高频率的更新引用会带来不小的开销
* 需要专门解决循环引用的情况。因为循环引用会导致引用计数无法更新到 0，造成对应的内存无法被回收的情况。

> 以上简单介绍了一些垃圾回收的方法，但是这些都是暂停用户程序，一段时间专注于进行垃圾回收（Stop The World）。这里有一个问题：用户真的可以接受长时间的暂停吗？
>
> 所以我们希望尽可能的缩短 `STW` 的时间。

## 增量式垃圾回收

如何缩短 STW 的时间，如果不能在算法层面上进行优化，那么就在使用层面上优化。

将用户程序与垃圾回收交替执行，将垃圾回收工作分多次来完成，这就是增量式垃圾回收。

这里解决方法也有一个问题：**误判垃圾**。

在交替执行的过程中，在这里一次垃圾回收工作中将某一个变量标记为垃圾，但是后面的程序运行时候，却需要使用到这个变量，产生了**误判**的结果，进而会影响程序正常执行。

### 原因

根据三色抽象可以很方便的描述出垃圾回收器错误回收可达对象的情况，需要满足以下两个条件：

* **黑色对象中存在对白色对象的引用**
* **不能从任何灰色对象追踪到该白色对象。**

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L7ic9zdgPfpuPSN3fyfAtlicUSeZOhD7gTlzF2wbBpUttB0dJwFGWMXHGcibx55PcIrNwyib2uz1EdLFA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

解释：黑色对象不会被回收期再次处理了，而回收器正在处理的灰色对象又不能抵达这个白色对象，那么就会被当成垃圾，实际上白色对象并不是垃圾，是可达的。

> 如何解决呢？

1. 在扫描过程中，直接不允许出现黑色对象到白色对象的引用。—**强三色不变式**
2. 允许出现黑色对象对白色对象的引用，但是可以保证通过灰色对象可以抵达到白色对象。—**弱三色不变式**

实现强/弱三色不变式的通常做法是**建立读/写屏障**。

### 写屏障

写屏障会在写操作中插入指令，目的是把数据对象的修改通知到垃圾回收器。那么写屏障通常有一个记录集。记录集的选择：

1. 选择顺序存储，还是哈希存储，
2. 是精确到修改的对象，还是只记录其所在的页

怎样选择都会带来不同的结果，需要根据具体的垃圾回收器类型，去设计写屏障的具体实现方案了。



