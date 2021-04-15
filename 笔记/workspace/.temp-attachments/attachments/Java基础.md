[TOC]




## 一、面向对象
#### Alan Kay对面向对象语言的基本特性的总结
1、 **万物皆对象**。将对象视为奇特的变量， 它可以存储数据，还可以要求它在自身上执行操作。理论上讲，可以抽取待求解问题的任何概念化构建（狗、建筑物、服务等），将其表示为程序中的对象
2、**程序时对象的集合，它们通过发送消息里告知彼此所要做的。**要想请求一个对象，就必须对该对象发送一条消息。更具体地说，可以把消息想象成对某个特定对象的方法的调用请求。
3、**每个对象都有自己的由其他对象所构成的存储**。可以动过创建包含现有对象的包的方式来创建新类型的对象。因此，可以在程序中构建复杂的体系，同时将其复杂性隐藏在对象的简单性背后。
4、**每个对象都拥有其类型**。每个对象都是某个类的一个实例。这个“类”就是“类型”的同义词。每个类最重要的区别于其他类的特性就是“可以发送什么样的消息给他”。
5、**某一特定类型的所有对象都可以接收同样的消息**。

**Booch对对象提出的简洁的描述：对象具有状态、行为和标识。这意味着每个对象都可以拥有内部数据（状态）和方法（行为），并且每一个对象都可以唯一地与其他对象区分开来，就是每一个对象在内存中都有一个唯一的地址。**

* * *

## 二、类初始化：

#### 1、虚拟机规定的立即对类进行初始化的情况

1、使用new关键字实例化对象时、读取或设置一个类的静态字段的时候，已经调用一个类的静态方法的时候（初始化在main方法之前完成）
2、使用`java.lang.reflect`包的方法对类进行反射调用的时候，如果类没有初始化，则需要先出发其初始化
3、当初始化一个类的时候，如果发现其父类没有被初始化就会先初始化它的父类
4、当虚拟机启动的时候，用户需要指定一个要执行的主类（包含`main()`方法的那个类），虚拟机会先初始化这个类
5、使用jdk1.7动态语言支持的时候的一些情况

**除了这5中情况，其他的所有引用类的方式都不会触发初始化，被称为被动引用。**

* 被动引用的三个例子：
  1、通过子类引用父类的静态字段，不会导致子类初始化
  2、通过数组定义来引用类，不会触发此类的初始化
```java
public class NotInitialization {    
    public static void main(String[] args) {            
        SuperClass[] sca = new SuperClass[10];         
    }  
}
```
3、常量在编译阶段会存入调用类的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化

```java
public class ConstClass {     
    static {         
        System.out.println("ConstClass init!");    
    }     
    public static final int value = 123; 
} 
public class NotInitialization{    
    public static void main(String[] args) {
        int x = ConstClass.value;     
    } 
} 
```
上述代码运行之后，没有输出“ConstClass init!”。这是因为虽然在Java源码中引用了ConstClass类中的常量value，但其实在编译阶段通过常量传播优化，已经将次常量的值存储到了NotInitialization类的常量池中，以后NotInitialization对常量ConstClass.value的引用实际都被转化为NotInitialization类对自身常量池的引用了。也就是说，实际上NotInitialization的Class文件之中并没有ConstClass类的符号引用入口，这两个类在编译成Class之后就不存在任何联系了

#### 2、Java类的初始化顺序

* 每个类的编译代码都存在于它自己的独立的文件中。该文件只在需要使用程序代码时才会被加载。但是，当访问static域或static方法时，也会被加载
* 在类的内部，变量定义的先后顺序决定了初始化的顺序。即使变量定义散布于方法定义之间，它们仍旧会**在任何方法（包括构造器）被调用之前得到初始化**。执行顺序为：变量初始化->构造器方法->普通方法。

* 构造器的调用顺序：
  1. 载入类后，初始化静态变量（包括静态代码块），先父类后子类
  2. 父类普通成员和非static块
  3. 父类构造函数
  4. 子类按照2、3步依次初始化
* 编写构造器的一条准则：用尽可能简单的方法使对象进入正常状态；如果可以的话，避免调用其他方法

## 三、垃圾回收

在java语言中，判断一块内存空间是否符合垃圾收集器收集的标准只有两个：
1、给对象赋值为null，以后没有调用过
2、给对象赋了新的值，重新分配了内存空间

* * *

## 四、内部类
#### 静态内部类
不可访问外部的非静态资源。可以有`public static abstract class Demo`
#### 成员内部类
可以访问外部**所有资源**，但是本身内部不可有静态属性
#### 局部内部类
不可被访问修饰符和static修饰，只能访问final变量和形参。
* 局部静态内部类：在外部类的静态方法中
* 局部内部类：在外部类的一般方法中
#### 匿名内部类
* 没有构造器，没有静态资源，无法被访问修饰符、static修饰
* **只能创建匿名内部类的一个实例**
* 创建的时候一定是在new的后面
#### 内部类的创建方式
* `Inner inner = outer.new Inner()`。在别的类中使用时需要导入内部类（比如：`import Outer.Inner`）
* 成员内部类：
  
    * `Outer.Inner inner = outer.new Inner()`
* 静态内部类：
  
    * `Outer.Inner inner = new Outer.Inner()`
    
      
    
      

## 五、重载与重写
#### 重载
Java允许重载任何方法，要完整的描述一个方法，需要指出方法名和参数类型，这就叫做方法的签名。而返回类型不属于签名的一部分，因此重载方法是**同名不同参**的方法

#### 重写
当父类中的方法对子类不适用，子类就需要提供一个新的方法来覆盖父类的方法
既然是对父类方法的重写，那么就不能去修改方法的签名，只能对内部进行调整，而且不仅是签名不能改，方法返回类型以及访问权限也不能改

## 六、Java线程状态转换

#### 1、 Java线程状态值

在Thread类源码中通过枚举为线程定义了6中状态值

```java
public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
```

```xml
NEW				---- 创建了线程对象但尚未调用start()方法时的状态
RUNNABLE		---- 线程对象调用start()方法后，线程处于可运行状态，此时线程等待获取CPU执行权
BLOCKED			---- 线程等待获取锁时的状态
WAITING			---- 线程处于等待状态，处于该状态表示当前线程需要等待其他线程做出一些特定的操作唤醒自己
TIME_WAITING	---- 超时等待状态，与WAITING不同，在等待指定的时间后会自行返回
TERMINATED		---- 终止状态，表示当前线程已执行完毕
```

#### 2、线程状态转换

<img src="F:\Java书和视频\笔记\images\Thread1.png" style="zoom:150%;" />

结合下图理解线程的状态转换就非常容易了

<img src="F:\Java书和视频\笔记\images\Thread2.png" style="zoom:150%;" />

<img src="F:\Java书和视频\笔记\images\Thread3.png" style="zoom:150%;" />

## 七、构造器

* 在构造器中可以用this调用另一个构造器，但却不能使用两次。

```java
public class Solution {
    int a, b;
    public Solution() {
        
    }
    public Solution(int a) {
        this();
        this.a = a;
        System.out.println("this调用无参构造器");
    }
    public Solution (int a, int b){
        this(a);
        this.b = b;
        System.out.println(a + b);
    }
}
```

## 八、编码方式

在Java语言中，中文字符所占的字节数取决于字符的编码方式，一般情况下，**采用ISO8859-1编码方式时，一个中文字符与一个英文字符相同只占1个字节**；**采用GB2312或GBK编码方是时，一个中文字符占2个字节**；而**采用UTF-8编码方式时，一个中文字符占3个字节**。

在Java中，**char和byte都是基础数据类型**，其中**byte占1个字节**，-128~127。而**char是16位，占两个字节**。

为什么java里的cha是2个字节？

因为**java内部都是用unicode**的，所以java其实是支持中文变量名的。

## 九、集合

### 1、 Map

Map家族的继承实现如下,Map与Collection之间是依赖关系

![image-20200315112838137](F:\Java书和视频\笔记\images\java基础\java1.png)

| Map集合类     | Key          | Value        |
| ------------- | ------------ | ------------ |
| HashMap       | 允许为null   | 允许为null   |
| TreeMap       | 不允许为null | 允许为null   |
| ConcurrentMap | 不允许为null | 不允许为null |

具体可看xmind思维导图

[java知识体系](F:\Java书和视频\笔记\Java知识体系.xmind )

#### 1.1 jdk 1.7源码实现

##### 1）存储结构

内部包含了一个Entry类型的数组table。Entry存储键值对，它包含了四个字段，Entry的实现形式是链表。即数组中的每一个位置被当做一个桶，每个桶用于存放一个链表。HashMap使用**拉链法**来解决冲突，同一个链表中存放哈希值和散列桶取模运算结果相同的Entry。这也就是为什么**hashcode()的值相同，但是对象不一定相同的原因**

```java
transient Entry<K,V>[] table;
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;
    int hash;

    /**
         * Creates new entry.
         */
    Entry(int h, K k, V v, Entry<K,V> n) {
        value = v;
        next = n;
        key = k;
        hash = h;
    }

    public final K getKey() {
        return key;
    }

    public final V getValue() {
        return value;
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry e = (Map.Entry)o;
        Object k1 = getKey();
        Object k2 = e.getKey();
        if (k1 == k2 || (k1 != null && k1.equals(k2))) {
            Object v1 = getValue();
            Object v2 = e.getValue();
            if (v1 == v2 || (v1 != null && v1.equals(v2)))
                return true;
        }
        return false;
    }

    public final int hashCode() {
        return (key==null   ? 0 : key.hashCode()) ^
            (value==null ? 0 : value.hashCode());
    }

    public final String toString() {
        return getKey() + "=" + getValue();
    }

    /**
         * This method is invoked whenever the value in an entry is
         * overwritten by an invocation of put(k,v) for a key k that's already
         * in the HashMap.
         */
    void recordAccess(HashMap<K,V> m) {
    }

    /**
         * This method is invoked whenever the entry is
         * removed from the table.
         */
    void recordRemoval(HashMap<K,V> m) {
    }
}
```

可以看到，在比较两个Entry是否相同的时候，即比较了键又比较了值。而且覆盖的hashcode()方法中，计算hashcode的方法为key^value。

##### 2）put操作

在向HashMap中添加键值对的时候，核心方法主要有：put,hash,addEntry,resize,indexFor,createEntry。put方法是核心方法，**它也是线程不安全的原因之一**

```java
//put
//这是我们调用的方法，传入要添加的键：key，和值：value
public V put(K key, V value) {
    if (key == null)
        //HashMap的键可以为null，键为null的Entry放在table的第一个位置
        return putForNullKey(value);
    //计算键的哈希值
    int hash = hash(key);
    //找到该哈希值在数组table中存放的位置，由此可见，HashMap是以键来确定Entry在数组中的位置的
    int i = indexFor(hash, table.length);
    //找到在table中存放的位置过后，查询是否有与这个key相同的键存在，如果存在则直接覆盖value
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        //哈希值相同，对象不一定相同，所以需要&&操作后面的语句
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            //覆盖value过后，返回被覆盖的value
            return oldValue;
        }
    }
	//每次有修改操作，modeCount都会自增
    modCount++;
    //如果没有找到与key相同的键存在，则调用addEntry()方法，传入的参数有key的哈希值，要存入的key，要存入的value，Entry在table中存放的位置i
    addEntry(hash, key, value, i);
    return null;
}

//hash
final int hash(Object k) {
    int h = 0;
    if (useAltHashing) {
        if (k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }
        h = hashSeed;
    }
	//^是异或运算，如果h=0，那么h的值就是k.hashCode()的值，如果自己没有定义，那hashCode()的计算方法就是Object类中的hashCode()计算方法
    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

//indexfor
//获得table中的存储位置
static int indexFor(int h, int length) {
    return h & (length-1);//这个值小于等于length-1
}

//addEntry
//hash是由key计算出来的
void addEntry(int hash, K key, V value, int bucketIndex) {
    //如果键值对的数量大于了阈值，并且在存放位置上遇到有碰撞的哈希值，那么就说明table已经存储得太满了，非常容易发生碰撞，此时就需要扩充table的容量至原来的两倍
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}

//resize
void resize(int newCapacity) {
    //把现在的table用一个叫做oldTable的变量保存起来
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    //如果现在table的大小已经是所设定的最大值，那table就不能再继续扩容了，就直接返回
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
	//新创建一个数组，旧table中的数据要放到新table中，需要看是否需要对每个元素重新做hash计算
    Entry[] newTable = new Entry[newCapacity];
    boolean oldAltHashing = useAltHashing;
    useAltHashing |= sun.misc.VM.isBooted() &&
        (newCapacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
    boolean rehash = oldAltHashing ^ useAltHashing;
    transfer(newTable, rehash);
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

//transfer
//这个方法造成了HashMap为线程不安全的
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    //遍历旧table中的每一个元素
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            //如果需要对元素重新做哈希运算，则先进行哈希运算
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            //找到元素在新table中的位置
            int i = indexFor(e.hash, newCapacity);
            //直接将旧table中的元素放入新table中，不创建新的Entry对象，这在多线程访问的时候会造成混乱，引发线程不安全性
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}

//createEntry
//采用头插法插入新的Entry(新的元素)
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```



#### 1.2 jdk 1.8.源码实现

