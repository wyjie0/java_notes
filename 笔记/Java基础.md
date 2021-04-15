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

内部类使得多继承的解决方案变得完整。

#### 静态内部类
静态内部类与非静态内部类的区别在于：非静态内部类在编译完成之后会隐含地保存着一个指向外围类的引用，而静态内部类没有，这就意味着：

* 静态内部类的创建不需要依赖于外围类
* 它不能使用任何外围类的非static成员变量和方法

静态内部类中可以存在静态资源。可以有`public static abstract class Demo`

#### 成员内部类
可以访问外部**所有资源**（包括private资源），但是**本身内部不可有静态属性**。因为，当我们在创建某个外围类的某个内部类对象的时候，内部类对象必定会捕获一个指向那个外围类的引用，只要我们在访问外围类成员时，就会用这个引用来选择外围类的成员。

但是**外围类要访问内部类的属性和方法则需要通过内部类实例来访问**。

应该注意两点：

* 成员内部类中不能有任何static的变量和方法
* 成员内部类是依附于外围类的，所以只有先创建了外围类才能够创建内部类

==推荐使用`getxxx()`来获取成员内部类，尤其是该内部类的构造函数无参时==

#### 局部内部类

局部内部类和成员内部类一样被编译，只是它的作用域发生了变化，它只能在该方法和属性中被使用，出了该方法和属性就会失效。

==不可被访问修饰符和static修饰，只能访问final变量和形参。==

* 局部静态内部类：在外部类的静态方法中
* 局部内部类：在外部类的一般方法中
#### 匿名内部类
* ==没有构造器，不能有任何静态资源，不能被static、访问修饰符修饰==

* **只能创建匿名内部类的一个实例**，不能够被重复使用匿名内部类的定义

* 创建的时候一定是在new的后面，而且这个类一定是要存在的，如果这个类不存在就会出现编译错误

* 匿名内部类不能是抽象的，它必须要实现继承的类或者实现的接口的所有抽象方法

* 当匿名内部类所在方法的形参需要被匿名内部类使用时，这个形参必须被final修饰

  ```java
  public class OuterClass {
      public void display(final String name,String age){
          class InnerClass{
              void display(){
                  System.out.println(name);
              }
          }
      }
  }
  ```

  为什么必须要为final的呢？

  在内部类编译成功后，它会产生一个class文件，该class文件与外部类并不是同一class文件，它仅仅只保留对外部类的引用。当外部类传入的参数需要被内部类调用时，从java程序的角度来看是直接被调用的，从上面代码也可看出，但是编译过后的代码如下：

  ```java
  public class OuterClass$InnerClass {
      public InnerClass(String name,String age){
          this.InnerClass$name = name;
          this.InnerClass$age = age;
      }
      
      
      public void display(){
          System.out.println(this.InnerClass$name + "----" + this.InnerClass$age );
      }
  }
  ```

  从上面的代码来看，==内部类并不是直接调用方法传递的参数，而是利用自身的构造器对传入的参数进行备份，自己内部方法调用的实际上是自己的属性，而不是外部方法传递进来的参数。==

  在内部类中的属性和外部方法中的参数两者从外表上看是同一个东西，但实际上却不是，所以它们两者是可以任意变化的，也就是说在内部类我对属性的改变并不会影响到外部的形参（成员内部类会影响）。所以为了保持参数的一致性，就规定使用final来避免形参的不改变。

  简单理解就是：拷贝引用，为了避免引用值发生变化，例如被外部类的方法修改等，而导致内部类得到的值不一致，于是用final来让该引用不可改变。因此，如果定义了一个匿名内部类，并且希望它使用一个其外部定义的参数，那么编译器会要求该参数引用时final的

* 虽然在匿名内部类中不能定义构造器，但是我们可以使用**代码块**来完成匿名内部类中实例的初始化工作。
#### 内部类的创建方式
* `Inner inner = outer.new Inner()`。在别的类中使用时需要导入内部类（比如：`import Outer.Inner`）
* 成员内部类：
  
    * `Outer.Inner inner = outer.new Inner()`
* 静态内部类：
  
    * `Outer.Inner inner = new Outer.Inner()`
    
    * 也可以之间创建静态内部类
    



## 五、重载与重写

#### 重载
Java允许重载任何方法，要完整的描述一个方法，需要指出方法名和参数类型，这就叫做方法的签名。而返回类型不属于签名的一部分，因此重载方法是**同名不同参**的方法

#### 重写
当父类中的方法对子类不适用，子类就需要提供一个新的方法来覆盖父类的方法既然是对父类方法的重写，那么就**==不能==去修改方法的签名**，只能对内部进行调整，而且不仅是签名不能改，方法返回类型以及访问权限也不能改。也就是说，**方法名、参数列表必须相同**，返回值范围**小于等于**父类，抛出的异常范围**小于等于**父类，访问修饰符的范围**大于等于**父类。

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

* 为什么HashMap是线程不安全的？

  ![image-20200404002627094](F:\Java书和视频\笔记\images\java基础\2.png)

如上图所示，假设有两个线程要进行``resize`操作，当线程1执行到`transfer()`方法的`Entry<K,V> next = e.next;`这一步时，如果线程1的时间片正好用完，此时线程1中的e执行[3,A]，next执行[7,B]；此时线程2被调度执行，并顺利完成了`resize`操作，这个时候[7,B]的next变为了[3,A]。此时线程1被重新调度运行，在它的执行过程中，又将[3,A]的next指向了[7,B]，然后[3,A]作为了头节点，这就已经形成了一个环。

#### 1.2 jdk 1.8.源码实现

##### 1) 存储结构

在jdk 1.8中也是采用数组加链表的存储方式，数组存放的是Node类型（和Entry相同），但是当单个链表的长度大于8时，会将链表转换成红黑树，减少搜索时间；如果红黑树中的节点小于6个，又转换回链表

```java
transient Node<K,V>[] table;

//1.8中存储的节点被称作Node，但是和1.7中的Entry是一样的
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}

//树节点类源码，也就是红黑树中的节点
//继承了LinkedHashMap中的Entry类
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }

    /**
         * Returns root of tree containing this node.
         */
    final TreeNode<K,V> root() {
        for (TreeNode<K,V> r = this, p;;) {
            if ((p = r.parent) == null)
                return r;
            r = p;
        }
    }
```

##### 2） put操作

该方法的思想为：

1. 如果定位到数组位置没有元素，就直接插入
2. 如果定位到数组位置有元素就要和插入的key作比较，如果key相同就直接覆盖；如果key不同，就判断p是否是一个树节点，如果是就调用`e=((TreeNode<K,V>)p).putTreeVal(this,tab,hash,key,value)`将元素添加进去，如果不是就遍历链表插入（尾插法）

```java
//put算法
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

//计算哈希
//此方法还是通过key来计算哈希，但是计算的具体方法与1.7不同
//该计算方法更简单，只进行了一次位运算，而1.7中进行了4次
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

/**
     * Implements Map.put and related methods.
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value 不覆盖存在的值
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
*/
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;//直接就是用Node[]数组来存储所有的桶了，没有segment数组了
    //判断table是否为空，或者长度是否为0，如果是就需要调用resize()方法进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //如果要插入的位置还没有其他节点插入，也就是说没有发生碰撞，那么就直接将元素放入该位置
    //i是节点应该存放的位置，tab是那个表，n是表的大小
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    //如果发生了碰撞，需要做更复杂的操作
    else {
        Node<K,V> e; K k;
        //如果要插入位置上的节点p与准备插入的节点键和值都相同，就先用一个变量e来保存节点p。后面的节点是否有相同的在第54行给出。处理方法在第61行给出
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //如果不相同，但p是树节点，就按树节点插入方式来插入这个节点
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //如果是链表节点
        else {
            for (int binCount = 0; ; ++binCount) {
                //如果插入位置的节点p后面没有节点，就直接将待插入节点加到p节点后面，如果插入过后数量大于了链表最大长度，则要将链表转换为红黑树。这里可以看出插入方法是尾插法，直到遍历到末尾都没有发现相同的节点，再插入
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //如果节点是相同的，直接退出
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //在这里处理相同节点
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //这里是先插入后扩容
    //如果插入后节点数量大于了阈值，就需要扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

//resize扩容
final Node<K,V>[] resize() {
    //先将现在的table用一个oldTab变量来保存
    Node<K,V>[] oldTab = table;
    //计算旧table的长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //保存旧的负载因子
    int oldThr = threshold;
    int newCap, newThr = 0;
    //如果旧的容量大于0
    if (oldCap > 0) {
        //如果旧table的容量已经大于了最大容量，就表示不能再继续扩容了，直接返回旧table
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //先设置扩容后的容量为旧容量的2倍，如果扩容后的容量小于设定的最大容量，并且旧的容量大于默认初始容量，就需要将新的负载因子也变为2倍，否则就不需要改变负载因子
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //如果旧的容量等于0，但是旧的负载因子大于0
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;//也就是使用resize方法扩容table为空或table长度为0时的table，这时扩容后的容量等于旧的负载因子
    else {               // zero initial threshold signifies using defaults
        //如果旧的table长度为0且旧的负载因子也为0，那么就使用默认的容量
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //这里从旧table把元素转移到新table中，因为旧的table可能为空，所以这里需要进行判断
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //遍历旧table中的元素，用e来保存，这时e就表示链表的头节点或红黑树的根节点
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                //如果在该位置上只有一个节点，那么就直接将这个节点放入到新的table中，而且没有在1.7中的专门计算下标值的方法，直接用一个表达式e.hash & (newCap - 1)计算下标值
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                //如果节点e是一个树节点，把以节点e为根节点的左右子树分开，传入的参数有新的table，e在旧table中的下标j，旧的容量，和this（指代正在使用的Map对象），在split方法中会调用其他方法来将树节点中的元素插入到新table中，在此不作展开，有点复杂
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //如果e.next不为空且e不是树节点，也就是说e为链表节点
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //如果(e.hash & oldCap) == 0，就用loHead来保存e，否则用hiHead来保存e
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);//这里是将以e作为头节点的链表又分为两个部分，区分的标准是(e.hash & oldCap)是否等于0（不清楚为什么这样做）
                    //loHead直接放在j处，j是以e为头节点的链表在旧table中的索引位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    //hiHead放在j+oldCap中,j的含义同上
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}

//newNode
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
    return new Node<>(hash, key, value, next);
}
```

### 2、ConcurrentHashMap

#### 2.1 jdk 1.7实现

##### 分段锁机制

ConcurrentHashMap在对象中保存了一个Segment数组，即将整个Hash表划分为多个分段；而每个Segment元素，即每个分段类似于一个HashTable；这样，在执行put操作时首先根据哈希算法定位到元素属于哪个Segment，然后对该Segment加锁即可。如下图所示：

![image-20200407081448277](F:\Java书和视频\笔记\images\java基础\5.png)

ConcurrentHashMap的高效并发机制又三个方面来保证：

1. 通过锁分段技术保证并发环境下的写操作
2. 通过HashEntry的不变性、Volatile变量的内存可见性和加锁重读机制保证高效、安全的读操作
3. 通过不加锁和加锁两种方案控制跨段操作的安全性

##### 成员变量定义

与HashMap相比，ConcurrentHashMap增加了两个属性用于定位段，分别是segmentMask和segmentShift。segmentMask的大小等于segments数组的大小减1，segmentShift大小等于32（hash值的位数）减去对segments的大小取以2为底的对数值。

定位方式：假设ConcurrentHashMap一共分为$2^n$个段，每个段有$2^m$个桶，那么段的定位方式是将key的hash值的高n位与($2^n - 1$)相与，在定位到某个段过后，再将key的hash值的低m位与$(2^m - 1)$相与，定位到具体的桶

##### 段的定义

Segment类继承于ReentrantLock类，从而使Segment对象能充当锁的角色。每个Segment对象用来守护它的成员变量table中包含的若干个桶。table是由HashEntry对象组成的链表数组，table数组中每一个数组成员就是一个桶。

在Segment类中，count变量是一个计数器，它表示每个Segment对象管理的table数组中包含的HashEntry对象的个数。**之所以每个Segment维护一个计数器，而不是在ConcurrentHashMap中使用全局的计数器，是因为当需要更新计数器时，不用锁定整个ConcurrentHashMap**。

##### HashEntry

HashEntry与Entry的不同之处在于，HashEntry内部的hash、key和next值都是final修饰的（1.7以后，next值不是final），value域被volatile修饰，因此HashEntry对象几乎是不可变的，这是ConcurrentHashMap读操作不需要加锁的一个重要原因。value域被volatile修饰，所以其可以确保被读线程读到最新的值。

与HashMap类似，在ConcurrentHashMap中，如果在散列时发生碰撞，也会将碰撞的HashEntry对象链成一个链表，由于HashEntry的next域是final的，所以只能采用头插法

##### 并发存取

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    //定位到特定的段。假设Segment的数量是2的n次方，那么segmentShift的值就是32-n，而segmentMask的值就是2^n-1（2进制为n个1）。所以，根据key的hash值的高n位就可确定元素到低在哪一个Segment中
    int j = (hash >>> segmentShift) & segmentMask;
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
                    //扩容
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
            if (next == null)   //  Single node on list
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                /*由于扩容是按照2的幂次方进行的，所以扩展前在同一个桶中的元素，现在要么还					是在原来的序号的桶里，或者就是原来的序号再加上一个2的幂次方，就这两种选				  择。根据本文前面对HashEntry的介绍，我们知道链接指针next是final的，因				   此看起来我们好像只能把该桶的HashEntry链中的每个节点复制到新的桶中(这意					 味着我们要重新创建每个节点)，但事实上JDK对其做了一定的优化。因为在理论					上原桶里的HashEntry链可能存在一条子链，这条子链上的节点都会被重哈希到					  同一个新的桶中，这样我们只要拿到该子链的头结点就可以直接把该子链放到新的					桶中，从而避免了一些节点不必要的创建，提升了一定的效率。因此，JDK为了提				   高效率，它会首先去查找这样的一个子链，而且这个子链的尾节点必须与原hash				  链的尾节点是同一个，那么就只需要把这个子链的头结点放到新的桶中，其后面跟					的一串子节点自然也就连接上了。对于这个子链头结点之前的结点，JDK会挨个遍				   历并把它们复制到新桶的链头(只能在表头插入元素)中*/
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
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

##### ConcurrentHashMap读操作不需要加锁的奥秘

* 用HashEntry对象的不变性来降低读操作对加锁的需求
* 用volatile变量协调读写线程间的内存可见性
* 若读时发生指令重排列现象，则加锁重读

##### ConcurrentHashMap跨段操作

在ConcurrentHashMap中，有些操作需要涉及到多个段，比如说size操作、containsValaue操作等。以size操作为例，如果我们要统计整个ConcurrentHashMap里元素的大小，那么就必须统计所有Segment里元素的大小后求和。我们知道，Segment里的全局变量count是一个volatile变量，那么在多线程场景下，我们是不是直接把所有Segment的count相加就可以得到整个ConcurrentHashMap大小了呢？显然不能，虽然相加时可以获取每个Segment的count的最新值，但是拿到之后可能累加前使用的count发生了变化，那么统计结果就不准了。所以最安全的做法，是在统计size的时候把所有Segment的put，remove和clean方法全部锁住，但是这种做法显然非常低效。

size方法主要思路是先在没有锁的情况下对所有段大小求和，这种求和策略最多执行RETRIES_BEFORE_LOCK次(默认是两次)：在没有达到RETRIES_BEFORE_LOCK之前，求和操作会不断尝试执行（这是因为遍历过程中可能有其它线程正在对已经遍历过的段进行结构性更新）；在超过RETRIES_BEFORE_LOCK之后，如果还不成功就在持有所有段锁的情况下再对所有段大小求和。事实上，在累加count操作过程中，之前累加过的count发生变化的几率非常小，所以ConcurrentHashMap的做法是先尝试RETRIES_BEFORE_LOCK次通过不锁住Segment的方式来统计各个Segment大小，如果统计的过程中，容器的count发生了变化，则再采用加锁的方式来统计所有Segment的大小。

　　那么，ConcurrentHashMap是如何判断在统计的时候容器的段发生了结构性更新了呢？我们在前文中已经知道，Segment包含一个modCount成员变量，在会引起段发生结构性改变的所有操作(put操作、 remove操作和clean操作)里，都会将变量modCount进行加1，因此，JDK只需要在统计size前后比较modCount是否发生变化就可以得知容器的大小是否发生变化。

#### 2.2 jdk 1.8实现

在1.7中查询遍历链表效率太低，因此，1.8在数据结构上做了调整：

![image-20200407110811428](F:\Java书和视频\笔记\images\java基础\6.png)

而且，抛弃了原有的Segment分段锁，而采用了CAS+Synchronized来保证并发安全性

##### put操作

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //判断是否需要进行初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //f 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //如果当前位置的 hashcode == MOVED == -1,则需要进行扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        //如果都不满足，则利用 synchronized 锁写入数据
        else {
            V oldVal = null;
            //使用synchronized来加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
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
                            //如果到最后都没有找到相同的元素，则在链表最后插入元素。（尾插法）
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

## 十、equals和hashcode方法



### 1、equals规范

#### 重写规则

`equals` 方法注重 **两个对象在逻辑上是否相等**。重写 `equals` 方法看似比较简单，但实际编写的时候还是要考虑具体类的意义。
一般来说，以下几种情况不需要重写 `equals` 方法：

- 一个类的每个实例在本质上都是独立的、不同的，比如 `Thread` 类
- 不需要 `equals` 方法，也就是判断对象相等的逻辑是没有必要的
- 父类已重写 `equals` 方法，并且子类的判断逻辑与父类相符
- 一个类的访问权限为 private 或 package-private，并且可以确定 `equals` 方法不会被调用

那么，另外的情况通常需要重写 `equals` 方法。一般来说，equals 方法遵循离散数学中的 **等价关系**（equivalence relation）。从 OOP 的角度来说，等价关系三原则如下：

- 自反性（Reflexive）：**一个对象与自身相等**，即 x=xx=x。对任何非空对象 x， `x.equals(x)` 必定为 true。
- 对称性（Symmetric）：**对象之间的等价关系是可交换的**，即 a=b⇔b=aa=b⇔b=a。对任何非空对象 x、y， `x.equals(y)` 为 true，当且仅当 `y.equals(x)` 为 true。
- 传递性（Transitive）：(a=b)∧(b=c)⇒(a=c)(a=b)∧(b=c)⇒(a=c)。对任何非空对象 x、y、z， 若 `x.equals(y)` 为 true 且 `y.equals(z)` 为 true，则 `x.equals(z)` 为 true。

除了遵循这三原则之外，还要遵循：

- 一致性（Consistent）：对任何非空对象 x、y，只要所比较的信息未变，则连续调用 `x.equals(y)` 总是得到一致的结果。这要求了 `equals` 所依赖的值必须是可靠的。
- 对任何非空对象 x， `x.equals(null)` 必定为 false。

所以，根据上面的原则，`equals` 函数的一个比较好的实践如下：

1. **首先先判断传入的对象与自身是否为同一对象**，如果是的话直接返回 `true`。这相当于一种性能优化，尤其是在各种比较操作代价高昂的时候，这种优化非常有效。
2. **判断对象是否为正确的类型**。若此方法接受子类，即子类判断等价的逻辑与父类相同，则可以用 `instanceof` 操作符；若逻辑不同，即仅接受当前类型，则可以用 `getClass` 方法获取 Class 对象来判断。注意使用 `getClass` 方法时必须保证非空，而用 `instanceof` 操作符则不用非空（`null instanceof o` 的值为 false）。
3. **将对象转换为相应的类型**：由于前面已经做过校验，因此这里做类型转换的时候不应当抛出 ClassCastException 异常。
4. 进行对应的判断逻辑

几个注意事项：

- 我们无法在扩展一个可实例化的类的同时，即增加新的成员变量，同时又保留原先的 `equals` 约定
- 注意不要写错 `equals` 方法的参数类型，标准的应该是 `public boolean equals(Object o)`，若写错就变成重载而不是重写了
- 不要让 `equals` 变得太“智能”而使其性能下降
- 如果重写了 `equals` 方法，则一定要重写 `hashCode` 方法（具体见下面）

#### 老生常谈的equals和==

上面提到过，`equals`方法用来判断 **两个对象在逻辑上是否相等**，而 `==` 用来判断两个引用对象是否指向同一个对象（是否为同一个对象）。

用烂了的例子，注意常量池：

```java
String str1 = "Fucking Scala";
String str2 = new String("Fucking Scala");
String str3 = new String("Fucking Scala");
String str4 = "Fucking Scala";
System.out.println(str1 == str2); // false
System.out.println(str2 == str3); // false
System.out.println(str2.equals(str3)); // true
System.out.println(str1 == str4); // true
str4 = "Fuck again!";
String str5 = "Fuck again!";
System.out.println(str1 == str4); // false
System.out.println(str4 == str5); // true
```

### 2、hashcode 方法

**==如果重写了equals方法，则一定要重写hashCode方法==**。

重写 `hashCode` 方法的原则如下：

- 在程序执行期间，只要equals方法的比较操作用到的信息没有被修改，那么对这同一个对象调用多次，hashCode方法必须始终如一地返回同一个整数
- 如果两个对象通过equals方法比较得到的结果是相等的，那么对这两个对象进行hashCode得到的值应该相同
- 两个不同的对象，hashCode的结果可能是相同的，这就是哈希表中的冲突。为了保证哈希表的效率，哈希算法应尽可能的避免冲突

关于相应的哈希算法，一个简单的算法如下:

- 永远不要让哈希算法返回一个常值，这时哈希表将退化成链表，查找时间复杂度也从 O(1)O(1) 退化到 O(N)O(N)
- 如果参数是boolean型，计算`(f ? 1 : 0)`
- 如果参数是byte, char, short或者int型，计算`(int) f`
- 如果参数是long型，计算`(int) (f ^ (f >>> 32))`
- 如果参数是float型，计算`Float.floatToIntBits(f)`
- 如果参数是double型，计算`Double.doubleToLongBits(f)`得到long类型的值，再根据公式计算出相应的hash值
- 如果参数是Object型，那么应计算其有用的成员变量的hash值，并按照下面的公式计算最终的hash值
- 如果参数是个数组，那么把数组中的每个值都当做单独的值，分别按照上面的方法单独计算hash值，最后按照下面的公式计算最终的hash值

组合公式：`result = 31 * result + c`

比如，String 类的 `hashCode` 方法如下（JDK 1.8）：

```java
public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
```

举个自定义类的 `hashCode` 例子：

```java
class Baz {
    private int id;
    private String name;
    private double weight;
    private float height;
    private String note;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        Baz baz = (Baz) o;

        if (id != baz.id) return false;
        if (Double.compare(baz.weight, weight) != 0) return false;
        if (Float.compare(baz.height, height) != 0) return false;
        if (name != null ? !name.equals(baz.name) : baz.name != null) return false;
        return !(note != null ? !note.equals(baz.note) : baz.note != null);

    }

    @Override
    public int hashCode() {
        int result;
        long temp;
        result = id;
        result = 31 * result + (name != null ? name.hashCode() : 0);
        temp = Double.doubleToLongBits(weight);
        result = 31 * result + (int) (temp ^ (temp >>> 32));
        result = 31 * result + (height != +0.0f ? Float.floatToIntBits(height) : 0);
        result = 31 * result + (note != null ? note.hashCode() : 0);
        return result;
    }
}
```

想找到一个不冲突的哈希算法不是非常容易，具体的属于数据结构部分知识，这里就不再赘述了。

## 十一、Class类型和反射机制

### 1、Class类型

Class类是Java用来表示运行时类型信息的类。Class类被创建后的对象就是Class对象，Class对象表示的是**自己手动编写类的类型信息**，比如创建了一个Shape类，那么JVM就会创建一个Shape对应Class类的Class对象，这个Class对象保存了Shape类相关的类型信息。

实际上，在Java中每个类都有一个Class对象，每当我们编写并且编译一个新创建的类，就会产生一个对应Class对象，并且这个Class对象会被保存在同名.class文件里（编译后的字节码文件保存的就是Class对象），为什么需要这样一个Class对象呢？

==当我们new一个新对象或者引用静态成员变量时，Java虚拟机中的类加载器子系统会将对应的Class对象加载到JVM中，然后JVM再根据这个类型信息相关的Class对象创建我们需要实例化的对象或提供静态变量的引用值。需要特别注意的是，手动编写的每个class类，无论创建多少个实例对象，在JVM中都只又一个Class对象，即在内存中每个类有且仅有一个相对应的Class对象==。

有三种方式来获得某个类的Class对象：

1. 使用Class.forName()方法，需要的参数为类的全限定名
2. 使用类对象的.getClass()方法
3. 使用Class字面量，也就是`类名.class.getClass()`。这个方法不会触发类的初始化，另外两个会触发

### 2、反射

#### 2.1 什么是反射？

反射时Java的特性之一，它允许运行中的Java程序获取自身的信息，并且==可以操作类或对象的内部属性==。通过反射，我们可以在运行时获得程序或程序集中每一个类型的成员和成员的信息。程序中一般的对象的类型都是在编译期就确定下来的，而 Java 反射机制可以动态地创建对象并调用其属性，这样的对象的类型在编译期是未知的。所以我们可以通过反射机制直接创建对象，即使这个对象的类型在编译期是未知的。

Java反射主要提供以下功能：

* 在**运行时**判断任意一个对象所属的类
* 在运行时构造任意一个类的对象
* 在运行时判断任意一个类所具有的成员变量和方法
* 在运行时调用任意一个对象的方法

#### 2.2 反射的用途

反射最重要的用途就是用来开发各种通用框架。很多框架（比如Spring）都是配置化的，为了保证框架的通用性，它们可能需要根据配置文件加载不同的对象或类，调用不同的方法，这个时候就必须使用到反射。

#### 2.3 反射的基本使用

##### 1. 通过反射来创建对象

通过反射来创建对象主要有两种方式：

* 使用Class对象的newInstance()方法来创建Class对象对应类的实例：

  ```java
  Class<?> c = String.class;
  Object str = c.newInstance();
  ```

* 先通过Class对象获取指定的Constructor对象，再调用Constructor对象的newInstance()方法来创建实例。这种方法可以用指定的构造器构造类的实例

  ```java
  //获取String所对应的Class对象
  Class<?> stringClass = String.class;
  //获取String类带一个String参数的构造器
  Constructor<?> constructor = stringClass.getConstructor(String.class);
  //根据构造器创建实例
  Object instance = constructor.newInstance("23333333");
  System.out.println(instance);
  ```

##### 2. 获取方法

获取某个Class对象的方法集合，有以下几种方式：

* `getDeclaredMethods`方法返回类或接口声明的所有方法，包括公共、保护、默认访问和私有方法，但不包括继承的方法：

  ```java
  public Method[] getDeclaredMethods() throws SecurityException
  ```

* `getMethods`方法返回某个类的所有共有方法，包括其父类的共有方法

  ```java
  public Method[] getMethods() throws SecurityException
  ```

* `getMethod`方法返回一个特定的方法，其中第一个参数为方法的名称，后面的参数为方法的参数对应的Class的对象

  ```java
  public Method getMethod(String name, Class<?>... parameterTypes)
  ```

##### 3. 获取类的成员变量的信息

* `getField`：访问共有的成员变量

* `getDeclaredField`：所有已声明的成员变量，但不能得到其父类的成员变量

  使用方法和上述类似

##### 4. 调用方法

当我们获取到一个方法后，我们可以调用`invoke()`方法来使用这个方法：

```java
public Object invoke(Object obj, Object... args)
    throws IllegalAccessException, IllegalArgumentException,
InvocationTargetException
```

##### 5. 利用反射创建数组

```java
public static void testArray() throws ClassNotFoundException {
    Class<?> aClass = Class.forName("java.lang.String");

    Object o = Array.newInstance(aClass, 25);
    Array.set(o, 0, "hello");
    Array.set(o, 1, "java");
    Array.set(o, 4, "hehe");
    System.out.println(Array.get(o, 4));
}
```

其中的`Array`类为`java.lang.reflect.Array`类，我们通过`Array.newInstance()`方法来创建数组对象，它的原型是：

```java
public static Object newInstance(Class<?> componentType, int length)
        throws NegativeArraySizeException {
        return newArray(componentType, length);
    }
```

#### 2.3 反射的一些注意事项

由于反射会造成一定的额外系统资源消耗，因此如果不需要动态地创建一个对象，那么就不要使用反射。

另外，反射调用方法时可以忽略权限检测，因此可能会破坏封装性而导致安全问题

### 3、深究反射的`invoke`方法

#### 3.1 invoke过程

invoke方法用来运行时动态调用某个对象的方法，它的实现如下：

```java
@CallerSensitive
public Object invoke(Object obj, Object... args)
    throws IllegalAccessException, IllegalArgumentException,
       InvocationTargetException
{
    if (!override) {
        if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            checkAccess(caller, clazz, obj, modifiers);
        }
    }
    MethodAccessor ma = methodAccessor;             // read volatile
    if (ma == null) {
        ma = acquireMethodAccessor();
    }
    return ma.invoke(obj, args);
}`
```

我们根据`invoke`的实现，将其分为以下几步：

##### 1. 权限检查

invoke方法会首先检查`AccessibleObject`的`override`属性的值。`AccessibleObject`类是`Field、Method和Constructor`的基类，它提供了 将反射的对象标记为在使用时取消默认`Java`语言访问控制检查 的能力。`override的值默认是false`，表示需要权限调用规则，调用方法时需要权限检查；我们也可以用`setAccessible`方法设置为`true`,若`override`的值为`true`，表示忽略权限规则，调用方法时无需检查权限（也就是说可以调用任意的private方法，违反了封装）。

如果`override`属性为默认值`false`，则进行进一步的权限检查：

(1)首先用`Reflection.quickCheckMemberAccess(clazz, modifiers)`方法检查方法是否为`public`，如果是的话跳过本步；如果不是`public`方法，那么用`Reflection.getCallerClass()`方法获取调用这个方法的`Class`对象，这是一个`native`方法

(2)获取了这个`Class`对象`caller`后用`checkAccess`方法做一次快速的权限检查

##### 2.调用`MethodAccessor`的`invoke`方法

`Method.invoke()`实际上并不是自己实现的反射调用逻辑，而是委托给`MethodAccessor`来处理。每个`Java`方法有且仅有一个`Method`对象作为`root`，它相当于跟对象，对用户不可见。当我们创建`Method`对象的时候，我们代码中获取到的`Method`对象实际上是它的副本（或引用）。`root`对象持有一个`MethodAccessor`对象，所以所有获取到的`Method`对象都共享这一个`MethodAccessor`对象，因此必须保证它在内存中的可见性。

![image-20200408152733504](F:\Java书和视频\笔记\images\java基础\7.png)

## 十二、Java动态代理

### 1、动态代理中的设计模式

**代理模式**

给某一个对象提供一个代理，并由代理对象来控制对真实对象的访问。

**抽象主题角色**

定义代理类和真实主题的公共对外接口，也是代理类代理真实主题的方法

**真实主题角色**

真正实现业务逻辑的类

**代理主题角色**

用来代理和封装真实主题

![image-20200408222656420](F:\Java书和视频\笔记\images\java基础\8.png)

### 2、代理分类

根据字节码的创建时期，可分为静态代理和动态代理

**静态代理**

在程序运行前就已经存在代理类的字节码文件，代理类和真实主题角色的关系在运行前就确定了

**动态代理**

动态代理的源码是在程序运行期间由JVM根据反射等机制动态生成的，所以在运行前并不存在代理类的字节码文件

#### 2.1 静态代理

```java
public interface UserService {
    public void select();   
    public void update();
}

public class UserServiceImpl implements UserService {  
    public void select() {  
        System.out.println("查询 selectById");
    }
    public void update() {
        System.out.println("更新 update");
    }
}

```

我们将通过静态代理对`UserServiceImpl`进行增强。写一个代理类`UserServiceProxy`，代理类需要实现`UserService`

```java
public class UserServiceProxy implements UserService {
    private UserService target; // 被代理的对象

    public UserServiceProxy(UserService target) {
        this.target = target;
    }
    public void select() {
        before();
        target.select();    // 这里才实际调用真实主题角色的方法
        after();
    }
    public void update() {
        before();
        target.update();    // 这里才实际调用真实主题角色的方法
        after();
    }

    private void before() {     // 在执行方法之前执行
        System.out.println(String.format("log start time [%s] ", new Date()));
    }
    private void after() {      // 在执行方法之后执行
        System.out.println(String.format("log end time [%s] ", new Date()));
    }
}
```

**静态代理的缺点**

从上面的代码可以看出：

1. 当需要多个类的时候，由于代理对象要实现与目标对象一致的接口，有两种方式：

   * 只维护一个代理类，让它实现多个接口，这样会导致**代理类过于庞大**

   * 实现多个代理类，每个目标对象对应一个代理类，**会产生过多的代理类**

2. 当接口需要增加、删除、修改方法的时候，目标对象和代理类都需要同时修改，**不易维护**

#### 2.2 动态代理

**为什么类可以动态生成？**

Java虚拟机类加载主要分为5个步骤：加载、验证、准备、解析、初始化。其中加载阶段需要完成下面三件事情：

1. 通过一个类的全限定名来获取定义此类的二进制字节流（.class文件）
2. 将这个字节流所代表的的静态存储结构转换为方法区的运行时数据结构
3. 在内存中生成一个代表这个类的`java.lang.Class`对象， 作为方法区这个类的各种数据访问入口

其中第一条获取类的二进制字节流就有很多途径：

* 从Zip包获取
* 从网络中获取
* 运行时计算，这种场景使用最多的是动态代理技术
* 由其他文件生成，比如JSP，又JSP生成对应的Class类
* 从数据库中获取

所以动态代理就是想办法，根据接口或目标对象，计算出代理类的字节码，然后再加载到JVM中使用

**动态代理两种实现方式**

1. 通过实现接口的方式：**JDK动态代理**
2. 通过继承类的方式：**cgLib柜台代理**

#### 2.3 JDK动态代理

主要涉及`java.lang.reflect.Proxy`和`java.lang.reflect.InvocationHandler`两个类

首先编写一个调用逻辑处理器`LogHandler`类，提供日志增强功能，并实现`InvocationHandler`接口；在`LogHandler`中维护一个目标对象，这个对象是被代理的对象（真实主题角色）；在`invoke`方法中编写方法调用的逻辑

```java
public class LogHandler implements InvocationHandler {
    //被代理的对象，实际的方法执行者
    Object target;
    
    public LogHandler(Object target) {
        this.target = target;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        
        before();
        Object result = method.invoke(target, args);//调用target的method方法
        after();
        return result;//返回方法的执行结果
    }
    
    public void before() {
        System.out.println(String.format("log start time [%s]", new Date()));
    }
    
    public void after() {
        System.out.println(String.format("log end time [%s]", new Date()));
    }
}

```

获取动态生成的代理类的对象必须借助`Proxy`类的`newInstance`()方法：

```java
public class Client {

    public static void main(String[] args) {
        //创建被代理类，UserService接口的实现类
        UserServiceImpl userService = new UserServiceImpl();
        //获取对应的ClassLoader
        ClassLoader classLoader = userService.getClass().getClassLoader();
        //获取所有接口的Class	
        Class<?>[] interfaces = userService.getClass().getInterfaces();
        //创建一个将传给代理类的调用请求处理器，处理所有代理对象上的方法调用
        InvocationHandler logHandler = new LogHandler(userService);
        //根据上面提供的信息，创建代理对象，在这个过程中：
        //  a.JDK会通过根据传入的参数信息动态地在内存中创建和.class 文件等同的字节码
        //  b.然后根据相应的字节码转换成对应的class
        //   c.然后调用newInstance()创建代理实例
        UserService proxyInstance = (UserService) Proxy.newProxyInstance(classLoader, interfaces, logHandler);
        
        proxyInstance.select();
        proxyInstance.update();
    }  
}
```

#### 2.4 cglib动态代理

首先创建一个Producer类作为被代理对象：

```java

/**
 * 生产者
 */
public class Producer{
    
    
    /**
     * 销售
     * @param money
     */
    public void saleProduct(float money) {
        System.out.println("拿到线，销售产品：" + money);
    }

    /**
     * 售后
     * @param money
     */
    public void afterService(float money) {
        System.out.println("提供售后服务，并拿到钱：" + money);
    }
}

```

创建代理对象：

```java
/**
 * 模拟一个消费者
 */
public class Client {

    public static void main(String[] args) {
        final Producer producer = new Producer();

        /**
         * 动态代理：
         *  特点：字节码随用随创建，随用随加载
         *  作用：不修改源码的基础上对方法增强
         *  分类：
         *      基于接口的动态代理
         *      基于子类的动态代理
         *  基于子类的动态代理
         *      涉及的类：Enhancer
         *      提供者：第三方cglib库
         *  如何创建代理对象：
         *      使用Enhancer类中的create方法
         *  创建代理对象的要求：
         *      被代理类不能是最终类
         *  create方法的参数：
         *      Class：字节码
         *          它是用于指定被代理对象的字节码
         *      Callback：用于提供增强的代码
         *          它是让我们写如何代理，我们一般写的都是该接口的子接口实现类：MethodInterceptor
         */
        Producer proxyProducer = (Producer) Enhancer.create(producer.getClass(), new MethodInterceptor() {
            /**
             * @param o
             * @param method
             * @param objects     
                以上三个参数和基于接口的动态代理中invoke方法的参数是一样的
             * @param methodProxy ：当前执行方法的代理对象
             * @return
             * @throws Throwable
             */
            @Override
            public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                Object returnValue = null;
                Float money = (Float) objects[0];
                if ("saleProduct".equals(method.getName())) {
                    returnValue = method.invoke(producer, money*0.8f);
                }
                return returnValue;
            }
        });
        proxyProducer.saleProduct(10000f);
    }
}
```

#### 2.5 JDK动态代理与CGLIB动态代理对比

JDK动态代理：基于Java反射机制实现，必须要实现了接口的业务类才能用这种方法生成代理对象

cglib动态代理：基于ASM机制实现，通过生成业务类的子类作为代理类

JDK动态代理的优势：

* 最小化依赖关系，减少依赖意味着简化开发和维护，JDK本身的支持，可能比cglib更可靠
* 平滑进行JDK版本升级，而字节码类库通常需要进行更新以保证在新版本Java上能够使用
* 代码实现简单

cglib动态代理的优势：

* 无需实现接口，达到代理类无侵入
* 只操作我们关系的类，而不必为其他相关类增加工作量
* 高性能

## 十三、泛型

[java泛型详解]: https://blog.csdn.net/s10461/article/details/53941091

### 1、概述

* 什么是泛型？为什么要使用泛型？

  泛型，即“参数化类型”。就是将类型由原来的具体的类型参数化，类似于方法中的变量参数，此时类型也定义成参数形式（可以称之为类型形参），然后在使用/调用时传入具体的类型（类型实参）

  泛型的本质是为了参数化类型（在不创建新的类型的情况下，通过泛型指定的不同类型来控制形参具体限制的类型）。也就是说，在泛型使用过程中，操作的数据类型被指定为一个参数这种参数类型可以用在类、接口和方法中，分别被称为泛型类、泛型接口、泛型方法



### 2、特性

泛型只在编译阶段有效，在编译过程中，编译器正确检验泛型结果后，会将泛型的相关信息擦除，并且在对象进入和离开方法的边界处添加类型检查和类型转换的方法，泛型信息不回进入到运行阶段。

## 十四、序列化

[Java序列化]: https://juejin.im/post/5ce3cdc8e51d45777b1a3cdf



### 1、序列化的含义、意义和使用场景

* 序列化：将对象写入到IO流中
* 反序列化：从IO流中恢复对象
* 意义：序列化机制允许将实现序列化的Java对象转换为字节序列，这些字节序列可以保存在磁盘上，或通过网络传输，以达到以后恢复成原来的对象。序列化机制使得对象可以脱离程序的运行而独立存在
* 使用场景：所有可在网络上传输的对象都必须是可序列化的；所有需要保存到磁盘的Java对象都必须是可序列化的

### 2、序列化实现的方式

#### 2.1 `Serializable`

##### 1. 普通序列化

`Serializable`接口是一个标记接口，不用实现任何方法。一旦实现了该接口，该类的对象就是可序列化的

1. 序列化的步骤：

* 步骤1：创建一个`ObjectOutputStream`输出流
* 步骤2：调用`ObjectOutputStream`对象的`writeObject`()方法输出可序列化的对象

2. 反序列化的步骤：

* 步骤1：创建一个`ObjectInputStream`输入流
* 步骤2：调用`ObjectOutputStream`对象的`readObject()`方法得到序列化的对象

##### 2. 成员是引用的序列化

如果一个可序列化的类的成员不是基本类型，也不是String类型，那这个引用类型也必须是可序列化的，否则会导致此类不能序列化

##### 3. 同一对象序列化多次的机制

对同一对象进行多次序列化，并不会将这个对象序列化多次

* Java序列化算法
  1. 所有保存到磁盘的对象都有一个序列化编码号
  2. 当程序试图序列化一个对象时，会先检查此对象是否已经被序列化过，只有此对象从来没有被序列化过（此虚拟机内），才会将此对象序列化为字节序列输出
  3. 如果此对象已经被序列化过，则直接输出编号即可

##### 4. java序列化算法潜在的问题

由于java序列化算法不会重复序列化一个对象，只会记录已序列化的编号。如果序列化一个可变对象（对象内的内容可更改）后，更改了对象内容，再次序列化时，并不会再次将此对象转换为字节序列，而只是保存序列化编号

##### 5. 可选的自定义序列化

可以使用`transient`关键字选择不需要序列化的字段

## 十五、Java注解

[java注解-最通俗易懂的讲解]: https://blog.csdn.net/qq1404510094/article/details/80577555
[深入理解Java注解类型(@Annotation)]: https://blog.csdn.net/javazejian/article/details/71860633



7. 

## 

