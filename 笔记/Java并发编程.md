# 一、Java并发机制的底层实现原理

## 1、volatile的应用

volatile是轻量级的synchronized，它在多处理器开发中保证了共享变量的**可见性**，可见性的意思是当一个线程修改了一个共享变量时，另外一个线程能够读到这个修改的值。在写volatile变量时，会将变量内容写入到主内存中，而不是CPU缓存中中；读volatile变量时会直接从主内存中获取。如果volatile修饰符使用恰当的话，它比synchronized的使用和执行成本更低，因为它不会引起线程上下文的切换和调度。

### 1.1 volatile的定义与实现原理

volatile的定义如下：Java编程语言支持多线程访问共享变量，为了确保共享变量能够被准确和一致地更新，线程应该确保通过排它锁来单独的获得这个变量。Java语言提供了volatile，在某些情况下比排它锁要更加方便。如果一个字段被声明为volatile，Java线程内存模型确保所有线程看到这个变量的值是一致的。

有volatile修饰的变量在执行写操作的时候，会多出现一个**Lock前缀指令**，这个指令在多核处理器下会引发两件事情：

1. 将当前处理器缓存的内容写回到**系统内存**中
2. 这个写回内存的操作会使在其他处理器里缓存了该内存地址的数据无效

为了提高处理速度，处理器不直接和内存进行通信，而是先将系统内存的数据读到内部缓存（L1，L2或其他）后再进行操作，但操作完不知道何时会写到内存。如果对声明了volatile的变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是，就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题。所以，在多处理器下，为了保证各个处理器的缓存是一致的，就会实现**缓存一致性协议**，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，**当处理器发现自己缓存行对应的内存地址被修改**，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。



## ==2、synchronized的实现原理和应用==

利用synchronized实现同步的基础：Java中的每一个对象都可以作为锁：

* 对于普通同步方法，锁对象是当前实例对象
* 对于静态同步方法，锁对象是当前类的Class对象
* 对于同步方法块，锁是synchronized括号里配置的对象

JVM基于**进入和退出Monitor**对象来实现方法同步和代码块同步，但两者的实现细节不一样。代码块同步是使用**monitorenter和monitorexit**指令实现的，而方法同步是使用另外一种方式实现的，细节在JVM规范里并没有详细说明。但是，方法的同步同样可以使用这两个指令来实现。

**monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结束处和异常处，JVM要保证每个monitorenter必须有对应的monitorexit与之配对**。**任何对象都有一个monitor与之关联**，当且一个monitor被持有后，它将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor的所有权，即尝试获得对象的锁。

### 2.1锁的升级与对比

Java SE 1.6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”（自旋锁），在Java SE 1.6中，锁一共有4种状态，**级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态**，这几个状态会随着竞争情况逐渐升级。**锁可以升级但不能降级**，意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率

#### 2.1.1 偏向锁

**在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。**当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时**不需要进行CAS操作来加锁和解锁**，只需简单地测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。如果测试失败，则需要再测试一下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁）：如果没有设置，则使用CAS竞争锁；如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

1. 偏向锁的撤销

   偏向锁使用了一种等到竞争出现才撤销锁的机制，所以**当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁**。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有正在执行的字节码）。它会首先**暂停拥有偏向锁的线程**，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置成无锁状态；如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。

<img src="F:\Java书和视频\笔记\images\并发编程\1.png" alt="image-20200417104719362" style="zoom:200%;" />

2. 关闭偏向锁

   偏向锁在Java 6和Java 7里是默认启用的，但是它在应用程序启动几秒钟之后才激活，如有必要可以使用JVM参数来关闭延迟：`-XX:BiasedLockingStartupDelay=0`。如果你确定应用程序里所有的锁通常情况下处于竞争状态，可以通过JVM参数关闭偏向锁：`-XX:-UseBiasedLocking=false`，那么程序默认会进入轻量级锁状态。

#### 2.1.2 轻量级锁

（1）轻量级锁加锁

线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。

（2）轻量级锁解锁

轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。

#### 2.1.3 锁的优缺点对比

|    锁    |                             优点                             |                      缺点                      |              适用场景              |
| :------: | :----------------------------------------------------------: | :--------------------------------------------: | :--------------------------------: |
|  偏向锁  | 加锁和解锁不需要额外的消耗，和执行非同步方法相比仅存在纳秒级的差距 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗 |  适用于只有一个线程访问同步块场景  |
| 轻量级锁 |           竞争的线程不会阻塞，提高了程序的响应速度           | 如果始终得不到锁竞争的线程，使用自旋 会消耗CPU | 追求响应时间，同步块执行速度非常快 |
| 重量级锁 |               线程竞争不使用自旋，不会消耗CPU                |             线程阻塞，响应时间缓慢             |   追求吞吐量，同步块执行速度较长   |

## 3、Java如何实现原子操作

在Java中**通过锁和循环CAS的方式来实现原子操作**

1. 使用循环CAS的方式来实现原子操作

   JVM中的CAS操作正是利用了处理器提供的CMPXCHG指令实现的。自旋CAS实现的基本思路就是循环进行CAS操作直到成功为止，以下代码实现了一个基于CAS线程安全的计数器方法safeCount和一个非线程安全的计数器count。

   ```java
   private AtomicInteger atomicI = new AtomicInteger(0);
   private int i = 0;
   public static void main(String[] args) {
       final Counter cas = new Counter();
       List<Thread> ts = new ArrayList<Thread>(600);
       long start = System.currentTimeMillis();
       for (int j = 0; j < 100; j++) {
           Thread t = new Thread(new Runnable() {
               @Override
               public void run() {
                   for (int i = 0; i < 10000; i++) {
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
       // 等待所有线程执行完成
       for (Thread t : ts) {
           try {
               t.join();
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
       System.out.println(cas.i);
       System.out.println(cas.atomicI.get());
       System.out.println(System.currentTimeMillis() - start);
   }
   /** * 使用CAS实现线程安全计数器 */
   private void safeCount() {
       for (;;) {
           int i = atomicI.get();
           boolean suc = atomicI.compareAndSet(i, ++i);
           if (suc) {
               break;
           }
       }
   }
   /**
   * 非线程安全计数器
   */
   private void count() {
       i++;
   }
   }
   ```

2. ==CAS实现原子操作的三大问题==

   * **ABA问题**

     因为CAS需要在操作值的时候，检查值是否发生了变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。**ABA问题的解决思路是使用版本号。**在变量前面追加上版本号，每次变量更新的时候版本号加一，那么A→B→A就会变成1A→2B→3A。从Java 1.5开始，JDK的Atomic包里提供了一个类AtomicStampedReference来解决ABA问题。这个类的compareAndSet方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

   * **循环时间长开销大**

     自旋CAS如果长时间不成功，会给CPU带来非常大的开销。如果JVM能支持处理器提供的pause指令，那么效率会有一定的提升。pause指令有两个作用：第
     一，它可以延迟流水线执行指令（de-pipeline），使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零；第二，它可以避免在退出循环的时候因内存顺序冲突（Memory Order Violation）而引起CPU流水线被清空（CPU Pipeline Flush），从而提高CPU的执行效率。

   * **只能保证一个共享变量的原子操作**

     当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁。还有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如，有两个共享变量i＝2，j=a，合并一下ij=2a，然后用CAS来操作ij。从Java 1.5开始，JDK提供了AtomicReference类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行CAS操作。

# 二、Java内存模型

## 1、Java内存模型的基础

### 1.1 Java内存模型的抽象结构

线程之间通信有两种机制：共享内存和消息传递

共享内存的线程是隐式进行通信的，而对于同步操作需要显示要求。而消息传递是显示通信的，对于同步操作是隐式要求的，因为消息发送必须在消息接收之前。

Java的并发采用共享内存模型。在Java中，所有实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享。

![image-20200417142756034](F:\Java书和视频\笔记\images\并发编程\2.png)

本地内存是JMM（Java Memory Model）的一个抽象概念，并不真实存在，它涵盖了**缓存、写缓冲区、寄存器**以及其他的硬件和编译器优化。

如果线程A和线程B要进行通信，必须经过下面两个步骤：

1. 线程A把本地内存中更新过的共享变量刷新到主内存中去
2. 线程B到主内存中去读取线程A之前已经更新过的共享变量

JMM通过控制**主内存与每个线程的本地内存之间的交互**，来为Java程序员提供内存可见性保证

### 1.2 指令序列的重排序

在执行程序时，为了提高性能，编译器和处理请会对指令做重排序，重排序分三种类型：

1. 编译器优化的重排序：编译器在不改变单线程程序语义的情况下，可以重新安排语句执行顺序
2. 指令级并行的重排序：现代处理器采用了**指令级并行技术**来将度哦条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序
3. 内存系统的重排序：由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

第一种是编译器重排序，第二三种是处理器重排序，这三种重排序都会对多线程造成内存可见性问题。对于编译器重排序，**JMM禁止特定类型的编译器重排序**。对于处理器重排序，**JMM要求编译器在生成指令序列时，插入特定类型的内存屏障指令来禁止特定类型的处理器重排序**

![image-20200417145756805](F:\Java书和视频\笔记\images\并发编程\3.png)

StoreLoad Barriers是一个“全能型”的屏障，它同时具有其他3个屏障的效果。现代的多处理器大多支持该屏障（其他类型的屏障不一定被所有处理器支持）。执行该屏障开销会很昂贵，因为当前处理器通常要把写缓冲区中的数据全部刷新到内存中（Buffer Fully Flush）。

对有数据依赖关系的指令不能进行重排序

### 1.3 happens-before

从JDK5 开始，Java使用新的JSR-133内存模型，该内存模式使用happens-before的概念来阐述操作之间的内存可见性。

有以下happens-before规则：

* 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作
* 监视器锁规则：对一个锁的**解锁**，happens-before于随后对这个锁的**加锁**
* volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读
* 传递性：如果A happens-before B，B happens-before C，那么A happens-before C

happens-before与JMM的关系：

![image-20200417150537874](F:\Java书和视频\笔记\images\并发编程\4.png)

==理想模型==

==顺序一致性模型：一个线程的所有操作必须按照程序的顺序来执行；不管程序是否同步，所有线程都只能看到一个单一的操作执行顺序，在顺序一致性内存模型中，每个操作都必须原子执行并且立即对所有线程可见==

衡量是否对多线程进行了正确同步，就是看执行结果和顺序一致性结果是否相同

## ==2、volatile的内存语义==

### 2.1 volatile的特性

即使是64位的long型和double型变量，只要它是volatile变量，对该变量的读/写操作就具有原子性。**如果是多个volatile操作或类似于volatile++这种复合操作，这些操作整体上不具有原子性**

volatile变量自身具有下列特性：

* 可见性：对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。基于JMM，被读取volatile变量，会去主内存中读取，而不是从工作内存中读取；写volatile变量会直接写入主内存。
* 禁止进行指令重排列

### 2.2 volatile写/读建立的happens-before关系

volatile的写-读操作与锁的释放-获取有相同的内存效果：volatile写和锁的释放有相同的内存语义；volatile读与锁的获取有相同的内存语义。

### 2.3 volatile写-读的内存语义

* volatile写的内存语义

  当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存中

* volatile读的内存语义：

  当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

### 2.4 volatile内存语义的实现

为了实现volatile内存语义，JMM会分别**限制编译器重排序和处理器重排序**。volatile重排序规则如下：

![image-20200417155254450](F:\Java书和视频\笔记\images\并发编程\5.png)

* **当第二个操作是volatile写时，不管第一个操作是什么，都不能重排序**。这个规则确保volatile写之前的操作不会被编译器重排序到volatile写之后。
* **当第一个操作是volatile读时，不管第二个操作是什么，都不能重排序**。这个规则确保volatile读之后的操作不会被编译器重排序到volatile读之前。
* 当第一个操作是volatile写，第二个操作是volatile读时，不能重排序。

**为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序==（增加一个lock前缀指令）==**：

* 在每个volatile写操作的前面插入一个StoreStore屏障。
* 在每个volatile写操作的后面插入一个StoreLoad屏障。
* 在每个volatile读操作的后面插入一个LoadLoad屏障。
* 在每个volatile读操作的后面插入一个LoadStore屏障。

**内存屏障提供三个功能:**

1. 它确保指令重排序时不会把屏障后面的操作放到屏障前面，屏障前面的操作放到屏障后面；即在执行内存屏障这条语句时，前面的操作已经全部完成了
2. 它会强制将对缓存的修改操作立即写入主内存
3. 如果是写操作，它会导致其他CPU中对应的缓存行无效

下面是保守策略下，volatile写插入内存屏障后生成的指令序列示意图

![image-20200417155721907](F:\Java书和视频\笔记\images\并发编程\6.png)

下面是在保守策略下，volatile读插入内存屏障后生成的指令序列示意图：

![image-20200417155856049](F:\Java书和视频\笔记\images\并发编程\7.png)

编译器会根据具体情况省略不必要的屏障

例子:

```java
//x、y为非volatile变量
//flag为volatile变量
 
x = 2;        //语句1
y = 0;        //语句2
flag = true;  //语句3
x = 4;         //语句4
y = -1;       //语句5
```

由于flag是volatile变量，那么在进行指令重排序时，3不会放到1，2前面，页不会放到4，5后面。但是1，2之间和4，5之间的顺序没有保证。

### ==2.5 使用Volatile关键字的典型应用-单例模式双重检查==

```java
public class Singleton {
    
    private static volatile Singleton instance = null;
    
    private Singleton(){}
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

**分析**

- 为什么要双重检查

    - 如果去掉第一次检查

        ```java
        public class Singleton {
            
            private static volatile Singleton instance = null;
            
            private Singleton(){}
            
            public static Singleton getInstance() {
                synchronized (Singleton.class) {
                	if (instance == null)
                		instance = new Singleton();
                }
                return instance;
            }
        }
        ```

        每个线程进入`getInstance()`方法都会先获取锁，这样的效率很低。因为即使对象已经存在，也会获取锁再去判断。

    - 如果去掉第二次检查

        ```java
        public class Singleton {
            
            private static volatile Singleton instance = null;
            
            private Singleton(){}
            
            public static Singleton getInstance() {
                if (instance == null) {
                    synchronized (Singleton.class) {
                    	instance = new Singleton();
                    }
                }
                return instance;
            }
        }
        ```

        假设有A、B两个线程，当线程A刚执行完`if(instance == null)`语句时间片用完，那么A会被挂起，B线程开始执行。那在B的时间片中执行完成了`getInstance()`方法，创建了一个Singleton实例。此时如果A继续执行，A已经判断过了instance是否等于null，那么它会直接执行后面的语句，又创建一个新的Singleton实例，那么这就违反了单例的设计原则。

- 不加volatile的后果

    ```java
    public class Singleton {
        
        private static Singleton instance = null;
        
        private Singleton(){}
        
        public static Singleton getInstance() {
            if (instance == null) {
                synchronized (Singleton.class) {//1
                    if (instance == null)//2
                        instance = new Singleton();//3
                }
            }
            return instance;
        }
    }
    ```

    分析下面这种场景：

    1. 线程A进入`getInstance()`方法，由于instance为null，所以进入到synchronized块中，执行到3处时可能会进行指令重排序。因为`new Singleton()`语句并不是一个原子操作，它先通过方法区找到Singleton类的全类名，然后看是否已经加载这个类，如果没有加载这个类会先进行加载，这个时候只是进行了类的初始化操作，并创建了class对象，但是没有进行实例化操作；然后在堆中开辟一个内存块，为该内存赋初值；然后填写对象头；最后执行构造方法填充数据。**如果在刚开辟一个内存后就返回内存的引用，此时instance指向的是一个不完整的Singleton对象，还并没有实例化。此时如果时间片用完，线程B执行发现instance不为null，直接就返回了这个不完整的对象，那么就会造成错误。**

## 3、锁的内存语义

### 3.1 锁的释放-获取的内存语义

当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中。当线程获取锁时，JMM会把该线程对应的本地内存置为无效，从而使得被监视器保护的临界区代码必须从主内存中获取共享变量。

锁的释放与获取和线程间通信的关系：

* 线程A释放一个锁，实质上是线程A向接下来将要获取这个锁的某个线程发出了（线程A对共享变量所做修改）的消息
* 线程B获取一个锁，实质上是线程B接收了之前某个线程发出的（在释放锁之前对共享变量所做修改的）消息
* 线程A释放锁，随后线程B获取这个锁，这个过程实质上是线程A通过主内存向 线程B发送消息

### 3.2 锁内存语义的实现

如下使用ReentrantLock的代码：

```java
class ReentrantLockExample {
    int a = 0;
    ReentrantLock lock = new ReentrantLock();
    public void writer() {
        lock.lock();　　　　 // 获取锁
        try {
            a++;
        } f　　inally {
            lock.unlock();　　// 释放锁
        }
    }
    public void reader () {
        lock.lock();　　　　 // 获取锁
        try {
            int i = a;
            ……
        } f　　inally {
            lock.unlock();　 // 释放锁
        }
    }
}
```

`ReentrantLock`的实现依赖于`Java`同步器框架`AbstractQueuedSynchronizer`（AQS）。AQS使用一个整型的**volatile变量（命名为state）**来维护同步状态，这个volatile变量是ReentrantLock内存语义实现的关键。

![image-20200417162517939](F:\Java书和视频\笔记\images\并发编程\8.png)

ReentrantLock分为公平锁和非公平锁。

**公平锁**

使用公平锁时，加锁方法lock()调用轨迹如下：

1. ReentrantLock:lock()
2. FairSync:lock()
3. AbstractQueuedSynchronizer:acquire(int arg)
4. ReentrantLock:tryAcquire(int acquires)

在第四步真正开始加锁，下面是该方法的源代码：

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();　　　　// 获取锁的开始，首先读volatile变量state
    if (c == 0) {
        if (isFirst(current) &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)　　
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

在使用公平锁时，解锁方法unlock()调用轨迹如下：

1. ReentrantLock:unlock()
2. AbstractQueuedSynchronizer:release(int arg)
3. Sync:tryRelease(int releases)

在第三步真正开始释放锁，下面是该方法的源代码：

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
    setState(c);　　　　　// 释放锁的最后，写volatile变量state
    return free;
}
```

**非公平锁**

使用非公平锁时，加锁方法lock()调用轨迹如下：

1. ReentrantLock:lock()
2. NonfairSync:lock()
3. AbstractQueuedSynchronizer:compareAndSetState(int expect,int update)

在第三步真正开始加锁，下面是该方法的源代码：

```java
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

该方法以原子操作的方式更新state变量：如果当前状态值等于预期值，泽一原子方式将同步状态设置为给定的更新值。此操作具有volatile读和写的内存语义。

对公平锁和非公平锁的内存语义的总结：

* 公平锁和非公平锁在释放时，最后都要写一个volatile变量state
* 公平锁获取时，首先回去读volatile变量
* 非公平锁获取时，首先会用CAS更新volatile变量，这个操作具有volatile读和volatile写到的内存语义

这就可以看出，**锁释放-获取的内存语义的实现至少有下面两种方式：**

1. 利用volatile变量的读-写所具有的内存语义
2. 利用CAS附带的volatile读和写的内存语义

### 3.3 concurrent包的实现

由于Java的CAS同时具有volatile读和写的内存语义，因此Java线程之间的通信现在有了下面4中方法：

1. A线程写volatile变量，随后B线程读这个volatile变量
2. A线程写volatile变量，随后B线程用CAS更新这个volatile变量
3. A线程用CAS更新一个volatile变量，随后B线程用CAS更新这个volatile变量
4. A线程用CAS更新一个volatile变量，随后B线程读这个volatile变量

concurrent包的一个通用化实现模式：

​	**首先，声明共享变量为volatile**

​	**然后，使用CAS的原子条件更新来实现线程之间的同步**

​	同时，配合以volatile的读/写和CAS所具有的volatile读和写内存语义来实现线程之间的通信

![image-20200417165400119](F:\Java书和视频\笔记\images\并发编程\9.png)

AQS，非阻塞数据结构和原子变量类（java.util.concurrent.atomic包中的类），这些concurrent包中的基础类都是使用这种模式来实现的，而concurrent包中的高层类又是依赖于这些基础类来实现的。

## 4、final域的内存语义

### 4.1 final域的重排序规则

对于final域，编译器和处理器要遵守两个重排序规则

1. 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序
2. 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序

#### 4.1.1 写final域的重排序规则

写final域的重排序规则禁止把final域的写重排序到构造函数之外。这个规则的实现包含下面2个方面。

1. JMM禁止编译器把final域的写重排序到构造函数之外。
2. 编译器会在final域的写之后，构造函数return之前，插入一个StoreStore屏障。这个屏障
   禁止处理器把final域的写重排序到构造函数之外。

写final域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保障

#### 4.1.2 读final域的重排序规则

读final域的重排序规则是，在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这两个操作。编译器会在读final域操作的前面插入一个LoadLoad屏障。

读final域的重排序规则可以确保：在读一个对象的final域之前，一定会先读包含这个final域的对象的引用

#### 4.1.3 final域的安全性保证

通过为final域增加写和读重排序规则，可以保证：只要对象是正确构造的（被构造对象的引用在构造函数中没有“逸出”），那么不需要使用同步（指lock和volatile的使用）就可以保证任意线程都能看到这个final域在构造函数中被初始化后的值。

## 5、happens-before

happens-before是JMM最核心的概念。对于重排序，可以分为下面两类：

1. 会改变程序结果的重排序
2. 不会改变程序执行结果的重排序

对于会改变程序执行结果的重排序，JMM要求编译器和处理器必须禁止这种重排序

对于不会改变程序执行结果的重排序，JMM对编译器和处理器不做任何要求（允许这种重排序）

![image-20200417173215628](F:\Java书和视频\笔记\images\并发编程\10.png)

只要不改变程序的执行结果，编译器和处理器怎么优化都行。

### 5.1 happens-before的定义

1. 如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前
2. 两个操作之间存在happens-before关系，并不意味着Java平台的具体实现必须要按照happens-before关系指定的顺序来执行。如果重排序之后的执行结果，与按happens-before关系来执行的结果一致，那么这种重排序并不非法（也就是说，JMM允许这种重排序）。

### 5.2 增添的happens-before规则

* start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
* join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。

## ==6、 双重检查锁定与延迟初始化==

### 6.1 双重检查锁定

在Java程序中，有时候可能需要推迟一些高开销的对象初始化操作，并且只有在使用这些对象时才进行初始化。此时，程序员可能会采用延迟初始化。但要正确实现线程安全的延迟初始化需要一些技巧，否则很容易出现问题。比如，下面是非线程安全的延迟初始化对象的示例代码。

```java
public class UnsafeLazyInitialization {
    private static Instance instance;
    public static Instance getInstance() {
        if (instance == null) // 1：A线程执行
            instance = new Instance(); // 2：B线程执行
        return instance;
    }
}
```

在UnsafeLazyInitialization类中，假设A线程执行代码1的同时，B线程执行代码2。此时，线
程A可能会看到instance引用的对象还没有完成初始化

对于UnsafeLazyInitialization类，我们可以对getInstance()方法做同步处理来实现线程安全的延迟初始化。示例代码如下

```java
public class SafeLazyInitialization {
    private static Instance instance;
    public synchronized static Instance getInstance() {
        if (instance == null)
            instance = new Instance();
        return instance;
    }
}
```

由于对getInstance()方法做了同步处理，synchronized将导致性能开销。如果getInstance()方
法被多个线程频繁的调用，将会导致程序执行性能的下降。反之，如果getInstance()方法不会被多个线程频繁的调用，那么这个延迟初始化方案将能提供令人满意的性能。

在早期的JVM中，synchronized的开销很大，所以提出了一种双重检查锁定来降低同步的开销：

```java
public class DoubleCheckedLocking { // 1
    private static Instance instance; // 2
    public static Instance getInstance() { // 3
        if (instance == null) { // 4:第一次检查
            synchronized (DoubleCheckedLocking.class) { // 5:加锁
                if (instance == null) // 6:第二次检查
                    instance = new Instance(); // 7:问题的根源出在这里
            } // 8
        } // 9
        return instance; // 10
    } // 11
}
```

不使用synchronized对方法进行加锁，而是在方法内部要创建对象的时候进行加锁。这可以大幅降低synchronized带来的性能开销。但是，当代码执行到第4行，代码读取到instance不为null时，instance引用的对象有可能还没有完成初始化。

### 6.2 问题的根源

前面的双重检查锁定示例代码的第7行（instance=new Singleton();）创建了一个对象。这一行代码可以分解为如下的3行伪代码。

```java
memory = allocate();　　// 1：分配对象的内存空间
ctorInstance(memory);　 // 2：初始化对象
instance = memory;　　 // 3：设置instance指向刚分配的内存地址
```

上面3行伪代码中的2和3之间，可能会被重排序。2和3之间重排序之后的执行时序如下。

```java
memory = allocate();　　// 1：分配对象的内存空间
instance = memory;　　 // 3：设置instance指向刚分配的内存地址
// 注意，此时对象还没有被初始化！
ctorInstance(memory);　 // 2：初始化对象
```

![image-20200417175030121](F:\Java书和视频\笔记\images\并发编程\11.png)

==DoubleCheckedLocking示例代码的第7行（instance=new Singleton();）如果发生重排序，另一个并发执行的线程B就有可能在第4行判断instance不为null。线程B接下来将访问instance所引用的对象，但此时这个对象可能还没有被A线程初始化==

下面是这个场景的具体执行时序：

![image-20200417175139436](F:\Java书和视频\笔记\images\并发编程\12.png)

这里A2和A3虽然重排序了，但Java内存模型的intra-thread semantics将确保A2一定会排在A4前面执行。因此，线程A的intra-thread semantics没有改变，但A2和A3的重排序，将导致线程B在B1处判断出instance不为空，线程B接下来将访问instance引用的对象。此时，线程B将会访问到一个还未初始化的对象。

有两种方法可以来实现线程安全的延迟初始化：

1. 不允许2和3的重排序
2. 允许2和3的重排序，但不允许其他线程“看到”这个重排序

### 6.3 基于volatile的解决方案

如下代码就是线程安全的延迟初始化：

```java
public class SafeDoubleCheckedLocking {
    private volatile static Instance instance;
    public static Instance getInstance() {
        if (instance == null) {
            synchronized (SafeDoubleCheckedLocking.class) {
                if (instance == null)
                    instance = new Instance(); // instance为volatile，现在没问题了
            }
        }
        return instance;
    }
}
```

当声明对象的引用为volatile过后，伪代码中2和3之间的重排序会被禁止

### 6.4 基于类初始化的解决方案

JVM在类的初始化阶段（即在Class被加载后，且被线程使用之前），会执行类的初始化。在执行类的初始化期间，JVM会去获取一个锁。这个锁可以同步多个线程对同一个类的初始化。

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

![image-20200417175712138](F:\Java书和视频\笔记\images\并发编程\13.png)

## 7、JDK 5中的增强

* 增强volatile的内存语义。旧内存模型允许volatile变量与普通变量重排序。JSR-133严格限制volatile变量与普通变量的重排序，使volatile的写-读和锁的释放-获取具有相同的内存语义。
* 增强final的内存语义。在旧内存模型中，多次读取同一个final变量的值可能会不相同。为此，JSR-133为final增加了两个重排序规则。在保证final引用不会从构造函数内逸出的情况下，final具有了初始化安全性。

# 三、并发编程基础

## 1、线程简介

### 1.1 什么是线程

操作系统中拥有资源的最小单位是进程，而调度的最小单位是线程，线程由进程创建，一个进程可以创建多个线程；这些被创建的线程都有各自的计数器、堆栈和局部变量等属性，并且能访问共享的内存变量。

### 1.2 为什么要使用多线程

1. 更多的处理核心

   使用多线程，可以充分利用处理器的多核心，提升效率

2. 更快的响应时间

   使用不同的线程完成一个复杂系统中不同的任务，可以使响应时间缩短，不用等待一个任务执行完成再执行下一个任务

3. 更好的编程模型

   开发人员可以更加专注于问题的解决，即为所遇到的问题建立合适的模型，而不是绞尽脑汁地考虑如何将其多线程化。一旦开发人员建立好了模型，稍做修改总是能够方便地映射到Java提供的多线程编程模型上。

### ==1.3 线程的状态==

Java线程在运行的生命周期中，只能处于下表中的6中状态之一：

|   状态名称   |                             说明                             |
| :----------: | :----------------------------------------------------------: |
|     NEW      |       初始状态，线程被创建，但是还没有调用start()方法        |
|   RUNNABLE   | 运行状态，Java线程将曹忠系统中的就绪和运行两种状态统称作“运行中” |
|   BLOCKED    |                    阻塞状态，表示线程阻塞                    |
|   WAITING    | 等待状态，线程进入该状态表示当前线程需要等待其他线程做出一些特定的动作（通知或中断） |
| TIME_WAITING |              超时等待状态，等待特定的时间就返回              |
|  TERMINATED  |              终止状态，表示当前线程已经执行完毕              |

线程在自身的生命周期中，随着代码的执行在不同的状态之间切换，Java线程状态变迁如下图：

![image-20200418100127538](F:\Java书和视频\笔记\images\并发编程\14.png)

yield()方法会使调用者线程进入就绪状态，让出自己的处理器时间片

join()方法会使当前线程进入等待状态，等待调用者线程执行完成

### 1.4 守护线程

Daemon线程是一种支持型线程，它主要被用于程序中后台调度以及支持性工作。当一个Java虚拟机中**不存在非Daemon线程的时候**，Java虚拟机将会退出，也就是说如果存在非Daemon线程，JVM不会退出。可以通过调用Thread.setDaemon(true)将线程设置为Daemon线程。

**注意**　Daemon属性需要在启动线程之前设置，不能在启动线程之后设置。

Daemon线程被用作完成支持性工作，但是在Java虚拟机退出时Daemon线程中的finally块并不一定会执行

## 2、启动和终止线程

### 2.1 构造线程

在运行线程之前首先要构造一个线程对象，线程对象在构造的时候需要提供线程所需要的属性，如线程所属的线程组、线程优先级、是否是Daemon线程等信息。

线程初始化源代码如下：

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

在上述过程中，一个新构造的线程对象是由其parent线程来进行空间分配的，而child线程继承了parent是否为Daemon、优先级和加载资源的contextClassLoader以及可继承的ThreadLocal，同时还会分配一个唯一的ID来标识这个child线程。至此，一个能够运行的线程对象就初始化好了，在堆内存中等待着运行。

### 2.2 启动线程

线程对象初始化过后，调用start()方法就会启动线程。start()方法的含义是：当前线程（即parent线程，比如在main线程创建线程，main线程就是parent线程）同步告知Java虚拟机，只要线程规划器空闲，应立即启动调用start()方法的线程。

### 2.3 中断

中断可以理解为一个线程的标识属性，它表示一个运行中的线程是否被其他线程执行了中断操作。也就是说，**一个运行中的线程ThreadA，可以被另一个线程ThreadB进行中断，也就是在ThreadB中调用ThreadA.interrupt()**。在B中断A过后，**A不会停止运行，A可以通过调用自身的isInterrupted()方法来进行判断是否被中断，如果被中断，则做出相应的响应**。也可以调用静态方法Thread.interrupted()对当前线程的中断标识位进行复位。如果A线程已经处于终结状态，即使A被中断过，在调用A对象的isInterrupted()方法时也会返回false。

就比如说，我在使用番茄工作法工作中，在一个番茄钟之内，有另外一个人给我打招呼，相当于对我进行了中断操作，我会记录一个符号表示我被中断，但此时我不会停止工作，我会在工作中的某个时间点检查这个符号，如果这个符号说我被中断过，我就会自行决定如何做出响应，要么不管这个中断，要么回复给我打招呼的人。

从Java的API中可以看到，许多声明抛出InterruptedException的方法（例如Thread.sleep(longmillis)方法）这些方法在抛出InterruptedException之前，Java虚拟机会先将该线程的中断标识位清除，然后抛出InterruptedException，此时调用isInterrupted()方法将会返回false。

```java
public class Interrupted {
    public static void main(String[] args) throws Exception {
        //sleep Thread 不停的尝试睡眠
        Thread sleepThread = new Thread(() -> {while (true) {
            SleepUtils.second(10);
        }}, "sleepThread");
        sleepThread.setDaemon(true);
        //busy thread 不停的运行
        Thread busyThread = new Thread(() -> {while (true);}, "busyThread");
        busyThread.setDaemon(true);
        sleepThread.start();
        busyThread.start();
        
        //休息5秒，让两个线程充分运行
        TimeUnit.SECONDS.sleep(5);
        sleepThread.interrupt();
        busyThread.interrupt();
        System.out.println("sleepThread interrupted is " + sleepThread.isInterrupted());
        System.out.println("busyThread interrupted is " + busyThread.isInterrupted());
        TimeUnit.SECONDS.sleep(2);
    }
    
}

```

### 2.4 安全地终止线程

中断状态是线程的一个标识位，而中断操作是一种简便的线程间交互方式，而这种交互方式最适合用来取消和停止任务。除了中断外，还可以利用一个boolean变量来控制是否需要停止任务并终止该线程。

## ==3、线程间通信==

### 3.1 volatile和synchronized关键字

关键字volatile可以用来修饰字段（成员变量），它告知程序任何对该变量的访问都需要从共享内存中获取，而对它的改变必须同步刷新到共享内存中，它保证所有线程对变量访问的可见性。

关键字synchronized可以修饰方法或者以同步块的形式来进行使用，它主要确保多个线程在同一个时刻，只能有一个线程处于方法或者同步块中，它保证了线程对变量访问的可见性和排他性。

### 3.2 等待和通知机制

![image-20200418105754613](F:\Java书和视频\笔记\images\并发编程\15.png)

* **使用wait()、notify()和notifyAll()时需要先对调用对象加锁**
* 调用wait()方法后，线程状态由RUNNING变为WAITING，并将当前线程放置到对象的**等待队列**。而且会**释放锁**
* notify()或notifyAll()方法调用后，等待线程依旧不会从wait()返回，需要调用notify()或notifAll()的线程释放锁之后，等待线程才有机会从wait()返回
* **notify()方法将等待队列中的一个等待线程从等待队列中移到同步队列中**，**而notifyAll()方法则是将等待队列中所有的线程全部移到同步队列，被移动的线程状态由WAITING变为BLOCKED**
* 从wait()方法返回的前提是获得了调用对象的锁

等待/通知机制依托于同步机制，其目的就是确保等待线程从wait()方法返回时能够感知到通知线程对变量做出的修改。

### 3.3 管道输入/输出流

管道输入/输出流和普通的文件输入/输出流或者网络输入/输出流不同之处在于，它主要用于线程之间的数据传输，而**传输的媒介为内存**。、

管道输入/输出流主要包括了如下4种具体实现：`PipedOutputStream`、`PipedInputStream`、`PipedReader`和`PipedWriter`，前两种面向字节，而后两种面向字符.

对于Piped类型的流，必须先要进行绑定，也就是调用connect()方法，如果没有将输入/输出流绑定起来，对于该流的访问将会抛出异常。

### 3.4 ThreadLocal的使用

ThreadLocal，即线程变量，是一个以ThreadLocal对象为键、任意对象为值的存储结构。这个结构被附带在线程上，也就是说一个线程可以根据一个ThreadLocal对象查询到绑定在这个线程上的一个值。可以通过set(T)方法来设置一个值，在当前线程下再通过get()方法获取到原先设置的值。



首先，在每个线程Thread内部有一个ThreadLocal.ThreadLocalMap类型的成员变量threadLocals，这个threadLocals就是用来存储实际的变量副本的，键值为当前ThreadLocal变量（不是以线程为变量），value为变量副本。

初始时，在Thread里面，threadLocals为空，当通过ThreadLocal变量调用get()方法或者set()方法时，就会对Thread类中的threadLocals进行初始化，并且以当前ThreadLocal变量为键值，以ThreadLocal要保存的副本变量为value，存到threadlocals中。

```java
public class ThreadLocalTest {
    ThreadLocal<Long> longLocal = new ThreadLocal<Long>();
    ThreadLocal<String> stringLocal = new ThreadLocal<String>();
    
    public void set() {
        longLocal.set(Thread.currentThread().getId());
        stringLocal.set(Thread.currentThread().getName());
    }
    
    public long getLong() {
        return longLocal.get();
    }
    
    public String getString() {
        return stringLocal.get();
    }

    public static void main(String[] args) throws InterruptedException {
        
        final ThreadLocalTest test = new ThreadLocalTest();
        test.set();
        System.out.println(test.getLong());
        System.out.println(test.getString());
        
        Thread thread = new Thread(() -> {
            test.set();
            System.out.println(test.getLong());
            System.out.println(test.getString());
        });
        thread.start();
        thread.join();
        System.out.println(test.getLong());
        System.out.println(test.getString());
    }
    
}

```

**从上述代码可以看出，在main线程中和thread线程中，longLocal和stringLocal保存的副本变量值都不一样。最后一次在main线程再次打印副本是为了证明在main线程中和thread线程中的副本值确实不同**

==总结:==

1. ==实际的通过ThreadLocal创建的副本是存储在每个线程自己的threadLocals中的。因为调用set方法时，会根据当前线程找到该线程中的threadLocals，然后放入当前线程的map中==
2. ==为何threadLocals的类型ThreadLocalMap的键为ThreadLocal对象？因为每个线程中可有多个ThreadLocal变量，如果设定为线程id，那么就只能有一个threadlocal==
3. ==在进行get之前，必须先set，否则会报空指针错。如果想在get之前不调用set方法，那么就必须重写initialValue()方法==



下面代码构建了一个常用的Profiler类，它具有begin()和end()两个方法，而end()方法返回从begin()方法调用开始到end()方法被调用时的时间差，单位是毫秒。

```java
public class Profiler {
    // 第一次get()方法调用时会进行初始化（如果set方法没有调用），每个线程会调用一次
    private static final ThreadLocal<Long> TIME_THREADLOCAL = new ThreadLocal<Long>() {
        protected Long initialValue() {
            return System.currentTimeMillis();
        }
    };
    public static final void begin() {
        TIME_THREADLOCAL.set(System.currentTimeMillis());
    }
    public static final long end() {
        return System.currentTimeMillis() - TIME_THREADLOCAL.get();
    }
    public static void main(String[] args) throws Exception {
        Profiler.begin();
        TimeUnit.SECONDS.sleep(1);
        System.out.println("Cost: " + Profiler.end() + " mills");
    }
}
```

**进程间的通信方式**

- 管道
- 有名管道
- 信号量
- 消息队列
- 信号
- 共享内存
- 套接字

## 4、线程应用实例

### 4.1 等待超时

假设超时时间段是T，那么可以推断出在当前时间now+T之后就会超时。定义如下变量。

* 等待持续时间：REMAINING=T
* 超时时间：FUTURE=now+T

这时仅需要`wait(REMAINING)`即可，在`wait(REMAINING)`返回之后会将执行：`REMAINING=FUTURE–now`。如果`REMAINING`小于等于0，表示已经超时，直接退出，否则将继续执行```wait(REMAINING)`

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

# ==四、Java中的锁==

## 1、Lock接口

Lock接口提供了synchronized关键字不具备的特性：

![image-20200418114913698](F:\Java书和视频\笔记\images\并发编程\16.png)

Lock接口的API如下所示：

![image-20200418114943517](F:\Java书和视频\笔记\images\并发编程\17.png)

Lock接口的实现基本都是通过聚合了一个同步器的子类来完成线程访问控制的。

## 2、==队列同步器==

队列同步器AbstractQueuedSynchronizer是用来构建锁或者其他同步组件的基础框架，它使用一个**int成员变量表示同步状态**，通过**内置的FIFO队列**来完成资源获取线程的排队工作。

同步器的主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态，在抽象方法的实现过程中免不了要对同步状态进行更改，这时就需要使用同步器提供的3个方法（`getState()`、`setState(int newState)`和`compareAndSetState(int expect,int update)）`来进行操作，因为它们能够保证状态的改变是安全的。

AQS定义两种资源共享方式：Exclusive（独占，只有一个线程能执行，如ReentrantLock）和share（共享，多个线程可以同时执行，如Semaphore和CountDownLatch）

子类推荐**被定义为自定义同步组件的静态内部类**，同步器自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获取同步状态，这样就可以方便实现不同类型的同步组件（`ReentrantLock`、`ReentrantReadWriteLock`和`CountDownLatch`等）。

### 2.1 AQS的接口与示例

AQS的设计是基于==**模板方法模式**==的，也就是说，使用者需要继承AQS并重写指定的方法，随后将AQS组合在自定义同步组件的实现中，并调用AQS提供的模板方法，而这些模板方法将会调用使用者重写的方法

重写同步器指定的方法时，需要使用同步器提供的如下3个方法来访问或修改同步状态。

* getState()：获取当前同步状态
* setState(int new State)：设置当前同步状态
* compareAndSetState(int expect, int update)：使用CAS设置当前状态，该方法能够保证状态设置的原子性

同步器可**重写的方法**（自定义同步器要实现的方法）：

![image-20200418143052832](F:\Java书和视频\笔记\images\并发编程\18.png)

**以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire独占该锁并将state+1。此后，其他线程再tryAcquire就会失败，直到A线程unlock到state=0（即释放锁）为止，其他线程才有机会获取到该锁。**

**再以CountDownLatch为例，任务分为N个子线程去执行，state也初始化为N。这N个子线程是并行执行的，每个子线程执行完后countDown一次，state会CAS减1。等到所有子线程都执行完后（state=0），会unpark主调用线程，然后主调用线程就会从await函数返回，然后做后面的动作。**

实现自定义同步组件时，将会调用同步器提供的模板方法，这些模板方法与描述如下：

![image-20200418143524592](F:\Java书和视频\笔记\images\并发编程\19.png)

AQS提供的模板方法基本上分为3类：独占式获取与释放同步状态、共享式获取与释放同步状态和查询同步队列中的等待线程情况。**自定义同步组件将使用AQS提供的模板方法来实现自己的同步语义。**

```java
public class Mutex implements Lock {
    
    //静态内部类，自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        //是否处于独占状态

        @Override
        protected boolean isHeldExclusively() {
            return getState() == -1;
        }
        
        //当状态为0的时候获取锁
        @Override
        protected boolean tryAcquire(int arg) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        
        //释放锁，将状态设置为0
        @Override
        protected boolean tryRelease(int arg) {
            if (getState() == 0)
                throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        //返回一个Condition，每个Condition都包含一个condition队列
        Condition newCondition() { return new ConditionObject();}
    }
    //将操作代理到Sync上即可
    private final Sync sync = new Sync();
    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}

```

上述示例中，独占锁Mutex是一个自定义同步组件，它在同一时刻只允许一个线程占有锁。Mutex中定义了一个静态内部类，该内部类继承了同步器并实现了独占式获取和释放同步状态。在`tryAcquire(int acquires)`方法中，如果经过CAS设置成功（同步状态设置为1），则代表获取了同步状态，而在`tryRelease(int releases)`方法中只是将同步状态重置为0。用户使用Mutex时并不会直接和内部同步器的实现打交道，而是调用Mutex提供的方法，在Mutex的实现中，以获取锁的`lock()`方法为例，只需要在方法实现中调用同步器的模板方法`acquire(int args)`即可，当前线程调用该方法获取同步状态失败后会被加入到同步队列中等待，这样就大大降低了实现一个可靠自定义同步组件的门槛。

### 2.2队列同步器的实现分析

#### 2.2.1 同步队列

同步器依赖内部的同步队列（一个FIFO双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。

同步队列中的节点（Node）用来保存获取同步状态失败的线程引用、等待状态以及前驱和后继节点

![image-20200418145433615](F:\Java书和视频\笔记\images\并发编程\20.png)

**节点的状态值中，负值表示节点处于有效等待状态，而正值表示节点已被取消。所以源码中很多地方用>0、<0来判断节点的状态是否正常。**

节点是构成同步队列的基础，同步器拥有首节点（head）和尾节点（tail），没有成功获取同步状态的线程将会成为节点加入该队列的**尾部**，同步队列的基本结构如图：

![image-20200418145558381](F:\Java书和视频\笔记\images\并发编程\21.png)

同步器包含了两个节点类型的引用，一个指向头节点，而另一个指向尾节点。试想一下，当一个线程成功地获取了同步状态（或者锁），其他线程将无法获取到同步状态，转而被构造成为节点并加入到同步队列中，而这个加入队列的过程必须要保证线程安全，因此同步器提供了一个基于CAS的设置尾节点的方法：compareAndSetTail(Node expect,Nodeupdate)，它需要传递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联。

**首节点是获取同步状态成功的节点**，首节点的线程在释放同步状态时，将会唤醒后继节点，而后继节点将会在获取同步状态成功时将自己设置为首节点

#### 2.2.2 独占式同步状态获取与释放

调用同步器的acquire(int arg)方法可以获取同步状态，该方法对中断不敏感，也就是由于线程获取同步状态失败后进入同步队列中，后续对线程进行中断操作，线程不会从同步队列中移出，该方法代码如下：

```java
public final void acquire(int arg) {//这是一个模板方法，调用了我们重写的方法
    if (!tryAcquire(arg) && //tryAcquire方法是需要我们重写的方法
        //如果同步状态获取失败则构造节点（独占式Node.EXCLUSIVE，同一时刻只能有一个线程成功获取同步状态），通过addWaiter方法将节点加入到同步队列尾部，调用acquireQueued方法使得该结点以死循环的方式获取同步状态
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
这段代码主要完成了同步状态获取，如果获取成功则直接返回，如果没有获取成功则需要构造节点，加入同步队列，以及在同步队列中自旋等待的相关工作。如果有中断信号，那么需要处理中断
```

函数流程如下：

1. tryAcquire()尝试直接去获取资源，如果成功则直接返回（**这里体现了非公平锁，每个线程获取锁时会尝试直接抢占加塞一次，而CLH队列中可能还有别的线程在等待**）；
2. addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
3. acquireQueued()使线程阻塞在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
4. **如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。**

```java
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
```

```java
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

```java
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

步骤:

1. 节点进入队尾后，检查状态，找到安全休息点
2. 调用park进入waiting状态，等待unpark或interrupt唤醒自己
3. 被唤醒后，看自己是不是有资格能拿到号，如果拿到，head指向当前节点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程1

**在acquireQueued方法的实现中，只有前驱节点是头节点的节点才能够尝试获取同步状态**，因为：

1. 头节点是成功获取到同步状态的节点，而头节点的线程释放了同步状态之后，将会唤醒其后继节点，后继节点的线程被唤醒后需要检查自己的前驱节点是否是头节点
2. **维护同步队列的FIFO原则。**

acquire方法的调用流程图如下：

![image-20200418151225751](F:\Java书和视频\笔记\images\并发编程\22.png)

**总结**

**在获取同步状态时，同步器维护一个同步队列，获取状态失败的线程都会被加入到队列中并在队列中进行自旋**；**移除队列（或停止自旋）的条件是前驱节点为头节点且成功获取到同步状态**。在释放同步状态时，同步器调用tryRelease()方法释放同步状态，然后唤醒头节点的后继节点

#### 2.2.3 共享式同步状态获取与释放

通过调用同步器的acquireShared(int arg)方法可以共享式地获取同步状态，该方法代码如下：

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)//小于0表示已经有线程在阻塞中了
        doAcquireShared(arg);//自旋获取state
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
                if (r >= 0) {//成功获取到资源
                    setHeadAndPropagate(node, r);//将head指向自己，还有剩余资源可以唤醒之后的线程
                    p.next = null;
                    //在此处要响应中断
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            //和acquireQueued中一样
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
在acquireShared(int arg)方法中，同步器调用tryAcquireShared(int arg)方法尝试获取同步状态，tryAcquireShared(int arg)方法返回值为int类型，当返回值大于等于0时，表示能够获取到同步状态。因此，在共享式获取的自旋过程中，成功获取到同步状态并退出自旋的条件就是tryAcquireShared(int arg)方法返回值大于等于0。可以看到，在doAcquireShared(int arg)方法的自旋过程中，如果当前节点的前驱为头节点时，尝试获取同步状态，如果返回值大于等于0，表示该次获取同步状态成功并从自旋过程中退出。
```

## ==3、重入锁==

重入锁支持一个线程对一个资源重复加锁，该锁还支持获取锁时的公平和非公平特性

**锁获取的公平性问题**

如果在绝对时间上，先对锁进行获取的请求一定先被满足，那么这个锁是公平的，反之是不公平的。公平的获取锁，也就是等待时间最长的线程最优先获取锁。ReentrantLock提供了一个构造函数，能够控制锁是否是公平的。但是**默认是非公平的**。

```java
public ReentrantLock() {
    sync = new NonfairSync();
}
```

### 3.1 ReentrantLock实现重进入（独占锁）

* 线程再次获取锁。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再次成功获取
* 锁的最终释放。线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到该锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而锁被释放时，计数自减，当计数等于0时表示锁已经成功释放。

ReentrantLock是通过组合自定义同步器来实现锁的获取与释放，以非公平性（默认的）实现为例，获取同步状态的代码如下：

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();//获取当前线程
    int c = getState();//获取同步状态
    if (c == 0) {//如果state为0，表示没有人获取
        if (compareAndSetState(0, acquires)) {//直接获取，不用管是否有其他线程在等待
            setExclusiveOwnerThread(current);//将当前线程设为拥有者
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {//如果当前线程是之前获取了状态的线程
        int nextc = c + acquires;//持续累加状态值
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

该方法增加了再次获取同步状态的处理逻辑：**通过判断当前线程是否为获取锁的线程来决定获取操作是否成功，如果是获取锁的线程再次请求，则将同步状态值进行增加并返回true，表示获取同步状态成功。**

成功获取锁的线程再次获取锁，只是增加了同步状态值，这也就要求ReentrantLock在释放同步状态时减少同步状态值，该方法的代码如下：

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

### 3.2 公平和非公平获取锁的区别

对于非公平锁，只要CAS设置同步状态成功，则表示当前线程获取了锁，而公平锁则不同，如代码：

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {//在能获取同步状态时，还要判断自己是否是当前等待队列的队首的后继队列
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

该方法与1比较，唯一不同的位置为判断条件多了`hasQueuedPredecessors()`方法，即加入了同步队列中当前节点是否有前驱节点的判断，如果该方法返回`true`，则表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。

**公平锁和非公平锁的区别**

公平性锁保证了锁的获取按照FIFO原则，而代价是进行大量的线程切换。非公平性锁虽然可能造成线程“饥饿”，但极少的线程切换，保证了其更大的吞吐量。

## ==4、读写锁==

读写锁维护了一对锁，一个读锁，一个写锁，通过分离读锁和写锁，使得并发性相比一般的排它锁有很大提升。Java并发包提供读写锁的实现是ReentrantReadWriteLock

![image-20200418182304165](F:\Java书和视频\笔记\images\并发编程\23.png)

### 4.1 读写锁的接口与示例

`ReadWriteLock`仅定义了获取读锁和写锁的两个方法，即`readLock()`方法和`writeLock()`方法，而其实现——`ReentrantReadWriteLock`，除了接口方法之外，还提供了一些便于外界监控其内部工作状态的方法

![image-20200418182448140](F:\Java书和视频\笔记\images\并发编程\24.png)

ReadLock是共享锁，WriteLock是独享锁

- ReadLock

```java
public void lock() {
	sync.acquireShared(1);//最终会调用sync中实现的tryAcquireShared方法
}

public boolean tryLock() {
	return sync.tryReadLock();//调用treReadLock方法
}
这两个方法的作用相同，不过tryReadLock中没有调用readerShouldBlock方法
```

- tryAcquireShared

    ```java
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        int c = getState();
        //如果存在写锁并且获取写锁的线程不是当前线程，那么就不能获取读锁。也就是说一个线程要获取读锁，要么没有其他线程获取写锁；要么写锁是由该线程获取的。同一个线程在获取写锁的情况下能够获取读锁。
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return -1;
        //获取了读锁的线程的数量
        int r = sharedCount(c);
        //如果读线程应该被阻塞，或者读线程的数量大于了最大限制那么就不执行这个方法体
        if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) {
            //如果获取读锁的线程数等于0，将该线程设置为第一个读线程
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return 1;
        }
        return fullTryAcquireShared(current);
    }
    ```

- tryReadLock

    ```java
    final boolean tryReadLock() {
        Thread current = Thread.currentThread();
        for (;;) {
            int c = getState();
            // 如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return false;
            int r = sharedCount(c);
            if (r == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            if (compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return true;
            }
        }
    }
    ```

- WriteLock

```java
protected final boolean tryAcquire(int acquires) {
    /*
    * Walkthrough:
    * 1. If read count nonzero or write count nonzero
    *    and owner is a different thread, fail.
    * 2. If count would saturate, fail. (This can only
    *    happen if count is already nonzero.)
    * 3. Otherwise, this thread is eligible for lock if
    *    it is either a reentrant acquire or
    *    queue policy allows it. If so, update state
    *    and set owner.
    */
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);//写锁的个数
    if (c != 0) {//如果已有线程持有了锁
        // (Note: if c != 0 and w == 0 then shared count != 0)
        // 如果写线程数（w）为0（换言之存在读锁） 或者持有锁的线程不是当前线程就返回失败。也就是说在由读锁的情况下不能获取写锁（但是在有写锁的情况下可以获取读锁）
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 如果写入锁的数量大于最大数（65535，2的16次方-1）就抛出一个Error。
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    // 如果当且写线程数为0，并且当前线程需要阻塞那么就返回失败；或者如果通过CAS增加写线程数失败也返回失败
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    // 如果c=0，w=0或者c>0，w>0（重入），则设置当前线程或锁的拥有者
    setExclusiveOwnerThread(current);
    return true;
}
```



下面是一个示例代码：

```java
public class Cache {
    static Map<String, Object> map = new HashMap<>();
    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    static Lock r = rwl.readLock();
    static Lock w = rwl.writeLock();
    
    //获取一个key对应的value
    public static final Object get(String key) {
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }
    
    //设置key对应的value，并返回旧的value
    public static final Object put(String key, Object value) {
        w.lock();
        try {
            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }
    
    //清空所有的内容
    public static final void clear() {
        w.lock();
        try {
            map.clear();
        } finally {
            w.unlock();
        }
    }
}
```

## 5、LockSupport工具

当需要阻塞或唤醒一个线程的时候，都会使用LockSupport工具类来完成相应工作。LockSupport定义了一组的公共静态方法，这些方法提供了最基本的线程阻塞和唤醒功能，而LockSupport也成为构建同步组件的基础工具。

![image-20200418184900522](F:\Java书和视频\笔记\images\并发编程\25.png)

## 6、Condition接口

任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括wait()、wait(long timeout)、notify()以及notifyAll()方法，这些方法与synchronized同步关键字配合，可以实现等待/通知模式。Condition接口也提供了类似Object的监视器方法，与Lock配合可以实现等待/通知模式，但是这两者在使用方式以及功能特性上还是有差别的。

下图是Condition接口和Object的监视器方法的比较：

![image-20200418185147862](F:\Java书和视频\笔记\images\并发编程\26.png)

### 6.1 Condition的实现分析

每个Condition对象都包含一个队列，该队列是Condition对象实现等待/通知的关键

1. 等待队列

   等待队列是一个FIFO队列，在队列中的每个节点都包含了一个线程引用，该线程就是在Condition对象上等待的线程，如果一个线程调用了Condition.await()方法，那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态。事实上，等待队列复用了AQS中的节点的定义，也就是说同步队列和等待队列中节点类型都是AQS的静态内部类AbstractQueuedSynchronizer.Node

   一个Condition包含一个等待队列，Condition拥有首节点和尾节点。当前线程调用Condition.await()方法，将会以当前线程构造节点，并将节点从尾部加入等待队列，等待队列的基本结构为：

   ![image-20200418191429664](F:\Java书和视频\笔记\images\并发编程\27.png)

   上述节点引用更新的过程并没有使用CAS保证，因为调用await()方法的线程必定是获取了锁的线程，也就是说该过程是由锁来保证线程安全的。

   在Object的监视器模型上，一个对象拥有一个同步队列和等待队列，而并发包中的Lock（更确切地说是同步器）拥有一个同步队列和多个等待队列，其对应关系如图所示。

2. 等待

   调用Condition的await()方法（或者以await开头的方法），会使当前线程进入等待队列并释放锁，同时线程状态变为等待状态。当从await()方法返回时，当前线程一定获取了Condition相关联的锁。

   如果从队列（同步队列和等待队列）的角度看await()方法，当调用await()方法时，相当于同步队列的首节点（获取了锁的节点）移动到Condition的等待队列中。

3. 通知

   调用Condition的signal()方法，将会唤醒在等待队列中等待时间最长的节点（首节点），在唤醒节点之前，会将节点移到同步队列中

![image-20200418192946786](F:\Java书和视频\笔记\images\并发编程\28.png)

# 五、Java并发容器和框架

## ==1、ConcurrentHashMap==

### 1.1 为什么要使用ConcurrentHashMap

并发编程中HashMap可能会造成死循环或写覆盖，而HashTable（对整个方法加锁）的效率很低，因此需要使用效率高的线程安全的HashMap。

ConcurrentHashMap在JDK 1.7 和1.8中实现的原理不一样。我们先来看1.7中的原理。

在1.7中，ConcurrentHashMap的所分段技术

### 1.2 ConcurrentHashMap的实现

#### 1.2.1 jdk 1.7实现

##### 分段锁机制

ConcurrentHashMap在对象中保存了一个Segment数组，即将整个Hash表划分为多个分段；而每个Segment元素，即每个分段类似于一个HashTable；这样，在执行put操作时首先根据哈希算法定位到元素属于哪个Segment，然后对该Segment加锁即可。如下图所示：

![image-20200407081448277](F:\Java书和视频\笔记\images\java基础\5.png)

ConcurrentHashMap的高效并发机制由三个方面来保证：

1. 通过锁分段技术保证并发环境下的写操作
2. 通过HashEntry的不变性、Volatile变量的内存可见性和加锁重读机制保证高效、安全的读操作
3. 通过不加锁和加锁两种方案控制跨段操作的安全性

##### 成员变量定义

与HashMap相比，ConcurrentHashMap增加了两个属性用于定位段，分别是**segmentMask**和**segmentShift**。segmentMask的大小等于segments数组的大小减1，segmentShift大小等于32（hash值的位数）减去对segments的大小取以2为底的对数值。

定位方式：假设ConcurrentHashMap一共分为$2^n$个段，每个段有$2^m$个桶，那么段的定位方式是将key的hash值的高n位与($2^n - 1$)相与，在定位到某个段过后，再将key的hash值的低m位与$(2^m - 1)$相与，定位到具体的桶。也就是说，key的哈希值的高n位用于表示段，低n位用于表示桶

##### 段的定义

Segment类继承于ReentrantLock类，从而使Segment对象能充当锁的角色。每个Segment对象用来守护它的成员变量table中包含的若干个桶。table是由HashEntry对象组成的链表数组，table数组中每一个数组成员就是一个桶。

在Segment类中，count变量是一个计数器，它表示每个Segment对象管理的table数组中包含的HashEntry对象的个数。**之所以每个Segment维护一个计数器，而不是在ConcurrentHashMap中使用全局的计数器，是因为当需要更新计数器时，不用锁定整个ConcurrentHashMap。只需要在读计数器的时候锁定整个Map就行了**。

##### HashEntry

HashEntry与Entry的不同之处在于，HashEntry内部的hash、key和next值都是final修饰的（1.7以后，next值不是final），**value域被volatile修饰**，因此HashEntry对象几乎是不可变的，**这是ConcurrentHashMap读操作不需要加锁的一个重要原因**。value域被volatile修饰，所以其可以确保被读线程读到最新的值。

与HashMap类似，在ConcurrentHashMap中，如果在散列时发生碰撞，也会将碰撞的HashEntry对象链成一个链表，**由于HashEntry的next域是final的，所以只能采用头插法**

##### 并发存取

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    //定位到特定的段。假设Segment的数量是2的n次方，那么segmentShift的值就是32-n，而segmentMask的值就是2^n-1（2进制为n个1）。所以，根据key的hash值的高n位就可确定元素到低在哪一个Segment中
    int j = (hash >>> segmentShift) & segmentMask;//定位在哪一个段中
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    //将put操作委托给内部类Segment
    return s.put(key, hash, value, false);
}
```

Segment类中的put操作：

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    //加锁。
    //Scans for a node containing given key while trying to acquire lock, creating and returning one if not found. Upon return, guarantees that lock is held.
    //在scanAndLockForPut内部会自旋获取锁，如果重试的次数达到了 MAX_SCAN_RETRIES 则改为阻塞锁获取，保证能获取成功。
    HashEntry<K,V> node = tryLock() ? null :
    scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        //定位到段中特定的桶
        int index = (tab.length - 1) & hash;
        //first指向段中的链表头节点
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                //如果已经存在键值对相同的元素，则直接覆盖
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            //如果到链表尾都没有找到相同的元素，就要添加此元素
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    //扩容，争对一个segment内部得数组进行扩容
                    rehash(node);
                else
                    //将元素放入到table相应的链表中
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

扩容操作：

```java
private void rehash(HashEntry<K,V> node) {
    /*
             * Reclassify nodes in each list to new table.  Because we
             * are using power-of-two expansion, the elements from
             * each bin must either stay at same index, or move with a
             * power of two offset. We eliminate unnecessary node
             * creation by catching cases where old nodes can be
             * reused because their next fields won't change.
             * Statistically, at the default threshold, only about
             * one-sixth of them need cloning when a table
             * doubles. The nodes they replace will be garbage
             * collectable as soon as they are no longer referenced by
             * any reader thread that may be in the midst of
             * concurrently traversing table. Entry accesses use plain
             * array indexing because they are followed by volatile
             * table write.
             */
    //先将旧table中的元素移到新table中，此时新添加的元素还没有添加进来，需要到最后才添加
    HashEntry<K,V>[] oldTable = table;
    //旧的容量
    int oldCapacity = oldTable.length;
    //新的容量
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    //临时变量newTable用来接收旧table中的数据
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    //用于定位桶
    int sizeMask = newCapacity - 1;
    //将oldTable中的数据转存到newTable中
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;//重哈希到新桶中的位置
            if (next == null)   //  桶中只有一个元素的时候直接移动过去
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                /*由于扩容是按照2的幂次方进行的，所以扩展前在同一个桶中的元素，现在要么还是在原来的序号的桶里，或者就是原来的序号再加上一个2的幂次方，就这两种选择。根据本文前面对HashEntry的介绍，我们知道链接指针next是final的，因此看起来我们好像只能把该桶的HashEntry链中的每个节点复制到新的桶中(这意味着我们要重新创建每个节点)，但事实上JDK对其做了一定的优化。因为在理论上原桶里的HashEntry链可能存在一条子链，这条子链上的节点都会被重哈希到同一个新的桶中，这样我们只要拿到该子链的头结点就可以直接把该子链放到新的桶中，从而避免了一些节点不必要的创建，提升了一定的效率。因此，JDK为了提高效率，它会首先去查找这样的一个子链，而且这个子链的尾节点必须与原hash链的尾节点是同一个，那么就只需要把这个子链的头结点放到新的桶中，其后面跟的一串子节点自然也就连接上了。对于这个子链头结点之前的结点，JDK会挨个遍历并把它们复制到新桶的链头(只能在表头插入元素)中*/
                HashEntry<K,V> lastRun = e;//桶中的头节点
                int lastIdx = idx;//这是新桶中的位置
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    //寻找与k值相同的子链，该子链尾节点与父节点的尾节点必须是同一个，相当于是从链尾开始向前数k值相同的子链
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                //将子链直接转存到新table中，不重新做hash，而且重用这些节点
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes，需要克隆节点，产生内存消耗。只遍历lastRun之前的节点，因为lastRun之后的节点已经移到新table中了
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);//头插法
                }
            }
        }
    }
    //插入新添加的元素
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```

**先扩容，再添加新的元素**

##### size操作

如果要统计整个`ConcurrentHashMap`里元素的大小，就必须统计所有`Segment`里元素的大小后求和。`Segment`里的全局变量`count`是一个`volatile`变量，那么在多线程场景下，是不是直接把所有`Segment`的`count`相加就可以得到整个`ConcurrentHashMap`大小了呢？不是的，虽然相加时可以获取每个`Segment`的`count`的最新值，但是可能累加前使用的`count`发生了变化，那么统计结果就不准了。所以，最安全的做法是在统计size的时候把所有`Segment`的`put`、`remove`和`clean``方法`全部锁住，但是这种做法显然非常低效。

因为在累加`count`操作过程中，之前累加过的`count`发生变化的几率非常小，所以`ConcurrentHashMap`的做法是**先尝试2次通过不锁住`Segment`的方式来统计各个`Segment`大小，如果统计的过程中，容器的`count`发生了变化，则再采用加锁的方式来统计所有`Segment`的大小。**那么``ConcurrentHashMap`是如何判断在统计的时候容器是否发生了变化呢？使用`modCount`变量，在`put`、`remove`和`clean`方法里操作元素前都会将变量`modCount`进行加1，那么在统计`size`前后比较`modCount`是否发生变化，从而得知容器的大小是否发生变化。

##### ConcurrentHashMap读操作不需要加锁的奥秘

* 用HashEntry对象的不变性来降低读操作对加锁的需求
* 用volatile变量协调读写线程间的内存可见性
* 若读时发生指令重排列现象，则加锁重读

##### ConcurrentHashMap跨段操作

在ConcurrentHashMap中，有些操作需要涉及到多个段，比如说size操作、containsValaue操作等。以size操作为例，如果我们要统计整个ConcurrentHashMap里元素的大小，那么就必须统计所有Segment里元素的大小后求和。我们知道，Segment里的全局变量count是一个volatile变量，那么在多线程场景下，我们是不是直接把所有Segment的count相加就可以得到整个ConcurrentHashMap大小了呢？显然不能，虽然相加时可以获取每个Segment的count的最新值，但是拿到之后可能累加前使用的count发生了变化，那么统计结果就不准了。所以最安全的做法，是在统计size的时候把所有Segment的put，remove和clean方法全部锁住，但是这种做法显然非常低效。

size方法主要思路是先在没有锁的情况下对所有段大小求和，这种求和策略最多执行RETRIES_BEFORE_LOCK次(默认是两次)：在没有达到RETRIES_BEFORE_LOCK之前，求和操作会不断尝试执行（这是因为遍历过程中可能有其它线程正在对已经遍历过的段进行结构性更新）；在超过RETRIES_BEFORE_LOCK之后，如果还不成功就在持有所有段锁的情况下再对所有段大小求和。事实上，在累加count操作过程中，之前累加过的count发生变化的几率非常小，所以ConcurrentHashMap的做法是先尝试RETRIES_BEFORE_LOCK次通过不锁住Segment的方式来统计各个Segment大小，如果统计的过程中，容器的count发生了变化，则再采用加锁的方式来统计所有Segment的大小。

　　那么，ConcurrentHashMap是如何判断在统计的时候容器的段发生了结构性更新了呢？我们在前文中已经知道，Segment包含一个modCount成员变量，在会引起段发生结构性改变的所有操作(put操作、 remove操作和clean操作)里，都会将变量modCount进行加1，因此，JDK只需要在统计size前后比较modCount是否发生变化就可以得知容器的大小是否发生变化。

#### 1.2.2 jdk 1.8实现

在1.7中查询遍历链表效率太低，因此，1.8在数据结构上做了调整：

![image-20200407110811428](F:\Java书和视频\笔记\images\java基础\6.png)

而且，**抛弃了原有的Segment分段锁，而采用了CAS+Synchronized来保证并发安全性**

##### put操作

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());//将hash进行扩展
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //判断是否需要进行初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //f 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,+++++++++++
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin。将元素插入空表的时候不用加锁，直接使用自旋
        }
        //如果当前位置的 hashcode == MOVED == -1,则需要进行扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        //如果都不满足，则利用 synchronized 锁写入数据
        else {
            V oldVal = null;
            //使用synchronized来加锁，f是该桶的头节点，锁住的是一个桶，所以其他桶有插入操作时也可以继续
            synchronized (f) {
                //再判断一次，免得被改了，被改了就自旋
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {//fh就是f的哈希值
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //如果存在相同的元素，就覆盖
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            //如果到最后都没有找到相同的元素，则在链表最后插入元素。（尾插法）next没有用final修饰了，所以可以用尾插法，但是1.7中不行
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                //如果数量大于 TREEIFY_THRESHOLD 则要转换为红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}

```

CAS在对空桶加元素的时候用到，synchronized在对非空桶加元素的时候使用。这样改进的目的还是在于减少线程等待时间，提高效率

## 2、ConcurrentLinkedQueue

ConcurrentLinkedQueue是使用非阻塞方式实现的线程安全队列，它是一个基于链接节点的无界队列，采用了CAS算法来实现。

### 2.1 结构

![image-20200419095509082](F:\Java书和视频\笔记\images\并发编程\29.png)

ConcurrentLinkedQueue由head节点和tail节点组成，每个节点（Node）由节点元素（item）和指向下一个节点（next）的引用组成，节点与节点之间就是通过这个next关联起来，从而组成一张链表结构的队列

### 2.2 入队列相关知识

#### 2.2.1 入队列的过程

入队列就是将节点添加到队列的尾部，现在假设我们要添加4个节点

![image-20200419095750434](F:\Java书和视频\笔记\images\并发编程\30.png)

* 添加元素1，队列更新head节点的next节点为元素1节点，又因为tail节点默认情况下等于head节点，因此它们的next节点都指向元素1节点
* 添加元素2，队列首先设置元素1节点的next节点为元素2节点，然后更新tail节点指向元素2节点
* 添加元素3，设置tail节点的next节点为元素3节点
* 添加元素4，设置元素3节点的next节点为元素4节点，然后将tail节点指向元素4节点

从这个过程我们可以看出，**如果tail节点的next节点为空，则将插入节点设置成tail的next节点，但是tail不改变；如果tail节点的next节点不为空，才将tail指向入队节点**。==所以tail节点不总是尾节点==

队列的插入算法如下：

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

整个入队过程主要做了两件事情：第一是定位出尾节点；第二是使用CAS算法将入队节点设置成尾节点的next节点，如果不成功则重试

#### 2.2.2 定位尾节点

tail节点不总是尾节点，因此每次入队都必须通过tail节点来找到尾节点。尾节点可能是tail节点，也可能是tail节点的next节点。代码中循环体的第一个if就是判断tail是否有next节点，有则表示next节点有可能是尾节点。获取tail节点的next节点需要注意的是p节点等于p的next节点的情况，只有一种可能就是p节点和p的next节点都等于空，表示这个队列刚初始化，证准备添加节点，所以需要返回head节点。

```java
final Node<E> succ(Node<E> p) {
    Node<E> next = p.getNext();
    return (p == next) head : next;
}
```

#### 2.2.3 HOPS的设计意图

使用hops变量来控制并减少tail节点的更新频率，并不是每次节点入队后都将tail节点更新成尾节点，而是当tail节点和尾节点的距离大于等于常量HOPS的值（默认等于1）时才更新tail节点，tail和尾节点的距离越长，使用CAS更新tail节点的次数就会越少，但是距离越长带来的负面效果就是每次入队时定位尾节点的时间就越长，因为循环体需要多循环一次来定位出尾节点，但是这样仍然能提高入队的效率，因为从本质上来看它通过增加对volatile变量的读操作来减少对volatile变量的写操作，而对volatile变量的写操作开销要远远大于读操作，所以入队效率会有所提升。

### 2.3 出队列

并不是每次出队时都更新head节点，**当head节点里有元素时，直接弹出head节点里的元素，而不会更新head节点。只有当head节点里没有元素时，出队操作才会更新head节点。这种做法也是通过hops变量来减少使用CAS更新head节点的消耗，从而提高出队效率**。

![image-20200419101541950](F:\Java书和视频\笔记\images\并发编程\31.png)

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

## 3、==CopyOnWriteArrayList==

### COW思想

ArrayList并不是线程安全的容器，而使用Collections来转换为线程安全的框架会使用synchronized来锁住整个表，这样做的效率太低了。如果是使用ReentrantReadWriteLock来进行封装的话，由于读写锁的特性，当写锁被写线程获取后，读写线程都会被阻塞（同一个获取写锁的线程可以获取读锁，但是其他线程不行）。如果要保证**读线程在任何时候都不会被阻塞**，那就需要使用COW思想。

**CopyOnWriteArrayList时在写数据时采用复制的思想来通过延时更新的策略实现数据的最终一致性，并且能够保证读线程间不阻塞。**在写数据的时候，先复制一个新的链表，往这个新的链表添加数据，添加完元素之后，再将原容器的引用指向新的容器。

- Add方法

    ```java
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        //添加的时候要加锁
        lock.lock();
        try {
            //获得现有的内置数组
            Object[] elements = getArray();
            int len = elements.length;
            //复制数组得到一个新的数组，新数组容量要加1
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            //向新数组中添加元素
            newElements[len] = e;
            //将旧数组的引用指向新的数组，由于数组被volatile修饰，所以写线程对数组引用的修改对读线程是可见的
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
    ```

- Get方法

    ```java
    public E get(int index) {
        return get(getArray(), index);
    }
    /**
     * Gets the array.  Non-private so as to also be accessible
     * from CopyOnWriteArraySet class.
     */
    final Object[] getArray() {
        return array;
    }
    private E get(Object[] a, int index) {
        return (E) a[index];
    }
    
    ```

    优点：实现简单，能够确保读线程不被阻塞

    缺点：1. 内存占用问题：写操作时会复制一个数组，可能会造成频繁的MinorGC和MajorGC；2. 数据一致性问题：CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。

## 4、CopyOnWriteArraySet

内部使用CopyOnWriteArrayList来实现

## 5、ConcurrentSkipListMap

## 6、ConcurrentSkipListSet

## 7、阻塞队列

### 7.1 Java里的阻塞队列

* ArrayBlockingQueue：一个由数组结构组成的有界阻塞队列
* LinkedBlockingQueue：一个由链表结构组成的有界阻塞队列
* PriorityBlockingQueue：一个支持优先级排序的**无界**阻塞队列（不能保证同优先级的顺序）
* DelayQueue：一个使用优先级队列实现的**无界**阻塞队列（支持延时获取元素，使用PriorityQueue来实现，在创建元素时可以指定多久才能从队列中获取当前元素，只有在延迟期满是才能从队列中提取元素）
* SynchronousQueue：一个不存储元素的阻塞队列
* LinkedTransferQueue：一个由链表结构组成的无界阻塞队列
* LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列

## 8、Fork/Join框架

Fork/Join框架是Java 7提供的一个用于并行执行任务的框架，是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。

### 8.1 工作窃取算法

当把一个大任务分为若干小任务，并把不同的小任务分派给不同的线程执行时，如果某一个线程先把任务做完，它可以去帮助其他线程执行任务。被窃取任务的线程从队列前端获取任务，窃取任务的线程从后端获取任务。

![image-20200419104528028](F:\Java书和视频\笔记\images\并发编程\32.png)

* 工作窃取算法的优点

  充分利用线程进行并行计算，减少了线程间的竞争

* 工作窃取算法的缺点

  在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且该算法会消耗更多的系统资源，比如创建多个线程和多个双端队列

### 8.2 使用框架

ForkJoinTask：我们要使用框架，必须首先创建一个ForkJoin任务，它提供在任务中执行fork()和join()操作的机制。通常情况下，我们不需要直接继承ForkJoinTask类，只需要继承它的子类：

* RecursiveAction：用于没有返回结果的任务
* RecursiveTask：用于有返回结果的任务

ForkJoinTask需要通过ForkJoinPool来执行。

**示例**

我们使用Fork/Join框架实现1+2+3+4

使用Fork/Join框架首先要考虑到的是如何分割任务，如果希望每个子任务最多执行两个数的相加，那么我们设置分割的阈值是2，由于是4个数字相加，所以Fork/Join框架会把这个任务fork成两个子任务，子任务一负责计算1+2，子任务二负责计算3+4，然后再join两个子任务的结果。因为是有结果的任务，所以必须继承ecursiveTask，实现代码如下。

```java
public class CountTask extends RecursiveTask<Integer> {

    public static final int THRESHOLD = 2;//阈值
    private int start;
    private int end;
    public CountTask(int start, int end) {
        this.start = start;
        this.end = end;
    }
    
    @Override
    protected Integer compute() {
        int sum = 0;
        //如果任务足够小就计算任务
        boolean canCompute = (end - start) <= THRESHOLD;
        if (canCompute) {
            for (int i = start; i <= end; i++) {
                sum += i;
            }
        } else {
            //如果任务大于阈值，就分裂成两个子任务计算
            int middle = (start + end) / 2;
            CountTask leftTask = new CountTask(start, middle);
            CountTask rightTask = new CountTask(middle + 1, end);
            //执行子任务
            leftTask.fork();
            rightTask.fork();
            //等待子任务执行完成，并得到结果
            int leftResult = leftTask.join();
            int rightResult = rightTask.join();
            //合并子任务
            sum = leftResult + rightResult;
        }
        return sum;
        
    }

    public static void main(String[] args) {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        //生成一个计算任务，负责计算1+2+3+4
        CountTask countTask = new CountTask(1, 4);
        //执行一个任务
        Future<Integer> result = forkJoinPool.submit(countTask);
        try {
            System.out.println(result.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

# 六、Java中的原子操作类

## 1、原子更新基本类型

* AtomicBoolean：原子更新布尔类型
* AtomicInteger：原子 更新整形
* AtomicLong：原子更新长整形

使用unsafe包中的compareAndSwapxxx方法来实现

## 2、原子更新数组

* AtomicIntegerArray：原子更新整型数组里的元素
* AtomicLongArray：原子更新长整型数组里的元素
* AtomicReferenceArray：原子更新引用类型数组里的元素

```java
public class AtomicIntegerArrayTest {
    static int[] value = new int[] { 1， 2 };
    static AtomicIntegerArray ai = new AtomicIntegerArray(value);
    public static void main(String[] args) {
        ai.getAndSet(0， 3);
        System.out.println(ai.get(0));
        System.out.println(value[0]);
    }
}
输出：
    3
    1
```

数组value通过构造方法传递进去，然后AtomicIntegerArray会将当前数组复制一份，所以当AtomicIntegerArray对内部的数组元素进行修改时，不会影响传入的数组。

## 3、原子更新引用类型

* AtomicReference：原子更新引用类型
* AtomicReferenceFieldUpdater：原子更新引用类型里的字段
* AtomicMarkableReference：原子更新带有标记位的引用类型

## 4、原子更新字段类

如果需要原子地更新某个类的某个字段时，就需要使用原子更新字段类

* AtomicIntegerFieldUpdater：原子更新整型的字段的更新器
* AtomicLongFieldUpdater：原子更新长整型字段的更新器
* AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于原子的更新数据和数据的版本号，可以解决使用CAS进行原子更新时可能出现的ABA问题。

# 七、并发工具类

## 1、等待多线程完成的CountDownLatch

CountDownLatch允许一个或多个线程等待其他线程完成工作。

CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果向等待N个点完成操作，那么就传入N。

当我们调用CountDownLatch的countDown方法的时候，N就会减1，CountDownLatch的await方法会阻塞当前线程，直到N变成0。**N为0后，tryAcquireShared返回1，然后就从自旋中返回了。主线程就可以继续执行后面的任务。**由于countDown方法可以用在任何地方，所以这里说的是N个点，可以是N个线程，也可以是1个线程里的N个执行步骤。**每调用一次countDown就是执行一次tryReleaseShared，也就是把count减一**。用在多线程时，只需要把这个CountDownLatch的引用传递到线程里即可。

```java
public class CountDownLatchTest {
    static CountDownLatch countDownLatch = new CountDownLatch(2);

    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            System.out.println(1);
            countDownLatch.countDown();
            System.out.println(2);
            countDownLatch.countDown();
        }).start();
        countDownLatch.await();
        System.out.println(3);
    }
}
```

**注意**

计数器必须大于等于0，只是等于0时候，计数器就是零，调用await方法时不会阻塞当前线程。CountDownLatch不可能重新初始化或者修改CountDownLatch对象的内部计数器的值。一个线程调用countDown方法happen-before，另外一个线程调用await方法。

## 2、同步屏障CyclicBarrier

主要思想是：让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会打开，所有被屏障阻塞的线程才会继续运行。

CyclicBarrier默认的构造方法是CyclicBarrier（int parties），其参数表示屏障拦截的线程数量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。

```java
public class CyclicBarrierTest {
    static CyclicBarrier c = new CyclicBarrier(2);

    public static void main(String[] args) {
        new Thread(() -> {
            try {
                c.await();//CyclicBarrier会在内部统计到达屏障的线程数
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
            System.out.println(1);
        }).start();
        try {
            c.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
        System.out.println(2);
    }
}

```

CyclicBarrier还提供一个更高级的构造函数CyclicBarrier（int parties，Runnable barrierAction），用于在线程到达屏障时，优先执行barrierAction，方便处理更复杂的业务场景

```java
public class CyclicBarrierTest {
    static CyclicBarrier c = new CyclicBarrier(2, () -> {
        System.out.println(3);
    });

    public static void main(String[] args) {
        new Thread(() -> {
            try {
                c.await();//CyclicBarrier会在内部统计到达屏障的线程数
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
            System.out.println(1);
        }).start();
        try {
            c.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
        System.out.println(2);
    }
    
}
```

因为CyclicBarrier设置了拦截线程的数量是2，所以必须等待代码中的第一个线程和主线程都到达屏障过后，这个屏障才会开启，开启过后首先会执行构造函数里面给定的操作，然后执行到达屏障的线程。

#### ==CountDownLatch和CyclicBarrier的区别==

CyclicBarrier是由Reentrant Lock来实现的，而CountDownLatch是直接继承AQS来实现的

CyclicBarrier和CyclicBarrier都是用于等待一组线程完成操作，但是CountDownLatch是：某个线程等待其他线程完成操作，然后该等待线程才能继续工作，等待的线程要显示调用await，而被等待的线程执行完过后要显示调用countDown。而CyclicBarrier是：所有的这些线程形成一组相互制约的关系，只有所有这些线程到达了屏障之后，屏障才开启，然后继续后续的操作；每个线程到达屏障后就调用await，表示它在等同组的其他线程到达。

CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset方法重置。所以CyclicBarrier能处理更为复杂的业务。例如，如果发生计算错误，可以重置计数器，并让线程重新执行一次。

## 3、控制线程并发数的Semaphore

Semaphore的构造方法Semaphore（int permits）接受一个整型的数字，表示可用的许可证数量。Semaphore（10）表示允许10个线程获取许可证，也就是最大并发数是10。Semaphore的用法也很简单，首先线程使用Semaphore的acquire()方法获取一个许可证，使用完之后调用release()方法归还许可证。还可以用tryAcquire()方法尝试获取许可证。

## 4、线程间交换数据的Exchanger

Exchanger（交换者）是一个用于线程间协作的工具类。Exchanger用于进行线程间的数据交换。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这两个线程通过exchange方法交换数据，如果第一个线程先执行exchange()方法，它会一直等待第二个线程也执行exchange方法，当两个线程都到达同步点时，这两个线程就可以交换数据，将本线程生产出来的数据传递给对方。

# ==八、Java中的线程池==

合理使用线程池的好处：

1. 降低资源消耗：通过重复利用已经创建好的线程降低线程创建和销毁造成的消耗
2. 提高响应速度：当任务到达时，任务可以不需要等到线程创建就能立即执行
3. 提高线程的可管理性

## 1、线程池的实现原理

线程池的主要处理流程如下：

![image-20200420101506344](F:\Java书和视频\笔记\images\并发编程\33.png)

ThreadPoolExecutor执行流程：

![image-20200420101618440](F:\Java书和视频\笔记\images\并发编程\34.png)

ThreadPoolExecutor执行execute方法分下面四种情况：

1. 如果当前运行的线程少于corePoolSize，则创建新的线程来执行任务（执行这一步骤需要获取全局锁）
2. 如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue
3. 如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务（需要获取全局锁）
4. 如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将拒绝执行，并调用RejectedExecutionHandler.rejectedExecution()方法

## 2、合理配置线程池



## 3、ThreadPoolExecutor

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler);
```

- corePollSize：线程池的大小，表示有多少个线程可以在线程池中，即使它们都是空闲的。核心线程不会回收，即使没有任务执行也会保持空闲状态。
- maximumPollSize：最多可以有多少个线程存储在池中
- keepAliveTime：当线程的数量大于池的大小时，核心线程池之外的线程最多可以有多少的空闲时间来等待任务的到来，如果超过了这个时间，那么线程就会被关闭
- unit：keepAliveTime时间的单位
- workQueue：用于没有线程执行的任务
- threadFactory：执行器创建一个线程时所用的工厂
- handler：如果线程数到达了maximumPoolSize，并且队列也满了，那么下一个任务再到达就会使用handler来处理，叫作**饱和策略**

### 1. 线程池中的运行状态

- RUNNING

    可以接收新的任务或者处理在队列中的任务

- SHUTDOWN

    指调用了shutdown()方法，不再接受新的任务（不能往队列中放任务），**阻塞队列中的任务会执行完毕**

- STOP

    指调用了shutdownnow()方法，不再接收新的任务，同时抛弃阻塞队列里的所有任务，并中断正在运行的任务

- TIDING

    所有任务都执行完了，在调用shutdown或shutdownnow中都会尝试更新为这个状态

- TERMINATED

    终止状态，当执行terminated()后会更新这个状态

### 2. 初始化线程池时可以预先创建线程吗？

初始化线程池时默认不会预先创建线程，但是可以调用`prestartAllCoreThreads()`方法，即可预先创建出corePoolSize数量的核心线程

```java
public int prestartAllCoreThreads() {
    int n = 0;
    while (addWorker(null, true))//创建没有任务执行的线程，直到创建了corePoolSize个核心线程
        ++n;
    return n;
}
```

`prestartCoreThreads()`同样可以预先创建线程，只不过**该方法只会创建一条线程**

### 3. 线程池的核心线程能被回收吗?

默认情况下，核心线程在创建过后就不会被回收了。如果把corePoolSize设置为0，虽然可以直接不使用核心线程，而直接使用临时线程；但是，任务到达时会先进入阻塞队列，只有阻塞队列满了过后才能创建临时线程。如果阻塞队列为无界队列，那么这些任务永远无法得到执行，直到内存溢出。

但是可以设置`allowCoreThreadTimeOut`，来让核心线程在一段时间过后被回收。

getTask源码：

```java
/**
* Performs blocking or timed wait for a task, depending on
* current configuration settings, or returns null if this worker
* must exit because of any of:
* 1. There are more than maximumPoolSize workers (due to
*    a call to setMaximumPoolSize).
* 2. The pool is stopped.
* 3. The pool is shutdown and the queue is empty.
* 4. This worker timed out waiting for a task, and timed-out
*    workers are subject to termination (that is,
*    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
*    both before and after the timed wait, and if the queue is
*    non-empty, this worker is not the last thread in the pool.
*
* @return task, or null if the worker must exit, in which case
*         workerCount is decremented
*/
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        int wc = workerCountOf(c);
        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
            workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

这里timed非常关键，当我们设置`allowCoreThreadTimeOut`为ture，或者工作线程大于了核心线程池数量时，timed会被设为true。如果timed为true，会调用poll()方法从阻塞队列中获取任务，否则调用take()方法获取任务。

- poll(long timeout, TimeUnit unit)方法：从BlockingQueue取出一个任务，如果不能立即取出，则可以等待timeout参数的时间，如果超过这个时间还没有取出任务，就返回true
- take()：从BlockingQueue中取出一个任务，如果不能立即取出（队列为空），会阻塞等待直到有任务到达

因此，**当`allowCoreThreadTimeOut`为true时，就算此时是核心线程，线程调用poll方法获取任务，如果超过keepAliveTime时间，则返回null，timeOut = true，则getTask会返回null，线程中的runWorker方法会退出while循环，线程接下来会被回收。**

### 4. `execute()`方法源码

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
    * Proceed in 3 steps:
    *
    * 1. If fewer than corePoolSize threads are running, try to
    * start a new thread with the given command as its first
    * task.  The call to addWorker atomically checks runState and
    * workerCount, and so prevents false alarms that would add
    * threads when it shouldn't, by returning false.
    *
    * 2. If a task can be successfully queued, then we still need
    * to double-check whether we should have added a thread
    * (because existing ones died since last checking) or that
    * the pool shut down since entry into this method. So we
    * recheck state and if necessary roll back the enqueuing if
    * stopped, or start a new thread if there are none.
    *
    * 3. If we cannot queue task, then we try to add a new
    * thread.  If it fails, we know we are shut down or saturated
    * and so reject the task.
    */
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}

```

整个执行流程如下：

1. 获取当前线程池的状态
2. 如果当前线程数量小于corePoolSize，则创建一个新的线程来执行
3. 如果当前运行的线程数量等于corePoolSize，就将任务写入阻塞队列中
4. 如果能放入阻塞队列中，需要双重检查，再次确认线程状态；如果线程状态变为了非运行状态(有可能执行了shutdown)，就需要从阻塞队列中移除任务，并尝试判断线程是否全部执行完毕，同时执行拒绝策略
5. 如果当前线程池为空就新创建一个线程
6. 如果第三步不能将任务加入到队列中（队列满了），就需要创建一个临时线程来执行任务，如果已经不能创建临时线程了，就执行拒绝策略

### 5. 饱和策略

- Abort Policy：默认策略，新任务提交时直接抛出RejectedExecutionException，该异常可由调用者捕获。

- CallerRuns：将任务分配给当前执行execute方法线程来处理

- Discard：直接抛弃，任务不执行

- DiscardOldestPolicy：丢弃队列中最老的任务

    我们也可以实现RejectedExecutionHandler接口来自定义饱和策略

### 6. 关闭线程池

可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池。它们的原理是遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止。但是它们存在一定的区别，shutdownNow首先将线程池的状态设置成STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表，而shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。
只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true。当所有的任务都已关闭后，才表示线程池关闭成功，这时调用isTerminaed方法会返回true。至于应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，通常调用shutdown方法来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow方法。

### 7. 合理的配置线程池

任务的特性：

* 任务的性质：CPU密集型任务、IO密集型任务和混合型任务
* 任务的优先级：高、中和低
* 任务的执行时间：长、中和短
* 任务的依赖性：是否依赖其他系统资源，如数据库连接

CPU密集型的任务应配置尽可能小的线程池，如配置$N_{cpu}+1$个线程的线程池，因为如果配置了太多的线程，那么可能会造成大量的线程上下文切换，非常消耗资源。由于IO密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程，如$2*N_{cpu}$。

### 8. 线程池的监控

如果在系统中大量使用线程池，则有必要对线程池进行监控，方便在出现问题时，可以根据线程池的使用状况快速定位问题。可以通过线程池提供的参数进行监控，在监控线程池的时候可以使用以下属性
·taskCount：线程池需要执行的任务数量。
·completedTaskCount：线程池在运行过程中已完成的任务数量，小于或等于taskCount。
·largestPoolSize：线程池里曾经创建过的最大线程数量。通过这个数据可以知道线程池是否曾经满过。如该数值等于线程池的最大大小，则表示线程池曾经满过。
·getPoolSize：线程池的线程数量。如果线程池不销毁的话，线程池里的线程不会自动销毁，所以这个大小只增不减。
·getActiveCount：获取活动的线程数。
通过扩展线程池进行监控。可以通过继承线程池来自定义线程池，重写线程池的beforeExecute、afterExecute和terminated方法，也可以在任务执行前、执行后和线程池关闭前执
行一些代码来进行监控。例如，监控任务的平均执行时间、最大执行时间和最小执行时间等。

# 九、Executor框架

## 1、两级调度模型

JVM把Java线程一对一映射为一个操作系统线程；在上层，Java多线程程序通常把应用分解为若干个任务，然后使用用户及调度器Executor将这些任务映射为固定数量的线程；在底层，操作系统内核将这些线程映射到硬件处理器上。

![image-20200421083939551](F:\Java书和视频\笔记\images\并发编程\35.png)

## 2、Executor框架的结构与成员

### 2.1 Executor框架的结构

`Executor`主要包含下面三大部分：

* 任务：包括被执行任务需要实现的`Runnable`或`Callable`接口
* 任务的执行：包括任务执行机制的核心接口`Executor`，以及继承自`Executor`的`ExecutorService`接口。`Executor`接口有两个关键实现类实现了`ExecutorService`（`ThreadPoolExecutor`和`ScheduledThreadPoolExecutor`）
* 异步计算的结果：包括接口`Future`和实现`Future`接口的`FutureTask`类

![image-20200421084520319](F:\Java书和视频\笔记\images\并发编程\36.png)

* `Executor`是一个接口，它是`Executor`框架的核心，它将任务的提交与执行分离开
* `ThreadPoolExecutor`是线程池的核心实现类，用来执行被提交的任务
* `ScheduledThreadPoolExecutor`是一个实现类，可以在给定的延迟后运行命令，或者定期执行命令。
* `Future`接口和`FutureTask`实现类，代表异步计算的结果
* `Runnable`接口和`Callable`接口的实现类，都可以被`ThreadPoolExecutor`或`ScheduledThreadPoolExecutor`执行

### 2.2 Executor框架的成员

1. ThreadPoolExecutor

   ThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建三种类型的ThreadPoolExecutor：SingleThreadExecutor、FixedThreadPool、CacheThreadPool

   1. SingleThreadExecutor

      SingleThreadExecutor适用于需要保证顺序地执行各个任务；并且在任意时间点不会有多个县城是活动的应用场景

   2. FixedThreadPool

      创建的线程池具有固定的线程数。适用于为了满足资源管理的需求，而需要限制当前线程数量的应用场景，适用于负载比较重的服务器

   3. CachedThreadPool

      该线程池可以根据需要来创建线程，它是大小无界的线程池，适用于执行很多的短期异步任务的小程序，或者是负载较轻的服务器

# 十、一些理论问题

## 1、什么是线程安全性？

当多个线程访问一个类时，如果不用考虑这些线程在运行时环境下的调度和交替执行，并且不需要额外的同步及在调用方代码不必做其他的协调，这个类的行为仍然是正确的，那么称这个类是线程安全的。

对于线程安全的类的实例进行顺序或并发的一系列操作，都不会导致实例处于无效状态。

## 2、乐观锁和悲观锁

* 悲观锁

  ==**互斥同步**==属于一种悲观的并发策略，其总是认为只要不去做正确的同步措施（例如加锁），那就肯定会出现问题，无论共享的数据是否真的会出现竞争，它都会进行加锁，这将会导致用户态到和心态转换、维护锁计数器和检查是否有被阻塞的线程需要被唤醒等开销。

* 乐观锁

  乐观并发策略是指，不管风险，先进行操作，如果没有其他线程争用共享数据，那操作就直接成功了（CAS判断或其他）；如果共享的数据被争用，产生了冲突，那再进行其他的补偿策略，最常用的补偿措施就是==**不断地重试，直到出现没有竞争的共享数据为止**==。

  这种乐观并发策略的实现不再需要把线程阻塞挂起，因此这种同步操作被称为==**非阻塞同步**==，使用这种措施的代码通常也被称为无锁编程