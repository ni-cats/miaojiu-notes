# 0 新版修改

![](https://cdn.nlark.com/yuque/0/2024/jpeg/38995097/1708251919712-144d164f-fb87-42d6-a815-b28c10285ec6.jpeg)

# 第一章 并发编程的挑战

## 1.1上下文切换

并发编程的目的：让程序运行的更快!

### 1.什么是上下文切换？

对于一个CPU，即使是单核的CPU也是开业支持多线程的，CPU通过给每一个超线程分配CPU时间片来实现这个机制。CPU通过时间片分配算法来执行多线程。当前线程执行完一个时间片之后会切换到下一个线程，在切换前需要保存当前线程的状态，以便下次切换回该线程时，可以加载该线程的状态。

线程从保存到再加载的过程就是一次上下文切换。

### 2.如何减少上下文切换？

+ 无锁并发编程

多线程竞争锁的时候会引起上下文切换。

+ CAS算法
+ 使用最少线程

避免创建不需要的线程。

+ 使用协程

在单线程里面实现多个任务的调度，并在单线程里维持多个任务间的切换。

# 第二章 Java并发机制的底层实现原理

Java中的并发机制依赖于JVM的实现和CPU的指令。

## 2.1volatile的应用

`**<font style="color:#DF2A3F;">volatile</font>**`**<font style="color:#DF2A3F;">是轻量级的</font>**`**<font style="color:#DF2A3F;">synchronized</font>**`**<font style="color:#DF2A3F;">，他在多处理器开发中保证了共享变量的“可见性”。</font>**

使用恰当时，会比`synchronized`的执行成本更低--->因为**它的实现不会引起线程上下文的切换！**

### 1.volatile的定义

java编程语言允许线程访问共享变量，为了确保共享变量能准确和一致的更新，线程应该确保通过排它锁单独获得这个变量。

Java语言提供了`volatile`，如果一个字段被声明成`volatile`的，java线程内存模型确保所有线程看到这个变量的值是一致的。

CPU的术语定义

|    术语    |        英文单词        | 术语描述                                                     |
| :--------: | :--------------------: | ------------------------------------------------------------ |
|  内存屏障  |    memory barriers     | 是一组处理器指令，用于实现对内存操作的顺序限制               |
|   缓冲行   |       cache line       | 缓存中可以分配的最小存储单位。处理器填写缓冲线时会加载整个缓冲线，需要使用多个主内存读周期 |
|  原子操作  |   atomic operations    | 不可中断的一个或者一些列操作                                 |
| 缓冲行填充 |    cache line fill     | 当处理器度却道从内存中读取操作数是可缓存的，处理器读取整个缓存行到适当的缓存（L1、L2、L2或者所有） |
|  缓存命中  |       cache hit        | 如果进行高速缓存行填充操作的内存位置任然是下一次处理器访问的地址时，处理器从缓存中读取操作数，而不是从内存中读取 |
|   写命中   |       write hit        | 当处理器将操作数写回到一个内存缓存的区域时，它首先会检查这个缓存的内存地址是否在缓存行中，如果存在一个有效的缓存行，则处理器将这个操作数写回到缓存，而不是写回到内存，这个操作被称为写命中。 |
|   写缺失   | write misses the cache | 一个有效的缓存行被写到一个不存在的内存区域。                 |


### 2.volatile是如何保证可见性的呢？

    我们用JIT工具转换成汇编代码，可以发现有`volatile`变量修饰的共享变量进行写操作的时候会多出第二行汇编代码：

```c
lock addl $0X0,(%esp)    ---将ESP寄存器的值加0
```

Lock前缀的指令在多核处理器下会引发两件事情：

+ 将当前处理器缓存行中的数据写回到系统内存。
+ 这个写回内存的操作会使在其它CPU里缓存了该内存地址的数据无效。

为了提高处理速度，处理器不直接和内存进行通信，而是先将内存中的数据读到内部缓存中后再进行操作，但是**操作完之后什么时候写回内存是不一定的。**如果我们使用了`volatile`关键字修饰该变量，那么在写该变量的时候，JVM就会向处理器发送一条lock指令，将这个变量所在的缓存行写回到内存，同时还需要借助**缓存一致性协议。**每个处理器通过**嗅探**在总线上传播的数据来检查自己的缓存值是不是过期了，如果处理器发现自己缓存行对应的内存地址发生变化了，则该缓存无效，再次使用的时候需要从内存中读取。

#### Lock前缀指令会引起处理器缓存回写到内存。

**<font style="color:#DF2A3F;">在多处理器环境中，</font>**`**<font style="color:#DF2A3F;">LOCK#</font>**`**<font style="color:#DF2A3F;">信号确保在声言该信号期间，处理器可以独占任何共享内存（以前是锁总线，后来是锁缓存</font>**）;

#### 一个处理器的缓存回写到内存会导致其它处理器的缓存无效。---缓存一致性协议

处理器使用**嗅探技术**保证它的内部缓存、系统内存和其它处理器的缓存的数据在总线上保持一致。

### 3.volatile的使用优化

有一位java大神在JDK7中的并发包中增加了一个队列集合类`LinkedTransferQueue`，它在使用volatile变量时，用一种追加字节的方式来优化队列的出队和入队的性能。

一个对象的引用为4个字节，追加64个字节可以让一些缓存行为64字节宽的处理器在读取队头和队尾节点时读取在不同的缓存行中，这样进行出队和入队操作就不会引起其它处理器的锁定。、

## 2.2synchronized的实现原理与应用

**<font style="color:#DF2A3F;">Java中的每一个对象都可以作为锁！</font>**

+ 对于普通同步方法，锁的是当前实例对象
+ 对于静态同步方法，锁的是当前类的class对象
+ 对于同步代码块中的，锁的是synchronized括号里面配置的对象

<font style="background-color:#FBDE28;">当前线程试图访问同步代码块时，它首先必须得到锁，退出或者抛出异常时，会自动释放锁。</font>

### 1.Java对象头

synchronized用的锁是存储在Java对象头里面的。如果对象是数组类型，则JVM用3个字宽存储对象头，如果是非数组类型，则用2个字宽存储对象头。在32位JVM中，一个字宽等于4个字节等于32位。

#### 1.偏向锁

大多数情况下，锁不仅仅是不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。

当一个线程需要获得锁时，首先会在对象头和栈帧中存储偏向锁偏向的线程ID，以后该线程再次需要持有该锁时，只需要简单地测试一下对象头里面是否存储指向当前线程的偏向锁，不需要进行CAS操作来加锁。如果测试失败，就需要再测试一下对象头里面偏向锁的标志是否是1（测试当前对象是否可偏向），如果不可偏向，则用CAS竞争锁，如果可偏向，直接将该对象偏向给该线程设置偏向锁即可。

偏向锁的撤销：偏向锁的撤销需要等到全局安全点（这个时间点上没有正在执行的字节码）。

#### 2.轻量级锁

线程在执行同步块之前，JVM会先在当前线程的栈帧中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word，然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针，如果成功则当前线程获取锁，失败，表明存在锁竞争，当前线程可以使用自旋来获取锁！

#### 3.各种锁的比较

|    锁    | 优点                                                         | 缺点                                           | 适用场景                              |
| :------: | ------------------------------------------------------------ | ---------------------------------------------- | ------------------------------------- |
|  偏向锁  | 加锁和解锁不需要额外的消耗，和执行非同步方法相比，仅存在纳秒级的差距 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗 | 适用于只有一个线程访问同步块的情况    |
| 轻量级锁 | 竞争的线程不会阻塞，提高了程序的响应速度                     | 如果始终得不到锁竞争的线程，使用自旋会消耗CPU  | 追求响应时间<br/>同步块执行速度非常快 |
| 重量级锁 | 线程竞争不使用自旋，不会消耗CPU                              | 线程阻塞，响应时间慢                           | 追求吞吐量<br/>同步块执行速度较长     |


## 2.3原子操作的实现原理

### 1.术语定义

|   术语名称   |          英文          | 解释                                                         |
| :----------: | :--------------------: | ------------------------------------------------------------ |
|    缓存行    |       cache line       | 缓存的最小操作单位                                           |
|  比较并交换  |    Compare and Swap    | CAS操作需要输人两个数值，一个旧值(期望操作前的值）和一个新值,在操作期间先比较旧值有没有发生变化，如果没有发生变化，才交换成新值，发生了变化则不交换 |
|  CPU流水线   |      CPU pipeline      | CPU流水线的工作方式就像.工业生产上的装配流水线，在 CPU中由5~6个不同功能的电路单元组成一条指令处理流水线，然后将一条X86指令分成5~6步后再由这些电路单元分别执行,这样就能实现在一个CPU时钟周期完成一条指令，因此提高CPU的运算速度 |
| 内存顺序冲突 | Memory order violation | 内存顺序冲突一般是由<font style="color:#DF2A3F;">假共享</font>引起的，假共享是指多个CPU同时修改同一个缓存行的不同部分而引起其中一个CPU的操作无效,当出现这个内存顺序冲突时，CPU必须清空流水线 |


### 2.处理器如何实现原子操作

**32位I-32处理器是使用基于****<font style="color:#0C68CA;">对缓存加锁</font>****或者****<font style="color:#0C68CA;">对总线加锁</font>****的方式来实现多处理器之间的原子操作的。**

首先处理器会自动保证基本的内存操作的原子性。<font style="color:#0C68CA;"></font>

处理器保证从系统内存中读取或者写入一个字节是原子的。（当一个处理器读取一个字节时，其它处理器不能访问这个字节的内存地址）

#### <font style="color:#DF2A3F;">a.使用总线锁来保证原子性</font>

所谓总线锁就是使用处理器提供的一个`LOCK#`信号，当一个处理器在总线上输出此信号时，其它处理器的请求将被阻塞住，那么该处理器可以独占共享内存。

#### <font style="color:#DF2A3F;">b.使用缓存锁来保证原子性</font>

在同一时刻，我们只需要保证对某个内存地址的操作是原子性的即可。

总线锁定把CPU和内存之间的通信锁住了，这使得锁定期间，其他处理器不能操作其他内存地址的数据，所以总线锁定的开销比较大,目前处理器在某些场合下使用缓存锁定代替总线锁定来进行优化。

频繁使用的内存会缓存在处理器的L1、L2和L3高速缓存里，那么<u>原子操作就可以直接在处理器内部缓存中进行</u>，并不需要声明总线锁，在Pentium 6和目前的处理器中可以使用“缓存锁定”的方式来实现复杂的原子性。**所谓“缓存锁定”是指内存区域如果被缓存在处理器的缓存行中，并且在Lock操作期间被锁定，那么当它执行锁操作回写到内存时，处理器不在总线上声言LOCK #信号，****<u>而是修改内部的内存地址（</u>**<font style="color:rgb(55, 65, 81);">标记内存区域为锁定状态</font>**<u>）</u>****，并允许它的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改由两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时，会使缓存行无效。**

#### c.两种处理器不会使用缓存锁定的情况

1. 当操作的数据不能被缓存在处理器内部，或者操作的数据跨越多个缓存行，则处理器会调用总线锁定。
2. 处理器不支持缓存锁定。

### 3.Java如何实现原子操作

在Java中可以通过**锁**和**循环CAS**的方式实现原子操作。

#### a.使用循环CAS实现锁操作

基本思路：循环进行CAS操作直到成功为止。

CAS举例：

```java
public class CASTest {
    public static void main(String[] args) {
        final Counter cas = new Counter();
        List<Thread> ts = new ArrayList<>(600);
        long start = System.currentTimeMillis();
        for (int i = 0; i < 100; i++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int j = 0; j < 10000; j++) {
                        cas.count();
                        cas.safeCount();
                    }
                }
            });
            ts.add(t);
        }
        for (Thread t : ts) {
            t.start();
        }

        for (Thread t : ts) {
            try {
                t.join();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
        System.out.println("线程不安全："+cas.i);
        System.out.println("原子类："+cas.atomicI.get());
        System.out.println(System.currentTimeMillis() - start);
    }
}

class Counter {
    public int i = 0;
    public AtomicInteger atomicI = new AtomicInteger(0);
    //現場不安全的计数器
    public void count() {
        i++;
    }
    //线程安全的计数器
    public void safeCount() {
        while (true) {
            int i = atomicI.get();
            boolean suc = atomicI.compareAndSet(i, ++i);
            if (suc) {
                break;
            }
        }
    }
}
```

您的代码演示了一个简单的多线程场景，其中包含一个不安全的计数器（`i`）和一个线程安全的计数器（`atomicI`，使用`AtomicInteger`实现）。在这个代码中，您创建了100个线程，每个线程都会对这两个计数器进行10000次的增加操作。

主要观察点和结果解释如下：

1.  `i` 是一个简单的整数变量，不具备线程安全性，因此多个线程同时对其进行修改可能会导致竞争条件和不确定的结果。这在 `cas.count()` 方法中被演示。
2.  `atomicI` 是一个使用 `AtomicInteger` 类型的线程安全计数器，它提供了一些原子操作，如 `get()` 和 `compareAndSet()`，确保了多线程环境下的安全访问。这在 `cas.safeCount()` 方法中被演示。
3.  最后，您打印了两个计数器的值以及程序的执行时间。

请注意以下几点：

+  使用 `i++` 语句时，应该使用后自增（`++i`）而不是前自增（`i++`），因为前自增会导致 `compareAndSet()` 永远不会成功。
+  `AtomicInteger` 的 `compareAndSet()` 方法用于比较当前值和期望值，如果相等，则将新值设置进去。这是一种原子操作，可以确保多个线程同时尝试更新计数器时只有一个成功，从而保证了线程安全。
+  通过观察程序的输出，您可以看到不安全的计数器 `i` 的结果是不确定的，而线程安全的计数器 `atomicI` 的结果是确定的且正确的。

这个示例演示了为什么在线程间共享数据时，使用线程安全的数据结构或同步机制是非常重要的，以避免竞争条件和不确定的结果。

## *<font style="color:#DF2A3F;">在并发编程中 ++i和i++有什么区别？</font>

在并发编程中，`++i` 和 `i++` 之间有重要的区别，因为它们在多线程环境中的行为不同：

1.  `++i`（前自增）：
    - `++i` 是前自增操作，它会先将变量 `i` 的值加一，然后返回增加后的值。
    - 在多线程环境中，`++i` 是原子操作，因为它是一步完成的，没有其他线程可以在中间插入操作。这意味着在多线程环境中，多个线程同时执行 `++i` 不会导致问题，因为每个线程会依次递增变量的值。
2.  `i++`（后自增）：
    - `i++` 是后自增操作，它会先返回变量 `i` 的当前值，然后再将 `i` 的值加一。
    - 在多线程环境中，`i++` 不是原子操作，因为它涉及两个步骤：读取当前值和增加值。这可能导致竞争条件，如果多个线程同时执行 `i++`，它们可能会读取相同的值，然后都尝试增加它，导致不确定的结果或数据不一致。

在并发编程中，为了确保线程安全，应优先选择使用原子操作，如 `++i`，而不是 `i++`。如果需要获取当前值并在之后增加，最好使用原子操作或适当的同步机制来保护共享变量。例如，可以使用 `AtomicInteger` 类来执行线程安全的自增操作，或者使用锁来保护临界区，以确保操作的原子性和线程安全性。

### 4.CAS实现原子操作的三大问题

#### a.ABA问题

解决：版本号机制

#### b.循环时间长，开销大

自旋 CAS如果长时间不成功，会给 CPU带米非常大的执行开销。如果**JVM能支持处理器提供的pause指令**，那么效率会有一足的提升。pause指令有两个作用:第一，它可以延迟流水线执行指令(de-pipeline)，使CPU不会消耗过多的执行资源,延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零;弟二它可以避免在退出循环的时候因内存顺序冲突(Memory Order Violation)血5起CPU流水线破特空（CPUPipeline Flush)、从而提高CPU 的执行效率。

#### c.<font style="color:#DF2A3F;">只能保证一个共享变量的原子操作</font>

当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS 就无法保证操作的原子性,这个时候就可以用锁。还有一个取巧的办法就是把多个共享变量合并成一个共享变量来操作。比如,有两个共享变量i=2 j=a，合并一下ij=2a，然后用 CAS来操作1j。从Java 1.5开始，JDK提供了AtomicReference类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行CAS操作。

### 5.使用锁机制实现原子操作

锁机制保证了只有获得锁的线程才能够操作锁定的内存区域。JVM内部实现了很多种锁机制，有偏向锁、轻量级锁和互斥锁。有意思的是除了偏向锁，JVM实现锁的方式都用了循环CAS,即当一个线程想进人同步块的时候使用循环CAS的方式来获取锁,当它退出同步块的时候使用循环CAS释放锁。

# 第三章 Java内存模型

## 3.1Java内存模型的基础

### 1.并发编程模型的两个关键问题

**在并发编程中，需要处理两个关键问题:****<font style="color:#DF2A3F;">线程之间如何通信</font>****及****<font style="color:#DF2A3F;">线程之间如何同步</font>**(这里的线程是指并发执行的活动实体)。通信是指线程之间以何种机制来交换信息。

在命令式编程中,线程之间的通信机制有两种:**共享内存**和**消息传递**。

+ 在共享内存的并发模型里,线程之间共享程序的公共状态，通过写-读内存中的公共状态进行隐式通信。
+ 在消息传递的并发模型里，线程之间没有公共状态，线程之间必须通过发送消息来显式进行通信。

<u>同步是指程序中用于控制不同线程间操作发生相对顺序的机制。</u>

在共享内存并发模型里,同步是显式进行的。程序员必须显式指定某个方法或某段代码需要在线程之间互斥执行。在消息传递的并发模型里，由于消息的发送必须在消息的接收之前，因此同步是隐式进行的。

**Java的并发采用的是共享内存模型**，Java线程之间的通信总是隐式进行的。

### 2.Java内存模型的抽象结构

在Java中，所有**实例域**、**静态域**和**数组元素**都存储在堆内存中,堆内存在线程之间共享(本章用“共享变量”这个术语代指实例域，静态域和数组元素)。

**局部变量**(Local和异常处理器参Variables).**方法定义参数**（Java语言规范称之为Formal Method Parameters）和**异常处理参数**（Exception Handler Parameters）不会在线程间共享，他们不会有内存可见性问题，也不会受内存模型的影响。

Java线程之间的通信由Java内存模型（本文简称为JMM)控制， **JMM决定一个线程对共享变量的写入何时对另一个线程可见**。从抽象的角度来看，JMM定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存（Main Memory)中,每个线程都有一个私有的本地内存（Local Memory)，本地内存中存储了该线程以读/写共享变量的副本。本地内存是 JMM的一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。Java 内存模型的抽象示意如图3-1所示。



**<font style="color:#DF2A3F;">JMM通过控制主内存与每个线程的本地内存之间的交互，来为Java程序员提供内存可见性保证。</font>**---   一直说JMM，JMM，JMM怎么控制并发？有什么作用？这就是答案！！！

### 3.从源代码到指令序列的重排序

+ 编译器优化的重排序
+ 指令级并行的重排序
+ 内存系统的重排序
+ （第一个是编译器级别，后面两个是处理器级别）

Java从源代码到最终的实际执行的指令序列要依次经过上面三种的重排序。

对编译器，JMM的编译器重排序规则会禁止特定类型的编译器重排序；

对处理器重排序，JMM的处理器重排序规则会要求Javav编译器在生成指令序列时插入特定类型的内存屏障指令，通过内存屏障指令来禁止特定类型的处理器重排序。

**JMM属于语言级的内存模型，它通过禁止特定类型的编译器重排序和处理器重排序，来提高内存可见性的保证。**

###  4.并发编程模型的分类

现代处理器会使用写缓冲区临时保存向内存中写入的数据，通过批处理方式刷新缓冲区，以及合并写缓冲区中对同一内存地址的多次写，减少对内存总线的占用。

<u>但是写缓冲区只对自己所在的处理器可见，会造成处理器对内存的读写操作的实现顺序与内存实际发生的读写操作顺序不一样！</u>

为了保证内存的可见性，Java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。

**StoreLoad指令**：会使该屏障之前的所有内存访问指令（存储和装载指令）完成之后，才执行该屏障之后的内存访问指令。

### 5.happens-before简介

+ 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
+ 监视器锁规则：对一个锁的解锁，happens-before于随后的对这个锁的加锁。
+ volatile变量规则：对一个volatile变量的写，happens-before于任意后续对这个volatile变量的读。
+ 传递性：如果A happens-before B，且B happens-before C，则A happens-before C。
+ （一个happens-before规则对应于一个或者多个编译器和处理器重排序规则）

## 3.2重排序

重排序是指编译器和处理器为了优化程序性能而对指令序列进行重排序的一种手段。

### 1.数据依赖性

如果两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在数据依赖性。

写后读	写后写	读后写

这三种情况，只要重排序两个操作的执行顺序，程序的执行结果就会发生改变！

单个处理器是可以满足这个要求的，但是多个处理器，也就是多线程下，这个要求就不一定得到满足！！！

### 2.as-if-serial语义

不管怎么重排序，单线程的程序的执行结果不能被改变。

为了达到这个要求，编译器和处理器不会对存在数据依赖关系的操作进行重排序。

### 3.程序顺序规则

### 4.重排序对多线程的影响

## 3.3顺序一致性

## 3.4volatile的内存语义

### 1.volatile的特性

**<font style="color:#DF2A3F;">理解volatile特性的一个好方法是把对volatile变量的单个读写看成是使用同一个锁对这些单个读写操作做了同步。</font>**

volatile变量自身具有如下特性：

+ 可见性：对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。
+ 原子性：对**<font style="color:#5C8D07;">任意单个volatile变量的读写具有原子性</font>**，但类似于volatile++这种符合操作不具有原子性。

### 2.volatile写-读建立的happens-before关系

volatile对线程的内存可见性的影响比volatile自身的特性更为重要。

volatile变量的读写可以实现线程之间的通信。

**<font style="color:#DF2A3F;">从内存语义的角度来说，volatile的的读写与锁的释放和获取有相同的内存效果：</font>**

+ volatile的写和锁的释放有相同的内存语义
+ volatile的读和锁的获取有相同的内存语义

### 3.volatile写-读的内存语义

volatile写的内存语义：当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存。

volatile读的内存语义：当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来见从主内存中读取共享变量。

总结：

+ 线程A写一个volatile变量，实质上是线程A向接下来将要读这个volatile变量的某个线程发出了（其对共享变量所做修改的）消息
+ 线程B读一个volatile变量，实质上是线程B接收了之前某个线程发出的（在写这个volatile变量之前对共享变量所做修改的）消息
+ 线程A写一个volatile变量，随后线程B读这个volatile变量，这个过程**实质上是线程A通过主内存向线程B发送消息。**

### 4.volatile内存语义的实现

<font style="color:#DF2A3F;">为了实现volatile内存语义，JMM会分别限制编译器重排序和处理器重排序；</font>

| 是否能重排序 | 第二个操作 |            |            |
| :----------: | :--------: | ---------- | ---------- |
|  第一个操作  |  普通读写  | volatile读 | volatile写 |
|   普通读写   |            |            | NO         |
|  volatile读  |     NO     | NO         | NO         |
|  volatile写  |            | NO         | NO         |


从表中我们可以看出：

+ 当第二个操作是 volatile写时、不管第一个操作是什么，都不能重排序。这个规则确保volatile写之前的操作不会被编译器重排序到 volatile写之后。
+ 当第一个操作是 volatile读时，不管第二个操作是什么，都不能重排序。这个规则确保volatile读之后的操作不会被编译器重排序到volatile读之前。
+ 当第一个操作是 volatile写,第二个操作是volatile读时,不能重排序。

为了实现volatile 的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。对于编译器来说，发现一个最优布置来最小化插入屏障的总数几乎不可能。为此，JMM采取保守策略。下面是基于保守策略的JMM内存屏障插入策略。

+ 在每个volatile写操作的<u>前面</u>插入一个 StoreStore屏障。
+ 在每个 volatile写操作的后面插入一个StoreLoad屏障。
+ 在每个 volatile读操作的后面插入一个 LoadLoad屏障。
+ 在每个 volatile 读操作的后面插人一个 LoadStore屏障。

上述内存屏障插入策略非常保守，但它可以保证在任意处理器平台，任意的程序中都能得到正确的volatile 内存语义。

###  5.JSR-133增强volatile内存语义

旧的Java内存模型允许volatile变量与普通变量之间的重排序。

因此,在旧的内存模型中，volatile的写-读没有锁的释放-获取所具有的内存语义。为了提供一种比锁更轻量级的线程之间通信的机制，JSR-133专家组决定增强 volatile 的内存语义:

**严格限制编译器和处理器对volatile变量与普通变量的重排序**，确保volatile的写-读和锁的释放-获取具有相同的内存语义。从编译器重排序规则和处理器内存屏障插入策略来看，只要volatile变量与普通变量之间的重排序可能会破坏volatile的内存语义，这种重排序就会被编译器重排序规则和处理器内存屏障插入策略禁止。

## 3.5锁的内存语义

### 1.锁写-读建立的happens-before关系

    线程A在释放锁之前所有可见的共享变量，在线程B获取同一个锁之后，将立刻变得对B线程可见。

### 2.锁的释放和获取的内存语义

+ 当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中。
+ 当线程获取锁时，JMM会把该线程对应的本地内存置为无效，从而使得被监视器保护的临界区代码必须从主内存中读取共享变量。

#### <font style="color:#DF2A3F;">锁释放和volatile变量写有相同的内存语义；锁的获取和volatile变量读有相同的内存语义！！！    </font>

### 总结：

+ 线程A释放一个锁，实质上是线程A想接下来要获取这个锁的某个线程发出了（线程A对某个共享变量所做修改的）消息。
+ 线程B获取一个锁，实质上就是线程B接收了之前某个线程发出的（在释放这个锁之前对共享变量所做的修改的）消息。
+ 线程A释放锁，随后线程B获取这个锁，这个过程实质上是线程A通过主内存向线程B发送消息。

### 3.锁内存语义的实现

CAS：如果当前状态值等于预期值，则以原子方式将同步状态设置为给定的更新值，**此操作具有volatile读和写的内存语义。**

CAS源代码的最终汇编代码：   --- P53

程序会根据当前处理器的类型来决定是否为cmpchg指令添加lock前缀，如果是在多处理器上运行，就会添加lock前缀，如果是在单处理器上运行，就不会添加lock前缀。

intel的手册对lock前缀的说明如下：

+ 确保对内存的读-改-写操作原子执行
+ 禁止该指令，与之前和之后的读和谐操作重排序
+ 把写缓冲区中的所有数据刷新到内存中。

上面亮点所具有的内存屏障效果，足以同时实现volatile读和volatile写的内存语义。

<font style="color:#DF2A3F;">现在我们知道，锁获取和释放的内存语义的实现至少有两种方式：</font>

+ 利用volatile变量的写-读所具有的内存语义
+ 利用CAS所附带的volatile读和写的内存语

### 4.concurrent包的实现

由于Java的CAS同时具有volatile读和volatile写的内存语义，因此java线程之间的通信现在有了下面四种方式。

1) A线程写volatile变量，随后B线程读这个volatile变量。

2) A线程写volatile变量，随后B线程用CAS更新这个volatile变量。

3) A线程用CAS更新一个volatile变量，随后B线程用CAS更新这个volatile变量。

4) A线程用CAS更新一一个volatile 变量，随后B线程读这个volatile 变量。

Java的CAS会使用现代处理器上提供的高效机器级别的原子指令，这些原子指令原子方式对内存执行读-改-写操作，这是在多处理器中实现同步的关键(从本质上来说，能够支持原子性读-改一写指令的计算机，是顺序计算图灵机的异步等价机器，因此任何现代的多处理器都会去支持某种能对内存执行原子性读-改-写操作的原子指令)。

同时，volatile变量的读/写和CAS可以实现线程之间的通信。把这些特性整合在一起，就形成了整个concurrent包得以实现的基石。

如果我们仔细分析concurrent包的源代码实现，会发现一个通用化的实现模式。

首先，声明共享变量为volatile。

然后，使用CAS的原子条件更新来实现线程之间的同步。

同时，配合以volatile的读/写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。

AQS,非阻塞数据结构和原子变量类（(java.util.concurrent.atomic包中的类）这些concurrent 包中的基础类都是使用这种模式来实现的，而concurrent包中的高层类又是依赖于这些基础类来实现的。从整体来看concurrent包的实现示意图如下图：

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696664307187-761a5f70-8354-48f7-a359-2a1ffc83cad3.png)

## 3.6final域的内存语义

### 1.final域的重排序规则

对于final域，编译器和处理器要遵守两个重排序规则。

+ 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
+ 初次读取一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。

### 2.写final域的重排序规则

写final域的重排序规则可以确保：在对应引用为任意线程可见之前，对象的final域已经被正确的初始化过了，而普通域不具有这个保障。

### 3.读final域的重排序规则

读final域的重排序规则可以确保：在读一个对象的final域之前，一定会先读包含这个final域对象的引用。

### 4.final域为引用类型

对于引用类型，对编译器和处理器增加了一个约束：

在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作不能重排序。

在构造函数内部，不能让这个被构造对象的引用为其它线程所见，也就是对象引用不能在构造函数中“溢出”。

## 3.7 happens-before

### JMM的设计

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696671354475-d6ef1b52-4bfc-4543-a17e-6285c7ca07ed.png)

从图3-33可以看出两点,如下。

+ JMM向程序员提供的 happens-before规则能满足程序员的需求。JMM的 happens-before规则不但简单易懂，而且也向程序员提高了足够的内存可见性保证(有些内存可见性保证其实并不一定真实存在，比如上面的 Ahappens-before B).
+ JMM对编译器和处理器的约束已经尽可能的少。对于编译器和处理器，只要能保证单线程的结果和正确的多线程同步，你们俩怎么优化都行！

### happens-before的定义

+ 如果一个操作happens-before另外一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。
+ 两个操作之间存在happens-before关系，并不意味着Java平台的具体实现必须要按照happens-before关系指定的顺序来执行。**如果重排序之后的执行结果，与按happens-before关系来执行的结果一致，那么这种重排序并不非法（也就是说，JMM允许这种重排序）。**

正如前面所言，JMM其实是在遵循一个基本原则：<font style="color:#DF2A3F;">只要不改变程序的执行结果（指的是单线程程序和正确同步的多线程程序），编译器和处理器怎么优化都行。</font>JMM这么做的原因是：<u>程序员对于这两个操作是否真的被重排序并不关心，程序员关心的是程序执行时的语义不能被改变</u>（即执行结果不能被改变）。因此，happens-before关系本质上和as-if-serial语义是一回事。

·as-if-serial语义保证单线程内程序的执行结果不被改变，happens-before关系保证正确同步的多线程程序的执行结果不被改变。

·as-if-serial语义给编写单线程程序的程序员创造了一个幻境：<u>单线程程序是按程序的顺序来执行的</u>。happens-before关系给编写正确同步的多线程程序的程序员创造了一个幻境：<u>正确同步的多线程程序是按happens-before指定的顺序来执行的。</u>

**as-if-serial语义和happens-before这么做的目的，都是为了在不改变程序执行结果的前提下，尽可能地提高程序执行的并行度。**

### happens-before规则

1）程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。

2）监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。

3）volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。

4）传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。

5）start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。

6）join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。  
<u>as-if-serial语义保证了程序顺序规则。因此，可以把程序顺序规则看成是对as-if-serial语义的“封装”。</u>

## 3.8双重检查锁定与延迟初始化

### 1.双重检查锁定的由来

双重检查锁是常见的**<font style="color:#DF2A3F;">延迟初始化技术</font>**！

懒加载的单例模式，怎么来的？ ---  线程安全的问题怎么解决？

首先看线程不安全的单例模式：

```java
//TODO 线程不安全的
public static Instance getINSTANCE1() {
    if (INSTANCE == null) {
        INSTANCE = new Instance();
    }
    return INSTANCE;
}
```

然后我们可以用synchronized加锁解决这个问题

```java
//TODO 线程安全的  ---  synchronized
public static synchronized Instance getINSTANCE2() {
    if (INSTANCE == null) {
        INSTANCE = new Instance();
    }
    return INSTANCE;
}
```

但是，这样在多线程并发执行的情况下，会存在一个资源消耗过大的缺点，这时候我们可能需要一个更优的解决方案    ---> 双重检查锁，应运而生！

```java
private static Instance INSTANCE;
//TODO 线程不安全的  ---  双重检查锁
public static Instance getINSTANCE3() {
    if (INSTANCE == null) {
        synchronized (SingleTon.class) {
            if (INSTANCE == null) {
                INSTANCE = new Instance();
            }
        }
    }
    return INSTANCE;
}
```

### 2.问题的原因

但是这里有个问题，就是第6行，这个`<font style="color:#DF2A3F;">INSTANCE = new Instance();</font>`，这行代码可以分解成如下的三行代码：

```c
memory = allocate();　　// 1：分配对象的内存空间
ctorInstance(memory);　 // 2：初始化对象
instance = memory;　　 // 3：设置instance指向刚分配的内存地址
```

上面3行伪代码中的2和3之间，可能会被重排序（在一些JIT编译器上，这种重排序是真实发生的，详情见参考文献1的“Out-of-order writes”部分）。2和3之间重排序之后的执行时序如下。

```c
memory = allocate();　　// 1：分配对象的内存空间
instance = memory;　　 // 3：设置instance指向刚分配的内存地址
// 注意，此时对象还没有被初始化！
ctorInstance(memory);　 // 2：初始化对象
```

这样，重排序之后，可能会导致多线程运行的时候，某个线程拿到初始化一半的对象

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696681344893-1fec38da-fb64-4530-9cd5-c90985753e54.png)

由于单线程内要遵守intra-thread semantics，从而能保证A线程的执行结果不会被改变。但是，当线程A和B按图3-38的时序执行时，B线程将看到一个还没有被初始化的对象。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696682951688-9d4ba325-e45e-4344-ad3c-303045555a02.png)

这里A2和A3虽然重排序了，但Java内存模型的intra-thread semantics将确保A2一定会排在A4前面执行。因此，线程A的intra-thread semantics没有改变，但A2和A3的重排序，将导致线程B在B1处判断出instance不为空，线程B接下来将访问instance引用的对象。此时，线程B将会访问到一个还未初始化的对象。

在知晓了问题发生的根源之后，我们可以想出两个办法来实现线程安全的延迟初始化。

1）不允许2和3重排序。

2）允许2和3重排序，但不允许其他线程“看到”这个重排序。

### 3.基于volatile的解决方案

对于前面的基于双重检查锁定来实现延迟初始化的方案（指DoubleCheckedLocking示例代码），只需要做一点小的修改（把instance声明为volatile型），就可以实现线程安全的延迟初始化。

```java
private static volatile Instance INSTANCE;
//TODO 线程安全的  ---  双重检查锁
public static Instance getINSTANCE4() {
    if (INSTANCE == null) {
        synchronized (SingleTon.class) {
            if (INSTANCE == null) {
                INSTANCE = new Instance();
            }
        }
    }
    return INSTANCE;
}
```

volatile在多线程环境中会禁止后面那两行代码的重排序！！

### 4.基于类初始化的解决方案

**JVM在类的初始化阶段（即在Class被加载后，且被线程使用之前），会执行类的初始化。在执行类的初始化期间，JVM会去获取一个锁。这个锁可以同步多个线程对同一个类的初始化。**

基于这个特性，可以实现另一种线程安全的延迟初始化方案（这个方案被称之为Initialization On Demand Holder idiom）。

```java
public class InstanceFactory {
    private static class InstanceHolder {
        public static Instance instance = new Instance();
    }
    public static Instance getInstance() {
        return InstanceHolder.instance ;　　// 这里将导致InstanceHolder类被初始化
    }
}
```

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696682930969-bbe41bfe-7755-4d70-80fc-1ed75568b283.png)

这个方案的实质是：允许3.8.2节中的3行伪代码中的2和3重排序，但不允许非构造线程（这里指线程B）“看到”这个重排序。

初始化一个类，包括执行这个类的静态初始化和初始化在这个类中声明的静态字段。根据Java语言规范，在首次发生下列任意一种情况时，一个类或接口类型T将被立即初始化。

1）T是一个类，而且一个T类型的实例被创建。

2）T是一个类，且T中声明的一个静态方法被调用。

3）T中声明的一个静态字段被赋值。

4）T中声明的一个静态字段被使用，而且这个字段不是一个常量字段。

5）T是一个顶级类（Top Level Class，见Java语言规范的§7.6），而且一个断言语句嵌套在T内部被执行。

在InstanceFactory示例代码中，首次执行getInstance()方法的线程将导致InstanceHolder类被初始化（符合情况4）。

由于Java语言是多线程的，多个线程可能在同一时间尝试去初始化同一个类或接口（比如这里多个线程可能在同一时刻调用getInstance()方法来初始化InstanceHolder类）。因此，在Java中初始化一个类或者接口时，需要做细致的同步处理。

**Java语言规范规定，对于每一个类或接口C，都有一个唯一的初始化锁LC与之对应。从C到LC的映射，由JVM的具体实现去自由实现。JVM在类初始化期间会获取这个初始化锁，并且每个线程至少获取一次锁来确保这个类已经被初始化过了（事实上，Java语言规范允许JVM的具体实现在这里做一些优化，见后文的说明）。**

对于类或接口的初始化，Java语言规范制定了精巧而复杂的类初始化处理过程。Java初始化一个类或接口的处理过程如下（这里对类初始化处理过程的说明，省略了与本文无关的部分；同时为了更好的说明类初始化过程中的同步处理机制，笔者人为的把类初始化的处理过程分为了5个阶段）。

**第1阶段**：通过控制在Class对象上的同步（即获取Class对象的初始化锁），来控制类或接口的初始化。这个获取锁的线程会一直等待，知道当前线程能够获取到这个初始化锁。

**第2阶段**：线程A执行类的初始化，同时线程B在初始化锁对应的condition上等待。

**第3阶段**：线程A设置state=initialized，然后唤醒在condition中等待的所有线程。

**第4阶段**：线程B结束类的初始化处理。

线程A在第2阶段的A1执行类的初始化，并在第3阶段的A4释放初始化锁；线程B在第4阶段的B1获取同一个初始化锁，并在第4阶段的B4之后才开始访问这个类。根据Java内存模型规范的锁规则，这里将存在如下的happens-before关系。

这个happens-before关系将保证：<u>线程A执行类的初始化时的写入操作（执行类的静态初始化和初始化类中声明的静态字段），线程B一定能看到。</u>

**第5阶段**：线程C执行类的初始化的处理。

在第3阶段之后，类已经完成了初始化。因此线程C在第5阶段的类初始化处理过程相对简单一些（前面的线程A和B的类初始化处理过程都经历了**两次**锁获取-锁释放，而线程C的类初始化处理只需要经历一次锁获取-锁释放）。



通过对比基于volatile的双重检查锁定的方案和基于类初始化的方案，我们会发现基于类初始化的方案的实现代码更简洁。但**基于volatile的双重检查锁定的方案有一个额外的优势：除了可以对静态字段实现延迟初始化外，还可以对实例字段实现延迟初始化。**

字段延迟初始化降低了初始化类或创建实例的开销，但增加了访问被延迟初始化的字段的开销。在大多数时候，正常的初始化要优于延迟初始化。如果确实需要对实例字段使用线程安全的延迟初始化，请使用上面介绍的基于volatile的延迟初始化的方案；如果确实需要对静态字段使用线程安全的延迟初始化，请使用上面介绍的基于类初始化的方案。

### 3.9Java内存模型综述

不同的处理器的优化方案都不一样，那么他们对读写组合操作的执行顺序的放松程度也不一样（是否允许读写组合操作的重排序）,不同的处理器的内存模型是不一样的，但是JMM保证了不同的处理器在最后生成的字节码文件一样（通过插入内存屏障来实现！）。

### 1.处理器内存模型

由于常见的处理器内存模型比JMM要弱，Java编译器在生成字节码时，会在执行指令序列的适当位置插入内存屏障来限制处理器的重排序。

JMM屏蔽了不同处理器内存模型的差异，它在不同的处理器平台之上为Java程序员呈现了一个一致的内存模型。

### 2.各种内存模型之间的关系

JMM是一个语言级的内存模型，处理器内存模型是硬件级的内存模型，顺序一致性内存模型是一个理论参考模型。

### 3.JMM的内存可见性保证

按程序类型，Java程序的内存可见性保证可以分为下列3类。

·单线程程序。单线程程序不会出现内存可见性问题。编译器、runtime和处理器会共同确保单线程程序的执行结果与该程序在顺序一致性模型中的执行结果相同。

·正确同步的多线程程序。正确同步的多线程程序的执行将具有顺序一致性（程序的执行结果与该程序在顺序一致性内存模型中的执行结果相同）。这是JMM关注的重点，JMM通过限制编译器和处理器的重排序来为程序员提供内存可见性保证。

·未同步/未正确同步的多线程程序。JMM为它们提供了最小安全性保障：线程执行时读取到的值，要么是之前某个线程写入的值，要么是默认值（0、null、false）。

**只要多线程程序是正确同步的，JMM保证该程序在任意的处理器平台上的执行结果，与该程序在顺序一致性内存模型中的执行结果一致。**

### 4.JSR-133对旧内存模型的修补

JSR-133对JDK 5之前的旧内存模型的修补主要有两个。

**·**增强volatile的内存语义。**旧内存模型允许volatile变量与普通变量重排序。**JSR-133严格限制volatile变量与普通变量的重排序，**使volatile的写-读和锁的释放-获取具有相同的内存语义。**

**·**增强final的内存语义。在旧内存模型中，多次读取同一个final变量的值可能会不相同。为此，JSR-133为final增加了两个重排序规则。<u>在保证final引用不会从构造函数内逸出的情况下，final具有了初始化安全性。</u>

# 第四章 Java并发编程基础

## 4.1线程简介

### 1.什么是线程

现代操作系统在运行一个程序时，会为其创建一个进程。例如，启动一个Java程序，操作系统就会创建一个Java进程。现代操作系统调度的最小单元是线程，也叫轻量级进程（Light Weight Process），在一个进程里可以创建多个线程，这些**线程都拥有各自的计数器、堆栈和局部变量等属性**，并且能够访问共享的内存变量。

一个Java程序从main()方法开始执行，然后按照既定的代码逻辑执行，看似没有其他线程参与，但实际上Java程序天生就是多线程程序，因为执行main()方法的是一个名称为main的线程。下面使用JMX来查看一个普通的Java程序包含哪些线程，如代码清单4-1所示。

```java
public class MultiThread{
    public static void main(String[] args) {
        // 获取Java线程管理MXBean
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        // 不需要获取同步的monitor和synchronizer信息，仅获取线程和线程堆栈信息
        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
        // 遍历线程信息，仅打印线程ID和线程名称信息
        for (ThreadInfo threadInfo : threadInfos) {
            System.out.println("[" + threadInfo.getThreadId() + "] " + threadInfo.
                               getThreadName());
        }
    }
}
```

输出如下所示:

```java
[4] Signal Dispatcher　 // 分发处理发送给JVM信号的线程
[3] Finalizer　　　　 // 调用对象finalize方法的线程
[2] Reference Handler // 清除Reference的线程
[1] main　 　　　　 // main线程，用户程序入口
```

可以看到，一个Java程序的运行不仅仅是main()方法的运行，而是main线程和多个其他线程的同时运行。

### 2.为什么要使用多线程

（1）更多的处理器核心

（2）更快的响应时间

（3）更好的编程模型

### 3.线程优先级

### 4.线程的状态

Java线程在运行的生命周期中可能处于表4-1所示的6种不同的状态，在给定的一个时刻，线程只能处于其中的一个状态。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696763484641-eea23e2d-b4fd-48a0-94ef-e22f4f8c1f01.png)

Java程序运行中线程状态的具体含义。线程在自身的生命周期中，并不是固定地处于某个状态，而是随着代码的执行在不同的状态之间进行切换，Java线程状态变迁如图4-1示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696763573356-e515ca1a-7396-4f23-817d-92f1fefac6fb.png)

由图4-1中可以看到，线程创建之后，调用start()方法开始运行。当线程执行wait()方法之后，线程进入等待状态。进入等待状态的线程需要依靠其他线程的通知才能够返回到运行状态，而超时等待状态相当于在等待状态的基础上增加了超时限制，也就是超时时间到达时将会返回到运行状态。当线程调用同步方法时，在没有获取到锁的情况下，线程将会进入到阻塞状态。线程在执行Runnable的run()方法之后将会进入到终止状态。

**<font style="color:#DF2A3F;">注意：</font>****Java将操作系统中的运行和就绪两个状态合并称为运行状态。**阻塞状态是线程阻塞在进入synchronized关键字修饰的方法或代码块（获取锁）时的状态，但是阻塞在java.concurrent包中Lock接口的线程状态却是等待状态，因为java.concurrent包中Lock接口对于阻塞的实现均使用了LockSupport类中的相关方法。

### 5.Daemon线程

Daemon线程是一种支持型线程，因为它**主要被用作程序中后台调度以及支持性工作**。这意味着，当一个Java虚拟机中不存在非Daemon线程的时候，Java虚拟机将会退出。可以通过调用Thread.setDaemon(true)将线程设置为Daemon线程。

Daemon属性需要在启动线程之前设置，不能在启动线程之后设置。

**Daemon线程被用作完成支持性工作，但是在Java虚拟机退出时Daemon线程中的finally块并不一定会执行。**

注意　在构建Daemon线程时，不能依靠finally块中的内容来确保执行关闭或清理资源的逻辑。

## 4.2启动和终止线程

### 1.构造线程

在运行线程之前首先要构造一个线程对象，线程对象在构造的时候需要提供线程所需要的属性，如线程所属的线程组、线程优先级、是否是Daemon线程等信息。代码清单4-6所示的代码摘自java.lang.Thread中对线程进行初始化的部分。

```java
private void init(ThreadGroup g, Runnable target, String name,long stackSize,
                  AccessControlContext acc) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }
    // 当前线程就是该线程的父线程
    Thread parent = currentThread();
    this.group = g;
    // 将daemon、priority属性设置为父线程的对应属性
    this.daemon = parent.isDaemon();
    this.priority = parent.getPriority();
    this.name = name.toCharArray();
    this.target = target;
    setPriority(priority);
    // 将父线程的InheritableThreadLocal复制过来
    if (parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals=ThreadLocal.createInheritedMap(parent.
                                                                    inheritableThreadLocals);
    // 分配一个线程ID
    tid = nextThreadID();
}
```

在上述过程中，**一个新构造的线程对象是由其parent线程来进行空间分配的**，而child线程继承了parent是否为Daemon、优先级和加载资源的contextClassLoader以及可继承的ThreadLocal，同时还会分配一个唯一的ID来标识这个child线程。至此，一个能够运行的线程对象就初始化好了，在堆内存中等待着运行。

### 2.启动线程

线程对象在初始化完成之后，调用start()方法就可以启动这个线程。线程start()方法的含义是：当前线程（即parent线程）同步告知Java虚拟机，只要线程规划器空闲，应立即启动调用start()方法的线程。

### 3.理解中断

中断可以理解为线程的一个标识位属性，它表示一个运行中的线程是否被其他线程进行了中断操作。中断好比其他线程对该线程打了个招呼，其他线程通过调用该线程的interrupt()方法对其进行中断操作。

线程通过检查自身是否被中断来进行响应，线程通过方法isInterrupted()来进行判断是否被中断，也可以调用静态方法Thread.interrupted()对当前线程的中断标识位进行复位。如果该线程已经处于终结状态，即使该线程被中断过，在调用该线程对象的isInterrupted()时依旧会返回false。

从Java的API中可以看到，许多声明抛出InterruptedException的方法（例如Thread.sleep(longmillis)方法）这些方法在抛出InterruptedException之前，Java虚拟机会先将该线程的中断标识位清除，然后抛出InterruptedException，此时调用isInterrupted()方法将会返回false。

### 4.过期的suspend()、resume()和stop()

不建议使用的原因主要有：以suspend()方法为例，**在调用后，线程不会释放已经占有的资源（比如锁）**，而是占有着资源进入睡眠状态，这样容易引发死锁问题。同样，stop()方法在终结一个线程时不会保证线程的资源正常释放，通常是没有给予线程完成资源释放工作的机会，因此会导致程序可能工作在不确定状态下。

注意　正因为suspend()、resume()和stop()方法带来的副作用，这些方法才被标注为不建议使用的过期方法，而暂停和恢复操作可以用后面提到的等待/通知机制来替代。

### 5.安全的终止线程

## 4.3　线程间通信

### 　1.volatile和synchronized关键字

关键字volatile可以用来修饰字段（成员变量），就是告知程序任何对该变量的访问均需要从共享内存中获取，而对它的改变必须同步刷新回共享内存，它能保证所有线程对变量访问的可见性。

但是，过多地使用volatile是不必要的，因为它会降低程序执行的效率。

关键字synchronized可以修饰方法或者以同步块的形式来进行使用，它主要确保多个线程在同一个时刻，只能有一个线程处于方法或者同步块中，它保证了线程对变量访问的可见性和排他性。

**<font style="color:#DF2A3F;">通过使用javap工具查看生成的class文件信息来分析synchronized关键字的实现细节！</font>**

```java
public class Synchronized {
    public static void main(String[] args) {
        // 对Synchronized Class对象进行加锁
        synchronized (Synchronized.class) {
            
        }
        // 静态同步方法，对Synchronized Class对象进行加锁
        m();
    }
    public static synchronized void m() {
        
    }
}
```

在Synchronized.class同级目录执行`javap–v Synchronized.class`，部分相关输出如下所示：

```java
public static void main(java.lang.String[]);
    // 方法修饰符，表示：public staticflags: ACC_PUBLIC, ACC_STATIC
    Code:
        stack=2, locals=1, args_size=1
        0: ldc #1　　// class com/murdock/books/multithread/book/Synchronized
        2: dup
        3: monitorenter　　// monitorenter：监视器进入，获取锁
        4: monitorexit　　 // monitorexit：监视器退出，释放锁
        5: invokestatic　　#16 // Method m:()V
        8: return
    public static synchronized void m();
    // 方法修饰符，表示： public static synchronized
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
    	Code:
            stack=0, locals=0, args_size=0
            0: return
```

上面class信息中，对于**<font style="color:#2F4BDA;">同步块</font>**的实现使用了**<font style="color:#DF2A3F;">monitorenter</font>**和**<font style="color:#DF2A3F;">monitorexit</font>**指令，而**<font style="color:#2F4BDA;">同步方法</font>**则是依靠**<font style="color:#DF2A3F;">方法修饰符上的ACC_SYNCHRONIZED</font>**来完成的。无论采用哪种方式，其<font style="color:#DF2A3F;">本质</font>是对一个对象的监视器（monitor）进行获取，而这个获取过程是排他的，也就是<u>同一时刻只能有一个线程获取到由synchronized所保护对象的监视器。</u>

任意一个对象都拥有自己的监视器，当这个对象由同步块或者这个对象的同步方法调用时，执行方法的线程必须先获取到该对象的监视器才能进入同步块或者同步方法，而没有获取到监视器（执行该方法）的线程将会被阻塞在同步块和同步方法的入口处，进入BLOCKED状态。

图4-2描述了对象、对象的监视器、同步队列和执行线程之间的关系。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696766326325-5434fcd1-a1e3-43d1-adf6-dac0cd6ded16.png)

从图4-2中可以看到，任意线程对Object（Object由synchronized保护）的访问，首先要获得Object的监视器。如果获取失败，线程进入同步队列，线程状态变为BLOCKED。当访问Object的前驱（获得了锁的线程）释放了锁，则该释放操作唤醒阻塞在同步队列中的线程，使其重新尝试对监视器的获取。

### 2.等待/通知机制

#### 我们需要实现线程之间的通信，对吧，那么一个线程发出消息（生产者），另外一个线程接收消息（消费者），怎么实现？怎么优化？

+ **首先想到的是。我们让消费者中设置一个不满足条件的循环，当条件满足，退出循环。**

```java
while (value != desire) {
    Thread.sleep(1000);
} doSomething();
```

上面这段伪代码在条件不满足时就睡眠一段时间，这样做的目的是防止过快的“无效”尝试，这种方式看似能够解实现所需的功能，但是却存在如下问题。

1）难以确保及时性。在睡眠时，基本不消耗处理器资源，但是如果睡得过久，就不能及时发现条件已经变化，也就是及时性难以保证。

2）难以降低开销。如果降低睡眠的时间，比如休眠1毫秒，这样消费者能更加迅速地发现条件变化，但是却可能消耗更多的处理器资源，造成了无端的浪费。

+ **wait()、notify()以及notifyAll()**

例子主要说明了调用wait()、notify()以及notifyAll()时需要注意的细节，如下。

1）使用wait()、notify()和notifyAll()时<font style="color:#DF2A3F;">需要先对调用对象加锁。</font>

2）调用wait()方法后，线程状态由RUNNING变为WAITING，并将当前线程放置到对象的等待队列。

3）notify()或notifyAll()方法调用后，等待线程依旧不会从wait()返回，需要调用notify()或notifAll()的线程释放锁之后，等待线程才有机会从wait()返回。

4）notify()方法将等待队列中的一个等待线程从等待队列中移到同步队列中，而notifyAll()方法则是将等待队列中所有的线程全部移到同步队列，被移动的线程状态由WAITING变为BLOCKED。

5）从wait()方法返回的前提是获得了调用对象的锁。

从上述细节中可以看到，等待/通知机制依托于同步机制，其目的就是确保等待线程从wait()方法返回时能够感知到通知线程对变量做出的修改。

### 3.等待/通知的经典范式   加锁 - 条件循环 - 通知和处理逻辑

从示例中可以提炼出等待/通知的经典范式，该范式分为两部分，分别针对等待方（消费者）和通知方（生产者）。

等待方遵循如下原则。

1）获取对象的锁。

2）如果条件不满足，那么调用对象的wait()方法，被通知后仍要检查条件。

3）条件满足则执行对应的逻辑。

对应的伪代码如下。

```java
synchronized(对象) {
    while(条件不满足) {
        对象.wait();
    }
    对应的处理逻辑
}
```

通知方遵循如下原则。

1）获得对象的锁。

2）改变条件。

3）通知所有等待在对象上的线程。

对应的伪代码如下。

```java
synchronized(对象) {
    改变条件
    对象.notifyAll();
}
```

### 4.管道输入/输出流

管道输入/输出流和普通的文件输入/输出流或者网络输入/输出流不同之处在于，它主要用于线程之间的数据传输，而传输的媒介为内存。

管道输入/输出流主要包括了如下4种具体实现：PipedOutputStream、PipedInputStream、PipedReader和PipedWriter，前两种面向字节，而后两种面向字符。

```java
public class Piped {
    public static void main(String[] args) throws Exception {
        PipedWriter out = new PipedWriter();
        PipedReader in = new PipedReader();
        // 将输出流和输入流进行连接，否则在使用时会抛出IOException
        out.connect(in);
        Thread printThread = new Thread(new Print(in), "PrintThread");
        printThread.start();
        int receive = 0;
        try {
            while ((receive = System.in.read()) != -1) {
                out.write(receive);
            }
        } finally {
            out.close();
        }
    }
    static class Print implements Runnable {
        private PipedReader in;
        public Print(PipedReader in) {
            this.in = in;
        }
        public void run() {
            int receive = 0;
            try {
                while ((receive = in.read()) != -1) {
                    System.out.print((char) receive);
                }
            } catch (IOException ex) {
            }
        }
    }
}
```

**<font style="color:#DF2A3F;">对于Piped类型的流，必须先要进行绑定，也就是调用connect()方法，如果没有将输入/输出流绑定起来，对于该流的访问将会抛出异常。</font>**

### 5.Thread.join()的使用

如果一个线程A执行了thread.join()语句，其含义是：当前线程A等待thread线程终止之后才从thread.join()返回。线程Thread除了提供join()方法之外，还提供了join(long millis)和join(long millis,int nanos)两个具备超时特性的方法。这两个超时方法表示，如果线程thread在给定的超时时间里没有终止，那么将会从该超时方法中返回。

JDK中Thread.join()方法的源码（进行了部分调整）。

```java
// 加锁当前线程对象
public final synchronized void join() throws InterruptedException {
    // 条件不满足，继续等待
    while (isAlive()) {
        wait(0);
    }
    // 条件符合，方法返回
}
```

当线程终止时，会调用线程自身的notifyAll()方法，会通知所有等待在该线程对象上的线程。可以看到join()方法的逻辑结构与4.3.3节中描述的等待/通知经典范式一致，即加锁、循环和处理逻辑3个步骤。

### 6.ThreadLocal的使用

ThreadLocal，即线程变量，是一个以ThreadLocal对象为键、任意对象为值的存储结构。这个结构被附带在线程上，也就是说一个线程可以根据一个ThreadLocal对象查询到绑定在这个线程上的一个值。

可以通过set(T)方法来设置一个值，在当前线程下再通过get()方法获取到原先设置的值。

## 4.4线程应用实例

### 1.等待超时模式

开发人员经常会遇到这样的方法调用场景：调用一个方法时等待一段时间（一般来说是给定一个时间段），如果该方法能够在给定的时间段之内得到结果，那么将结果立刻返回，反之，超时返回默认结果。

前面的章节介绍了等待/通知的经典范式，即加锁、条件循环和处理逻辑3个步骤，而这种范式无法做到超时等待。而超时等待的加入，只需要对经典范式做出非常小的改动，改动内容如下所示。

假设超时时间段是T，那么可以推断出在当前时间now+T之后就会超时。

定义如下变量。

·等待持续时间：REMAINING=T。

·超时时间：FUTURE=now+T。

这时仅需要wait(REMAINING)即可，在wait(REMAINING)返回之后会将执行：

REMAINING=FUTURE–now。如果REMAINING小于等于0，表示已经超时，直接退出，否则将继续执行wait(REMAINING)。

```java
// 对当前对象加锁
public synchronized Object get(long mills) throws InterruptedException {
    long future = System.currentTimeMillis() + mills;
    long remaining = mills;
    // 当超时大于0并且result返回值不满足要求
    while ((result == null) && remaining > 0) {
        wait(remaining);
        remaining = future - System.currentTimeMillis();
    }
    return result;
}
```

可以看出，等待超时模式就是在等待/通知范式基础上增加了超时控制，这使得该模式相比原有范式更具有灵活性，因为即使方法执行时间过长，也不会“永久”阻塞调用者，而是会按照调用者的要求“按时”返回。

### 2.手撕线程池

一个简单的线程池接口定义:

```java
public interface ThreadPool<Job extends Runnable> {
    // 执行一个Job，这个Job需要实现Runnable
    void execute(Job job);
    // 关闭线程池
    void shutdown();
    // 增加工作者线程
    void addWorkers(int num);
    // 减少工作者线程
    void removeWorker(int num);
    // 得到正在等待执行的任务数量
    int getJobSize();
}
```

接下来是线程池接口的默认实现:

```java
public class DefaultThreadPool<Job extends Runnable> implements ThreadPool<Job> {
    // 线程池最大限制数
    private static final int MAX_WORKER_NUMBERS = 10;
    // 线程池默认的数量
    private static final int DEFAULT_WORKER_NUMBERS = 5;
    // 线程池最小的数量
    private static final int MIN_WORKER_NUMBERS = 1;
    // 这是一个工作列表，将会向里面插入工作
    private final LinkedList<Job> jobs = new LinkedList<Job>();
    // 工作者列表
    private final List<Worker> workers = Collections.synchronizedList(new ArrayList<Worker>());
    // 工作者线程的数量
    private int workerNum = DEFAULT_WORKER_NUMBERS;
    // 线程编号生成
    private AtomicLong threadNum = new AtomicLong();
    public DefaultThreadPool() {
        initializeWokers(DEFAULT_WORKER_NUMBERS);
    }
    public DefaultThreadPool(int num) {
        workerNum = num > MAX_WORKER_NUMBERS MAX_WORKER_NUMBERS : num < MIN_WORKER_
        NUMBERS MIN_WORKER_NUMBERS : num;
        initializeWokers(workerNum);
    }
    public void execute(Job job) {
        if (job != null) {
            // 添加一个工作，然后进行通知
            synchronized (jobs) {
                jobs.addLast(job);
                jobs.notify();
            }
        }
    }
    public void shutdown() {
        for (Worker worker : workers) {
            worker.shutdown();
        }
    }
    public void addWorkers(int num) {
        synchronized (jobs) {
            // 限制新增的Worker数量不能超过最大值
            if (num + this.workerNum > MAX_WORKER_NUMBERS) {
                num = MAX_WORKER_NUMBERS - this.workerNum;
            }
            initializeWokers(num);
            this.workerNum += num;
        }
    }
    public void removeWorker(int num) {
        synchronized (jobs) {
            if (num >= this.workerNum) {
                throw new IllegalArgumentException("beyond workNum");
            }
            // 按照给定的数量停止Worker
            int count = 0;
            while (count < num) {
                Worker worker = workers.get(count)
                if (workers.remove(worker)) {
                    worker.shutdown();
                    count++;
                }
            }
            this.workerNum -= count;
        }
    }
    public int getJobSize() {
        return jobs.size();
    }
    // 初始化线程工作者
    private void initializeWokers(int num) {
        for (int i = 0; i < num; i++) {
            Worker worker = new Worker();
            workers.add(worker);
            Thread thread = new Thread(worker, "ThreadPool-Worker-" + threadNum.
                                       incrementAndGet());
            thread.start();
        }
    }
    // 工作者，负责消费任务
    class Worker implements Runnable {
        // 是否工作
        private volatile boolean running = true;
        public void run() {
            while (running) {
                Job job = null;
                synchronized (jobs) {
                    // 如果工作者列表是空的，那么就wait
                    while (jobs.isEmpty()) {
                        try {
                            jobs.wait();
                        } catch (InterruptedException ex) {
                            // 感知到外部对WorkerThread的中断操作，返回
                            Thread.currentThread().interrupt();
                            return;
                        }
                    }
                    // 取出一个Job
                    job = jobs.removeFirst();
                }
                if (job != null) {
                    try {
                        job.run();
                    } catch (Exception ex) {
                        // 忽略Job执行中的Exception
                    }
                }
            }
        }
        public void shutdown() {
            running = false;
        }
    }
}
```

从线程池的实现可以看到，当客户端调用execute(Job)方法时，会不断地向任务列表jobs中添加Job，而每个工作者线程会不断地从jobs上取出一个Job进行执行，当jobs为空时，工作者线程进入等待状态。

添加一个Job后，对工作队列jobs调用了其notify()方法，而不是notifyAll()方法，因为能够确定有工作者线程被唤醒，这时使用notify()方法将会比notifyAll()方法获得更小的开销（避免将等待队列中的线程全部移动到阻塞队列中）。

可以看到，**线程池的本质就是使用了一个线程安全的工作队列连接工作者线程和客户端线程**，客户端线程将任务放入工作队列后便返回，而工作者线程则不断地从工作队列上取出工作并执行。当工作队列为空时，所有的工作者线程均等待在工作队列上，当有客户端提交了

一个任务之后会通知任意一个工作者线程，随着大量的任务被提交，更多的工作者线程会被唤醒。

# 第五章 Java中的锁

## 5.1Lock接口

锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能够防止多个线程同时访问共享资源（但是有些锁可以<u>允许多个线程并发的访问共享资源</u>，比如读写锁）。在Lock接口出现之前，Java程序是靠synchronized关键字实现锁功能的，而Java SE 5之后，并发包中新增了Lock接口（以及相关实现类）用来实现锁功能，它提供了与synchronized关键字类似的同步功能，只是在使用时需要显式地获取和释放锁。

虽然它缺少了（通过synchronized块或者方法所提供的）隐式获取释放锁的便捷性，但是却拥有了锁获取与释放的可操作性、可中断的获取锁以及超时获取锁等多种synchronized关键字所不具备的同步特性。

使用synchronized关键字将会隐式地获取锁，但是它将锁的获取和释放固化了，也就是先获取再释放。当然，这种方式简化了同步的管理，可是扩展性没有显示的锁获取和释放来的好。

lock的使用方式：

```java
Lock lock = new ReentrantLock();
lock.lock();
try {
} finally {
    lock.unlock();
}
```

在finally块中释放锁，目的是保证在获取到锁之后，最终能够被释放。

**<font style="color:#0C68CA;">不要将获取锁的过程写在try块中，因为如果在获取锁（自定义锁的实现）时发生了异常，异常抛出的同时，也会导致锁无故释放。</font>**

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696933271862-a8cb7809-0b67-43f1-9d77-c5240b8b1a8c.png)

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696933429396-d89b4551-7e3a-498c-8869-566db57403ca.png)

这里先简单介绍一下Lock接口的API，随后的章节会详细介绍同步器AbstractQueuedSynchronizer以及常用Lock接口的实现ReentrantLock。**Lock接口的实现基本都是通过聚合了一个同步器的子类**来完成线程访问控制的。

## 5.2队列同步器

队列同步器AbstractQueuedSynchronizer（以下简称同步器），是用来构建锁或者其他同步组件的基础框架，它**使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作**，并发包的（Doug Lea）期望它能够成为实现大部分同步需求的基础。

<u>同步器的主要使用方式是继承</u>，子类通过继承同步器并实现它的抽象方法来管理同步状态，在抽象方法的实现过程中免不了要对同步状态进行更改，这时就需要使用同步器提供的3个方法（<u>getState()、setState(int newState)和compareAndSetState(int expect,int update)</u>）来进行操作，因为它们能够保证状态的改变是安全的。

<u>子类推荐被定义为</u><u><font style="background-color:#F9EFCD;">自定义同步组件（就是自己业务代码定义的那个锁）</font></u><u>的静态内部类</u>，同步器自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获取同步状态，这样就可以方便实现不同类型的同步组件（ReentrantLock、ReentrantReadWriteLock和CountDownLatch等）。

<u>同步器是实现锁（也可以是任意同步组件）的关键</u>，在锁的实现中聚合同步器，利用同步器实现锁的语义。可以这样理解二者之间的关系：**锁是面向使用者的**，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；**同步器面向的是锁的实现者**，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。锁和同步器很好地隔离了使用者和实现者所需关注的领域。

### 1.队列同步器的接口与示例

同步器的设计是基于模板方法模式的，也就是说，**使用者需要继承同步器并重写指定的方法(表5-3)**，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法（表5-4），而这些模板方法将会调用使用者重写的方法。

**重写同步器指定的方法时，需要使用同步器提供的如下3个方法来访问或修改同步状态。**

+ getState()：获取当前同步状态。
+ setState(int newState)：设置当前同步状态。
+ compareAndSetState(int expect,int update)：使用CAS设置当前状态，该方法能够保证状态设置的原子性。

同步器可重写的方法与描述如表5-3所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696937495828-a3860361-5a75-4c02-a739-a70b96451c44.png)

实现自定义同步组件时，将会调用同步器提供的模板方法，这些（部分）模板方法与描述如表5-4所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1696937720653-8885c061-5c0d-4a9f-ba45-18e717c33728.png)

同步器提供的模板方法基本上分为3类：

+ 独占式获取与释放同步状态；
+ 共享式获取与释放同步状态
+ 查询同步队列中的等待线程情况。

自定义同步组件将使用同步器提供的模板方法来实现自己的同步语义。

只有掌握了同步器的工作原理才能更加深入地理解并发包中其他的并发组件，所以下面通过一个独占锁的示例来深入了解一下同步器的工作原理。

顾名思义，独占锁就是在同一时刻只能有一个线程获取到锁，而其他获取锁的线程只能处于同步队列中等待，只有获取锁的线程释放了锁，后继的线程才能够获取锁。

```java
class Mutex implements Lock {
    // 静态内部类，自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        // 是否处于占用状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        // 当状态为0的时候获取锁
        public boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        // 释放锁，将状态设置为0
        protected boolean tryRelease(int releases) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        // 返回一个Condition，每个condition都包含了一个condition队列
        Condition newCondition() {
            return new ConditionObject();
        }
    }

    // 仅需要将操作代理到Sync上即可
    private final Sync sync = new Sync();

    public void lock() {
        sync.acquire(1);
    }

    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    public void unlock() {
        sync.release(1);
    }

    public Condition newCondition() {
        return sync.newCondition();
    }

    public boolean isLocked() {
        return sync.isHeldExclusively();
    }

    public boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
}
```

上述示例中，独占锁Mutex是一个自定义同步组件，它在同一时刻只允许一个线程占有锁。

**Mutex中定义了一个静态内部类，该内部类继承了同步器并实现了独占式获取和释放同步状态。**

在tryAcquire(int acquires)方法中，如果经过CAS设置成功（同步状态设置为1），则代表获取了同步状态，而在tryRelease(int releases)方法中只是将同步状态重置为0。

用户使用Mutex时并不会直接和内部同步器的实现打交道，而是调用Mutex提供的方法，在Mutex的实现中，以获取锁的lock()方法为例，只需要在方法实现中调用同步器的模板方法acquire(int args)即可，当前线程调用该方法获取同步状态失败后会被加入到同步队列中等待，这样就大大降低了实现一个可靠自定义同步组件的门槛。

### 2.队列同步器的实现分析

**<font style="color:#DF2A3F;">首先明确一个名词：获取同步状态或者，处于同步状态就是获取锁！！！</font>**

接下来将从实现角度分析同步器是如何完成线程同步的，主要包括：

+ 同步队列；
+ 独占式同步状态获取与释放；
+ 共享式同步状态获取与释放；
+ 超时获取同步状态；

等同步器的核心数据结构与模板方法。

#### a)同步队列

**同步器依赖内部的同步队列（一个FIFO双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。**

同步队列中的节点（Node）用来保存获取同步状态失败的线程引用、等待状态以及前驱和后继节点，节点的属性类型与名称以及描述如表5-5所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697019146302-9cb7166f-905f-4a8f-baae-f76e07993cee.png)

节点是构成同步队列（等待队列，在5.6节中将会介绍）的基础，**<font style="color:#DF2A3F;">同步器拥有首节点（head）和尾节点（tail）</font>**，没有成功获取同步状态的线程将会成为节点加入该队列的尾部，同步队列的基本结构如图5-1所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697019612871-1378fa68-cb70-4acd-b930-5e705e3a41ab.png)

在图5-1中，同步器包含了两个节点类型的引用，一个指向头节点，而另一个指向尾节点。试想一下，当一个线程成功地获取了同步状态（或者锁），其他线程将无法获取到同步状态，转而被构造成为节点并加入到同步队列中，而这个加入队列的过程必须要保证线程安全，因此同步器提供了一个基于CAS的设置尾节点的方法：compareAndSetTail(Node expect,Node update)，它需要传递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联。

同步器将节点加入到同步队列的过程如图5-2所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697020417075-8ed9af8a-be48-4947-b4e7-7134eb8af8fb.png)

同步队列遵循FIFO，**首节点是获取同步状态成功的节点**，首节点的线程在释放同步状态时，将会唤醒后继节点，而后继节点将会在获取同步状态成功时将自己设置为首节点，该过程如图5-3所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697020812366-7afdd8ed-8340-4da1-9018-57b75d138703.png)

在图5-3中，**设置首节点是通过获取同步状态成功的线程来完成的**，由于只有一个线程能够成功获取到同步状态，因此设置头节点的方法并不需要使用CAS来保证，它只需要将首节点设置成为原首节点的后继节点并断开原首节点的next引用即可。

#### b)独占式同步状态获取与释放

通过调用同步器的acquire(int arg)方法可以获取同步状态，**该方法对中断不敏感，也就是由于线程获取同步状态失败后进入同步队列中，后续对线程进行中断操作时，线程不会从同步队列中移出**，该方法代码如代码清单5-3所示。

```java
public final void acquire(int arg) {
if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

上述代码主要完成了**同步状态获取**、**节点构造**、**加入同步队列**以及**在同步队列中****<font style="background-color:#FBDE28;">自旋等待</font>**的相关工作，其主要逻辑是：首先调用自定义同步器实现的tryAcquire(int arg)方法，该方法保证线程安全的获取同步状态，如果同步状态获取失败，则构造同步节点（独占式Node.EXCLUSIVE，同一时刻只能有一个线程成功获取同步状态）并通过addWaiter(Node node)方法将该节点加入到同步队列的尾部，**<font style="color:#DF2A3F;">最后调用acquireQueued(Node node,int arg)方法，使得该节点以“死循环”的方式获取同步状态。</font>**如果获取不到则阻塞节点中的线程，而被阻塞线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现。

下面分析一下相关工作。首先是节点的构造以及加入同步队列：

同步器的addWaiter和enq方法:

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // 快速尝试在尾部添加
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (; ; ) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

上述代码通过使用compareAndSetTail(Node expect,Node update)方法来确保节点能够被线程安全添加。试想一下：如果使用一个普通的LinkedList来维护节点之间的关系，那么当一个线程获取了同步状态，而其他多个线程由于调用tryAcquire(int arg)方法获取同步状态失败而并发地被添加到LinkedList时，LinkedList将难以保证Node的正确添加，最终的结果可能是节点的数量有偏差，而且顺序也是混乱的。

在enq(final Node node)方法中，同步器通过“死循环”来保证节点的正确添加，在“死循环”中只有通过CAS将节点设置成为尾节点之后，当前线程才能从该方法返回，否则，当前线程不断地尝试设置。可以看出，**enq(final Node node)方法将并发添加节点的请求通过CAS变得“串行化”了。**

**<font style="color:#07787E;">节点进入同步队列之后，就进入了一个自旋的过程，每个节点（或者说每个线程）都在自省地观察，当条件满足，获取到了同步状态，就可以从这个自旋过程中退出，否则依旧留在这个自旋过程中（并会阻塞节点的线程）</font>**，如代码清单5-5所示。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
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

在acquireQueued(final Node node,int arg)方法中，**当前线程在“死循环”中尝试获取同步状态，而只有前驱节点是头节点才能够尝试获取同步状态**，这是为什么？原因有两个，如下。

+ 第一，头节点是成功获取到同步状态的节点，而头节点的线程释放了同步状态之后，将会唤醒其后继节点，后继节点的线程被唤醒后需要检查自己的前驱节点是否是头节点。
+ 第二，维护同步队列的FIFO原则。该方法中，节点自旋获取同步状态的行为如图5-4所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697029546936-a362ab9c-3558-4ccf-af33-f40104953c90.png)

在图5-4中，由于非首节点线程前驱节点出队或者被中断而从等待状态返回，随后检查自己的前驱是否是头节点，如果是则尝试获取同步状态。可以看到节点和节点之间在循环检查的过程中基本不相互通信，而是简单地判断自己的前驱是否为头节点，这样就使得节点的释放规则符合FIFO，并且也便于对过早通知的处理（**过早通知是指前驱节点不是头节点的线程由于中断而被唤醒**）。

独占式同步状态获取流程，也就是acquire(int arg)方法调用流程，如图5-5所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697029906804-6960aac2-a6a9-41e4-87b1-6f287763e654.png)

在图5-5中，前驱节点为头节点且能够获取同步状态的判断条件和线程进入等待状态是获取同步状态的自旋过程。当同步状态获取成功之后，当前线程从acquire(int arg)方法返回，如果对于锁这种并发组件而言，代表着当前线程获取了锁。

当前线程获取同步状态并执行了相应逻辑之后，就需要释放同步状态，使得后续节点能够继续获取同步状态。通过调用同步器的release(int arg)方法可以释放同步状态，该方法在释放了同步状态之后，会唤醒其后继节点（进而使后继节点重新尝试获取同步状态）。该方法代码如代码清单5-6所示。

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

该方法执行时，会唤醒头节点的后继节点线程，unparkSuccessor(Node node)方法使用LockSupport（在后面的章节会专门介绍）来唤醒处于等待状态的线程。

分析了独占式同步状态获取和释放过程后，适当做个总结：在获取同步状态时，同步器维护一个同步队列，获取状态失败的线程都会被加入到队列中并在队列中进行自旋；移出队列（或停止自旋）的条件是前驱节点为头节点且成功获取了同步状态。在释放同步状态时，同步器调用tryRelease(int arg)方法释放同步状态，然后唤醒头节点的后继节点。

##### 问：在同步队列中，队头节点获得锁，当其释放锁的时候，只有其后一个节点才能获取锁，那么再后面的节点为什么要自旋？而不是阻塞呢？

您提出的问题是非常合理的，让我们进一步澄清一下同步队列中的情况。

在典型的同步队列中，只有队头节点能够直接获得锁。当队头节点释放锁时，通常只有其后的一个节点（紧随其后的节点）才能获得锁。这是为了保持公平性，即等待时间较长的线程有更高的机会获得锁，避免出现线程饥饿的情况。

对于队头之后的节点，它们需要自旋等待的原因包括：

1.  **竞争锁**：即使队头节点释放了锁，后续的节点仍然必须竞争，因为在队头节点释放锁和后续节点获取锁之间的时间间隔是非常短暂的。这种竞争是为了确保多个线程争夺锁的公平性和效率。
2.  **减少上下文切换**：在许多情况下，自旋等待可以减少线程之间的上下文切换开销。如果一个线程进入休眠状态，等待锁的释放，操作系统可能会将CPU资源分配给其他线程，然后再唤醒它。这会涉及到线程的切换开销。自旋等待可以减少这种开销，因为线程一直在CPU上自旋等待，不需要进入休眠状态。
3.  **降低等待时间**：自旋等待通常是短暂的，因为队头节点通常会很快释放锁。这意味着后续节点只需进行短暂的自旋，以获得锁，而不必进入较长的休眠状态。

虽然队头之后的节点需要自旋，但通常情况下，自旋的时间非常短暂，因为锁的释放和获取是非常迅速的。这种机制可以在保持公平性的同时，提高锁的并发性能。如果自旋等待时间过长或出现频繁的锁争用，可能会导致性能问题，但在许多情况下，自旋等待是一种有效的方式来管理同步。

##### <font style="color:rgb(52, 53, 65);">问：如果后面的节点被打断唤醒，是不是也得等到前面所有的节点都出队列之后，才能响应中断？</font>

在同步队列中，如果一个后续节点（非队头节点）被打断唤醒，它**不需要等到前面所有的节点都出队列才能响应中断。**唤醒后，该节点会尝试获取锁，但在获取之前，它会检查中断标志并在必要时响应中断。

如果线程在自旋等待过程中被打断，通常它会检查中断标志（例如通过`Thread.interrupted()`或`Thread.currentThread().isInterrupted()`方法），然后决定是否继续等待或中断自己的执行。这允许线程能够在被打断后及时响应中断请求，而不必等待前面的节点都出队列。

中断处理是一个重要的线程管理和协作机制，能够帮助避免线程无限等待，陷入死锁，或者能够及时停止执行任务以响应外部的中断请求。因此，**在同步队列中，被打断唤醒的线程可以根据自身需要和中断状态来响应中断，而不必等待前面的节点都出队列。这有助于提高线程协作和响应性。**

#### c)共享式同步状态获取与释放

共享式获取与独占式获取最主要的区别在于同一时刻能否有多个线程同时获取到同步状态。以文件的读写为例，如果一个程序在对文件进行读操作，那么这一时刻对于该文件的写操作均被阻塞，而读操作能够同时进行。写操作要求对资源的独占式访问，而读操作可以是共享式访问，两种不同的访问模式在同一时刻对文件或资源的访问情况，如图5-6所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697033030804-ac8632cc-290c-454e-b500-bb62db3c6867.png)

在图5-6中，左半部分，共享式访问资源时，其他共享式的访问均被允许，而独占式访问被阻塞，右半部分是独占式访问资源时，同一时刻其他访问均被阻塞。通过调用同步器的acquireShared(int arg)方法可以共享式地获取同步状态，该方法代码如代码清单5-7所示。

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
} 
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null;
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
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

在acquireShared(int arg)方法中，同步器调用tryAcquireShared(int arg)方法尝试获取同步状态，tryAcquireShared(int arg)方法返回值为int类型，当返回值大于等于0时，表示能够获取到同步状态。因此，在共享式获取的自旋过程中，成功获取到同步状态并退出自旋的条件就是tryAcquireShared(int arg)方法返回值大于等于0。可以看到，在doAcquireShared(int arg)方法的自旋过程中，如果当前节点的前驱为头节点时，尝试获取同步状态，如果返回值大于等于0，表示该次获取同步状态成功并从自旋过程中退出。

与独占式一样，共享式获取也需要释放同步状态，通过调用releaseShared(int arg)方法可以释放同步状态，该方法代码如代码清单5-8所示。

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

该方法在释放同步状态之后，将会唤醒后续处于等待状态的节点。对于能够支持多个线程同时访问的并发组件（比如Semaphore），<font style="background-color:#FBDE28;">它和独占式主要区别在于tryReleaseShared(int arg)方法必须确保同步状态（或者资源数）线程安全释放，一般是通过循环和CAS来保证的，因为释放同步状态的操作会同时来自多个线程。</font>

##### <font style="color:rgb(52, 53, 65);">问：共享锁也是按照同步队列一个个来获取锁的吗</font>

共享锁通常不是按照同步队列中的顺序一个个来获取锁的，与独占锁的方式有所不同。在共享锁的情况下，多个线程可以同时获取锁，而不需要像独占锁那样一个一个等待。这是共享锁的关键特性之一。

当一个线程持有共享锁时，其他线程也可以同时持有相同的共享锁，而不会相互排斥。**这种方式允许多个线程同时执行具有共享锁的代码段，通常用于支持并发读取操作，提高性能。**

因此，共享锁不会像独占锁一样按照同步队列的顺序一个个来获取锁。取而代之的是，当多个线程尝试获取共享锁时，它们通常会同时成功获取锁，而不需要按照顺序等待。这是共享锁的核心特性，有助于提高多线程读取操作的并发性。

#### d)独占式超时获取同步状态

通过调用同步器的doAcquireNanos(int arg,long nanosTimeout)方法可以超时获取同步状态，即在指定的时间段内获取同步状态，如果获取到同步状态则返回true，否则，返回false。该方法提供了传统Java同步操作（比如synchronized关键字）所不具备的特性。

在分析该方法的实现前，先介绍一下响应中断的同步状态获取过程。在Java 5之前，当一个线程获取不到锁而被阻塞在synchronized之外时，对该线程进行中断操作，此时该线程的中断标志位会被修改，但线程依旧会阻塞在synchronized上，等待着获取锁。在Java 5中，同步器提供了acquireInterruptibly(int arg)方法，这个方法在等待获取同步状态时，如果当前线程被中断，会立刻返回，并抛出InterruptedException。

超**时获取同步状态过程可以被视作响应中断获取同步状态过程的“增强版”**，doAcquireNanos(int arg,long nanosTimeout)方法在支持响应中断的基础上，增加了超时获取的特性。针对超时获取，主要需要计算出需要睡眠的时间间隔nanosTimeout，为了防止过早通知，nanosTimeout计算公式为：nanosTimeout-=now-lastTime，其中now为当前唤醒时间，lastTime为上次唤醒时间，如果nanosTimeout大于0则表示超时时间未到，需要继续睡眠nanosTimeout纳秒，反之，表示已经超时，该方法代码如代码清单5-9所示。

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
throws InterruptedException {
    long lastTime = System.nanoTime();
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            if (nanosTimeout <= 0)
                return false;
            if (shouldParkAfterFailedAcquire(p, node)
                && nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            long now = System.nanoTime();
            //计算时间，当前时间now减去睡眠之前的时间lastTime得到已经睡眠
            //的时间delta，然后被原有超时时间nanosTimeout减去，得到了
            //还应该睡眠的时间
            nanosTimeout -= now - lastTime;
            lastTime = now;
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

该方法在自旋过程中，当节点的前驱节点为头节点时尝试获取同步状态，如果获取成功则从该方法返回，这个过程和独占式同步获取的过程类似，但是在同步状态获取失败的处理上有所不同。如果当前线程获取同步状态失败，则判断是否超时（nanosTimeout小于等于0表示已经超时），如果没有超时，重新计算超时间隔nanosTimeout，然后使当前线程等待nanosTimeout纳秒（当已到设置的超时时间，该线程会从LockSupport.parkNanos(Object

blocker,long nanos)方法返回）。

如果nanosTimeout小于等于spinForTimeoutThreshold（1000纳秒）时，将不会使该线程进行超时等待，而是进入快速的自旋过程。原因在于，非常短的超时等待无法做到十分精确，如果这时再进行超时等待，相反会让nanosTimeout的超时从整体上表现得反而不精确。因此，**在超时非常短的场景下，同步器会进入无条件的快速自旋。**

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697100110330-cbe27eef-6d84-4392-af32-77f551784d96.png)

从图5-7中可以看出，独占式超时获取同步状态doAcquireNanos(int arg,long nanosTimeout)和独占式获取同步状态acquire(int args)在流程上非常相似，其主要区别在于未获取到同步状态时的处理逻辑。acquire(int args)在未获取到同步状态时，将会使当前线程一直处于等待状态，而doAcquireNanos(int arg,long nanosTimeout)会使当前线程等待nanosTimeout纳秒，如果当前线程在nanosTimeout纳秒内没有获取到同步状态，将会从等待逻辑中自动返回。

#### e).自定义同步组件——TwinsLock

设计一个同步工具：该工具在同一时刻，只允许至多两个线程同时访问，超过两个线程的访问将被阻塞，我们将这个同步工具命名为TwinsLock。

首先，确定访问模式。TwinsLock能够在同一时刻支持多个线程的访问，这显然是共享式访问，因此，需要使用同步器提供的acquireShared(int args)方法等和Shared相关的方法，这就要求TwinsLock必须重写tryAcquireShared(int args)方法和tryReleaseShared(int args)方法，这样才能保证同步器的共享式同步状态的获取与释放方法得以执行。

其次，定义资源数。TwinsLock在同一时刻允许至多两个线程的同时访问，表明同步资源数为2，这样可以设置初始状态status为2，当一个线程进行获取，status减1，该线程释放，则status加1，状态的合法范围为0、1和2，其中0表示当前已经有两个线程获取了同步资源，此时再有其他线程对同步状态进行获取，该线程只能被阻塞。在同步状态变更时，需要使用compareAndSet(int expect,int update)方法做原子性保障。

最后，组合自定义同步器。前面的章节提到，自定义同步组件通过组合自定义同步器来完成同步功能，一般情况下自定义同步器会被定义为自定义同步组件的内部类。

```java
public class TwinsLock implements Lock {
    private final Sync sync = new Sync(2);
    private static final class Sync extends AbstractQueuedSynchronizer {
        Sync(int count) {
            if (count <= 0) {
                throw new IllegalArgumentException("count must large
                than zero.");
            }
            setState(count);
        }
        public int tryAcquireShared(int reduceCount) {
            for (;;) {
                int current = getState();
                int newCount = current - reduceCount;
                if (newCount < 0 || compareAndSetState(current, newCount)) {
                    return newCount;
                }
            }
        }
        public boolean tryReleaseShared(int returnCount) {
            for (;;) {
                int current = getState();
                int newCount = current + returnCount;
                if (compareAndSetState(current, newCount)) {
                    return true;
                }
            }
        }
    }
    public void lock() {
        sync.acquireShared(1);
    }
    public void unlock() {
        sync.releaseShared(1);
    }
    // 其他接口方法略
}
```

在上述示例中，TwinsLock实现了Lock接口，提供了面向使用者的接口，使用者调用lock()方法获取锁，随后调用unlock()方法释放锁，而同一时刻只能有两个线程同时获取到锁。

TwinsLock同时包含了一个自定义同步器Sync，而该同步器面向线程访问和同步状态控制。以共享式获取同步状态为例：同步器会先计算出获取后的同步状态，然后通过CAS确保状态的正确设置，当tryAcquireShared(int reduceCount)方法返回值大于等于0时，当前线程才获取同步状态，对于上层的TwinsLock而言，则表示当前线程获得了锁。

同步器作为一个桥梁，连接线程访问以及同步状态控制等底层技术与不同并发组件（比如Lock、CountDownLatch等）的接口语义。

## 5.3重入锁

重入锁ReentrantLock，顾名思义，就是支持重进入的锁，它表示该锁能够支持一个线程对资源的重复加锁。除此之外，该锁的还支持获取锁时的公平和非公平性选择。

而synchronized关键字隐式的支持重进入，比如一个synchronized修饰的递归方法，在方法执行时，执行线程在获取了锁之后仍能连续多次地获得该锁，而不像Mutex由于获取了锁，而在下一次获取锁时出现阻塞自己的情况。

ReentrantLock虽然没能像synchronized关键字一样支持隐式的重进入，但是在调用lock()方法时，已经获取到锁的线程，能够再次调用lock()方法获取锁而不被阻塞。

### 1.实现重进入

重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞，该特性的实现需要解决以下两个问题。

1）线程再次获取锁。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再次成功获取。

2）锁的最终释放。线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到该锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而锁被释放时，计数自减，当计数等于0时表示锁已经成功释放。

ReentrantLock是通过组合自定义同步器来实现锁的获取与释放，以非公平性（默认的）实现为例，获取同步状态的代码如代码清单5-12所示。

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

该方法增加了再次获取同步状态的处理逻辑：**通过判断当前线程是否为获取锁的线程来决定获取操作是否成功**，如果是获取锁的线程再次请求，则将同步状态值进行增加并返回true，表示获取同步状态成功。

成功获取锁的线程再次获取锁，只是增加了同步状态值，这也就要求ReentrantLock在释放同步状态时减少同步状态值，该方法的代码如代码清单5-13所示。

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

如果该锁被获取了n次，那么前(n-1)次tryRelease(int releases)方法必须返回false，而只有同步状态完全释放了，才能返回true。可以看到，该方法将同步状态是否为0作为最终释放的条件，当同步状态为0时，将占有线程设置为null，并返回true，表示释放成功。

### 2.公平与非公平获取锁的区别

公平性与否是针对获取锁而言的，如果一个锁是公平的，那么锁的获取顺序就应该符合请求的绝对时间顺序，也就是FIFO。

回顾上一小节中介绍的nonfairTryAcquire(int acquires)方法，对于非公平锁，只要CAS设置同步状态成功，则表示当前线程获取了锁，而公平锁则不同，如代码清单5-14所示。

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

该方法与nonfairTryAcquire(int acquires)比较，唯一不同的位置为判断条件多了hasQueuedPredecessors()方法，即**加入了同步队列中当前节点是否有前驱节点的判断，如果该方法返回true，则表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。**

## 5.4读写锁

之前提到锁（如Mutex和ReentrantLock）基本都是排他锁，这些锁在同一时刻只允许一个线程进行访问，而读写锁在同一时刻可以允许多个读线程访问，但是在写线程访问时，所有的读线程和其他写线程均被阻塞。读写锁维护了一对锁，一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升。

**除了保证写操作对读操作的可见性以及并发性的提升之外，读写锁能够简化读写交互场景的编程方式。**

一般情况下，读写锁的性能都会比排它锁好，因为大多数场景读是多于写的。在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量。Java并发包提供读写锁的实现是ReentrantReadWriteLock，它提供的特性如表5-8所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697532287065-d336f343-59fe-4e48-bd99-116dbb559190.png)

### 1.读写锁的接口与示例

ReadWriteLock仅定义了获取读锁和写锁的两个方法，即readLock()方法和writeLock()方法，而其实现——ReentrantReadWriteLock，除了接口方法之外，还提供了一些便于外界监控其内部工作状态的方法，这些方法以及描述如表5-9所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697532351717-93dc5f1b-12c5-4f24-9635-c1b84acc1aa3.png)

### 2.读写锁的实现分析

接下来分析ReentrantReadWriteLock的实现，主要包括：

+ 读写状态的设计
+ 写锁的获取与释放
+ 读锁的获取与释放
+ 锁降级（以下没有特别说明读写锁均可认为是ReentrantReadWriteLock）。

#### 读写状态的设计

读写锁同样依赖自定义同步器来实现同步功能，而读写状态就是其同步器的同步状态。

回想ReentrantLock中自定义同步器的实现，同步状态表示锁被一个线程重复获取的次数，而读写锁的自定义同步器需要在同步状态（一个整型变量）上维护多个读线程和一个写线程的状态，使得该状态的设计成为读写锁实现的关键。

**如果在一个整型变量上维护多种状态，就一定需要“按位切割使用”这个变量，读写锁将变量切分成了两个部分，高16位表示读，低16位表示写**，划分方式如图5-8所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697538389656-ba6dbbef-f01b-4e11-b9b1-e750d04db85a.png)

当前同步状态表示一个线程已经获取了写锁，且重进入了两次，同时也连续获取了两次读锁。读写锁是如何迅速确定读和写各自的状态呢？答案是通过位运算。**<font style="color:#00346B;">假设当前同步状态值为S，写状态等于S&0x0000FFFF（将高16位全部抹去），读状态等于S>>>16（无符号补0右移16位）。当写状态增加1时，等于S+1，当读状态增加1时，等于S+(1<<16)，也就是S+0x00010000。</font>**

根据状态的划分能得出一个推论：S不等于0时，当写状态（S&0x0000FFFF）等于0时，则读状态（S>>>16）大于0，即读锁已被获取。

#### 写锁的获取与释放

**写锁是一个支持重进入的排它锁。**如果当前线程已经获取了写锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取（读状态不为0）或者该线程不是已经获取写锁的线程，则当前线程进入等待状态，<font style="color:#DF2A3F;">获取写锁的代码</font>如代码清单5-17所示。

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // 存在读锁或者当前获取线程不是已经获取写锁的线程
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() || !compareAndSetState(c, c + acquires)) {
        return false;
    }
    setExclusiveOwnerThread(current);
    return true;
}
```

该方法除了重入条件（当前线程为获取了写锁的线程）之外，增加了一个读锁是否存在的判断。**如果存在读锁，则写锁不能被获取**，原因在于：**读写锁要确保写锁的操作对读锁可见，如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。**<u>因此，只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。</u>

写锁的释放与ReentrantLock的释放过程基本类似，每次释放均减少写状态，当写状态为0时表示写锁已被释放，从而等待的读写线程能够继续访问读写锁，同时前次写线程的修改对后续读写线程可见。

#### 读锁的获取与释放

读锁是一个支持重进入的共享锁，它能够被多个线程同时获取，在没有其他写线程访问（或者写状态为0）时，读锁总会被成功地获取，而所做的也只是（线程安全的）增加读状态。如果当前线程已经获取了读锁，则增加读状态。如果当前线程在获取读锁时，写锁已被其他线程获取，则进入等待状态。获取读锁的实现从Java 5到Java 6变得复杂许多，主要原因是新增了一些功能，例如getReadHoldCount()方法，作用是返回当前线程获取读锁的次数。读状态是所有线程获取读锁次数的总和，而每个线程各自获取读锁的次数只能选择保存在ThreadLocal中，由线程自身维护，这使获取读锁的实现变得复杂。因此，这里将获取读锁的代码做了删减，保留必要的部分，如代码清单5-18所示。

```java
protected final int tryAcquireShared(int unused) {
    for (;;) {
        int c = getState();
        int nextc = c + (1 << 16);
        if (nextc < c)
            throw new Error("Maximum lock count exceeded");
        if (exclusiveCount(c) != 0 && owner != Thread.currentThread())
            return -1;
        if (compareAndSetState(c, nextc))
            return 1;
    }
}
```

在tryAcquireShared(int unused)方法中，如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态。**如果当前线程获取了写锁或者写锁未被获取，则当前线程（线程安全，依靠CAS保证）增加读状态，成功获取读锁。**

读锁的每次释放（线程安全的，可能有多个读线程同时释放读锁）均减少读状态，减少的值是（1<<16）。

#### 锁降级

锁降级指的是写锁降级成为读锁。如果当前线程拥有写锁，然后将其释放，最后再获取读锁，这种分段完成的过程不能称之为锁降级。**锁降级是指把持住（当前拥有的）写锁，再获取到读锁，随后释放（先前拥有的）写锁的过程。**

锁降级中读锁的获取是否必要呢？答案是必要的。<u>主要是为了保证数据的可见性，如果当前线程不获取读锁而是直接释放写锁，假设此刻另一个线程（记作线程T）获取了写锁并修改了数据，那么当前线程无法感知线程T的数据更新。如果当前线程获取读锁，即遵循锁降级的步骤，则线程T将会被阻塞，直到当前线程使用数据并释放读锁之后，线程T才能获取写锁进行数据更新。</u>

RentrantReadWriteLock不支持锁升级（把持读锁、获取写锁，最后释放读锁的过程）。目的也是保证数据可见性，如果读锁已被多个线程获取，其中任意线程成功获取了写锁并更新了数据，则其更新对其他获取到读锁的线程是不可见的。

## 5.5LockSupport工具

回顾5.2节，当需要阻塞或唤醒一个线程的时候，都会使用LockSupport工具类来完成相应工作。LockSupport定义了一组的公共静态方法，这些方法提供了最基本的线程阻塞和唤醒功能，而LockSupport也成为构建同步组件的基础工具。

LockSupport定义了一组以park开头的方法用来阻塞当前线程，以及unpark(Thread thread)方法来唤醒一个被阻塞的线程。Park有停车的意思，假设线程为车辆，那么park方法代表着停车，而unpark方法则是指车辆启动离开，这些方法以及描述如表5-10所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697550915520-46cbb87a-a9a8-42cb-b07a-1436754c4015.png)

在Java 6中，LockSupport增加了park(Object blocker)、parkNanos(Object blocker,long nanos)和parkUntil(Object blocker,long deadline)3个方法，用于实现阻塞当前线程的功能，其中参数blocker是用来标识当前线程在等待的对象（以下称为阻塞对象），该对象主要用于问题排查和系统监控。

## 5.6Condition接口

任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括wait()、wait(long timeout)、notify()以及notifyAll()方法，这些方法与synchronized同步关键字配合，可以实现等待/通知模式。Condition接口也提供了类似Object的监视器方法，与Lock配合可以实现等待/通知模式，但是这两者在使用方式以及功能特性上还是有差别的。

通过对比Object的监视器方法和Condition接口，可以更详细地了解Condition的特性，对比项与结果如表5-12所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697551223137-8e56e309-cdc2-4190-a911-68cbefe1f9a3.png)

### 1.Condition接口与示例

Condition定义了等待/通知两种类型的方法，当前线程调用这些方法时，需要提前获取到Condition对象关联的锁。Condition对象是由Lock对象（调用Lock对象的newCondition()方法）创建出来的，换句话说，**Condition是依赖Lock对象的。**

Condition的使用方式比较简单，需要注意在调用方法前获取锁，使用方式如代码清单5-20所示。

```java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
public void conditionWait() throws InterruptedException {
    lock.lock();
    try {
        condition.await();
    } finally {
        lock.unlock();
}
} public void conditionSignal() throws InterruptedException {
    lock.lock();
    try {
        condition.signal();
    } finally {
        lock.unlock();
    }
}
```

如示例所示，一般都会将Condition对象作为成员变量。当调用await()方法后，当前线程会释放锁并在此等待，而其他线程调用Condition对象的signal()方法，通知当前线程后，当前线程才从await()方法返回，**并且在返回前已经获取了锁。**

Condition定义的（部分）方法以及描述如表5-13所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697551930113-c13c7a1e-68d1-4f2d-9236-f40da6438c03.png)

获取一个Condition必须通过Lock的newCondition()方法。下面通过一个有界队列的示例来深入了解Condition的使用方式。有界队列是一种特殊的队列，当队列为空时，队列的获取操作将会阻塞获取线程，直到队列中有新增元素，当队列已满时，队列的插入操作将会阻塞插入线程，直到队列出现“空位”，如代码清单5-21所示。

```java
public class BoundedQueue<T> {
    private Object[] items;
    // 添加的下标，删除的下标和数组当前数量
    private int addIndex, removeIndex, count;
    private Lock lock = new ReentrantLock();
    private Condition notEmpty = lock.newCondition();
    private Condition notFull = lock.newCondition();
    public BoundedQueue(int size) {
        items = new Object[size];
    }
    // 添加一个元素，如果数组满，则添加线程进入等待状态，直到有"空位"
    public void add(T t) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length){
                notFull.await();
            }
            items[addIndex] = t;
            if (++addIndex == items.length)
                addIndex = 0;
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }
    // 由头部删除一个元素，如果数组空，则删除线程进入等待状态，直到有新添加元素
    @SuppressWarnings("unchecked")
    public T remove() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0){
                notEmpty.await();
            }
            Object x = items[removeIndex];
            if (++removeIndex == items.length)
                removeIndex = 0;
            --count;
            notFull.signal();
            return (T) x;
        } finally {
            lock.unlock();
        }
    }
}
```

上述示例中，BoundedQueue通过add(T t)方法添加一个元素，通过remove()方法移出一个元素。以添加方法为例。

首先需要获得锁，目的是确保数组修改的可见性和排他性。当数组数量等于数组长度时，表示数组已满，则调用notFull.await()，当前线程随之释放锁并进入等待状态。如果数组数量不等于数组长度，表示数组未满，则添加元素到数组中，同时通知等待在notEmpty上的线程，数组中已经有新元素可以获取。

**在添加和删除方法中使用while循环而非if判断，目的是防止过早或意外的通知，只有条件符合才能够退出循环。回想之前提到的等待/通知的经典范式，二者是非常类似的。**

### 2.Condition的实现分析

ConditionObject是同步器AbstractQueuedSynchronizer的内部类，因为Condition的操作需要获取相关联的锁，所以作为同步器的内部类也较为合理。每个Condition对象都包含着一个队列（以下称为等待队列），该队列是Condition对象实现等待/通知功能的关键。

下面将分析Condition的实现，主要包括：等待队列、等待和通知，下面提到的Condition如果不加说明均指的是ConditionObject。

#### 等待队列

等待队列是一个FIFO的队列，在队列中的每个节点都包含了一个线程引用，该线程就是在Condition对象上等待的线程，如果一个线程调用了Condition.await()方法，那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态。事实上，节点的定义复用了同步器中节点的定义，也就是说，同步队列和等待队列中节点类型都是同步器的静态内部类AbstractQueuedSynchronizer.Node。

一个Condition包含一个等待队列，Condition拥有首节点（firstWaiter）和尾节点（lastWaiter）。当前线程调用Condition.await()方法，将会以当前线程构造节点，并将节点从尾部加入等待队列，等待队列的基本结构如图5-9所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697552590494-e783b6da-61b7-4c26-830e-7f79eb16dd84.png)

如图所示，<font style="color:#DF2A3F;">Condition拥有首尾节点的引用</font>，而新增节点只需要将原有的尾节点nextWaiter指向它，并且更新尾节点即可。**上述节点引用更新的过程并没有使用CAS保证，原因在于调用await()方法的线程必定是获取了锁的线程，也就是说该过程是由锁来保证线程安全的。**

在Object的监视器模型上，一个对象拥有一个同步队列和等待队列，而并发包中的Lock（更确切地说是同步器）拥有一个同步队列和多个等待队列，其对应关系如图5-10所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697609131262-8c85a6f8-3433-4e80-8d39-7a5fcb66245f.png)

如图所示，Condition的实现是同步器的内部类，因此每个Condition实例都能够访问同步器提供的方法，相当于每个Condition都拥有所属同步器的引用。

#### 等待

调用Condition的await()方法（或者以await开头的方法），会使当前线程进入等待队列并释放锁，同时线程状态变为等待状态。当从await()方法返回时，当前线程一定获取了Condition相关联的锁。

**如果从队列（同步队列和等待队列）的角度看await()方法，当调用await()方法时，相当于同步队列的首节点（获取了锁的节点）移动到Condition的等待队列中。**

Condition的await()方法，如代码清单5-22所示。

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 当前线程加入等待队列
    Node node = addConditionWaiter();
    // 释放同步状态，也就是释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

调用该方法的线程成功获取了锁的线程，也就是同步队列中的首节点，该方法会将当前线程构造成节点并加入等待队列中，然后释放同步状态，唤醒同步队列中的后继节点，然后当前线程会进入等待状态。

当等待队列中的节点被唤醒，则唤醒节点的线程开始尝试获取同步状态。如果不是通过其他线程调用Condition.signal()方法唤醒，而是对等待线程进行中断，则会抛出InterruptedException。

如果从队列的角度去看，当前线程加入Condition的等待队列，该过程如图5-11示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697609471249-976f5bd2-83b1-429d-8a2c-c73f7556c0d8.png)

**如图所示，同步队列的首节点并不会直接加入等待队列，而是通过addConditionWaiter()方法把当前线程构造成一个新的节点并将其加入等待队列中。**

#### 通知

调用Condition的signal()方法，**将会唤醒在等待队列中等待时间最长的节点（首节点）**，在唤醒节点之前，会将节点移到同步队列中。

Condition的signal()方法，如代码清单5-23所示。

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

调用该方法的前置条件是当前线程必须获取了锁，可以看到signal()方法进行了isHeldExclusively()检查，也就是当前线程必须是获取了锁的线程。接着获取等待队列的首节点，将其移动到同步队列并使用LockSupport唤醒节点中的线程。

节点从等待队列移动到同步队列的过程如图5-12所示。

![](https://cdn.nlark.com/yuque/0/2023/png/38995097/1697609566572-c031e3e7-22f0-4577-aa0b-0125f9d2857e.png)

**通过调用同步器的enq(Node node)方法，等待队列中的头节点线程安全地移动到同步队列。当节点移动到同步队列后，当前线程再使用LockSupport唤醒该节点的线程。**

被唤醒后的线程，将从await()方法中的while循环中退出（isOnSyncQueue(Node node)方法返回true，节点已经在同步队列中），进而调用同步器的acquireQueued()方法加入到获取同步状态的竞争中。

成功获取同步状态（或者说锁）之后，被唤醒的线程将从先前调用的await()方法返回，此时该线程已经成功地获取了锁。

<u>Condition的signalAll()方法，相当于对等待队列中的每个节点均执行一次signal()方法，效果就是将等待队列中所有节点全部移动到同步队列中，并唤醒每个节点的线程。</u>

## 5.7本章小结

本章介绍了Java并发包中与锁相关的API和组件，通过示例讲述了这些API和组件的使用方式以及需要注意的地方，并在此基础上详细地剖析了队列同步器、重入锁、读写锁以及Condition等API和组件的实现细节，只有理解这些API和组件的实现细节才能够更加准确地运用它们。

# 第六章 java并发容器和框架

ConcurrentHashMap是线程安全且高效的HashMap。

## 6.0 总结

这一章，主要是这几个点：

根据名字我们也能看出来，就是一个容器和一个框架：

容器：

    - ConcurrentHashMap
        * 这个主要是从一开始的hashmap的线程不安全开始介绍，然后对比线程安全的HashTable。
        * 后者主要是用的synchronized，整个map都用的一个锁，这样比较低效。
        * 然后引入分段锁 --- 这里主要是讲之前的一个容器一个锁变成一个容器多个锁（分段锁），这样就可以提高并发。
        * 然后就是如何解决HashMap的并发的问题做了哪些进步：
            + 首先一点就是为了将元素更加散列开，用了全新的散列函数；
            + 一个ConcurrentHashMap中有多个segment数组（代表着多把锁），然后一个segment里面有多个hashEntry。
            + get操作：
                - 主要是通过**<font style="color:#DF2A3F;">再散列</font>**+哈希定位元素位置；
                - 同时我们知道其实，在get操作中是没有加锁的，为什么   ---> 这里就是一个**<font style="color:#DF2A3F;">volatile</font>**变量的作用，我们给每个共享变量都加上volatile关键字，这样根据happens-before原则，永远读的是最新值。
                - get操作的高效之处在于整个get过程不需要加锁，除非**读到的值是空才会加锁重读**。我们知道HashTable容器的get方法是需要加锁的，那么ConcurrentHashMap的get操作是如何做到不加锁的呢？原因是**<font style="color:#601BDE;">它的get方法里将要使用的共享变量都定义成volatile类型</font>**，如<font style="color:#DF2A3F;">用于统计当前Segement大小的count字段和用于存储值的HashEntry的value。定义成volatile的变量</font>，能够在线程之间保持可见性，能够被多线程同时读，并且保证不会读到过期的值，但是<u>只能被单线程写（有一种情况可以被多线程写，就是写入的值不依赖于原值）</u>，在get操作里只需要读不需要写共享变量count和value，所以可以不用加锁。之所以不会读到过期的值，是因为根据Java内存模型的happen before原则，对volatile字段的写入操作先于读操作，即使两个线程同时修改和获取volatile变量，get操作也能拿到最新的值，这是<font style="background-color:#FBDE28;">用volatile替换锁的经典应用场景</font>。
            + 扩容操作：
                - **<font style="color:#DF2A3F;">由于put方法里需要对共享变量进行写入操作，所以为了线程安全，在操作共享变量时必须加锁。</font>**put方法首先定位到Segment，然后在Segment里进行插入操作。插入操作需要经历两个步骤，第一步判断是否需要对Segment里的HashEntry数组进行扩容，第二步定位添加元素的位置，然后将其放在HashEntry数组里。
                - **为了高效，ConcurrentHashMap不会对整个容器进行扩容，而只对某个segment进行扩容。**
            + size操作：
                - 我们在统计ConcurrentHashMap的大小的时候，就需要把每个segment中的元素都加起来，但是在多线程并发的情况下，这样直接相加是会出现误差的。
                - ConcurrentHashMap的做法是**<font style="color:#601BDE;">先尝试2次</font>****通过不锁住Segment的方式来统计各个Segment大小，如果统计的过程中，容器的count发生了变化，则再采用加锁的方式来统计所有Segment的大小。**
    - ConcurrentLinkedQueue
        * 如果要实现一个线程安全的队列有两种方式：一种是使用阻塞算法，另一种是使用非阻塞算法。使用阻塞算法的队列可以用一个锁（入队和出队用同一把锁）或两个锁（入队和出队用不同的锁）等方式来实现。非阻塞的实现方式则可以使用循环CAS的方式来实现。
        * **<font style="background-color:#FBDE28;">---ConcurrentLinkedQueue是基于FIFO的链表实现的线程安全的非加锁的队列！---</font>**
        * **<font style="background-color:#FBDE28;">入队主要做两件事情：第一是将入队节点设置成当前队列尾节点的下一个节点；</font>****第二是更新tail节点，如果tail节点的next节点不为空，则将入队节点设置成tail节点，如果tail节点的next节点为空，则将入队节点设置成tail的next节点，****<font style="color:#DF2A3F;">所以tail节点不总是尾节点。</font>**
        * 这里主要是想说明一个hops思想：
        * 让tail节点永远作为队列的尾节点，这样实现代码量非常少，而且逻辑清晰和易懂。但是，这么做有个缺点，每次都需要使用循环CAS更新tail节点。如果能减少CAS更新tail节点的次数，就能提高入队的效率，所以doug lea使用hops变量来控制并减少tail节点的更新频率，并不是每次节点入队后都将tail节点更新成尾节点，而是当tail节点和尾节点的距离大于等于常量HOPS的值（默认等于1）时才更新tail节点，**tail和尾节点的距离越长，使用CAS更新tail节点的次数就会越少，但是距离越长带来的负面效果就是每次入队时定位尾节点的时间就越长，因为循环体需要多循环一次来定位出尾节点**，但是这样仍然能提高入队的效率，因为**<font style="color:#601BDE;">从本质上来看它通过增加对volatile变量的读操作来减少对volatile变量的写操作</font>**<font style="color:#DF2A3F;">，</font>而对volatile变量的写操作开销要远远大于读操作，所以入队效率会有所提升。   --- 这里因为tail节点是用volatile变量修饰的
        * 出队：**<font style="color:#DF2A3F;">并不是每次出队时都更新head节点，当head节点里有元素时，直接弹出head节点里的元素，而不会更新head节点。只有当head节点里没有元素时，出队操作才会更新head节点。</font>**这种做法也是通过hops变量来减少使用CAS更新head节点的消耗，从而提高出队效率。

+ 阻塞队列：
    - 阻塞队列（BlockingQueue）是一个支持两个附加操作的队列。这两个附加的操作**<font style="color:#DF2A3F;">支持阻塞的插入和移除方法。</font>**
        * 1）支持阻塞的插入方法：意思是当队列满时，队列会阻塞插入元素的线程，直到队列不满。
        * 2）支持阻塞的移除方法：意思是在队列为空时，获取元素的线程会等待队列变为非空。
        * **阻塞队列常用于生产者和消费者的场景**，生产者是向队列里添加元素的线程，消费者是从队列里取元素的线程。阻塞队列就是生产者用来存放元素、消费者用来获取元素的**容器**。
    - 首先明白一点，阻塞队列是用来干嘛的   --->  <font style="color:rgb(13, 13, 13);">是一个用于在多线程环境下进行任务调度的 !</font>
        * <font style="color:rgb(13, 13, 13);">阻塞队列可以用于任务的生产者-消费者模型，其中生产者将任务放入队列，而消费者从队列中取出任务并执行。在这个模型中，</font><font style="color:#DF2A3F;background-color:#FBDE28;">队列充当了缓冲区</font><font style="color:rgb(13, 13, 13);">，协调了生产者和消费者之间的任务传递。</font>
+ 一个小问题：阻塞队列和消息中间件的区别
+ 阻塞队列和消息中间件都是用于在多线程或分布式系统中实现异步通信的工具，但它们有一些关键的区别。

1.  **用途和领域：**
    - **阻塞队列：** 主要用于<font style="color:#DF2A3F;">线程之间的数据传递和协调</font>，是在单个应用程序内部的多线程环境中使用的。阻塞队列通常用于实现生产者-消费者模型，其中生产者将数据放入队列，而消费者从队列中取出数据。
    - **消息中间件：** 更通用，<font style="color:#DF2A3F;">用于在不同应用程序或系统之间进行异步消息传递</font>。消息中间件允许分布式系统中的不同组件通过消息进行通信，**<font style="color:#DF2A3F;">可以跨越不同的进程、主机甚至是不同的语言。</font>**
2.  **通信方式：**
    - **阻塞队列：** 提供了一种简单的、直接的、同步的方式，其中线程在队列为空或已满时会被阻塞。
    - **消息中间件：** 提供了**异步的、解耦的**通信方式，通过发布-订阅或者点对点的消息传递模式，允许发送者和接收者不直接耦合，提高系统的灵活性和可维护性。
3.  **跨越边界：**
    - **阻塞队列：** 通常用于在同一进程内的多个线程之间进行通信，数据传递在同一应用程序内。
    - **消息中间件：** 用于跨越不同进程、不同主机、不同应用程序之间的通信，适用于构建分布式系统。
4.  **可靠性和持久性：**
    - **阻塞队列：** 通常是内存中的数据结构，对于应用程序重启或故障恢复的支持有限。
    - **消息中间件：** 可以提供更强大的消息可靠性和持久性支持，通过消息的存储和重发机制，可以确保消息在发送和接收之间的可靠传递。

总体而言，阻塞队列和消息中间件在某种程度上有相似之处，但它们的设计目标和使用场景不同。选择使用哪种方式取决于应用程序的具体需求和架构设计。

+ 框架：
    - 框架主要是介绍了一个Fork/Join框架。Fork/Join框架是Java 7提供的一个**<font style="color:#DF2A3F;">用于并行执行任务的框架</font>**，**是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。**
    - **工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行。**
    - 基于Fork/Join框架的特性，也就是大任务拆成小任务，然后分给多个线程执行，我们可以使用双端队列来做这个事情，原线程从队头拿消息，窃取线程从队尾拿消息。

## 6.1ConcurrentHashMap的实现原理与使用

### 6.1.1为什么要使用ConcurrentHashMap*

在并发编程中使用HashMap可能导致程序死循环。而使用线程安全的HashTable效率又非常低下，基于以上两个原因，便有了ConcurrentHashMap的登场机会。

（1）线程不安全的HashMap

在多线程环境下，使用HashMap进行put操作会引起死循环，导致CPU利用率接近100%，所以在并发情况下不能使用HashMap。例如，执行以下代码会引起死循环。

```java
final HashMap<String, String> map = new HashMap<String, String>(2);
        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10000; i++) {
                    new Thread(new Runnable() {
                        @Override
                        public void run() {
                        map.put(UUID.randomUUID().toString(), "");
                        }
                    }, "ftf" + i).start();
                }
            }
        }, "ftf");
        t.start();
        t.join();
```

HashMap在并发执行put操作时会引起死循环，是因为多线程会导致HashMap的Entry链表形成环形数据结构，一旦形成环形数据结构，Entry的next节点永远不为空，就会产生死循环获取Entry。

（2）效率低下的HashTable

HashTable容器使用synchronized来保证线程安全，但在线程竞争激烈的情况下HashTable的效率非常低下。因为当一个线程访问HashTable的同步方法，其他线程也访问HashTable的同步方法时，会进入阻塞或轮询状态。如线程1使用put进行元素添加，线程2不但不能使用put方法添加元素，也不能使用get方法来获取元素，所以竞争越激烈效率越低。

（3）ConcurrentHashMap的锁分段技术可有效提升并发访问率

HashTable容器在竞争激烈的并发环境下表现出<font style="background-color:#FBDE28;">效率低下的原因</font>是<u>所有访问HashTable的线程都必须竞争同一把锁</u>，**假如容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效提高并发访问效率，**这就是ConcurrentHashMap所使用的**<font style="color:#DF2A3F;">锁分段技术</font>**。首先将数据分成一段一段地存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。

--- 所以我们又能总结出一个问题：那就是concurrentHashMap的演进过程：

+ 首先有线程不安全的HashMap；  --- 并发死链和复制问题；
+ 然后就有线程安全的hashTable;   --- 但是有个问题就是，它的线程安全是基于在每个方法上面加synchronized来实现的，在并发的情况下，会出现效率低下的问题；
+ <font style="background-color:#FBDE28;">根因：</font>其实是由于所有访问容器的线程都必须去获取那一把锁，这样造成效率低下！ --->  采用分段锁技术：其实就是引入多把锁，每把锁只管一部分。

### 6.1.2 ConcurrentHashMap的结构

通过ConcurrentHashMap的类图来分析ConcurrentHashMap的结构，如图6-1所示。

ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。**Segment是一种可重入锁（ReentrantLock），在ConcurrentHashMap里扮演锁的角色；**HashEntry则用于存储键值对数据。一个ConcurrentHashMap里包含一个Segment数组。Segment的结构和HashMap类似，是一种数组和链表结构。一个Segment里包含一个HashEntry数组，每个HashEntry是一个链表结构的元素，每个Segment守护着一个HashEntry数组里的元素，当对HashEntry数组的数据进行修改时，必须首先获得与它对应的Segment锁，如图6-2所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708057397573-20f0a999-f3ce-4445-bdc3-7aae9bb19d69.png)

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708057406795-d9dc48d4-024c-4088-9ecc-801c9a66631f.png)

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708057438923-5bd4718b-2563-4881-bdc9-b95ba3dcf427.png)

### 6.1.3　ConcurrentHashMap的初始化

ConcurrentHashMap初始化方法是通过initialCapacity、loadFactor和concurrencyLevel等几个参数来初始化segment数组、段偏移量segmentShift、段掩码segmentMask和每个segment里的HashEntry数组来实现的。

#### 1.初始化segments数组

让我们来看一下初始化segments数组的源代码。

```java
if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        int sshift = 0;
        int ssize = 1;
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
        segmentShift = 32 - sshift;
        segmentMask = ssize - 1;
        this.segments = Segment.newArray(ssize);
```

由上面的代码可知，segments数组的长度ssize是通过concurrencyLevel计算得出的。**为了能通过按位与的散列算法来定位segments数组的索引，必须保证segments数组的长度是2的N次方（power-of-two size）**，所以<u>必须计算出一个大于或等于concurrencyLevel的最小的2的N次方值来作为segments数组的长度。</u>假如concurrencyLevel等于14、15或16，ssize都会等于16，即容器里锁的个数也是16。

注意　concurrencyLevel的最大值是65535，这意味着segments数组的长度最大为65536，对应的二进制是16位。

#### 2.初始化segmentShift和segmentMask

这两个全局变量需要在定位segment时的散列算法里使用，sshift等于ssize从1向左移位的次数，在默认情况下concurrencyLevel等于16，1需要向左移位移动4次，所以sshift等于4。

segmentShift用于定位参与散列运算的位数，segmentShift等于32减sshift，所以等于28，这里之所以用32是因为ConcurrentHashMap里的hash()方法输出的最大数是32位的，后面的测试中我们可以看到这点。segmentMask是散列运算的掩码，等于ssize减1，即15，掩码的二进制各个位的值都是1。因为ssize的最大长度是65536，所以segmentShift最大值是16，segmentMask最大值是65535，对应的二进制是16位，每个位都是1。

#### 3.初始化每个segment

输入参数initialCapacity是ConcurrentHashMap的初始化容量，loadfactor是每个segment的负载因子，在构造方法里需要通过这两个参数来初始化数组中的每个segment。

```java
if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = 1;
    while (cap < c)
        cap <<= 1;
    for (int i = 0; i < this.segments.length; ++i)
        this.segments[i] = new Segment<K,V>(cap, loadFactor);
```

上面代码中的变量cap就是segment里HashEntry数组的长度，它等于initialCapacity除以ssize

的倍数c，如果c大于1，就会取大于等于c的2的N次方值，所以cap不是1，就是2的N次方。

segment的容量threshold＝（int）cap*loadFactor，默认情况下initialCapacity等于16，loadfactor等于

0.75，通过运算cap等于1，threshold等于零。

### 6.1.4 定位Segment*

既然ConcurrentHashMap使用分段锁Segment来保护不同段的数据，那么在插入和获取元素的时候，必须先通过散列算法定位到Segment。可以看到ConcurrentHashMap会首先使用Wang/Jenkins hash的变种算法对元素的hashCode进行一次再散列。

```java
private static int hash(int h) {
        h += (h << 15) ^ 0xffffcd7d;
        h ^= (h >>> 10);
        h += (h << 3);
        h ^= (h >>> 6);
        h += (h << 2) + (h << 14);
        return h ^ (h >>> 16);
    }
```

**<font style="color:#DF2A3F;">之所以进行再散列，目的是减少散列冲突，使元素能够均匀地分布在不同的Segment上，从而提高容器的存取效率。</font>**假如散列的质量差到极点，那么所有的元素都在一个Segment中，不仅存取元素缓慢，分段锁也会失去意义。笔者做了一个测试，不通过再散列而直接执行散列计算。

```java
System.out.println(Integer.parseInt("0001111", 2) & 15);
System.out.println(Integer.parseInt("0011111", 2) & 15);
System.out.println(Integer.parseInt("0111111", 2) & 15);
System.out.println(Integer.parseInt("1111111", 2) & 15);
```

计算后输出的散列值全是15，通过这个例子可以发现，如果不进行再散列，散列冲突会非常严重，因为只要低位一样，无论高位是什么数，其散列值总是一样。我们再把上面的二进制数据进行再散列后结果如下（为了方便阅读，不足32位的高位补了0，每隔4位用竖线分割下）。

```r
0100｜0111｜0110｜0111｜1101｜1010｜0100｜1110
1111｜0111｜0100｜0011｜0000｜0001｜1011｜1000
0111｜0111｜0110｜1001｜0100｜0110｜0011｜1110
1000｜0011｜0000｜0000｜1100｜1000｜0001｜1010
```

可以发现，每一位的数据都散列开了，通过这种再散列能让数字的每一位都参加到散列运算当中，从而减少散列冲突。ConcurrentHashMap通过以下散列算法定位segment。

```java
final Segment<K,V> segmentFor(int hash) {
    return segments[(hash >>> segmentShift) & segmentMask];
}
```

默认情况下segmentShift为28，segmentMask为15，再散列后的数最大是32位二进制数据，向右无符号移动28位，<font style="background-color:#FBDE28;">意思是让高4位参与到散列运算中</font>，`**（hash>>>segmentShift）&segmentMask**`的运算结果分别是4、15、7和8，可以看到散列值没有发生冲突。

### 6.1.5　ConcurrentHashMap的操作*

本节介绍ConcurrentHashMap的3种操作——get操作、put操作和size操作。

#### 1.get操作*

Segment的get操作实现非常简单和高效。先经过一次再散列，然后使用这个散列值通过散列运算定位到Segment，再通过散列算法定位到元素，代码如下。

```java
public V get(Object key) {
    int hash = hash(key.hashCode());
    return segmentFor(hash).get(key, hash);
}
```

get操作的高效之处在于整个get过程不需要加锁，除非**读到的值是空才会加锁重读**。我们知道HashTable容器的get方法是需要加锁的，那么ConcurrentHashMap的get操作是如何做到不加锁的呢？原因是**它的get方法里将要使用的共享变量都定义成volatile类型**，如<font style="color:#DF2A3F;">用于统计当前Segement大小的count字段和用于存储值的HashEntry的value。定义成volatile的变量</font>，能够在线程之间保持可见性，能够被多线程同时读，并且保证不会读到过期的值，但是<u>只能被单线程写（有一种情况可以被多线程写，就是写入的值不依赖于原值）</u>，在get操作里只需要读不需要写共享变量count和value，所以可以不用加锁。之所以不会读到过期的值，是因为根据Java内存模型的happen before原则，对volatile字段的写入操作先于读操作，即使两个线程同时修改和获取volatile变量，get操作也能拿到最新的值，这是<font style="background-color:#FBDE28;">用volatile替换锁的经典应用场景</font>。

```java
transient volatile int count;
volatile V value;
```

在定位元素的代码里我们可以发现，定位HashEntry和定位Segment的散列算法虽然一样，都与数组的长度减去1再相“与”，但是相“与”的值不一样，定位Segment使用的是元素的hashcode通过再散列后得到的值的高位，而定位HashEntry直接使用的是再散列后的值。其目的是避免两次散列后的值一样，虽然元素在Segment里散列开了，但是却没有在HashEntry里散列开。

```java
hash >>> segmentShift) & segmentMask　　 // 定位Segment所使用的hash算法
int index = hash & (tab.length - 1);　　 // 定位HashEntry所使用的hash算法
```

#### 2.put操作

**<font style="color:#DF2A3F;">由于put方法里需要对共享变量进行写入操作，所以为了线程安全，在操作共享变量时必须加锁。</font>**put方法首先定位到Segment，然后在Segment里进行插入操作。插入操作需要经历两个步骤，第一步判断是否需要对Segment里的HashEntry数组进行扩容，第二步定位添加元素的位置，然后将其放在HashEntry数组里。

（1）是否需要扩容

在插入元素前会先判断Segment里的HashEntry数组是否超过容量（threshold），如果超过阈值，则对数组进行扩容。值得一提的是，Segment的扩容判断比HashMap更恰当，因为HashMap是在插入元素后判断元素是否已经到达容量的，如果到达了就进行扩容，但是很有可能扩容之后没有新元素插入，这时HashMap就进行了一次无效的扩容。  ---   相当于提前准备，就像饿汉式一样，提前扩容不一定会有人用啊，懒汉式（懒加载）更能提升效率。

（2）如何扩容

在扩容的时候，首先会创建一个容量是原来容量两倍的数组，然后将原数组里的元素进行再散列后插入到新的数组里。**为了高效，ConcurrentHashMap不会对整个容器进行扩容，而只对某个segment进行扩容。**

#### 3.size操作*

如果要统计整个ConcurrentHashMap里元素的大小，就必须统计所有Segment里元素的大小后求和。Segment里的全局变量count是一个volatile变量，那么在多线程场景下，是不是直接把所有Segment的count相加就可以得到整个ConcurrentHashMap大小了呢？不是的，虽然相加时可以获取每个Segment的count的最新值，但是可能累加前使用的count发生了变化，那么统计结果就不准了。所以，最安全的做法是在统计size的时候把所有Segment的put、remove和clean方法全部锁住，但是这种做法显然非常低效。

因为在累加count操作过程中，之前累加过的count发生变化的几率非常小，所以ConcurrentHashMap的做法是**先尝试2次通过不锁住Segment的方式来统计各个Segment大小，如果统计的过程中，容器的count发生了变化，则再采用加锁的方式来统计所有Segment的大小。**

那么ConcurrentHashMap是如何判断在统计的时候容器是否发生了变化呢？使用**modCount**变量，在put、remove和clean方法里操作元素前都会将变量modCount进行加1，那么在统计size前后比较modCount是否发生变化，从而得知容器的大小是否发生变化。

## 6.2 ConcurrentLinkedQueue*

在并发编程中，有时候需要使用线程安全的队列。如果要实现一个线程安全的队列有两种方式：一种是使用阻塞算法，另一种是使用非阻塞算法。使用阻塞算法的队列可以用一个锁（入队和出队用同一把锁）或两个锁（入队和出队用不同的锁）等方式来实现。非阻塞的实现方式则可以使用循环CAS的方式来实现。本节让我们一起来研究一下`Doug Lea`是如何使用非阻塞的方式来实现线程安全队列ConcurrentLinkedQueue的，相信从大师身上我们能学到不少并发编程的技巧。

ConcurrentLinkedQueue是一个基于链接节点的无界线程安全队列，它采用先进先出的规则对节点进行排序，当我们添加一个元素的时候，它会添加到队列的尾部；当我们获取一个元素时，它会返回队列头部的元素。它采用了“wait-free”算法（即CAS算法）来实现，该算法在Michael&Scott算法上进行了一些修改。

**<font style="background-color:#FBDE28;">---ConcurrentLinkedQueue是基于FIFO的链表实现的线程安全的非加锁的队列！---</font>**

### 6.2.1　ConcurrentLinkedQueue的结构

通过ConcurrentLinkedQueue的类图来分析一下它的结构，如图6-3所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708067978274-790bf745-001f-48d9-82f8-12af91d7dd1e.png)

ConcurrentLinkedQueue由head节点和tail节点组成，每个节点（Node）由节点元素（item）和指向下一个节点（next）的引用组成，节点与节点之间就是通过这个next关联起来，从而组成一张链表结构的队列。默认情况下head节点存储的元素为空，tail节点等于head节点。

```java
private transient volatile Node<E> tail = head;
```

### 6.2.2　入队列

本节将介绍入队列的相关知识。

#### 1.入队列的过程

入队列就是将入队节点添加到队列的尾部。为了方便理解入队时队列的变化，以及head节点和tail节点的变化，这里以一个示例来展开介绍。假设我们想在一个队列中依次插入4个节点，为了帮助大家理解，每添加一个节点就做了一个队列的快照图，如图6-4所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708068353412-276fb173-bee0-43c6-beda-45e8412113d5.png)

图6-4所示的过程如下。

+ 添加元素1。队列更新head节点的next节点为元素1节点。又因为tail节点默认情况下等于head节点，所以它们的next节点都指向元素1节点。
+ 添加元素2。队列首先设置元素1节点的next节点为元素2节点，然后更新tail节点指向元素2节点。
+ 添加元素3，设置tail节点的next节点为元素3节点。
+ 添加元素4，设置元素3的next节点为元素4节点，然后将tail节点指向元素4节点。

通过调试入队过程并观察head节点和tail节点的变化，发现入队主要做两件事情：第一是将入队节点设置成当前队列尾节点的下一个节点；**第二是更新tail节点，如果tail节点的next节点不为空，则将入队节点设置成tail节点，如果tail节点的next节点为空，则将入队节点设置成tail的next节点，****<font style="color:#DF2A3F;">所以tail节点不总是尾节点</font>**（理解这一点对于我们研究源码会非常有帮助）。

通过对上面的分析，我们从单线程入队的角度理解了入队过程，但是多个线程同时进行入队的情况就变得更加复杂了，因为可能会出现其他线程插队的情况。如果有一个线程正在入队，那么它必须先获取尾节点，然后设置尾节点的下一个节点为入队节点，但这时可能有另外一个线程插队了，那么队列的尾节点就会发生变化，这时当前线程要暂停入队操作，然后重新获取尾节点。让我们再通过源码来详细分析一下它是如何使用CAS算法来入队的。

```java
public boolean offer(E e) {
            if (e == null) throw new NullPointerException();
            // 入队前，创建一个入队节点
            Node<E> n = new Node<E>(e);
            retry:
            // 死循环，入队不成功反复入队。
            for (;;) {
                // 创建一个指向tail节点的引用
                Node<E> t = tail;
                // p用来表示队列的尾节点，默认情况下等于tail节点。
                Node<E> p = t;
                for (int hops = 0; ; hops++) {
                // 获得p节点的下一个节点。
                    Node<E> next = succ(p);
        // next节点不为空，说明p不是尾节点，需要更新p后在将它指向next节点
                if (next != null) {
                    // 循环了两次及其以上，并且当前节点还是不等于尾节点
                    if (hops > HOPS && t != tail)
                    continue retry;
                    p = next;
                }
                // 如果p是尾节点，则设置p节点的next节点为入队节点。
                else if (p.casNext(null, n)) {
                    /*如果tail节点有大于等于1个next节点，则将入队节点设置成tail节点，
                    更新失败了也没关系，因为失败了表示有其他线程成功更新了tail节点*/
        if (hops >= HOPS)
                    casTail(t, n); // 更新tail节点，允许失败
                return true;
            }
            // p有next节点,表示p的next节点是尾节点，则重新设置p节点
            else {
                p = succ(p);
            }
        }
    }
}
```

从源代码角度来看，整个入队过程主要做两件事情：

+ 第一是定位出尾节点；
+ <font style="color:#DF2A3F;">第二是使用CAS算法将入队节点设置成尾节点的next节点，如不成功则重试。</font>

#### 2.定位尾节点

**tail节点并不总是尾节点，所以每次入队都必须先通过tail节点来找到尾节点。****<font style="color:#DF2A3F;">尾节点可能是tail节点，也可能是tail节点的next节点。</font>**代码中循环体中的第一个if就是判断tail是否有next节点，有则表示next节点可能是尾节点。获取tail节点的next节点需要注意的是p节点等于p的next节点的情况，只有一种可能就是p节点和p的next节点都等于空，表示这个队列刚初始化，正准备添加节点，所以需要返回head节点。获取p节点的next节点代码如下。

```java
final Node<E> succ(Node<E> p) {
    Node<E> next = p.getNext();
    return (p == next) head : next;
}
```

#### 3.设置入队节点为尾节点

p.casNext（null，n）方法用于将入队节点设置为当前队列尾节点的next节点，如果p是null，表示p是当前队列的尾节点，如果不为null，表示有其他线程更新了尾节点，则需要重新获取当前队列的尾节点。

#### 4.HOPS的设计意图*

上面分析过对于先进先出的队列入队所要做的事情是将入队节点设置成尾节点，doug lea写的代码和逻辑还是稍微有点复杂。那么，我用以下方式来实现是否可行？

```java
public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        Node<E> n = new Node<E>(e);
        for (;;) {
            Node<E> t = tail;
            if (t.casNext(null, n) && casTail(t, n)) {
                return true;
            }
        }
}
```

让tail节点永远作为队列的尾节点，这样实现代码量非常少，而且逻辑清晰和易懂。但是，这么做有个缺点，每次都需要使用循环CAS更新tail节点。如果能减少CAS更新tail节点的次数，就能提高入队的效率，所以doug lea使用hops变量来控制并减少tail节点的更新频率，并不是每次节点入队后都将tail节点更新成尾节点，而是当tail节点和尾节点的距离大于等于常量HOPS的值（默认等于1）时才更新tail节点，**tail和尾节点的距离越长，使用CAS更新tail节点的次数就会越少，但是距离越长带来的负面效果就是每次入队时定位尾节点的时间就越长，因为循环体需要多循环一次来定位出尾节点**，但是这样仍然能提高入队的效率，因为**<font style="color:#DF2A3F;">从本质上来看它通过增加对volatile变量的读操作来减少对volatile变量的写操作</font>**<font style="color:#DF2A3F;">，</font>而对volatile变量的写操作开销要远远大于读操作，所以入队效率会有所提升。   --- 这里因为tail节点是用volatile变量修饰的，我们可以看下面的源码：

```java
    private transient volatile Node<E> head;
    /**
     * A node from which the last node on list (that is, the unique
     * node with node.next == null) can be reached in O(1) time.
     * Invariants:
     * - the last node is always reachable from tail via succ()
     * - tail != null
     * Non-invariants:
     * - tail.item may or may not be null.
     * - it is permitted for tail to lag behind head, that is, for tail
     *   to not be reachable from head!
     * - tail.next may or may not be self-pointing to tail.
     */
    private transient volatile Node<E> tail;
```

```java
private static final int HOPS = 1;
```

注意　入队方法永远返回true，所以不要通过返回值判断入队是否成功。

### 6.2.3　出队列

出队列的就是从队列里返回一个节点元素，并清空该节点对元素的引用。让我们通过每个节点出队的快照来观察一下head节点的变化，如图6-5所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708070245509-51b8e90b-4b34-44da-8779-7ff184ecf4f0.png)

从图中可知，**<font style="color:#DF2A3F;">并不是每次出队时都更新head节点，当head节点里有元素时，直接弹出head节点里的元素，而不会更新head节点。只有当head节点里没有元素时，出队操作才会更新head节点。</font>**这种做法也是通过hops变量来减少使用CAS更新head节点的消耗，从而提高出队效率。让我们再通过源码来深入分析下出队过程。

```java
public E poll() {
        Node<E> h = head;
        // p表示头节点，需要出队的节点
        Node<E> p = h;
        for (int hops = 0;; hops++) {
            // 获取p节点的元素
            E item = p.getItem();
            // 如果p节点的元素不为空，使用CAS设置p节点引用的元素为null,
            // 如果成功则返回p节点的元素。
            if (item != null && p.casItem(item, null)) {
                if (hops >= HOPS) {
                // 将p节点下一个节点设置成head节点
                Node<E> q = p.getNext();
                updateHead(h, (q != null) q : p);
                }
                return item;
                }
            // 如果头节点的元素为空或头节点发生了变化，这说明头节点已经被另外
            // 一个线程修改了。那么获取p节点的下一个节点
            Node<E> next = succ(p);
            // 如果p的下一个节点也为空，说明这个队列已经空了
            if (next == null) {
                // 更新头节点。
                updateHead(h, p);
                break;
            }
            // 如果下一个元素不为空，则将头节点的下一个节点设置成头节点
            p = next;
        }
        return null;
}
```

首先获取头节点的元素，然后判断头节点元素是否为空，如果为空，表示另外一个线程已经进行了一次出队操作将该节点的元素取走，如果不为空，则使用CAS的方式将头节点的引用设置成null，如果CAS成功，则直接返回头节点的元素，如果不成功，表示另外一个线程已经进行了一次出队操作更新了head节点，导致元素发生了变化，需要重新获取头节点。

## 6.3 Java中的阻塞队列

本节将介绍什么是阻塞队列，以及Java中阻塞队列的4种处理方式，并介绍Java 7中提供的7种阻塞队列，最后分析阻塞队列的一种实现方式。

### 6.3.1　什么是阻塞队列

阻塞队列（BlockingQueue）是一个支持两个附加操作的队列。这两个附加的操作**<font style="color:#DF2A3F;">支持阻塞的插入和移除方法。</font>**

1）支持阻塞的插入方法：意思是当队列满时，队列会阻塞插入元素的线程，直到队列不满。

2）支持阻塞的移除方法：意思是在队列为空时，获取元素的线程会等待队列变为非空。

**阻塞队列常用于生产者和消费者的场景**，生产者是向队列里添加元素的线程，消费者是从队列里取元素的线程。阻塞队列就是生产者用来存放元素、消费者用来获取元素的容器。

在阻塞队列不可用时，这两个附加操作提供了4种处理方式，如表6-1所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708076785988-688d0595-5e52-4b9b-8526-70c23c7b1e3e.png)

+ 抛出异常：当队列满时，如果再往队列里插入元素，会抛出IllegalStateException（"Queuefull"）异常。当队列空时，从队列里获取元素会抛出NoSuchElementException异常。
+ 返回特殊值：当往队列插入元素时，会返回元素是否插入成功，成功返回true。如果是移除方法，则是从队列里取出一个元素，如果没有则返回null。
+ 一直阻塞：当阻塞队列满时，如果生产者线程往队列里put元素，队列会一直阻塞生产者线程，直到队列可用或者响应中断退出。当队列空时，如果消费者线程从队列里take元素，队列会阻塞住消费者线程，直到队列不为空。
+ 超时退出：当阻塞队列满时，如果生产者线程往队列里插入元素，队列会阻塞生产者线程一段时间，如果超过了指定的时间，生产者线程就会退出。

这两个附加操作的4种处理方式不方便记忆，所以我找了一下这几个方法的规律。put和take分别尾首含有字母t，offer和poll都含有字母o。

注意　如果是无界阻塞队列，队列不可能会出现满的情况，所以使用put或offer方法永远不会被阻塞，而且使用offer方法时，该方法永远返回true。

### 6.3.2　Java里的阻塞队列

首先明白一点，阻塞队列是用来干嘛的   --->  <font style="color:rgb(13, 13, 13);">是一个用于在多线程环境下进行任务调度的 !</font>

<font style="color:rgb(13, 13, 13);">阻塞队列可以用于任务的生产者-消费者模型，其中生产者将任务放入队列，而消费者从队列中取出任务并执行。在这个模型中，</font><font style="color:#DF2A3F;background-color:#FBDE28;">队列充当了缓冲区</font><font style="color:rgb(13, 13, 13);">，协调了生产者和消费者之间的任务传递。</font>

JDK 7提供了7个阻塞队列，如下。

+ ArrayBlockingQueue：一个由数组结构组成的有界阻塞队列。
+ LinkedBlockingQueue：一个由链表结构组成的有界阻塞队列。
+ PriorityBlockingQueue：一个**支持优先级排序**的**无界**阻塞队列。
+ DelayQueue：一个使用**优先级队列**实现的**无界**阻塞队列。
+ SynchronousQueue：一个不存储元素的阻塞队列。
+ LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
+ LinkedBlockingDeque：一个由链表结构组成的**双向**阻塞队列。

#### 1.ArrayBlockingQueue

ArrayBlockingQueue是一个用数组实现的有界阻塞队列。此队列按照先进先出（FIFO）的原则对元素进行排序。

**默认情况下不保证线程公平的访问队列**，所谓公平访问队列是指阻塞的线程，可以按照阻塞的先后顺序访问队列，即先阻塞线程先访问队列。非公平性是对先等待的线程是非公平的，当队列可用时，阻塞的线程都可以争夺访问队列的资格，有可能先阻塞的线程最后才访问队列。为了保证公平性，通常会降低吞吐量。我们可以使用以下代码创建一个公平的阻塞队列。

```java
ArrayBlockingQueue fairQueue = new ArrayBlockingQueue(1000,true);
```

访问者的公平性是使用可重入锁实现的，代码如下。

```java
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull = lock.newCondition();
}
```

注：

**虽然 `ArrayBlockingQueue` 默认是非公平的，但它仍然保持了FIFO（先进先出）的特性。**FIFO 是指队列中的元素按照它们被插入队列的顺序进行排序，也就是说，先插入的元素会在队列的前面，后插入的元素会在队列的后面。

即使是在非公平模式下，当队列有多个线程在等待访问时，队列会按照元素的插入顺序来选择下一个执行的线程。这样，尽管非公平性可能导致较晚等待的线程有机会比较早地访问队列，但在整体上，队列仍然遵循了插入顺序，因此仍然保持了FIFO的性质。

总的来说，FIFO 是 `ArrayBlockingQueue` 的基本特性，即使在非公平模式下，该队列仍然按照插入顺序对元素进行排序。

#### 2.LinkedBlockingQueue

LinkedBlockingQueue是一个用链表实现的有界阻塞队列。此队列的默认的最大长度为Integer.MAX_VALUE。此队列按照先进先出的原则对元素进行排序。

#### 3.PriorityBlockingQueue

PriorityBlockingQueue是一个**支持优先级的无界阻塞队列**。默认情况下元素采取自然顺序升序排列。也可以自定义类实现compareTo()方法来指定元素排序规则，或者初始化PriorityBlockingQueue时，指定构造参数Comparator来对元素进行排序。需要注意的是不能保证同优先级元素的顺序。

#### 4.DelayQueue*   --- 定时任务调度 保存缓存数据

DelayQueue是一个**支持延时获取元素的无界阻塞队列**。队列使用<font style="color:#DF2A3F;">PriorityQueue（优先级队列）</font>来实现。队列中的元素必须实现Delayed接口，在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。

<font style="background-color:#FBDE28;">DelayQueue非常有用，可以将DelayQueue运用在以下应用场景。</font>

+ 缓存系统的设计：可以用DelayQueue保存缓存元素的有效期，使用一个线程循环查询DelayQueue，一旦能从DelayQueue中获取元素时，表示缓存有效期到了。
+ 定时任务调度：使用DelayQueue保存当天将会执行的任务和执行时间，一旦从DelayQueue中获取到任务就开始执行，比如TimerQueue就是使用DelayQueue实现的。

（1）如何实现Delayed接口

DelayQueue队列的元素必须实现Delayed接口。我们可以参考ScheduledThreadPoolExecutor里ScheduledFutureTask类的实现，一共有三步。

第一步：在对象创建的时候，初始化基本数据。使用time记录当前对象延迟到什么时候可以使用，使用sequenceNumber来标识元素在队列中的先后顺序。代码如下。

```java
private static final AtomicLong sequencer = new AtomicLong(0);
ScheduledFutureTask(Runnable r, V result, long ns, long period) {
ScheduledFutureTask(Runnable r, V result, long ns, long period) {
        super(r, result);
        this.time = ns;
        this.period = period;
        this.sequenceNumber = sequencer.getAndIncrement();
}
```

第二步：实现getDelay方法，该方法返回当前元素还需要延时多长时间，单位是纳秒，代码如下。

```java
public long getDelay(TimeUnit unit) {
        return unit.convert(time - now(), TimeUnit.NANOSECONDS);
}
```

通过构造函数可以看出延迟时间参数ns的单位是纳秒，自己设计的时候最好使用纳秒，因为实现getDelay()方法时可以指定任意单位，一旦以秒或分作为单位，而延时时间又精确不到纳秒就麻烦了。使用时请注意当time小于当前时间时，getDelay会返回负数。

第三步：实现compareTo方法来指定元素的顺序。例如，让延时时间最长的放在队列的末尾。实现代码如下。

```java
public int compareTo(Delayed other) {
        if (other == this)　　// compare zero ONLY if same object
            return 0;
        if (other instanceof ScheduledFutureTask) {
            ScheduledFutureTask<> x = (ScheduledFutureTask<>)other;
            long diff = time - x.time;
            if (diff < 0)
            return -1;
            else if (diff > 0)
            return 1;
            else if (sequenceNumber < x.sequenceNumber)
            return -1;
            else
            return 1;
        }
        long d = (getDelay(TimeUnit.NANOSECONDS) -
                other.getDelay(TimeUnit.NANOSECONDS));
        return (d == 0) 0 : ((d < 0) -1 : 1);
}
```

（2）如何实现延时阻塞队列

延时阻塞队列的实现很简单，当消费者从队列里获取元素时，如果元素没有达到延时时间，就阻塞当前线程。

```java
long delay = first.getDelay(TimeUnit.NANOSECONDS);
if (delay <= 0)
    return q.poll();
else if (leader != null)
    available.await();
else {
    Thread thisThread = Thread.currentThread();
    leader = thisThread;
        try {
            available.awaitNanos(delay);
        } finally {
            if (leader == thisThread)
                leader = null;
        }
}
```

代码中的变量leader是一个等待获取队列头部元素的线程。如果leader不等于空，表示已经有线程在等待获取队列的头元素。所以，使用await()方法让当前线程等待信号。如果leader等于空，则把当前线程设置成leader，并使用awaitNanos()方法让当前线程等待接收信号或等待delay时间。

#### 5.SynchronousQueue

SynchronousQueue是一个不存储元素的阻塞队列。每一个put操作必须等待一个take操作，否则不能继续添加元素。

它**支持公平访问队列**。默认情况下线程采用非公平性策略访问队列。使用以下构造方法可以创建公平性访问的SynchronousQueue，如果设置为true，则等待的线程会采用先进先出的顺序访问队列。

```java
public SynchronousQueue(boolean fair) {
    transferer = fair new TransferQueue() : new TransferStack();
}
```

**SynchronousQueue可以看成是一个传球手，负责把生产者线程处理的数据直接传递给消费者线程。队列本身并不存储任何元素，非常适合传递性场景。****<font style="color:#DF2A3F;">SynchronousQueue的吞吐量高于LinkedBlockingQueue和ArrayBlockingQueue。</font>**

#### 6.LinkedTransferQueue

LinkedTransferQueue是一个由链表结构组成的无界阻塞TransferQueue队列。相对于其他阻塞队列，LinkedTransferQueue多了tryTransfer和transfer方法。

（1）transfer方法

**如果当前有消费者正在等待接收元素（消费者使用take()方法或带时间限制的poll()方法时），transfer方法可以把生产者传入的元素立刻transfer（传输）给消费者。如果没有消费者在等待接收元素，transfer方法会将元素存放在队列的tail节点，并等到该元素被消费者消费了才返回。**transfer方法的关键代码如下。

```java
Node pred = tryAppend(s, haveData);
return awaitMatch(s, pred, e, (how == TIMED), nanos);
```

第一行代码是试图把存放当前元素的s节点作为tail节点。第二行代码是让CPU自旋等待消费者消费元素。因为自旋会消耗CPU，所以自旋一定的次数后使用Thread.yield()方法来暂停当前正在执行的线程，并执行其他线程。

（2）tryTransfer方法

tryTransfer方法是用来试探生产者传入的元素是否能直接传给消费者。如果没有消费者等待接收元素，则返回false。和transfer方法的区别是tryTransfer方法无论消费者是否接收，方法立即返回，而transfer方法是必须等到消费者消费了才返回。

对于带有时间限制的tryTransfer（E e，long timeout，TimeUnit unit）方法，试图把生产者传入的元素直接传给消费者，但是如果没有消费者消费该元素则等待指定的时间再返回，如果超时还没消费元素，则返回false，如果在超时时间内消费了元素，则返回true。

#### 7.LinkedBlockingDeque

LinkedBlockingDeque是一个由链表结构组成的**双向阻塞队列**。所谓双向队列指的是可以从队列的两端插入和移出元素。<u>双向队列因为多了一个操作队列的入口，在多线程同时入队时，也就减少了一半的竞争。</u>相比其他的阻塞队列，LinkedBlockingDeque多了addFirst、addLast、offerFirst、offerLast、peekFirst和peekLast等方法，以First单词结尾的方法，表示插入、获取（peek）或移除双端队列的第一个元素。以Last单词结尾的方法，表示插入、获取或移除双端队列的最后一个元素。另外，插入方法add等同于addLast，移除方法remove等效于

removeFirst。但是take方法却等同于takeFirst，不知道是不是JDK的bug，使用时还是用带有First和Last后缀的方法更清楚。

在初始化LinkedBlockingDeque时可以设置容量防止其过度膨胀。另外，双向阻塞队列可以运用在“工作窃取”模式中。

### 6.3.3　阻塞队列的实现原理

如果队列是空的，消费者会一直等待，当生产者添加元素时，消费者是如何知道当前队列有元素的呢？如果让你来设计阻塞队列你会如何设计，如何让生产者和消费者进行高效率的通信呢？让我们先来看看JDK是如何实现的。

**<font style="color:#DF2A3F;">使用通知模式实现。</font>**所谓通知模式，就是当生产者往满的队列里添加元素时会阻塞住生产者，当消费者消费了一个队列中的元素后，会通知生产者当前队列可用。通过查看JDK源码发现ArrayBlockingQueue使用了Condition来实现，代码如下。

```java
private final Condition notFull;
private final Condition notEmpty;
public ArrayBlockingQueue(int capacity, boolean fair) {
        // 省略其他代码
        notEmpty = lock.newCondition();
        notFull = lock.newCondition();
}
public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
            notFull.await();
            insert(e);
        } finally {
            lock.unlock();
        }
} 
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
        try {
        while (count == 0)
        notEmpty.await();
        return extract();
    } finally {
        lock.unlock();
    }
}
private void insert(E x) {
    items[putIndex] = x;
    putIndex = inc(putIndex);
    ++count;
    notEmpty.signal();
}
```

当往队列里插入一个元素时，如果队列不可用，那么阻塞生产者主要通过**<font style="color:#DF2A3F;background-color:#FBDE28;">LockSupport.park（this）</font>**来实现。

```java
public final void await() throws InterruptedException {
        if (Thread.interrupted())
                throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            LockSupport.park(this);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
}
```

继续进入源码，发现调用setBlocker先保存一下将要阻塞的线程，然后调用**<font style="color:#DF2A3F;background-color:#FBDE28;">unsafe.park</font>**阻塞当前线程。

```java
public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        unsafe.park(false, 0L);
        setBlocker(t, null);
}
```

unsafe.park是个native方法，代码如下。

```java
public native void park(boolean isAbsolute, long time);
```

park这个方法会阻塞当前线程，只有以下4种情况中的一种发生时，该方法才会返回。

+ 与park对应的unpark执行或已经执行时。“已经执行”是指unpark先执行，然后再执行park的情况。
+ 线程被中断时。
+ 等待完time参数指定的毫秒数时。
+ 异常现象发生时，这个异常现象没有任何原因。

继续看一下JVM是如何实现park方法：**park在不同的操作系统中使用不同的方式实现，在Linux下使用的是系统方法pthread_cond_wait实现。**实现代码在JVM源码路径src/os/linux/vm/os_linux.cpp里os::PlatformEvent::park方法，代码如下。

```java
void os::PlatformEvent::park() {
    int v ;
        for (;;) {
        v = _Event ;
        if (Atomic::cmpxchg (v-1, &_Event, v) == v) break ;
        }
        guarantee (v >= 0, "invariant") ;
        if (v == 0) {
        // Do this the hard way by blocking ...
        int status = pthread_mutex_lock(_mutex);
        assert_status(status == 0, status, "mutex_lock");
        guarantee (_nParked == 0, "invariant") ;
        ++ _nParked ;
        while (_Event < 0) {
        status = pthread_cond_wait(_cond, _mutex);
        // for some reason, under 2.7 lwp_cond_wait() may return ETIME ...
        // Treat this the same as if the wait was interrupted
        if (status == ETIME) { status = EINTR; }
        assert_status(status == 0 || status == EINTR, status, "cond_wait");
        }
        -- _nParked ;
        // In theory we could move the ST of 0 into _Event past the unlock(),
        // but then we'd need a MEMBAR after the ST.
        _Event = 0 ;
        status = pthread_mutex_unlock(_mutex);
        assert_status(status == 0, status, "mutex_unlock");
        }
        guarantee (_Event >= 0, "invariant") ;
        }
}
```

pthread_cond_wait是一个多线程的条件变量函数，cond是condition的缩写，字面意思可以理解为线程在等待一个条件发生，这个条件是一个全局变量。这个方法接收两个参数：一个共享变量_cond，一个互斥量_mutex。而unpark方法在Linux下是使用pthread_cond_signal实现的。

park方法在Windows下则是使用WaitForSingleObject实现的。想知道pthread_cond_wait是如何实现的，可以参考glibc-2.5的nptl/sysdeps/pthread/pthread_cond_wait.c。

当线程被阻塞队列阻塞时，线程会进入WAITING（parking）状态。我们可以使用jstack dump阻塞的生产者线程看到这点，如下。

```java
"main" prio=5 tid=0x00007fc83c000000 nid=0x10164e000 waiting on condition [0x000000010164d000]
    java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for <0x0000000140559fe8> (a java.util.concurrent.locks.
        AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.
        await(AbstractQueuedSynchronizer.java:2043)
        at java.util.concurrent.ArrayBlockingQueue.put(ArrayBlockingQueue.java:324)
        at blockingqueue.ArrayBlockingQueueTest.main(ArrayBlockingQueueTest.java
```

## 6.4 Fork/Join框架

### 6.4.1　什么是Fork/Join框架

Fork/Join框架是Java 7提供的一个**<font style="color:#DF2A3F;">用于并行执行任务的框架</font>**，**是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。**

我们再通过Fork和Join这两个单词来理解一下Fork/Join框架。Fork就是把一个大任务切分为若干子任务并行的执行，Join就是合并这些子任务的执行结果，最后得到这个大任务的结果。比如计算1+2+…+10000，可以分割成10个子任务，每个子任务分别对1000个数进行求和，最终汇总这10个子任务的结果。Fork/Join的运行流程如图6-6所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708078250192-3010db5e-ba76-4d27-ac9c-377e2373173b.png)

### 6.4.2　工作窃取算法

**工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行。**那么，为什么需要使用工作窃取算法呢？假如我们需要做一个比较大的任务，可以把这个任务分割为若干互不依赖的子任务，为了减少线程间的竞争，把这些子任务分别放到不同的队列里，并<u>为每个队列创建一个单独的线程来执行队列里的任务</u>，线程和队列一一对应。比如A线程负责处理A队列里的任务。但是，有的线程会先把自己队列里的任务干完，而其他线程对应的队列里还有任务等待处理。干完活的线程与其等着，不如去帮其他线程干活，于是它就去其他线程的队列里窃取一个任务来执行。而在这时它们会访问同一个队列，所以**<u>为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。</u>**

工作窃取的运行流程如图6-7所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708078308285-7dbbd4ef-79c6-42e0-896e-e3ee17093239.png)

工作窃取算法的优点：充分利用线程进行并行计算，减少了线程间的竞争。

工作窃取算法的缺点：在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且该算法会消耗了更多的系统资源，比如创建多个线程和多个双端队列。

### 6.4.3　Fork/Join框架的设计

我们已经很清楚Fork/Join框架的需求了，那么可以思考一下，如果让我们来设计一个Fork/Join框架，该如何设计？这个思考有助于你理解Fork/Join框架的设计。

**步骤1**　分割任务。首先我们需要有一个fork类来把大任务分割成子任务，有可能子任务还是很大，所以还需要不停地分割，直到分割出的子任务足够小。

**步骤2**　执行任务并合并结果。分割的子任务分别放在双端队列里，然后几个启动线程分别从双端队列里获取任务执行。子任务执行完的结果都统一放在一个队列里，启动一个线程从队列里拿数据，然后合并这些数据。

Fork/Join使用两个类来完成以上两件事情。

①ForkJoinTask：我们要使用ForkJoin框架，必须首先创建一个ForkJoin任务。它提供在任务中执行fork()和join()操作的机制。通常情况下，我们不需要直接继承ForkJoinTask类，只需要继承它的子类，Fork/Join框架提供了以下两个子类。

+ RecursiveAction：用于没有返回结果的任务。
+ RecursiveTask：用于有返回结果的任务。

②ForkJoinPool：ForkJoinTask需要通过ForkJoinPool来执行。

任务分割出的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。当一个工作线程的队列里暂时没有任务时，它会随机从其他工作线程的队列的尾部获取一个任务。

### 6.4.4　使用Fork/Join框架

让我们通过一个简单的需求来使用Fork/Join框架，需求是：计算1+2+3+4的结果。

使用Fork/Join框架首先要考虑到的是如何分割任务，如果希望每个子任务最多执行两个数的相加，那么我们设置分割的阈值是2，由于是4个数字相加，所以Fork/Join框架会把这个任务fork成两个子任务，子任务一负责计算1+2，子任务二负责计算3+4，然后再join两个子任务的结果。因为是有结果的任务，所以必须继承RecursiveTask，实现代码如下。

```java
package fj;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.Future;
import java.util.concurrent.RecursiveTask;
public class CountTask extends RecursiveTask<Integer> {
    private static final int THRESHOLD = 2;　　// 阈值
    private int start;
    private int end;
    public CountTask(int start, int end) {
        this.start = start;
        this.end = end;
    }
    @Override
    protected Integer compute() {
        int sum = 0;
        // 如果任务足够小就计算任务
        boolean canCompute = (end - start) <= THRESHOLD;
        if (canCompute) {
            for (int i = start; i <= end; i++) {
                sum += i;
            }
        } else {
            // 如果任务大于阈值，就分裂成两个子任务计算
            int middle = (start + end) / 2;
            CountTask leftTask = new CountTask(start, middle);
            CountTask rightTask = new CountTask(middle + 1, end);
            // 执行子任务
            leftTask.fork();
            rightTask.fork();
            // 等待子任务执行完，并得到其结果
            int leftResult=leftTask.join();
            int rightResult=rightTask.join();
            // 合并子任务
            sum = leftResult + rightResult;
        }
        return sum;
    }
    public static void main(String[] args) {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        // 生成一个计算任务，负责计算1+2+3+4
        CountTask task = new CountTask(1, 4);
        // 执行一个任务
        Future<Integer> result = forkJoinPool.submit(task);
        try {
            System.out.println(result.get());
        } catch (InterruptedException e) {
        } catch (ExecutionException e) {
        }
    }
}
```

通过这个例子，我们进一步了解ForkJoinTask，**ForkJoinTask与一般任务的主要区别在于它需要实现compute方法**，在这个方法里，首先需要判断任务是否足够小，如果足够小就直接执行任务。如果不足够小，就必须分割成两个子任务，每个子任务在调用fork方法时，又会进入compute方法，看看当前子任务是否需要继续分割成子任务，如果不需要继续分割，则执行当前子任务并返回结果。使用join方法会等待子任务执行完并得到其结果。

### 6.4.5　Fork/Join框架的异常处理

ForkJoinTask在执行的时候可能会抛出异常，但是我们没办法在主线程里直接捕获异常，所以ForkJoinTask提供了isCompletedAbnormally()方法来检查任务是否已经抛出异常或已经被取消了，并且可以通过ForkJoinTask的getException方法获取异常。使用如下代码。

```java
if(task.isCompletedAbnormally())
{
    System.out.println(task.getException());
}
```

getException方法返回Throwable对象，如果任务被取消了则返回CancellationException。如果任务没有完成或者没有抛出异常则返回null。

### 6.4.6　Fork/Join框架的实现原理

ForkJoinPool由ForkJoinTask数组和ForkJoinWorkerThread数组组成，

+ ForkJoinTask数组负责将存放程序提交给ForkJoinPool的任务；
+ ForkJoinWorkerThread数组负责执行这些任务。

（1）ForkJoinTask的fork方法实现原理

当我们调用ForkJoinTask的fork方法时，程序会调用ForkJoinWorkerThread的pushTask方法异步地执行这个任务，然后立即返回结果。代码如下。

```java
public final ForkJoinTask<V> fork() {
    ((ForkJoinWorkerThread) Thread.currentThread())
    .pushTask(this);
    return this;
}
```

pushTask方法把当前任务存放在ForkJoinTask数组队列里。然后再调用ForkJoinPool的signalWork()方法唤醒或创建一个工作线程来执行任务。代码如下。

```java
final void pushTask(ForkJoinTask<> t) {
    ForkJoinTask<>[] q; int s, m;
    if ((q = queue) != null) {　　　　// ignore if queue removed
        long u = (((s = queueTop) & (m = q.length - 1)) << ASHIFT) + ABASE;
        UNSAFE.putOrderedObject(q, u, t);
        queueTop = s + 1;　　　　　　// or use putOrderedInt
        if ((s -= queueBase) <= 2)
            pool.signalWork();
        else if (s == m)
            growQueue();
    }
}
```

（2）ForkJoinTask的join方法实现原理

Join方法的主要作用是阻塞当前线程并等待获取结果。让我们一起看看ForkJoinTask的join方法的实现，代码如下。

```java
public final V join() {
    if (doJoin() != NORMAL)
        return reportResult();
    else
        return getRawResult();
} private V
reportResult() {
    int s; Throwable ex;
    if ((s = status) == CANCELLED)
        throw new CancellationException();
    if (s == EXCEPTIONAL && (ex = getThrowableException()) != null)
        UNSAFE.throwException(ex);
    return getRawResult();
}
```

首先，它调用了doJoin()方法，通过doJoin()方法得到当前任务的状态来判断返回什么结果，任务状态有4种：已完成（NORMAL）、被取消（CANCELLED）、信号（SIGNAL）和出现异常（EXCEPTIONAL）。

·如果任务状态是已完成，则直接返回任务结果。

·如果任务状态是被取消，则直接抛出CancellationException。

·如果任务状态是抛出异常，则直接抛出对应的异常。

让我们再来分析一下doJoin()方法的实现代码。

```java
private int doJoin() {
    Thread t; ForkJoinWorkerThread w; int s; boolean completed;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) {
        if ((s = status) < 0)
            return s;
        if ((w = (ForkJoinWorkerThread)t).unpushTask(this)) {
            try {
                completed = exec();
            } catch (Throwable rex) {
                return setExceptionalCompletion(rex);
            }
            if (completed)
                return setCompletion(NORMAL);
        }
        return w.joinTask(this);
    }
    else
        return externalAwaitDone();
}
```

在doJoin()方法里，首先通过查看任务的状态，看任务是否已经执行完成，如果执行完成，则直接返回任务状态；如果没有执行完，则从任务数组里取出任务并执行。如果任务顺利执行完成，则设置任务状态为NORMAL，如果出现异常，则记录异常，并将任务状态设置为EXCEPTIONAL。

# 第七章 Java中的13个原子操作类

当程序更新一个变量时，如果多线程同时更新这个变量，可能得到期望之外的值，比如变量i=1，A线程更新i+1，B线程也更新i+1，经过两个线程操作之后可能i不等于3，而是等于2。因为A和B线程在更新变量i的时候拿到的i都是1，这就是线程不安全的更新操作，通常我们会使用synchronized来解决这个问题，synchronized会保证多线程不会同时更新变量i。

而Java从JDK 1.5开始提供了java.util.concurrent.atomic包（以下简称Atomic包），这个包中的原子操作类提供了一种用法简单、性能高效、线程安全地更新一个变量的方式。

因为变量的类型有很多种，所以在Atomic包里一共提供了13个类，属于4种类型的原子更新方式，分别是原子更新基本类型、原子更新数组、原子更新引用和原子更新属性（字段）。Atomic包里的类基本都是使用Unsafe实现的包装类。

# 第八章 Java中的并发工具类

## 8.0 总结

## 8.1 等待多线程完成的CountDownLatch*

CountDownLatch允许一个或多个线程等待其他线程完成操作。

假如有这样一个需求：我们需要解析一个Excel里多个sheet的数据，此时可以考虑使用多线程，每个线程解析一个sheet里的数据，等到所有的sheet都解析完之后，程序需要提示解析完成。在这个需求中，要实现主线程等待所有线程完成sheet的解析操作，最简单的做法是使用join()方法，如代码清单8-1所示。

```java
public class JoinCountDownLatchTest {
    public static void main(String[] args) throws InterruptedException {
        Thread parser1 = new Thread(new Runnable() {
            @Override
            public void run() {
            }
        });
        Thread parser2 = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("parser2 finish");
            }
        });
        parser1.start();
        parser2.start();
        parser1.join();
        parser2.join();
        System.out.println("all parser finish");
    }
}
```

join用于让当前执行线程等待join线程执行结束。其实现原理是不停检查join线程是否存活，如果join线程存活则让当前线程永远等待。其中，wait（0）表示永远等待下去，代码片段如下。

```java
while (isAlive()) {
    wait(0);
}
```

直到join线程中止后，线程的this.notifyAll()方法会被调用，调用notifyAll()方法是在JVM里实现的，所以在JDK里看不到，大家可以查看JVM源码。

在JDK 1.5之后的并发包中提供的CountDownLatch也可以实现join的功能，并且比join的功能更多，如代码清单8-2所示。

```java
public class CountDownLatchTest {
    staticCountDownLatch c = new CountDownLatch(2);
    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(1);
                c.countDown();
                System.out.println(2);
                c.countDown();
            }
        }).start();
        c.await();
        System.out.println("3");
    }
}
```

CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果你想等待N个点完成，这里就传入N。

当我们调用CountDownLatch的countDown方法时，N就会减1，CountDownLatch的await方法会阻塞当前线程，直到N变成零。由于countDown方法可以用在任何地方，所以这里说的N个点，可以是N个线程，也可以是1个线程里的N个执行步骤。用在多个线程时，只需要把这个CountDownLatch的引用传递到线程里即可。

如果有某个解析sheet的线程处理得比较慢，我们不可能让主线程一直等待，所以可以使用另外一个带指定时间的await方法——await（long time，TimeUnit unit），这个方法等待特定时间后，就会不再阻塞当前线程。join也有类似的方法。

注意　计数器必须大于等于0，只是等于0时候，计数器就是零，调用await方法时不会阻塞当前线程。CountDownLatch不可能重新初始化或者修改CountDownLatch对象的内部计数器的值。一个线程调用countDown方法happen-before，另外一个线程调用await方法。

## 8.2 同步屏障CyclicBarrier

CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

### 8.2.1　CyclicBarrier简介

CyclicBarrier默认的构造方法是CyclicBarrier（int parties），其参数表示屏障拦截的线程数量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。示例

代码如代码清单8-3所示。

```java
public class CyclicBarrierTest {
    staticCyclicBarrier c = new CyclicBarrier(2);
    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    c.await();
                } catch (Exception e) {
                }
                System.out.println(1);
            }
        }).start();
        try {
            c.await();
        } catch (Exception e) {
        }
        System.out.println(2);
    }
}
```

因为主线程和子线程的调度是由CPU决定的，两个线程都有可能先执行，所以会产生两种输出，第一种可能输出如下。

```c
1
2    
```

第二种可能的输出如下:

```c
2
1
```

如果把new CyclicBarrier(2)修改成new CyclicBarrier(3)，则主线程和子线程会永远等待，因为没有第三个线程执行await方法，即没有第三个线程到达屏障，所以之前到达屏障的两个线程都不会继续执行。

CyclicBarrier还提供一个更高级的构造函数CyclicBarrier（int parties，Runnable barrier-Action），用于在线程到达屏障时，优先执行barrierAction，方便处理更复杂的业务场景，如代码清单8-4所示。

```java
import java.util.concurrent.CyclicBarrier;
public class CyclicBarrierTest2 {
    static CyclicBarrier c = new CyclicBarrier(2, new A());
    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    c.await();
                } catch (Exception e) {
                }
                System.out.println(1);
            }
        }).start();
        try {
            c.await();
        } catch (Exception e) {
        }
        System.out.println(2);
    }
    static class A implements Runnable {
        @Override
        public void run() {
            System.out.println(3);
        }
    }
}
```

因为CyclicBarrier设置了拦截线程的数量是2，所以必须等代码中的第一个线程和线程A都执行完之后，才会继续执行主线程，然后输出2，所以代码执行后的输出如下。

```c
3
1
2
```

### 8.2.2　CyclicBarrier的应用场景

CyclicBarrier可以用于多线程计算数据，最后合并计算结果的场景。例如，用一个Excel保存了用户所有银行流水，每个Sheet保存一个账户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet的日均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个Excel的日均银行流水，如代码清单8-5所示。

```java
import java.util.Map.Entry;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;
/**
* 银行流水处理服务类
*
* @authorftf
*
*/
publicclass BankWaterService implements Runnable {
    /**
* 创建4个屏障，处理完之后执行当前类的run方法
*/
    private CyclicBarrier c = new CyclicBarrier(4, this);
    /**
* 假设只有4个sheet，所以只启动4个线程
*/
    private Executor executor = Executors.newFixedThreadPool(4);
    /**
* 保存每个sheet计算出的银流结果
*/
    private ConcurrentHashMap<String, Integer>sheetBankWaterCount = new
    ConcurrentHashMap<String, Integer>();
    privatevoid count() {
        for (inti = 0; i< 4; i++) {
            executor.execute(new Runnable() {
                @Override
                publicvoid run() {
                    // 计算当前sheet的银流数据，计算代码省略
                    sheetBankWaterCount
                    .put(Thread.currentThread().getName(), 1);
                    // 银流计算完成，插入一个屏障
                    try {
                        c.await();
                    } catch (InterruptedException |
                             BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }
    @Override
    publicvoid run() {
        intresult = 0;
        // 汇总每个sheet计算出的结果
        for (Entry<String, Integer>sheet : sheetBankWaterCount.entrySet()) {
            result += sheet.getValue();
        }
        // 将结果输出
        sheetBankWaterCount.put("result", result);
        System.out.println(result);
    }
    publicstaticvoid main(String[] args) {
        BankWaterService bankWaterCount = new BankWaterService();
        bankWaterCount.count();
    }
}
```

使用线程池创建4个线程，分别计算每个sheet里的数据，每个sheet计算结果是1，再由BankWaterService线程汇总4个sheet计算出的结果，输出结果如下。

```c
4
```

8.2.3　CyclicBarrier和CountDownLatch的区别

CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置。所以CyclicBarrier能处理更为复杂的业务场景。例如，如果计算发生错误，可以重置计数器，并让线程重新执行一次。

CyclicBarrier还提供其他有用的方法，比如getNumberWaiting方法可以获得Cyclic-Barrier阻塞的线程数量。isBroken()方法用来了解阻塞的线程是否被中断。代码清单8-5执行完之后会返回true，其中isBroken的使用代码如代码清单8-6所示。

```java
importjava.util.concurrent.BrokenBarrierException;
importjava.util.concurrent.CyclicBarrier;
public class CyclicBarrierTest3 {
    staticCyclicBarrier c = new CyclicBarrier(2);
    public static void main(String[] args) throws InterruptedException，
    BrokenBarrierException {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    c.await();
                } catch (Exception e) {
                }
            }
        });
        thread.start();
        thread.interrupt();
        try {
            c.await();
        } catch (Exception e) {
            System.out.println(c.isBroken()); // true
        }
    }
}
```

## 8.3 控制线程并发数的Semaphore

Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。

多年以来，我都觉得从字面上很难理解Semaphore所表达的含义，只能把它比作是控制流量的红绿灯。比如××马路要限制流量，只允许同时有一百辆车在这条路上行使，其他的都必须在路口等待，所以前一百辆车会看到绿灯，可以开进这条马路，后面的车会看到红灯，不能驶入××马路，但是如果前一百辆中有5辆车已经离开了××马路，那么后面就允许有5辆车驶入马路，这个例子里说的车就是线程，驶入马路就表示线程在执行，离开马路就表示线程执行完成，看见红灯就表示线程被阻塞，不能执行。

### 1.应用场景

Semaphore可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库连接。假如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发地读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有10个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连接。这个时候，就可以使用Semaphore来做流量控制，如代码清单8-7所示。

```java
public class SemaphoreTest {
    private static final int THREAD_COUNT = 30;
    private static ExecutorServicethreadPool = Executors
    .newFixedThreadPool(THREAD_COUNT);
    private static Semaphore s = new Semaphore(10);
    public static void main(String[] args) {
        for (inti = 0; i< THREAD_COUNT; i++) {
            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        s.acquire();
                        System.out.println("save data");
                        s.release();
                    } catch (InterruptedException e) {
                    }
                }
            });
        }
        threadPool.shutdown();
    }
}
```

在代码中，虽然有30个线程在执行，但是只允许10个并发执行。Semaphore的构造方法Semaphore（int permits）接受一个整型的数字，表示可用的许可证数量。Semaphore（10）表示允许10个线程获取许可证，也就是最大并发数是10。Semaphore的用法也很简单，首先线程使用Semaphore的acquire()方法获取一个许可证，使用完之后调用release()方法归还许可证。还可以用tryAcquire()方法尝试获取许可证。

### 2.其他方法

Semaphore还提供一些其他方法，具体如下。

+ intavailablePermits()：返回此信号量中当前可用的许可证数。
+ intgetQueueLength()：返回正在等待获取许可证的线程数。
+ booleanhasQueuedThreads()：是否有线程正在等待获取许可证。
+ void reducePermits（int reduction）：减少reduction个许可证，是个protected方法。
+ Collection getQueuedThreads()：返回所有等待获取许可证的线程集合，是个protected方法。

## 8.4 线程间交换数据的Exchanger

Exchanger（交换者）是一个用于线程间协作的工具类。Exchanger用于进行线程间的数据交换。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这两个线程通过exchange方法交换数据，如果第一个线程先执行exchange()方法，它会一直等待第二个线程也执行exchange方法，当两个线程都到达同步点时，这两个线程就可以交换数据，将本线程生产出来的数据传递给对方。

下面来看一下Exchanger的应用场景。

Exchanger可以用于遗传算法，遗传算法里需要选出两个人作为交配对象，这时候会交换两人的数据，并使用交叉规则得出2个交配结果。Exchanger也可以用于校对工作，比如我们需要将纸制银行流水通过人工的方式录入成电子银行流水，为了避免错误，采用AB岗两人进行录入，录入到Excel之后，系统需要加载这两个Excel，并对两个Excel数据进行校对，看看是否录入一致，代码如代码清单8-8所示。

```java
public class ExchangerTest {
    private static final Exchanger<String>exgr = new Exchanger<String>();
    private static ExecutorServicethreadPool = Executors.newFixedThreadPool(2);
    public static void main(String[] args) {
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    String A = "银行流水A";　　　　// A录入银行流水数据
                    exgr.exchange(A);
                } catch (InterruptedException e) {
                }
            }
        });
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    String B = "银行流水B";　　　　// B录入银行流水数据
                    String A = exgr.exchange("B");
                    System.out.println("A和B数据是否一致：" + A.equals(B) + "，A录入的是："
                                       + A + "，B录入是：" + B);
                } catch (InterruptedException e) {
                }
            }
        });
        threadPool.shutdown();
    }
}
```

如果两个线程有一个没有执行exchange()方法，则会一直等待，如果担心有特殊情况发生，避免一直等待，可以使用exchange（V x，longtimeout，TimeUnit unit）设置最大等待时长。

## 8.5 本章小结

本章配合一些应用场景介绍JDK中提供的几个并发工具类，大家记住这个工具类的用途，一旦有对应的业务场景，不妨试试这些工具类。

# 第九章 Java中的线程池

Java中的线程池是运用场景最多的**并发框架**，几乎所有需要**<font style="color:#DF2A3F;">异步</font>**或**<font style="color:#DF2A3F;">并发执行任务</font>**的程序都可以使用线程池。在开发过程中，合理地使用线程池能够带来3个好处。

第一：**降低资源消耗。**通过重复利用已创建的线程降低线程创建和销毁造成的消耗。

第二：**提高响应速度。**当任务到达时，任务可以不需要等到线程创建就能立即执行。

第三：**提高线程的可管理性。**线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源，

还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。但是，要做到合理利用线程池，必须对其实现原理了如指掌。

## 9.0 总结

+ 首先最基本的，创建线程的4种方式。
+ 其次基本的，线程池的工作流程。
+ 然后我们要知道线程的创建和销毁比较消耗资源，其中很重要的一个点就是**<font style="color:#D22D8D;">要获取全局锁</font>**！
+ ThreadPoolExecutor采取运行步骤的总体设计思路，**<font style="color:#4C16B1;">是为了在执行execute()方法时，尽可能地避免获取全局锁</font>**（那将会是一个严重的可伸缩瓶颈）。**在ThreadPoolExecutor完成预热之后（****<font style="color:#DF2A3F;">当前运行的线程数大于等于corePoolSize</font>****），几乎所有的execute()方法调用都是执行步骤2，而步骤2不需要获取全局锁。**
+ **线程池中有一个全局变量**`**ctl**`**的数据结构是一个**`**AtomicInteger**`**，通过位运算来保存线程池的状态、工作线程数量等信息。这个复杂的设计是**<u>为了支持线程池的各种状态转换和线程数量的动态调整。</u>
+ _<font style="color:#0C68CA;background-color:#FAFAFA;">---好的，那么这里是否说明了一个问题，那就是我们在向线程池提交任务的时候，</font>__**<font style="color:#0C68CA;background-color:#FAFAFA;">如果我们的线程数没有达到核心线程数的时候，不管是否有线程空闲，都会新建线程用于执行当前提交的任务，</font>**__<u><font style="color:#0C68CA;background-color:#FAFAFA;">当创建线程的数目达到核心线程数的时候，才会将新的任务加入到任务队列中。</font></u>__<font style="color:#0C68CA;background-color:#FAFAFA;">核心线程会不断从任务队列中取任务进行执行，当任务队列满之后，才会创建救急线程。   ---> 其实中间有一个很重要的点就是，我们创建线程是需要获取同步锁的，这是一个重量级操作，所以我们更希望能少创建线程，亦或是我们可以说更希望用核心线程去任务队列中取线程这样的方式（这样我们在执行任务的时候其实都没有创建锁），效率是最高的。---</font>_
+ <u>通过扩展线程池进行监控。可以通过继承线程池来自定义线程池，重写线程池的beforeExecute、afterExecute和terminated方法，也可以在任务执行前、执行后和线程池关闭前执行一些代码来进行监控。</u>
+ 线程池的任务提交有两种方式，submit和exectue两种方式，submit有返回值。
+ **关于线程池，直接去看Executor那一章叭，讲的很详细！**

## 9.1 线程池的实现原理

**<font style="color:#DF2A3F;">当向线程池提交一个任务之后，线程池是如何处理这个任务的呢？</font>**本节来看一下线程池的主要处理流程，处理流程图如图9-1所示。

从图中可以看出，当提交一个新任务到线程池时，线程池的处理流程如下。

1）线程池判断核心线程池里的线程是否都在执行任务。如果不是，则创建一个新的工作线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下个流程。

2）线程池判断工作队列是否已经满。如果工作队列没有满，则将新提交的任务存储在这个工作队列里。如果工作队列满了，则进入下个流程。

3）线程池判断线程池的线程是否都处于工作状态。如果没有，则创建一个新的工作线程来执行任务。如果已经满了，则交给饱和策略来处理这个任务。

ThreadPoolExecutor执行execute()方法的示意图：

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707970502430-9d26d80d-aa18-494d-9d4f-ff5d161637c0.png)

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707970560628-4a3652bc-a35c-45d3-8fa5-a08d5e68b0bf.png)

ThreadPoolExecutor执行execute方法分下面4种情况。

1）如果当前运行的线程少于corePoolSize，则创建新线程来执行任务（注意，执行这一步骤**<font style="color:#DF2A3F;">需要获取全局锁</font>**）。

2）如果**运行的线程等于或多于corePoolSize**，则将任务加入BlockingQueue。

3）如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务（注意，执行这一步骤需要<font style="color:#DF2A3F;">获取全局锁</font>）。

4）如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用RejectedExecutionHandler.rejectedExecution()方法。

ThreadPoolExecutor采取上述步骤的总体设计思路，**<font style="color:#4C16B1;">是为了在执行execute()方法时，尽可能地避免获取全局锁</font>**（那将会是一个严重的可伸缩瓶颈）。**在ThreadPoolExecutor完成预热之后（****<font style="color:#DF2A3F;">当前运行的线程数大于等于corePoolSize</font>****），几乎所有的execute()方法调用都是执行步骤2，而步骤2不需要获取全局锁。**

**源码分析：**上面的流程分析让我们很直观地了解了线程池的工作原理，让我们再通过源代码来看看是如何实现的，线程池执行任务的方法如下。

```java
public void execute(Runnable command) {
    if (command == null){
    	throw new NullPointerException();
    }

    // 如果线程数小于基本线程数，则创建线程并执行当前任务
    
	if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
	// 如线程数大于等于基本线程数或线程创建失败，则将当前任务放到工作队列中。
    
        if (runState == RUNNING && workQueue.offer(command)) {
            if (runState != RUNNING || poolSize == 0)
                ensureQueuedTaskHandled(command);
    } // 如果线程池不处于运行中，或者任务无法放入队列，并且当前线程数量小于最大允许的线程数量
    // 则创建一个线程执行任务。
    
    else if (!addIfUnderMaximumPoolSize(command))
    // 抛出RejectedExecutionException异常
    reject(command); // is shutdown or saturated
    }
}
```

工作线程：**线程池创建线程时，会将线程封装成工作线程Worker**，**Worker在执行完任务后，还会循环获取工作队列里的任务来执行。**我们可以从Worker类的run()方法里看到这点。

```java
public void run() {
    try {
        Runnable task = firstTask;
        firstTask = null;
        while (task != null || (task = getTask()) != null) {
            runTask(task);
            task = null;
        }
    } finally {
        workerDone(this);
    }
}
```

ThreadPoolExecutor中线程执行任务的示意图如图9-3所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707979149508-1b177114-048b-4482-b690-ab0daf66304a.png)

线程池中的线程执行任务分两种情况，如下。

<font style="color:#2F4BDA;">1）</font>**<font style="color:#2F4BDA;">在execute()方法中创建一个线程时，会让这个线程执行当前任务。</font>**

<font style="color:#2F4BDA;">2）</font>**<font style="color:#2F4BDA;">这个线程执行完上图中1的任务后，会反复从BlockingQueue获取任务来执行。</font>**

## 9.2 线程池的使用

### 9.2.1线程池的创建*

实际上，线程池的状态、核心线程数和救急线程数通常是通过多个变量来保存的，而不是单一的`long`型变量。Java中的`ThreadPoolExecutor`类内部会使用多个成员变量来维护这些信息。

例如，`ThreadPoolExecutor`中的一些关键成员变量包括：

1. `<font style="color:#DF2A3F;">ctl</font>`<font style="color:#DF2A3F;">（原子变量，保存了线程池的状态和工作线程数量等信息）</font>
2. `corePoolSize`（线程池的基本大小，保存核心线程数）
3. `maximumPoolSize`（线程池的最大大小，保存最大线程数）

`ctl`的数据结构是一个`AtomicInteger`**，通过位运算来保存线程池的状态、工作线程数量等信息。**这个复杂的设计是<u>为了支持线程池的各种状态转换和线程数量的动态调整。</u>

所以，总体而言，并非一个简单的`long`型变量，而是通过多个变量来保存线程池的状态和相关信息。

我们可以通过ThreadPoolExecutor来创建一个线程池。

```java
new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime,
					   milliseconds,runnableTaskQueue, handler);
```

创建一个线程池时需要输入几个参数，如下。

1）corePoolSize（线程池的基本大小）：**当提交一个任务到线程池时，线程池会创建一个线程来执行任务，****<font style="color:#DF2A3F;">即使其他空闲的基本线程能够执行新任务也会创建线程</font>****，等到需要执行的任务数大于线程池基本大小时就不再创建。**<u>如果调用了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有基本线程。</u>

_<font style="color:#0C68CA;background-color:#FAFAFA;">---好的，那么这里是否说明了一个问题，那就是我们在向线程池提交任务的时候，</font>__**<font style="color:#0C68CA;background-color:#FAFAFA;">如果我们的线程数没有达到核心线程数的时候，不管是否有线程空闲，都会新建线程用于执行当前提交的任务，</font>**__<u><font style="color:#0C68CA;background-color:#FAFAFA;">当创建线程的数目达到核心线程数的时候，才会将新的任务加入到任务队列中。</font></u>__<font style="color:#0C68CA;background-color:#FAFAFA;">核心线程会不断从任务队列中取任务进行执行，当任务队列满之后，才会创建救急线程。   ---> 其实中间有一个很重要的点就是，</font>__**<font style="color:#DF2A3F;background-color:#FAFAFA;">我们创建线程是需要获取同步锁的</font>**__<font style="color:#0C68CA;background-color:#FAFAFA;">，这是一个重量级操作，所以我们更希望能少创建线程，亦或是我们可以说更希望用核心线程去任务队列中取线程这样的方式（这样我们在执行任务的时候其实都没有创建锁），效率是最高的。---</font>_

2）runnableTaskQueue（任务队列）：用于保存等待执行的任务的阻塞队列。可以选择以下几个阻塞队列。

+ ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按FIFO（先进先出）原则对元素进行排序。
+ LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
+ SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
+ PriorityBlockingQueue：一个具有优先级的无限阻塞队列。

3）maximumPoolSize（线程池最大数量）：线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是，如果使用了无界的任务队列这个参数就没什么效果。

4）ThreadFactory：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。使用开源框架guava提供的ThreadFactoryBuilder可以快速给线程池里的线程设置有意义的名字，代码如下。

```java
new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build();
```

5）RejectedExecutionHandler（饱和策略）：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是`AbortPolicy`，表示无法处理新任务时抛出异常。在JDK 1.5中Java线程池框架提供了以下4种策略。

+ AbortPolicy：<font style="color:#DF2A3F;">直接抛出异常。</font>
+ CallerRunsPolicy：只用调用者所在线程来运行任务。
+ DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
+ DiscardPolicy：不处理，丢弃掉。

当然，也可以根据应用场景需要来实现`RejectedExecutionHandler`接口自定义策略。如记录日志或持久化存储不能处理的任务。

6）keepAliveTime（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以，

如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率。

7）TimeUnit（线程活动保持时间的单位）：可选的单位有天（DAYS）、小时（HOURS）、分钟

（MINUTES）、毫秒（MILLISECONDS）、微秒（MICROSECONDS，千分之一毫秒）和纳秒

（NANOSECONDS，千分之一微秒）。

`keepAliveTime`（线程活动保持时间）是指线程池的工作线程空闲后保持存活的时间。对于线程池而言，它通常分为核心线程和救急线程（也叫最大线程数的线程）。

+  对于核心线程来说，它们是始终保持活动的，不会被销毁，即使它们空闲。<font style="color:#DF2A3F;">这是为了提供一种快速响应任务的机制</font>，避免频繁创建和销毁线程的开销。
+  对于救急线程，当线程池中的线程数量超过核心线程数时，会根据情况创建新的线程来执行任务。而`keepAliveTime`的作用则是规定了当救急线程在空闲一段时间后会被销毁，以节省系统资源。这可以避免在任务繁忙的时候保持大量空闲线程。

因此，`keepAliveTime`主要影响救急线程的存活时间，而核心线程是不会被销毁的。如果任务执行时间短且任务量大，可以适当调大`keepAliveTime`，以提高线程的利用率，并减少线程的创建和销毁频率。

### 9.2.2向线程池提交任务

可以使用两个方法向线程池提交任务，分别为execute()和submit()方法。

execute()方法用于提交不需要返回值的任务，所以<font style="color:#DF2A3F;">无法判断任务是否被线程池执行成功</font>。

通过以下代码可知**execute()方法输入的任务是一个Runnable类的实例。**

```java
threadsPool.execute(new Runnable() {
                        @Override
                        public void run() {
                        	// TODO Auto-generated method stub
                        }
                });
```

submit()方法用于提交需要返回值的任务。**线程池会返回一个future类型的对象**，通过这个future对象可以判断任务是否执行成功，并且可以通过future的get()方法来获取返回值，get()方法会阻塞当前线程直到任务完成，而使用get（long timeout，TimeUnit unit）方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。

```java
Future<Object> future = executor.submit(harReturnValuetask);
//上面代码提交任务执行之后，返回一个future对象，我们调用future对象的get方法可以获得任务执行的结果
try {
    Object s = future.get();
} catch (InterruptedException e) {
    // 处理中断异常
} catch (ExecutionException e) {
    // 处理无法执行任务异常
} finally {
    // 关闭线程池
    executor.shutdown();
}
```

### 9.2.3关闭线程池

可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池。它们的**原理**是**遍历线程池中的工作线程，然后逐个调用线程的****<font style="color:#DF2A3F;background-color:#FBDE28;">interrupt</font>****方法来中断线程，****<font style="color:#DF2A3F;">所以无法响应中断的任务可能永远无法终止</font>****。**但是它们存在一定的区别：

+ shutdownNow首先将线程池的状态设置成STOP，然后**尝试停止所有的正在执行或暂停任务的线程**，并返回等待执行任务的列表；
+ 而shutdown只是将线程池的状态设置成SHUTDOWN状态，然后**中断所有没有正在执行任务的线程。**

只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true。当所有的任务都已关闭后，才表示线程池关闭成功，这时调用isTerminaed方法会返回true。至于应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，通常调用shutdown方法来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow方法。

### 9.2.4合理配置线程池

要想合理地配置线程池，就必须首先分析任务特性，可以从以下几个角度来分析。

+ 任务的性质：CPU密集型任务、IO密集型任务和混合型任务。
+ 任务的优先级：高、中和低。
+ 任务的执行时间：长、中和短。
+ 任务的依赖性：是否依赖其他系统资源，如数据库连接。

性质不同的任务可以用不同规模的线程池分开处理。

+ CPU密集型任务应配置尽可能小的线程，如配置Ncpu+1个线程的线程池。
+ 由于IO密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程，如2*Ncpu。
+ 混合型的任务，如果可以拆分，将其拆分成一个CPU密集型任务和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐量将高于串行执行的吞吐量。如果这两个任务执行时间相差太大，则没必要进行分解。

可以通过Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数。

**优先级不同的任务可以使用优先级队列PriorityBlockingQueue来处理。它可以让优先级高的任务先执行。**

注意：

如果一直有优先级高的任务提交到队列里，那么优先级低的任务可能永远不能执行。

执行时间不同的任务可以交给不同规模的线程池来处理，或者可以使用优先级队列，让执行时间短的任务先执行。

依赖数据库连接池的任务，因为线程提交SQL后需要等待数据库返回结果，**等待的时间越长，则CPU空闲时间就越长，那么线程数应该设置得越大，这样才能更好地利用CPU。**

**建议使用有界队列。**有界队列能增加系统的稳定性和预警能力，可以根据需要设大一点儿，比如几千。有一次，我们系统里后台任务线程池的队列和线程池全满了，不断抛出抛弃任务的异常，通过排查发现是数据库出现了问题，导致执行SQL变得非常缓慢，因为后台任务线程池里的任务全是需要向数据库查询和插入数据的，所以导致线程池里的工作线程全部阻塞，任务积压在线程池里。如果当时我们设置成无界队列，那么线程池的队列就会越来越多，有可能会撑满内存，导致整个系统不可用，而不只是后台任务出现问题。当然，我们的系统所有的任务是用单独的服务器部署的，我们使用不同规模的线程池完成不同类型的任务，但是出现这样问题时也会影响到其他任务。

### 9.2.5线程池的监控

如果在系统中大量使用线程池，则有必要对线程池进行监控，方便在出现问题时，可以根据线程池的使用状况快速定位问题。可以通过线程池提供的参数进行监控，在监控线程池的时候可以使用以下属性。

+ taskCount：线程池需要执行的任务数量。
+ completedTaskCount：线程池在运行过程中已完成的任务数量，小于或等于taskCount。
+ largestPoolSize：线程池里曾经创建过的最大线程数量。**通过这个数据可以知道线程池是否曾经满过**。如该数值等于线程池的最大大小，则表示线程池曾经满过。
+ getPoolSize：线程池的线程数量。如果线程池不销毁的话，线程池里的线程不会自动销毁，所以这个大小只增不减。
+ getActiveCount：获取活动的线程数。

<u>通过扩展线程池进行监控。可以通过继承线程池来自定义线程池，重写线程池的beforeExecute、afterExecute和terminated方法，也可以在任务执行前、执行后和线程池关闭前执行一些代码来进行监控。</u>例如，监控任务的平均执行时间、最大执行时间和最小执行时间等。

这几个方法在线程池里是空方法。

```java
protected void beforeExecute(Thread t, Runnable r) { }
```

# 第十章 Executor框架

在Java中，使用线程来异步执行任务。Java线程的创建与销毁需要一定的开销，如果我们为每一个任务创建一个新线程来执行，这些线程的创建与销毁将消耗大量的计算资源。同时，为每一个任务创建一个新线程来执行，这种策略可能会使处于高负荷状态的应用最终崩溃。

Java的线程既是工作单元，也是执行机制。**从JDK 5开始，****<font style="color:#DF2A3F;">把工作单元与执行机制分离开来。</font>****工作单元包括Runnable和Callable，而执行机制由Executor框架提供。**

## 10.0 总结

+ 首先我们要理解的是这个两级调度模型，也就是我们将任务提交给线程池，然后线程池的Executor框架会将任务进行划分，划分后为一个个的可以被单个线程执行的任务，然后将任务分配给线程，Java的线程又和操作系统的线程一一对应，这个对应关系有一个操作系统的线程调度系统来做。
+ _<font style="color:#0C68CA;">---我们来理解一下，也就是我们在提交任务到线程池的时候，</font>__**<font style="color:#0C68CA;">好像是一个任务提交给一个线程，其实不然</font>**__<font style="color:#0C68CA;">，</font>__<u><font style="color:#0C68CA;">我们提交的任务会由Executor给我们进行划分</font></u>__<font style="color:#0C68CA;">，然后再分配给线程池中的线程，然后线程池中的一个线程对应操作系统中的一个线程，这样就可以实现一个任务的无感执行，开发者只用提交任务即可，细节有Executor框架帮我们实现。---</font>_
+ 然后就是这个Executor框架的一些内容了：
    - 首先是任务模型：Runnable和Callable接口（Runnable可以转换成Callable）；
    - 然后是两个线程池模型**<font style="color:#DF2A3F;">ThreadPoolExecutor</font>****和ScheduledThreadPoolExecutor（**定时任务的线程池模型或者延时任务的线程池模型**）；**
    - 获取返回结果的Future和FutureTask类（可用于异步计算）；
+

## 10.1 Executor框架简介*

### 10.1.1 Executor框架的两级调度模型

在HotSpot VM的线程模型中，**Java线程（java.lang.Thread）被一对一映射为本地操作系统线程。**<font style="color:#DF2A3F;">Java线程启动时会创建一个本地操作系统线程；当该Java线程终止时，这个操作系统线程也会被回收。</font>**操作系统会调度所有线程并将它们分配给可用的CPU。**

+ 在上层，**Java多线程程序通常把应用分解为若干个任务**，<u>然后使用用户级的调度器（Executor框架）将这些任务映射为固定数量的线程；</u>
+ 在底层，操作系统内核将这些线程映射到硬件处理器上。这种两级调度模型的示意图如图10-1所示。

从图中可以看出，**应用程序通过Executor框架控制上层的调度**；

而**下层的调度由操作系统内核控制**，下层的调度不受应用程序的控制。

_<font style="color:#0C68CA;">---我们来理解一下，也就是我们在提交任务到线程池的时候，</font>__**<font style="color:#0C68CA;">好像是一个任务提交给一个线程，其实不然</font>**__<font style="color:#0C68CA;">，</font>__<u><font style="color:#0C68CA;">我们提交的任务会由Executor给我们进行划分</font></u>__<font style="color:#0C68CA;">，然后再分配给线程池中的线程，然后线程池中的一个线程对应操作系统中的一个线程，这样就可以实现一个任务的无感执行，开发者只用提交任务即可，细节有Executor框架帮我们实现。---</font>_

### 10.1.2 Executor框架的结构与成员

本文将分两部分来介绍Executor：Executor的结构和Executor框架包含的成员组件。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707989282348-6daecb2e-a0c6-452a-826c-e0be72b58074.png)

#### 1）Executor框架的结构*

Executor框架主要由3大部分组成如下。

+ 任务。包括被执行任务需要实现的接口：Runnable接口或Callable接口。
+ 任务的执行。包括**<font style="color:#DF2A3F;">任务执行机制的核心接口Executor</font>**，以及继承自Executor的ExecutorService接口。Executor框架有两个关键类实现了ExecutorService接口（ThreadPoolExecutor和ScheduledThreadPoolExecutor）。
+ 异步计算的结果。包括接口Future和实现Future接口的FutureTask类。

Executor框架包含的主要的类与接口如图10-2所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707989366840-52ed3041-bc8a-4afe-8d2b-439ff96862a7.png)

下面是这些类和接口的简介。

+ Executor是一个接口，它是Executor框架的基础，**它将任务的提交与任务的执行分离开来。**
+ **<font style="color:#DF2A3F;">ThreadPoolExecutor是线程池的核心实现类</font>**，用来执行被提交的任务。
+ ScheduledThreadPoolExecutor是一个实现类，**可以在给定的延迟后运行命令，或者定期执行命令。**ScheduledThreadPoolExecutor比Timer更灵活，功能更强大。
+ Future接口和实现Future接口的FutureTask类，**代表异步计算的结果**。
+ Runnable接口和Callable接口的实现类，都可以被ThreadPoolExecutor或ScheduledThreadPoolExecutor执行。

Executor框架的使用示意图如图10-3所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707989382027-efecbf80-0d8a-4324-9f6e-0c5c025a65f2.png)

主线程首先要创建实现Runnable或者Callable接口的任务对象。工具类Executors可以把一个Runnable对象封装为一个Callable对象（Executors.callable（Runnable task）或Executors.callable（Runnable task，Object resule））。

然后可以把Runnable对象直接交给ExecutorService执行（ExecutorService.**execute**（Runnable command））；或者也可以把Runnable对象或Callable对象提交给ExecutorService执行（Executor-

Service.**submit**（Runnable task）或ExecutorService.**submit**（Callable<T>task））。

<u>如果执行ExecutorService.</u>**<u>submit</u>**<u>（…），ExecutorService将</u>**<u>返回</u>**<u>一个实现Future接口的对象</u>

（到目前为止的JDK中，返回的是FutureTask对象）。由于FutureTask实现了Runnable，程序员也可

以创建FutureTask，然后直接交给ExecutorService执行。

最后，主线程可以执行FutureTask.get()方法来等待任务执行完成。主线程也可以执行FutureTask.cancel（boolean mayInterruptIfRunning）来取消此任务的执行。

<font style="background-color:#FBDE28;">---好的我们再来总结一下：</font>

+ 首先我们会发现，线程池提交任务一般都是将任务和线程分开的，用的Runnable或者Callable接口，一般是不推荐使用继承Thread类来实现的。
+ 其次我们发现提交后的任务，Runnable是可以由Executors工具类转换成Callable任务的。（Executors.callable（Runnable task）。
+ Runnable对象可以用exectue或者submit方法执行；但是Callable对象只能用submit方法执行；
+ exectue方法是没有返回值的，submit方法是有返回值的，返回一个Future接口的FutureTask对象。

<font style="background-color:#FBDE28;">---总结完毕！</font>

#### 2）Executor框架的成员

本节将介绍Executor框架的主要成员：ThreadPoolExecutor、ScheduledThreadPoolExecutor、

Future接口、Runnable接口、Callable接口和Executors。

##### （1）ThreadPoolExecutor

ThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建3种类型的

ThreadPoolExecutor：SingleThreadExecutor、FixedThreadPool和CachedThreadPool。

下面分别介绍这3种ThreadPoolExecutor。

1）FixedThreadPool。下面是Executors提供的，创建使用固定线程数的FixedThreadPool的API。

```java
public static ExecutorService newFixedThreadPool(int nThreads)
public static ExecutorService newFixedThreadPool(
                                int nThreads, ThreadFactorythreadFactory ThreadFactory)
```

FixedThreadPool适用于为了满足资源管理的需求，而**需要限制当前线程数量的应用场景**，它适用于负载比较重的服务器。

2）SingleThreadExecutor。下面是Executors提供的，创建使用单个线程的SingleThreadExecutor的API。

```java
public static ExecutorService newSingleThreadExecutor()
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory)
```

SingleThreadExecutor适用于**需要保证顺序地执行各个任务**；并且在任意时间点，不会有多个线程是活动的应用场景。

3）CachedThreadPool。下面是Executors提供的，创建一个会根据需要创建新线程的CachedThreadPool的API。

```java
public static ExecutorService newCachedThreadPool()
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory)
```

CachedThreadPool是**大小无界的线程池**，**适用于执行很多的短期异步任务的小程序**，或者是负载较轻的服务器。

##### （2）ScheduledThreadPoolExecutor

ScheduledThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建2种类型的ScheduledThreadPoolExecutor，如下。

+ ScheduledThreadPoolExecutor。**包含若干个线程**的ScheduledThreadPoolExecutor。
+ SingleThreadScheduledExecutor。**只包含一个线程**的ScheduledThreadPoolExecutor。

下面分别介绍这两种ScheduledThreadPoolExecutor。

下面是工厂类Executors提供的，创建固定个数线程的ScheduledThreadPoolExecutor的API。

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)
public static ScheduledExecutorService newScheduledThreadPool(
                                            int corePoolSize,ThreadFactory threadFactory)
```

ScheduledThreadPoolExecutor适用于**需要多个后台线程执行周期任务，同时为了满足资源**

**管理的需求而需要限制后台线程的数量的应用场景。**下面是Executors提供的，创建单个线程的SingleThreadScheduledExecutor的API。

```java
public static ScheduledExecutorService newSingleThreadScheduledExecutor()
public static ScheduledExecutorService newSingleThreadScheduledExecutor(
                                                            ThreadFactory threadFactory)
```

SingleThreadScheduledExecutor**适用于需要单个后台线程执行周期任务**，同时需要保证顺序地执行各个任务的应用场景。

##### （3）Future接口

Future接口和实现Future接口的FutureTask类用来表示异步计算的结果。当我们把Runnable接口或Callable接口的实现类提交（**<font style="color:#DF2A3F;">submit</font>**）给ThreadPoolExecutor或ScheduledThreadPoolExecutor时，ThreadPoolExecutor或ScheduledThreadPoolExecutor会向我们返回一个FutureTask对象。下面是对应的API。

```java
<T> Future<T> submit(Callable<T> task)
<T> Future<T> submit(Runnable task, T result)
Future<> submit(Runnable task)
```

有一点需要读者注意，到目前最新的JDK 8为止，Java通过上述API返回的是一个FutureTask对象。但从API可以看到，Java仅仅保证返回的是一个实现了Future接口的对象。在将来的JDK实现中，返回的可能不一定是FutureTask。

##### （4）Runnable接口和Callable接口

Runnable接口和Callable接口的实现类，都可以被ThreadPoolExecutor或ScheduledThreadPoolExecutor执行。**它们之间的区别是Runnable不会返回结果，而Callable可以返回结果。**

除了可以自己创建实现Callable接口的对象外，还**可以使用工厂类Executors来把一个Runnable包装成一个Callable。**

下面是Executors提供的，把一个Runnable包装成一个Callable的API。

```java
public static Callable<Object> callable(Runnable task) // 假设返回对象Callable1
```

下面是Executors提供的，把一个Runnable和一个待返回的结果包装成一个Callable的API。

```java
public static <T> Callable<T> callable(Runnable task, T result) // 假设返回对象Callable2
```

前面讲过，当我们把一个Callable对象（比如上面的Callable1或Callable2）提交给ThreadPoolExecutor或ScheduledThreadPoolExecutor执行时，submit（…）会向我们返回一个FutureTask对象。我们可以执行FutureTask.get()方法来等待任务执行完成。当任务成功完成后FutureTask.get()将返回该任务的结果。例如，如果提交的是对象Callable1，FutureTask.get()方法将返回null；如果提交的是对象Callable2，FutureTask.get()方法将返回result对象。

## 10.2 ThreadPoolExecutor详解*

Executor框架最核心的类是ThreadPoolExecutor，它是线程池的实现类，主要由下列4个组件构成。

+ corePool：核心线程池的大小。
+ maximumPool：最大线程池的大小。
+ BlockingQueue：用来暂时保存任务的工作队列。
+ RejectedExecutionHandler：当ThreadPoolExecutor已经关闭或ThreadPoolExecutor已经饱和时（达到了最大线程池大小且工作队列已满），execute()方法将要调用的Handler。

通过Executor框架的工具类Executors，可以创建3种类型的ThreadPoolExecutor。

    - FixedThreadPool。
    - SingleThreadExecutor。
    - CachedThreadPool。

下面将分别介绍这3种ThreadPoolExecutor。

### 10.2.1 FixedThreadPool详解*

FixedThreadPool被称为**可重用固定线程数的线程池**。下面是FixedThreadPool的源代码实现。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
}
```

FixedThreadPool的corePoolSize和maximumPoolSize都被设置为创建FixedThreadPool时指定的参数nThreads。

当线程池中的线程数大于corePoolSize时，keepAliveTime为多余的空闲线程等待新任务的最长时间，超过这个时间后多余的线程将被终止。这里把keepAliveTime设置为0L，意味着多余的空闲线程会被立即终止。

FixedThreadPool的execute()方法的运行示意图如图10-4所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707997139605-1565547d-2ec4-43ab-b1b3-987b85f4ca11.png)

对图10-4的说明如下。

1）**如果当前运行的线程数少于corePoolSize，****<font style="color:#DF2A3F;">则创建新线程来执行任务</font>****。--- 这句很重要！！！好好理解！**

2）在线程池完成预热之后（当前运行的线程数等于corePoolSize），将任务加入`**LinkedBlockingQueue**`。

3）线程执行完1中的任务后，会在循环中反复从LinkedBlockingQueue获取任务来执行。

FixedThreadPool使用无界队列LinkedBlockingQueue作为线程池的工作队列（队列的容量为Integer.MAX_VALUE）。使用无界队列作为工作队列会对线程池带来如下影响。

a.当线程池中的线程数达到corePoolSize后，新任务将在无界队列中等待，因此**线程池中的线程数不会超过corePoolSize。**

b.由于a，使用无界队列时maximumPoolSize将是一个无效参数。

c.由于a和b，使用无界队列时keepAliveTime将是一个无效参数。

d.由于使用无界队列，运行中的FixedThreadPool（未执行方法shutdown()或shutdownNow()）不会拒绝任务（不会调用RejectedExecutionHandler.rejectedExecution方法）。

### 10.2.2 SingleThreadExecutor详解*

SingleThreadExecutor是使用单个worker线程的Executor。下面是SingleThreadExecutor的源代码实现。

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

SingleThreadExecutor的**corePoolSize和maximumPoolSize被设置为1**。其他参数与FixedThreadPool相同。SingleThreadExecutor使用无界队列LinkedBlockingQueue作为线程池的工作队列（队列的容量为Integer.MAX_VALUE）。SingleThreadExecutor使用无界队列作为工作队列对线程池带来的影响与FixedThreadPool相同，这里就不赘述了。

SingleThreadExecutor的运行示意图如图10-5所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707997990242-d599a0c7-7cc9-47ff-8ae3-a04d25142230.png)

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707997997479-7e01fefb-fb3d-43d6-9165-cdeac7a29e12.png)

对图10-5的说明如下。

1）如果当前运行的线程数少于corePoolSize（即线程池中无运行的线程），则创建一个新线程来执行任务。

2）在线程池完成预热之后（当前线程池中有一个运行的线程），将任务加入Linked-BlockingQueue。

3）线程执行完1中的任务后，会在一个无限循环中反复从LinkedBlockingQueue获取任务来执行。

### <font style="color:#DF2A3F;">10.2.3 CachedThreadPool详解*</font>

CachedThreadPool是一个会根据需要创建新线程的线程池。下面是创建CachedThread-Pool的源代码。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}
```

CachedThreadPool的**corePoolSize被设置为0**，即corePool为空；

**maximumPoolSize被设置为Integer.MAX_VALUE**，即**maximumPool是无界的**。

这里把keepAliveTime设置为60L，意味着CachedThreadPool中的空闲线程等待新任务的最长时间为60秒，空闲线程超过60秒后将会被终止。

FixedThreadPool和SingleThreadExecutor使用无界队列LinkedBlockingQueue作为线程池的工作队列。**CachedThreadPool使用没有容量的SynchronousQueue作为线程池的工作队列**，但CachedThreadPool的maximumPool是无界的。这意味着，<u>如果主线程提交任务的速度高于maximumPool中线程处理任务的速度时，CachedThreadPool会不断创建新线程。</u>极端情况下，CachedThreadPool会因为创建过多线程而耗尽CPU和内存资源。

CachedThreadPool的execute()方法的执行示意图如图10-6所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707998298169-ec4027e9-8495-4219-a2ab-e68c12dd852c.png)

对图10-6的说明如下。

1）首先执行SynchronousQueue.offer（Runnable task）。如果当前maximumPool中有空闲线程正在执行SynchronousQueue.poll（keepAliveTime，TimeUnit.NANOSECONDS），那么**主线程执行offer操作与空闲线程执行的poll操作****<font style="color:#DF2A3F;">配对成功</font>****，****<font style="color:#4C16B1;">主线程把任务交给空闲线程执行</font>**，execute()方法执行完成；否则执行下面的步骤2）。

2）当初始maximumPool为空，或者maximumPool中当前没有空闲线程时，将没有线程执行SynchronousQueue.poll（keepAliveTime，TimeUnit.NANOSECONDS）。这种情况下，步骤1）将失败。此时CachedThreadPool会创建一个新线程执行任务，execute()方法执行完成。

3）<u>在步骤2）中新创建的线程将任务执行完后，会执行SynchronousQueue.poll（keepAliveTime，</u>TimeUnit.NANOSECONDS）。这个poll操作会让空闲线程最多在SynchronousQueue中等待60秒钟。如果60秒钟内主线程提交了一个新任务（主线程执行步骤1）），那么这个空闲线程将执行主线程提交的新任务；否则，这个空闲线程将终止。<u>由于空闲60秒的空闲线程会被终止，因此长时间保持空闲的CachedThreadPool不会使用任何资源。</u>

前面提到过，**SynchronousQueue是一个没有容量的阻塞队列。每个插入操作必须等待另一个线程的对应移除操作，反之亦然。**CachedThreadPool使用SynchronousQueue，**<font style="color:#4C16B1;">把主线程提交的任务传递给空闲线程执行（</font>**<font style="color:#DF2A3F;">这句话很有深度哈，好好理解！！！</font>**<font style="color:#4C16B1;">）</font>**。CachedThreadPool中任务传递的示意图如图10-7所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707998373755-476a6cc0-6920-4435-9287-dce6409123c8.png)

## 10.3 ScheduledThreadPoolExecutor详解*

ScheduledThreadPoolExecutor继承自ThreadPoolExecutor。它**<font style="color:#DF2A3F;">主要用来在给定的延迟之后运行任务，或者定期执行任务。</font>**ScheduledThreadPoolExecutor的功能与Timer类似，但ScheduledThreadPoolExecutor功能更强大、更灵活。<u>Timer对应的是单个后台线程，而ScheduledThreadPoolExecutor可以在构造函数中指定多个对应的后台线程数。</u>

### 10.3.1 ScheduledThreadPoolExecutor的运行机制

ScheduledThreadPoolExecutor的执行示意图（本文基于JDK 6）如图10-8所示。

**DelayQueue是一个****<font style="color:#DF2A3F;">无界队列</font>**，所以ThreadPoolExecutor的maximumPoolSize在Scheduled-ThreadPoolExecutor中没有什么意义（设置maximumPoolSize的大小没有什么效果）。

ScheduledThreadPoolExecutor的执行主要分为两大部分。

1）当调用ScheduledThreadPoolExecutor的scheduleAtFixedRate()方法或者scheduleWith-FixedDelay()方法时，会向ScheduledThreadPoolExecutor的DelayQueue添加一个实现了RunnableScheduledFuture接口的ScheduledFutureTask。

2）线程池中的线程从DelayQueue中获取ScheduledFutureTask，然后执行任务。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1707999567958-551b4124-c595-4990-a132-0d11236f1ad4.png)

ScheduledThreadPoolExecutor为了实现周期性的执行任务，对ThreadPoolExecutor做了如下

的修改。

+ <font style="color:#DF2A3F;">使用DelayQueue作为任务队列。</font>
+ <font style="color:#DF2A3F;">获取任务的方式不同（后文会说明）。</font>
+ <font style="color:#DF2A3F;">执行周期任务后，增加了额外的处理（后文会说明）。</font>

### 10.3.2 ScheduledThreadPoolExecutor的实现*

前面我们提到过，ScheduledThreadPoolExecutor会把待调度的任务（ScheduledFutureTask）放到一个DelayQueue中。

ScheduledFutureTask主要包含3个成员变量，如下。

+ long型成员变量time，表示这个任务将要被执行的具体时间。
+ long型成员变量sequenceNumber，表示这个任务被添加到ScheduledThreadPoolExecutor中

的序号。

+ long型成员变量period，表示任务执行的间隔周期。

**DelayQueue封装了一个PriorityQueue**，这个PriorityQueue会对队列中的Scheduled-FutureTask进行排序。排序时，time小的排在前面（时间早的任务将被先执行）。如果两个ScheduledFutureTask的time相同，就比较sequenceNumber，sequenceNumber小的排在前面（也就是说，如果两个任务的执行时间相同，那么先提交的任务将被先执行）。

首先，让我们看看ScheduledThreadPoolExecutor中的线程执行周期任务的过程。图10-9是ScheduledThreadPoolExecutor中的线程1执行某个周期任务的4个步骤。

<font style="background-color:#FBDE28;">---这里又可以总结一下了！</font>

+ 我们知道ScheduledThreadPoolExecutor主要是想实现一个延时调度或者周期调度；
+ 其次我们要实现这个延时调度该怎么办？
    - 我们需要三个变量:
        * time ： 表示这个任务要被执行的具体时间；
        * sequenceNumber：被添加到队列中的序号；
        * period：任务执行的时间间隔周期；
    - 这里说明一个问题，我们为什么要这三个变量？time用于标识任务执行的顺序，period是周期调度的间隔。
+ 然后我们怎么对提交到周期线程池的任务进行一个排序？  **----> 这里用到了****<font style="color:#DF2A3F;">优先级队列</font>****！！！**

<font style="background-color:#FBDE28;">--- 总结完毕！</font>

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708000369779-5c722ff9-0a0c-4b1e-a5ab-abf68ee6e678.png)

下面是对这4个步骤的说明。

1）线程1从DelayQueue中获取已到期的ScheduledFutureTask（DelayQueue.take()）。到期任务是指ScheduledFutureTask的time大于等于当前时间。

2）线程1执行这个ScheduledFutureTask。

3）线程1修改ScheduledFutureTask的time变量为下次将要被执行的时间。

4）线程1把这个修改time之后的ScheduledFutureTask放回DelayQueue中（Delay-Queue.add()）。

接下来，让我们看看上面的**<font style="color:#DF2A3F;background-color:#FBDE28;">步骤1</font>**）获取任务的过程。下面是DelayQueue.take()方法的源代码实现。

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();　　　　　　　　　　　　 // 1
    try {
        for (;;) {
            E first = q.peek();
            if (first == null) {
                available.await();　　　　　　　　　　 // 2.1
            } else {
                long delay = first.getDelay(TimeUnit.NANOSECONDS);
                if (delay > 0) {
                    long tl = available.awaitNanos(delay);　　 // 2.2
                } else {
                    E x = q.poll();　　　　　　　　　　 // 2.3.1
                    assert x != null;
                if (q.size() != 0)
                    available.signalAll();　　　　　　　　 // 2.3.2
                return x;
                }
            }
        }
    } finally {
    lock.unlock();　　　　　　　　　　　　　　 // 3
    }
}
```

图10-10是DelayQueue.take()的执行示意图。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708000803928-98e15d59-e4fd-4b22-9817-62af4811da77.png)

如图所示，获取任务分为3大步骤。

**1）获取Lock。 ****<font style="color:#DF2A3F;"> --- 第一步首先要获取锁！  ---> 为什么要加锁？ 线程同步的问题！</font>**<font style="color:rgb(13, 13, 13);">加锁的原因主要是因为多线程的并发操作，特别是在获取优先级队列的队头元素时可能存在竞争问题。</font>

2）获取周期任务。

+ 如果PriorityQueue为空，当前线程到Condition中等待；否则执行下面的2.2。
+ 如果PriorityQueue的头元素的time时间比当前时间大，到Condition中等待到time时间；否则执行下面的2.3。
+ 获取PriorityQueue的头元素（2.3.1）；如果PriorityQueue不为空，则唤醒在Condition中等待的所有线程（2.3.2）。

3）释放Lock。

ScheduledThreadPoolExecutor在一个循环中执行步骤2，直到线程从PriorityQueue获取到一个元素之后（执行2.3.1之后），才会退出无限循环（结束步骤2）。

最后，让我们看看ScheduledThreadPoolExecutor中的线程执行任务的步骤4，把ScheduledFutureTask放入DelayQueue中的过程。下面是DelayQueue.add()的源代码实现。

```java
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();　　　　　　　　　　 // 1
    try {
        E first = q.peek();
        q.offer(e);　　　　　　　　 // 2.1
        if (first == null || e.compareTo(first) < 0)
        available.signalAll();　　　 // 2.2
        return true;
    } finally {
        lock.unlock();　　　　　　　　 // 3
    }
}
```

图10-11是DelayQueue.add()的执行示意图。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708000897896-fabea049-9a83-4e09-85e7-3661e349a99f.png)

如图所示，添加任务分为3大步骤。

1）获取Lock。

2）添加任务。

+ 向PriorityQueue添加任务。
+ 如果在上面2.1中添加的任务是PriorityQueue的头元素，唤醒在Condition中等待的所有线程。

3）释放Lock。

## 10.4 FutureTask详解

Future接口和实现Future接口的FutureTask类，代表异步计算的结果。

---  FutureTask其实可以理解为一个可以被线程池执行的任务，然后执行之后我们可以调用他的get方法获取执行结果。---

### 10.4.0 FutureTask的使用场景

`FutureTask` 是 Java 提供的一个实现了 `RunnableFuture` 接口的类，它可用于异步获取计算结果。`**FutureTask**`** 可以用于包装一个 **`**Callable**`** 或者 **`**Runnable**`** 对象，作为一个可取消的异步计算任务。**

主要的使用场景包括：

1.  **异步任务的执行：** 当你有一些计算密集型或者耗时较长的任务时，你可以将这些任务包装在 `FutureTask` 中，然后提交给一个线程池去执行。这样可以避免阻塞主线程，充分利用多线程来提高程序的性能。

```java
Callable<Integer> myTask = new MyTask();
FutureTask<Integer> futureTask = new FutureTask<>(myTask);

ExecutorService executor = Executors.newFixedThreadPool(1);
executor.submit(futureTask);

// 在需要结果的地方，通过 futureTask.get() 获取异步任务的执行结果
Integer result = futureTask.get();
```

2.  **异步加载资源：** 在某些场景下，你可能需要异步加载一些资源，例如图片、文件等。`FutureTask` 可以用于在后台加载资源，然后在需要使用这些资源的地方获取加载完成的结果。

```java
Callable<Image> loadImageTask = new LoadImageTask();
FutureTask<Image> loadImageFutureTask = new FutureTask<>(loadImageTask);

ExecutorService executor = Executors.newFixedThreadPool(1);
executor.submit(loadImageFutureTask);

// 在需要使用图片的地方，通过 loadImageFutureTask.get() 获取异步加载的图片
Image image = loadImageFutureTask.get();
```

3.  **并行计算：** 当你有一些可以并行计算的子任务时，你可以使用 `FutureTask` 将这些子任务提交给不同的线程执行，然后在主线程等待所有子任务的完成，并汇总结果。

```java
Callable<Integer> task1 = new MyTask();
Callable<Integer> task2 = new MyTask();

FutureTask<Integer> futureTask1 = new FutureTask<>(task1);
FutureTask<Integer> futureTask2 = new FutureTask<>(task2);

ExecutorService executor = Executors.newFixedThreadPool(2);
executor.submit(futureTask1);
executor.submit(futureTask2);

// 在需要获取计算结果的地方，通过 futureTask1.get() 和 futureTask2.get() 获取子任务的执行结果
Integer result1 = futureTask1.get();
Integer result2 = futureTask2.get();
```

总的来说，`FutureTask` 可以用于**<font style="background-color:#FBDE28;">处理异步任务的执行</font>**、**<font style="background-color:#FBDE28;">资源的加载</font>**、**<font style="background-color:#FBDE28;">并行计算</font>**等场景，提供了一种方便的方式来处理多线程下的异步操作。

### 10.4.1 FutureTask简介

**FutureTask除了实现Future接口外，还实现了Runnable接口。因此，FutureTask可以交给Executor执行，也可以由调用线程直接执行（FutureTask.run()）。**根据FutureTask.run()方法被执行的时机，FutureTask可以处于下面3种状态。

1）未启动。FutureTask.run()方法还没有被执行之前，FutureTask处于未启动状态。当创建一个FutureTask，且没有执行FutureTask.run()方法之前，这个FutureTask处于未启动状态。

2）已启动。FutureTask.run()方法被执行的过程中，FutureTask处于已启动状态。

3）已完成。FutureTask.run()方法执行完后正常结束，或被取消（FutureTask.cancel（…）），或执行FutureTask.run()方法时抛出异常而异常结束，FutureTask处于已完成状态。

图10-12是FutureTask的状态迁移的示意图。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708001851367-249e93e7-ede2-4b44-9572-756c60b626aa.png)

当FutureTask处于未启动或已启动状态时，执行FutureTask.get()方法将导致调用线程阻塞；

当FutureTask处于已完成状态时，执行FutureTask.get()方法将导致调用线程立即返回结果或抛出异常。

当FutureTask处于未启动状态时，执行FutureTask.cancel()方法将导致此任务永远不会被执行；

当FutureTask处于已启动状态时，执行FutureTask.cancel（true）方法将以中断执行此任务线程的方式来试图停止任务；

当FutureTask处于已启动状态时，执行FutureTask.cancel（false）方法将不会对正在执行此任务的线程产生影响（让正在执行的任务运行完成）；

当FutureTask处于已完成状态时，执行FutureTask.cancel（…）方法将返回false。

图10-13是get方法和cancel方法的执行示意图。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708001893168-8f1f64f7-6435-413e-83c5-9077f5ea4783.png)

### 10.4.2 FutureTask的使用

可以把FutureTask交给Executor执行；也可以通过ExecutorService.submit（…）方法返回一个FutureTask，然后执行FutureTask.get()方法或FutureTask.cancel（…）方法。除此以外，还可以单独使用FutureTask。

**当一个线程需要等待另一个线程把某个任务执行完后它才能继续执行，此时可以使用FutureTask。**假设有多个线程执行若干任务，每个任务最多只能被执行一次。当多个线程试图同时执行同一个任务时，只允许一个线程执行任务，其他线程需要等待这个任务执行完后才能继续执行。下面是对应的示例代码。

```java
private final ConcurrentMap<Object, Future<String>> taskCache =
    new ConcurrentHashMap<Object, Future<String>>();
private String executionTask(final String taskName)
    throws ExecutionException, InterruptedException {
    while (true) {
        Future<String> future = taskCache.get(taskName);　　 // 1.1,2.1
        if (future == null) {
            Callable<String> task = new Callable<String>() {
                public String call() throws InterruptedException {
                    return taskName;
                }
            }; 
            FutureTask<String> futureTask = new FutureTask<String>(task);
            future = taskCache.putIfAbsent(taskName, futureTask);　 // 1.3
            if (future == null) {
                future = futureTask;
                futureTask.run();　　　　　　　　 // 1.4执行任务
            }
        }
        try {
            return future.get();　　　　　　 // 1.5,2.2} catch (CancellationException e) {
            taskCache.remove(taskName, future);
        }
    }
}
```

上述代码的执行示意图如图10-14所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708002006930-1c7f377d-d4fa-41fd-9d8e-e6d0cf78a92f.png)

当两个线程试图同时执行同一个任务时，如果Thread 1执行1.3后Thread 2执行2.1，那么接下来Thread 2将在2.2等待，直到Thread 1执行完1.4后Thread 2才能从2.2（FutureTask.get()）返回。

### 10.4.3 FutureTask的实现*

FutureTask的实现基于AbstractQueuedSynchronizer（以下简称为AQS）。java.util.concurrent中的很多可阻塞类（比如ReentrantLock）都是基于AQS来实现的。AQS是一个同步框架，它提供通用机制来原子性管理同步状态、阻塞和唤醒线程，以及维护被阻塞线程的队列。JDK 6中AQS被广泛使用，基于AQS实现的同步器包括：ReentrantLock、Semaphore、ReentrantReadWriteLock、CountDownLatch和FutureTask。

**每一个基于AQS实现的同步器都会包含两种类型的操作**，如下。

+ **至少一个acquire操作。**这个操作阻塞调用线程，除非/直到AQS的状态允许这个线程继续执行。FutureTask的acquire操作为<font style="background-color:#FBDE28;">get()</font>/get（long timeout，TimeUnit unit）方法调用。
+ **至少一个release操作。**这个操作改变AQS的状态，改变后的状态可允许一个或多个阻塞线程被解除阻塞。FutureTask的release操作包括<font style="background-color:#FBDE28;">run()</font>方法和cancel（…）方法。

基于“复合优先于继承”的原则，**<font style="color:#DF2A3F;">FutureTask声明了一个内部私有的继承于AQS的子类Sync</font>**，**对FutureTask所有公有方法的调用都会委托给这个内部子类。**

AQS被作为“模板方法模式”的基础类提供给FutureTask的内部子类Sync，这个内部子类只需要实现状态检查和状态更新的方法即可，这些方法将控制FutureTask的获取和释放操作。具体来说，Sync实现了AQS的tryAcquireShared（int）方法和tryReleaseShared（int）方法，Sync通过这两个方法来检查和更新同步状态。

FutureTask的设计示意图如图10-15所示。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708002213368-48130f15-69e0-46a3-a37b-48561e39fdd4.png)

如图所示，**Sync是FutureTask的内部私有类，它继承自AQS。**创建FutureTask时会创建内部私有的成员对象Sync，**FutureTask所有的的公有方法都直接委托给了内部私有的Sync。**

<font style="background-color:#FBDE28;">FutureTask.get()</font>方法会调用AQS.acquireSharedInterruptibly（int arg）方法，这个方法的执行过程如下。

1）调用AQS.acquireSharedInterruptibly（int arg）方法，这个方法首先会回调在子类Sync中实现的tryAcquireShared()方法来判断acquire操作是否可以成功。acquire操作可以成功的条件为：state为执行完成状态RAN或已取消状态CANCELLED，且runner不为null。

2）如果成功则get()方法立即返回。如果失败则到<u>线程等待队列</u>中去等待其他线程执行release操作。

3）当其他线程执行release操作（比如FutureTask.run()或FutureTask.cancel（…））唤醒当前线程后，当前线程再次执行tryAcquireShared()将返回正值1，当前线程将离开线程等待队列并唤醒它的后继线程（这里会产生级联唤醒的效果，后面会介绍）。

4）最后返回计算的结果或抛出异常。

<font style="background-color:#FBDE28;">FutureTask.run()</font>的执行过程如下。

1）执行在构造函数中指定的任务（Callable.call()）。

2）以原子方式来更新同步状态（调用AQS.compareAndSetState（int expect，int update），设置state为执行完成状态RAN）。如果这个原子操作成功，就设置代表计算结果的变量result的值为Callable.call()的返回值，然后调用AQS.releaseShared（int arg）。

3）AQS.releaseShared（int arg）首先会回调在子类Sync中实现的tryReleaseShared（arg）来执行release操作（设置运行任务的线程runner为null，然会返回true）；AQS.releaseShared（int arg），然后唤醒线程等待队列中的第一个线程。

4）调用FutureTask.done()。

当执行FutureTask.get()方法时，如果FutureTask不是处于执行完成状态RAN或已取消状态CANCELLED，当前执行线程将到AQS的线程等待队列中等待（见下图的线程A、B、C和D）。当某个线程执行FutureTask.run()方法或FutureTask.cancel（...）方法时，会唤醒线程等待队列的第一个线程（见图10-16所示的线程E唤醒线程A）。

![](https://cdn.nlark.com/yuque/0/2024/png/38995097/1708002318361-408113ca-b790-4add-b4ae-4383c3bf2f5c.png)

假设开始时FutureTask处于未启动状态或已启动状态，等待队列中已经有3个线程（A、B和C）在等待。此时，线程D执行get()方法将导致线程D也到等待队列中去等待。

当线程E执行run()方法时，会唤醒队列中的第一个线程A。线程A被唤醒后，首先把自己从队列中删除，然后唤醒它的后继线程B，最后线程A从get()方法返回。线程B、C和D重复A线程的处理流程。最终，在队列中等待的所有线程都被级联唤醒并从get()方法返回。

# 第十一章 Java并发编程实战

略。

# 第十二章 分布式编程基础

## 12.1 分布式CAP原则

## 12.2 分布式事务：两阶段提交

## 12.3 分布式事务：TCC

## 12.4 分布式协议：RAFT

## 12.5 分布式协议：Paxos

# 第十三章 分布式锁

## 13.1 什么是分布式锁

## 13.2 实现分布式锁会遇到的问题

## 12.3 分布式锁框架

## 13.4 拉模式的分布式锁

## 13.5 推模式的分布式锁

## 13.6 再看分布式锁

# 第十四章 分布式架构系统