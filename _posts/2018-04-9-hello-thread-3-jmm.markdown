---
layout:     post
title:      "并发基础知识（三）"
subtitle:   " \"java内存模型的基础认知（二）\""
date:       2018-04-9 23:30:00
author:     "Luob"
header-img: "img/post/post-2018-hello-thread-3.jpg"
catalog: false
tags:
    - 并发
    - java
---

>在拿到第二个之前，千万别扔掉第一个。


## 前言

在多核时代，如何提高CPU的性能成为了一个永恒的话题，而这个话题的讨论主要就是如何定义一个高性能的内存模型，内存模型用于定义处理器的各层缓存与共享内存的同步机制及线程和内存交互的规则。

这一篇我们接[上一篇](http://lifetrut.com/2018/04/08/hello-thread-2-jmm/)继续来讨论 JAVA 内存模型（JMM）之重排序。


---
## 正文

由于[前面](http://lifetrut.com/2018/04/08/hello-thread-2-jmm/)所说的，写缓冲区仅对自己的处理器可见，它会导致处理器执行内存操作的顺序可能会与内存实际的操作执行顺序不一致。<br>而现代的处理器都会使用写缓冲区，因此现代的处理器都会允许对写-读操作重排序。

以下是常见处理器允许的重排序类型的列表：

|        | Load-Load | Load-Store | Store-Store | Store-Load | 数据依赖 |
| ------ | ------  | ------ | ------ | ------ | ------ |
|sparc-TSO|N|N|N|Y|N|
|x86|N|N|N|Y|N|
|ia64|Y|Y|Y|Y|N|
|PowerPC|Y|Y|Y|Y|N|

从上表我们可以看出：常见的处理器都允许Store-Load重排序；常见的处理器都不允许对存在数据依赖的操作做重排序。sparc-TSO和x86（包括x64及AMD64）拥有相对较强的处理器内存模型，它们仅允许对写-读操作做重排序（因为它们都使用了写缓冲区）。

为了保证内存可见性，java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。JMM把内存屏障指令分为下列四类：

|屏障类型|指令示例|说明|
| ------ | ------ | ------ |
| LoadLoad<br>Barriers|Load1;<br> LoadLoad;<br> Load2|确保Load1数据的装载，之前于Load2及所有后续装载指令的装载。|
|StoreStore<br>Barriers|Store1;<br> StoreStore;<br> Store2|确保Store1数据对其他处理器可见（刷新到内存），<br>之前于Store2及所有后续存储指令的存储。|
|LoadStore<br>Barriers|Load1;<br> LoadStore;<br> Store2|确保Load1数据装载，之前于Store2及所有后续的存储指令刷新到内存。|
StoreLoad<br>Barriers|Store1;<br> StoreLoad;<br> Load2|确保Store1数据对其他处理器变得可见（指刷新到内存），<br>之前于Load2及所有后续装载指令的装载。<br>StoreLoad Barriers会使该屏障之前的所有内存访问指令（存储和装载指令）完成之后，才执行该屏障之后的内存访问指令。|


其中，StoreLoad Barriers是一个 “全能型” 的屏障，它同时具有其他三个屏障的效果。现代的多处理器大都支持该屏障（其他类型的屏障不一定被所有处理器支持）。执行该屏障开销会很昂贵，因为当前处理器通常要把写缓冲区中的数据全部刷新到内存中。<br><br>

### happens-before
从JDK5开始，java使用lJSR -133内存模型。JSR-133使用happens-before的概念来阐述操作之间的内存可见性。在JMM中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在happens-before关系（这里提到的两个操作既可以在一个线程之内，也可以是在不同线程之间）。

与程序员相关的四个happens-before规则:
* **程序顺序规则**：一个线程中的每个操作，happens- before 于该线程中的任意后续操作。
* **监视器锁规则**：对一个监视器锁的解锁，happens- before 于随后对这个监视器锁的加锁。
* **volatile变量规则**：对一个volatile域的写，happens- before 于任意后续对这个volatile域的读。
* **传递性**：如果A happens- before B，且B happens- before C，那么A happens- before C。

**注意**：*两个操作之间具有happens-before关系，并不意味着前一个操作必须要在后一个操作之前执行！happens-before仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前。*

happens-before与JMM的关系:

![Aaron Swartz](http://ifeve.com/wp-content/uploads/2013/01/552.png)

如上图所示，一个happens-before规则通常对应于多个编译器和处理器重排序规则。happens-before规则它避免java程序员为了理解JMM提供的内存可见性保证而去学习复杂的重排序规则以及这些规则的具体实现。


接下来我们再来看看数据依赖性、as-if-serial 语义、程序顺序规则，然后我们再回过头来看看重排序对多线程的影响。
<br><br>


### 数据依赖性
数据依赖性是指：如果两个操作访问同一个变量，且这两个操作中有一个为写操作，那么这两个操作之间就存在数据依赖性。

数据依赖分三种类型:

|名称|代码示例|说明|
| ------ | ------ | ------ |
|写后读|a = 1;b = a;|写一个变量之后，再读这个位置。|
|写后写|a = 1;a = 2;|写一个变量之后，再写这个变量。|
|读后写|a = b;b = 1;|读一个变量之后，再写这个变量。|

以上三种情况，只要重排序两个操作的执行顺序，程序的执行结果将会被改变。

前面提到过，编译器和处理器可能会对操作做重排序。**编译器和处理器在重排序时，会遵守数据依赖性，编译器和处理器不会改变存在数据依赖关系的两个操作的执行顺序。**

**注意**：*这里所说的数据依赖性仅针对单个处理器中执的指令序列和单个线程中执行的操作，不同处理器之间和不同线程之间的数据依赖性不被编译器和处理器考虑。* <br><br>


### as-if-serial 语义
as-if-serial 语义的意思指：不管怎么重排序(编译器和处理器为了提高并行度)，(单线程)程序的执行结果不能被改变。编译器，runtime 和处理器都必须遵守as-if-serial 语义。

为了遵守 as-if-serial 语义，编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变执行结果。但是，如果操作之间不存在数据依赖关系，这些操作就可能被编译器和处理器重排序。

举个例子：
```java
double pi = 3.14; //A
double r = 1.0; //B
double area = pi * r * r; //C
```
上面三个操作的数据依赖关系如下图所示:

![Aaron Swartz](https://segmentfault.com/img/remote/1460000013474319?w=550&h=384)

A 和 C 之间存在数据依赖关系，同时 B 和 C 之间也存在数据依赖关系。因此在最终执行的指令序列中，C 不能被重排序到 A 和 B 的前面(C 排到 A 和B 的前面，程序的结果将会被改变)。但 A 和 B 之间没有数据依赖关系，编译器和处理器可以重排序 A 和 B 之间的执行顺序。下图是该程序的两种执行顺序:

![Aaron Swartz](https://segmentfault.com/img/remote/1460000013474320?w=1290&h=464)

as-if-serial 语义把单线程程序保护了起来，遵守 as-if-serial 语义的编译器，runtime 和处理器共同为编写单线程程序的程序员创建了一个幻觉:单线程程序是按程序的顺序来执行的。as-if-serial 语义使单线程程序员无需担心重排序会干扰他们，也无需担心内存可见性问题。<br><br>


### 程序顺序规则
根据 happens- before 的程序顺序规则，上面计算圆的面积的示例代码存在三个happens- before 关系:
1. A happens- before B;
2. B happens- before C;
3. A happens- before C;

其中第 3 个 happens- before 关系，是根据 happens- before 的传递性推导出来的。

这里 A happens- before B，但实际执行时 B 可以排在 A 之前执行(看上面的重排序后的执行顺序)。前面提到过，A happens- before B，JMM 并不会要求 A 一定要在 B 之前执行。JMM 只要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前。在这里操作 A 的执行结果不
需要对操作 B 可见;而且重排序操作 A 和操作 B 后的执行结果，与操作 A 和操作 B 按 happens- before 顺序执行的结果一致。在这种情况下，JMM 会认为这种重排序并不非法，JMM 允许这种重排序。

**在计算机中，软件技术和硬件技术有一个共同的目标：在不改变程序执行结果的前提下，尽可能的开发并行度**。从 happens- before 的定义我们可以看出编译器和处理器都在遵从这一目标，JMM 同样遵从这一目标。<br><br>


### 接下来我们在来回到重排序对多线程的影响
首先我们来看一段代码：
```java
public class ReorderExample {
    
    int a = 0;
    boolean flag = false;

    public void writer() {
        a = 1;          //1
        flag = true;    //2
    }

    public void reader(){
        if(flag){
            int i = a * a;
        }
    }
}
```
上面代码中，flag 变量是个标记，用来标识变量 a 是否已被写入。<br>

我们假设有两个线程 A 和 B，A 首先执行 writer()方法，随后 B 线程接着执行 reader()方法。线程 B 在执行操作 4 时，能否看到线程 A 在操作 1 对共享变量 a 的写入?

答案是：不一定能看到。

因为操作 1 和操作 2 没有数据依赖关系，编译器和处理器可以对这两个操作重排序;同样，操作 3 和操作 4 没有数据依赖关系，编译器和处理器也可以对这两个操作重排序。让我们先来看一下，当操作 1 和操作 2 重排序时，可能会产生什么效果?

![Aaron Swartz](https://segmentfault.com/img/remote/1460000013474321?w=1076&h=738)

如上图所示，操作 1 和操作 2 做了重排序。程序执行时，线程 A 首先写标记变量flag，随后线程 B 读这个变量。由于条件判断为真，线程 B 将读取变量 a。此时，变量 a 还根本没有被线程 A 写入，在这里多线程程序的语义被重排序破坏了!

接下来让再我们看看，当操作 3 和操作 4 重排序时会产生什么效果：

![Aaron Swartz](https://segmentfault.com/img/remote/1460000013474322?w=1294&h=896)

在程序中，操作 3 和操作 4 存在控制依赖关系。当代码中存在控制依赖性时，会影响指令序列执行的并行度。为此，编译器和处理器会采用猜测执行来克服控制相关性对并行度的影响。以处理器的猜测执行为例，执行线程 B 的处理器可以提前读取并计算 a*a，然后把计算结果临时保存到一个名为重排序缓冲的硬件缓存中。当接下来操作 3 的条件判断为真时，就把该计算结果写入变量 i 中。

从上图中我们可以看出，猜测执行实质上对操作 3 和 4 做了重排序。重排序在这里破坏了多线程程序的语义!

#### 总结：在单线程程序中，对存在控制依赖的操作重排序，不会改变执行结果(这也是 as-if-serial 语义允许对存在控制依赖的操作做重排序的原因);但在多线程程序中，对存在控制依赖的操作重排序，可能会改变程序的执行结果。

<br>

---
<br>

## 后记
今天因为下班看出租屋去了，所以回来时比较晚。
但是写博客贵在坚持，每天学一点东西写一点东西总能感觉没有辜负这一天。

后天终于搬家了，以后就不用每天六点多就起床了。
想想一天确实还是 有点辛苦。。

睡觉睡觉还有5个多小时又得起床开始忙碌的一天了！

—— Luob 后记于 2018.4