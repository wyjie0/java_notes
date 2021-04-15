## 一、Lambda表达式

可以把Lambda表达式看作匿名功能，它基本上是没有声明名称的方法，但和匿名类一样，它也可以作为参数传递给一个方法：它没有名称，但有参数列表，函数主体，返回类型可能还有一个可以抛出的异常列表。Lambda表达式还可以存储在变量当中。

==只有在接受函数式编程接口的地方才可以使用Lambda表达式==

比如我们可以利用Lambda表达式简洁地自定义一个Comparator对象：

```java
(Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight());
```

从上面的形式可以看出，Lambda表达式由：==参数、箭头==和主体组成

其基本语法是：

```java
(parameters) -> expression;

或
    
(parameters) -> {statements;};
```

### 1、 函数式接口

函数式接口就是**只定义一个抽象方法**的接口。在JDK 1.8中，接口还可以拥有默认方法（在类没有对方法进行实现时，其主体为方法提供默认实现的方法）。哪怕有很多默认方法，只要接口只定义了一个抽象方法，它就是一个函数式接口

Lambda表达式允许我们直接以内联的形式为函数式接口的方法提供实现，并把整个表达式作为函数式接口的实例。

在Java 8以前，从文件中读取一行的模板代码如下：

```java
public class LambdaTest {
    public static String processFile() throws IOException {
        try (BufferedReader br = new BufferedReader(new FileReader("test"))){
            return br.readLine();
        }
    }
}
```

我们将上述代码改为Lambda表达式的形式：

1. 参数行为化：

   现在的代码有局限性。我们只能读取文件的一行，如果我们要返回头两行，甚至是返回使用最频繁的词，在理想状况下，我们要重用执行设置和清理的代码，并告诉`processFile`方法对文件执行不同的操作。也就是说，我们需要把`processFile`的行为参数化。我们需要把行为传递给`processFile`，以便它利用`BufferedReader`执行不同的行为

   下面就是从`BufferedReader`中打印两行的方法：

   ```java
   String result = processFile((BufferedReader br) -> br.readLine() + br.readLine());
   ```

2. 使用函数式接口来传递行为

   Lambda仅可使用于上下文是函数式接口的情况，我们需要创建一个能匹配`BufferedReader –> String`，还可以抛出`IOException`异常的接口，我们把这一接口称作`BufferedReaderProcessor`

   ```java
   @FunctionalInterface
   public interface BufferedReaderProcessor {
       String process(BufferedReader b) throws IOException;
   }
   ```

   现在就可以将这个接口作为新的`ProcessFile`方法的参数了：

   ```java
   public static String processFile(BufferedReaderProcessor processor) throws IOException {
   ```

3. 执行一个行为

   任何`BufferedReader -> String`形式的`Lambda`都可以作为参数来传递，因为它们符合`BufferedReaderProcessor`接口中定义的`process`方法的签名。现在我们只需要一种方法在`processFile`主体内执行`Lambda`所代表的代码。注意：`Lambda`表达式允许我们直接内联，为函数式接口的抽象方法提供实现，并且将整个表达式作为函数式接口的一个实例。因此，我们可以在`processFile`主体内，对得到的`BufferedReaderProcessor`对象调用`process`方法执行处理：

   ```java
       public static String processFile(BufferedReaderProcessor processor) throws IOException {
           try (BufferedReader br = new BufferedReader(new FileReader("test"))){
               return processor.process(br);
           }
       }
   ```

4. 传递Lambda表达式

   现在我们就可以通过传递不同的Lambda重用processFile方法，并以不同的方式处理文件

### 2、JDK中的函数式接口

#### 2.1 Predicate

`java.util.function.Predicate<T>`接口定义了一个名叫test的抽象方法，它接受泛型T对象，并返回一个`boolean`。在我们需要表示一个涉及类型T的布尔表达式时，就可以使用这个接口。

```java
public class PredicateTest {
    public static <T> List<T> filter(List<T> list, Predicate<T> p) {
        List<T> results = new ArrayList<>();

        for (T s : list) {
            if (p.test(s)) {
                results.add(s);
            }
        }
        return results;
    }



    public static void main(String[] args) {
        Predicate<String> nonEmptyStringPredicate = (String s) -> !s.isEmpty();
        List<String> nonEmpty = filter(Arrays.asList("123", "456", "", "789"), nonEmptyStringPredicate);
        System.out.println(nonEmpty);
    }
}
```



#### 2.2 Consumer

`java.util.function.Consumer<T>`定义了一个名叫accept的抽象方法，它接受泛型T的对象，没有返回值。如果我们需要访问类型T的对象，并对其执行某些操作，就可以使用这个接口。比如我们可以用它来创建一个`forEach`方法，接受一个Integers的列表，并对其中每个元素执行操作。

```java
public class ConsumerTest {
    
    public static <T> void forEach(List<T> list, Consumer<T> c) {
        for (T i : list) {
            c.accept(i);
        }
    }

    public static void main(String[] args) {
        forEach(Arrays.asList(1,2,3,4,5), (Integer i) -> System.out.println(i));//Lambda是Consumer中accept的实现
    }
    
}
```

#### 2.3 Function

`java.util.function.Function<T, R>`接口定义了一个叫作apply的方法，它接受一个泛型T的对象，并返回一个泛型R的对象。如果你需要定义一个Lambda，将输入对象的信息映射到输出，就可以使用这个接口（比如提取苹果的重量，或把字符串映射为它的长度）

```java
public class FunctionTest {
    public static <T, R> List<R> map(List<T> list, Function<T, R> f) {
        List<R> result = new ArrayList<R>();
        for (T s : list) {
            result.add(f.apply(s));
        }
        return result;
    }

    public static void main(String[] args) {
        List<Integer> l = map(Arrays.asList("lambda", "in", "action"),
                (String s) -> s.length());//Lambda是Function接口的apply方法的实现
    }
}
```

#### 2.4 Java 8中常用的函数式编程接口

![image-20200413112713905](F:\Java书和视频\笔记\images\java基础\14.png)

![image-20200413112733571](F:\Java书和视频\笔记\images\java基础\13.png)



### 3、方法引用

方法引用可以被看作仅仅调用特定方法的Lambda的一种快捷写法。它的基本思想是，如果一个Lambda代表的只是“直接调用这个方法”，那最好还是使用名称来调用它，而不是去描述如何调用它。事实上，方法调用就是让你根据已有的方法实现来创建Lambda表达式。显示地指明方法的名称，我们的代码的可读性可能会更好。

当我们使用方法引用时，目标引用放在分隔符`::`前，方法的名称放在后面。比如，`Apple::getWeight`，就是应用了`Apple`类中定义的方法`getWeight`。我们不需要添加括号，因为没有实际调用这个方法。方法引用就是`Lambda`表达式`(Apple a) -> a.getWeight()`的快捷写法。

![image-20200413103814298](F:\Java书和视频\笔记\images\java基础\9.png)

我们可以把方法引用看作针对仅仅涉及单一方法的Lambda的语法糖。

方法引用主要有三类：

1. 指向静态方法的方法引用（例如`Integer`的`parseInt`方法，写作：`Integer::parseInt`）

2. 指向任意类型实例方法的方法引用（例如`String`的`length`方法，写作：`String::length`）
3. 指向现有对象的实例方法的方法引用（假如我们有一个局部变量`expensiveTransaction`用于存放`Transaction`类型的对象，它支持实例方法`getValue`，那我们就可以写`expensiveTransaction::getValue`）

![image-20200413104359954](F:\Java书和视频\笔记\images\java基础\10.png)

![image-20200413104418754](F:\Java书和视频\笔记\images\java基础\11.png)

![image-20200413104434826](F:\Java书和视频\笔记\images\java基础\12.png)

#### 3.1 构造函数的引用

对于一个现有构造函数，我们可以利用它的名称和关键字new来创建它的一个引用：`ClassName::new`。它的功能与指向静态方法的引用类似。

### 4、Lambda和方法引用实战

用不同的排序策略给一个Apple列表排序

#### 4.1 传递代码

List的sort方法的签名如下：

```java
void sort(Comparator<? super E> c)
```

它需要一个`Compartor`对象来比较两个`Apple`。这就是在`Java`中传递策略的方式：它们必须包裹在一个对象里。我们说`sort`的行为被参数化了：传递给它的排序策略不同，其行为也会不同。

第一个解决方案如下：

```java
public class AppleComparator implements Comparator<Apple> {
    public int compare(Apple a1, Apple a2){
        return a1.getWeight().compareTo(a2.getWeight());
    }
}
inventory.sort(new AppleComparator());
```

#### 4.2 使用匿名类

我们可以使用匿名类来改进解决方案，而不是实现一个`Compartor`却只实例化一次：

```java
inventory.sort(new Comparator<Apple>() {
    public int compare(Apple a1, Apple a2){
        return a1.getWeight().compareTo(a2.getWeight());
    }
});
```

#### 4.3 使用Lambda表达式

在需要函数式接口的地方可以使用`Lambda`表达式。函数式接口就是仅仅定义了一个抽象方法的接口。抽象方法的签名（称为函数描述符）描述了`Lambda`表达式的签名。在这个例子里，`Compartor`代表了函数描述符`（T，T） -> int`。

改进后的解决方案为：

```java
inventory.sort((Apple a1, Apple a2)
               -> a1.getWeight().compareTo(a2.getWeight())
              );
```

Java编译器可以根据`Lambda`出现的上下文来推断`Lambda`表达式参数的类型，那么我们的方案可以重写为以下形式：

```java
inventory.sort((a1, a2) -> a1.getWeight().compareTo(a2.getWeight()));
```

`Comparator`具有一个叫做`comparing`的静态辅助方法，它可以接受一个`Function`来提取`Comparable`键值，并生成一个`Compartor`对象，它可以像下面这样使用：

```java
Comparator<Apple> c = Comparator.comparing((Apple a) -> a.getWeight());
```

我们再次修改以前的代码：

```java
import static java.util.Comparator.comparing;
inventory.sort(comparing((a) -> a.getWeight()));
```

#### 4.4 使用方法引用

我们可以使用方法引用让代码变得更加简洁：

```java
inventory.sort(comparing(Apple::getWeight));
```

## 二、引入流

### 1、流是什么

流允许我们以声明式方式处理集合（通过查询语句来表达，而不是临时编写一个实现）。这就是说，我们可以把流看作遍历数据集的高级迭代器。

下面两段代码都是用来返回低热量的菜名，并按照卡路里排序。一个是用java 7来编写，另一个使用java 8 来编写。

Java 7版本：

```java
List<Dish> lowCaloricDishes = new ArrayList<>();
for(Dish d: menu){
    if(d.getCalories() < 400){ ←─用累加器筛选元素
        lowCaloricDishes.add(d);
                             }
}
Collections.sort(lowCaloricDishes, new Comparator<Dish>() { ←─用匿名类对菜肴排序
    public int compare(Dish d1, Dish d2){
    return Integer.compare(d1.getCalories(), d2.getCalories());
}
                                                          });
List<String> lowCaloricDishesName = new ArrayList<>();
for(Dish d: lowCaloricDishes){
    lowCaloricDishesName.add(d.getName()); ←─处理排序后的菜名列表
}
```

在这段代码中，我们使用了一个垃圾变量`lowCaloricDishes`。它唯一的作用就是作为一次性的中间容器。

Java 8版本：

```java
import static java.util.Comparator.comparing;
import static java.util.stream.Collectors.toList;
List<String> lowCaloricDishesName =
    menu.stream()
    .filter(d -> d.getCalories() < 400) ←─选出400卡路里以下的菜肴
    .sorted(comparing(Dish::getCalories)) ←─按照卡路里排序
    .map(Dish::getName) ←─提取菜肴的名称
    .collect(toList()); ←─将所有名称保存在List中
```

为了利用多核架构并行执行这段代码，我们只需将`stream()`换成`parallelStream()`：

```java
List<String> lowCaloricDishesName =
    menu.parallelStream()
    .filter(d -> d.getCalories() < 400)
    .sorted(comparing(Dishes::getCalories))
    .map(Dish::getName)
    .collect(toList()); 
```

* 代码是以声明性方式写的：说明想要完成什么（筛选热量低的菜肴）而不是说明如何实现一个操作（利用循环和if条件等控制流语句）。这种方法加上行为参数化我们可以轻松应对变化的需求：很容易再创建一个代码版本，利用Lambda表达式来筛选高卡路里的菜肴，而用不着去复制粘贴代码
* 我们可以把几个基础操作连接起来，来表达复杂的数据处理流水线（在`filter`后面接上`sortted`、`map`和`collect`操作），同时保持代码清晰可读。`filter`的结果被传给了`sorted`方法，再传给`map`方法，最后传给``collect``方法

Java 8的Stream API的好处：

* 声明性——更简洁、更易读
* 可复合——更灵活
* 可并行——性能更好

### 2、流简介

流简单的定义是：**从支持数据处理操作的源生成的元素序列**

* 元素序列——就像集合一样，流也提供了一个接口，可以访问特定元素类型的一组有序值。因为集合是数据结构，所以它的目的在于特定的时间/空间复杂度存储和访问元素。但流的目的在于表达计算。
* 源——流会使用一个提供数据的源，如集合、数组或输入/输出资源。注意，**从有序集合生成的流会保持原有的顺序。由列表生成的流，其元素顺序和列表一致**
* 数据处理操作——流的数据处理功能支持类似于数据库的操作，以及函数式编程语言中的常用操作，如`filter、map、reduce、find、match、sort`等。流操作可顺序执行，也可并行执行

此外，流操作有两个特别的特点：

* 流水线——很多流操作本身会返回一个流，这样多个操作就可以链接起来，形成一个大的流水线。流水线的操作可以看作对数据源进行数据库式查询

* 内部迭代——与使用迭代器显示迭代的集合不同，流的迭代操作是在内部执行的。

  ```java
  import static java.util.stream.Collectors.toList;
  List<String> threeHighCaloricDishNames =
      menu.stream() ←─从menu获得流（菜肴列表）
      .filter(d -> d.getCalories() > 300) ←─建立操作流水线：首先选出高热量的菜肴
      .map(Dish::getName) ←─获取菜名
      .limit(3) ←─只选择头三个
      .collect(toList()); ←─将结果保存在另一个List中
      System.out.println(threeHighCaloricDishNames); ←─结果是[pork, beef,chicken]
  ```

  在本例中，我们先是对`menu`调用`stream`方法，由菜单得到一个流。数据源是菜肴列表（菜单），它给流提供一个元素序列。接下来，对流应用一系列数据处理操作：`filter、map、limit`和`collect`。除了`collect`之外，所有这些操作都会返回另一个流，这样它们就可以接成一条流水线，于是就可以看作对源的一个查询。最后，`collect`操作开始处理流水线，并返回结果（它和别的操作不一样，因为它返回的不是流，在这里是一个`List`）。

  在调用`collect`之前，没有任何结果产生，实际上根本就没有从`menu`中选择元素。我们可以理解为：链中的方法正在排队等待，直到调用`collect`。

  * `filter`——接受`Lambda`，从流中排除某些元素。在本例中，通过传递`Lambda d->d.getCalories() > 300`，选择出热量高于`300`卡路里的菜
  * `map`——接受一个`Lambda`，将元素转换成其他形式或提取信息。在本例中，通过传递方法引用`Dish::getName`，相当于``Lambda d -> d.getName()`，提取每道菜的名字`
  * `limit`——截断流，使其元素不超过给定数量
  * `collect`——将流转换为其他形式。在本例中，流被转换成一个列表。

### 3、流与集合

流与集合之间的差异在于：

* 集合是一个内存中的数据结构，它包含数据结构中目前所有的值——集合中的每个元素都得先算出来才能添加到集合中。（我们可以往集合里加东西或者删东西，但是不管什么时候，集合中的每个元素都是存在内存中的，元素都得先算出来才能称为集合的一部分）
* 流则是在概念上固定的数据结构（我们不能添加或者删除元素），其元素是按需计算的

![image-20200414092350620](F:\Java书和视频\笔记\images\java基础\15.png)

另一个例子是用浏览器进行互联网搜索。假设你搜索的短语在Google或是网店里面有很多匹配项。你用不着等到所有结果和照片的集合下载完，而是得到一个流，里面有最好的10个或20个匹配项，还有一个按钮来查看下面10个或20个。当你作为消费者点击“下面10个”的时候，供应商就按需计算这些结果，然后再送回你的浏览器上显示。

#### 3.1 只能遍历一次

和迭代器类似，流只能遍历一次。遍历完之后，我们就说这个流已经被消费掉了。你可以从原始数据源那里再获得一个新的流来重新遍历一遍，就像迭代器一样（这里假设它是集合之类的可重复的源，如果是I/O通道就没戏了）

#### 3.2 外部迭代和内部迭代

使用Collection接口需要用户去做迭代（比如用for-each），这称为外部迭代。 相反，Streams库使用内部迭代——它帮你把迭代做了，还把得到的流值存在了某个地方，我们只要给出一个函数说要干什么就可以了

Stream库的内部迭代可以自动选择一种适合硬件的数据表示和并行实现，

### 4、流操作

流操作可分为中间操作和终端操作，可以连接起来的流操作称为中间操作，关闭流的操作称为终端操作。

![image-20200414094154653](F:\Java书和视频\笔记\images\java基础\16.png)

#### 4.1 中间操作

中间操作会返回一个流，这让多个操作可以连接起来形成一个查询。重要的是，除非流水线上触发一个终端操作，否则中间操作不会执行任何处理——它们很懒。这是因为中间操作一般都可以合并起来，在终端操作时一次性全部处理。

#### 4.2 终端操作

终端操作会从流的流水线生成结果。其结果是任何不是流的值，比如List、Integer，甚至void

#### 4.3 使用流

流的使用一般包括三件事：

* 一个数据源（如集合）来执行一个查询；
* 一个中间操作链，形成一条流的流水线；
* 一个终端操作，执行流水线，并能生成结果。

流的流水线背后的理念类似于构建器模式。在构建器模式中有一个调用链用来设置一套配置（对流来说这就是一个中间操作链），接着是调用built方法（对流来说就是终端操作）。

## 三、使用流

### 1、筛选和切片

#### 1.1 用谓词筛选

Streams接口支持filter方法。该操作会接受一个谓词（一个返回boolean的函数）作为参数，并返回一个包括所有符合谓词的元素的流

```java
List<Dish> vegetarianMenu = menu.stream()
    .filter(Dish::isVegetarian) ←─方法引用检查菜肴是否适合素食者
    .collect(toList());
```

#### 1.2 筛选各异的流

流还支持一个叫作`distinct`的方法，它会返回一个元素各异（根据流所生成元素的`hashCode`和`equals`方法实现）的流。例如，以下代码会筛选出列表中所有的偶数，并确保没有重复

```java
List<Integer> numbers = Arrays.asList(1, 2, 1, 3, 3, 2, 4);
numbers.stream()
    .filter(i -> i % 2 == 0)
    .distinct()
    .forEach(System.out::println);
```

#### 1.3 截断流

流支持`limit(n)`方法，该方法会返回一个不超过给定长度的流。所需的长度作为参数传递给`limit`。如果流是有序的，则最多会返回前`n`个元素。

请注意`limit`也可以用在无序流上，比如源是一个`Set`。这种情况下，`limit`的结果不会以任何顺序排列

#### 1.4 跳过元素

流还支持`skip(n)`方法，返回一个扔掉了前`n`个元素的流。如果流中元素不足`n`个，则返回一个空流。请注意，`limit(n)`和`skip(n)`是互补的！

```java
List<Dish> dishes = menu.stream()
    .filter(d -> d.getCalories() > 300)
    .skip(2)
    .collect(toList());
```

### 2、映射

一个非常常见的数据处理套路就是从某些对象中选择信息。比如在`SQL`里，你可以从表中选择一列。`Stream API`也通过`map`和`flatMap`方法提供了类似的工具。

#### 2.1 对流中每一个元素应用函数

流支持`map`方法，它会接受一个函数作为参数。这个函数会被应用到每个元素上，并将其映射成一个新的元素（使用映射一词，是因为它和转换类似，但其中的细微差别在于它是“创建一个新版本”而不是去“修改”）。例如，下面的代码把方法引用`Dish::getName`传给了`map`方法，来提取流中菜肴的名称：

```java
List<String> dishNames = menu.stream()
    .map(Dish::getName)
    .collect(toList());
```

#### 2.2 流的扁平化

你已经看到如何使用map方法返回列表中每个单词的长度了。让我们拓展一下：对于一张单词表，如何返回一张列表，列出里面各不相同的字符呢？例如，给定单词列表["Hello","World"]，你想要返回列表["H","e","l", "o","W","r","d"]。

我们可以使用`flatMap`来解决这个问题

1. 尝试使用`map`和`Arrays.stream()`

   首先，你需要一个字符流，而不是数组流。有一个叫作`Arrays.stream()`的方法可以接受一个数组并产生一个流，例如：

```java
String[] arrayOfWords = {"Goodbye", "World"};
Stream<String> streamOfwords = Arrays.stream(arrayOfWords);
```

​	把它用在前面的那个流水线里，看看会发生什么：

```java
words.stream()
    .map(word -> word.split("")) ←─将每个单词转换为由其字母构成的数组
    .map(Arrays::stream) ←─让每个数组变成一个单独的流
    .distinct()
    .collect(toList());
```

当前的解决方案仍然搞不定！这是因为，你现在得到的是一个流的列表（更准确地说是Stream<String>）！的确，你先是把每个单词转换成一个字母数组，然后把每个数组变成了一个独立的流。

2. 使用`flatMap`

   可以像下面这样使用`flatMap`来解决这个问题：

   ```java
   List<String> uniqueCharacters =
       words.stream()
       .map(w -> w.split("")) ←─将每个单词转换为由其字母构成的数组
       .flatMap(Arrays::stream) ←─将各个生成流扁平化为单个流
       .distinct()
       .collect(Collectors.toList());
   ```

   使用`flatMap`方法的效果是，各个数组并不是分别映射成一个流，而是映射成流的内容。所有使用`map(Arrays::stream)`时生成的单个流都被合并起来，即扁平化为一个流。

![image-20200414101834221](F:\Java书和视频\笔记\images\java基础\17.png)

`flatmap`方法让你把一个流中的每个值都换成另一个流，然后把所有的流连接起来成为一个流。

练习：

1. 给定两个数字列表，如何返回所有的数对呢？例如，给定列表[1, 2, 3]和列表[3, 4]，应该返回[(1, 3), (1, 4), (2, 3), (2, 4), (3, 3), (3, 4)]

   答案：你可以使用两个map来迭代这两个列表，并生成数对。但这样会返回一个`Stream<Stream<Integer[]>>`。你需要让生成的流扁平化，以得到一个`Stream<Integer[]>`

   ```java
   List<Integer> numbers1 = Arrays.asList(1, 2, 3);
   List<Integer> numbers2 = Arrays.asList(3, 4);
   List<int[]> pairs =
       numbers1.stream()
       .flatMap(i -> numbers2.stream()
                .map(j -> new int[]{i, j})
               )
       .collect(toList());
   ```

2. 如何扩展前一个例子，只返回总和能被3整除的数对呢？例如(2, 4)和(3, 3)是可以的。

   答案：你在前面看到了，`filter`可以配合谓词使用来筛选流中的元素。因为在`flatMap`操作后，你有了一个代表数对的`int[]`流，所以你只需要一个谓词来检查总和是否能被3整除就可以了：

   ```java
   List<Integer> numbers1 = Arrays.asList(1, 2, 3);
   List<Integer> numbers2 = Arrays.asList(3, 4);
   List<int[]> pairs =
       numbers1.stream()
       .flatMap(i ->
                numbers2.stream()
                .filter(j -> (i + j) % 3 == 0)
                .map(j -> new int[]{i, j})
               )
       .collect(toList());
   ```

### 3、查找和匹配

另一个常见的数据处理套路是看看数据集中的某些元素是否匹配一个给定的属性。`StreamAPI`通过`allMatch、anyMatch、noneMatch、findFirst和findAny`方法提供了这样的工具。

#### 3.1 检查谓词是否至少匹配一个元素

`anyMatch`方法可以回答“流中是否有一个元素能匹配给定的谓词”。比如，你可以用它来看看菜单里面是否有素食可选择：

```java
if(menu.stream().anyMatch(Dish::isVegetarian)){
    System.out.println("The menu is (somewhat) vegetarian friendly!!");
}
```

`anyMatch`方法返回一个`boolean`，因此是一个终端操作。

#### 3.2 检查谓词是否匹配所有元素

`allMatch`方法的工作原理和`anyMatch`类似，但它会看看流中的元素是否都能匹配给定的谓词。比如，你可以用它来看看菜品是否有利健康（即所有菜的热量都低于1000卡路里）：

```java
boolean isHealthy = menu.stream()
    .allMatch(d -> d.getCalories() < 1000);
```

#### 3.3 查找元素

`findAny`方法将返回当前流中的任意元素。它可以与其他流操作结合使用。比如，你可能想找到一道素食菜肴。你可以结合使用`filter`和`findAny`方法来实现这个查询：

```java
Optional<Dish> dish =
    menu.stream()
    .filter(Dish::isVegetarian)
    .findAny();
```

**Optional简介**

`Optional<T>类（java.util.Optional）`是一个容器类，代表一个值存在或不存在。在上面的代码中，`findAny`可能什么元素都没找到。`Java 8`的库设计人员引入了`Optional<T>`，这样就不用返回众所周知容易出问题的`null`了。

* `isPresent()`将在`Optional`包含值的时候返回`true`, 否则返回`false`。
* `ifPresent(Consumer<T> block)`会在值存在的时候执行给定的代码块。
* `T get()`会在值存在时返回值，否则抛出一个`NoSuchElement`异常。
* `T orElse(T other)`会在值存在时返回值，否则返回一个默认值

例如，在前面的代码中你需要显式地检查`Optional`对象中是否存在一道菜可以访问其名称：

```java
menu.stream()
    .filter(Dish::isVegetarian)
    .findAny() ←─返回一个Optional<Dish>
    .ifPresent(d -> System.out.println(d.getName()); ←─如果包含一个值就打印它，否则什么都不做
```

#### 3.4 查找第一个元素

有些流有一个出现顺序（`encounter order`）来指定流中项目出现的逻辑顺序（比如由List或排序好的数据列生成的流）。对于这种流，你可能想要找到第一个元素。为此有一个`findFirst`方法，它的工作方式类似于`findany`。例如，给定一个数字列表，下面的代码能找出第一个平方能被3整除的数：

```java
List<Integer> someNumbers = Arrays.asList(1, 2, 3, 4, 5);
Optional<Integer> firstSquareDivisibleByThree =
    someNumbers.stream()
    .map(x -> x * x)
    .filter(x -> x % 3 == 0)
    .findFirst(); // 9
```

### 4、归约

#### 4.1 元素求和

在我们研究如何使用reduce方法之前，先来看看如何使用for-each循环来对数字列表中的元素求和：

```java
int sum = 0;
for (int x : numbers) {
    sum += x;
}
```

numbers中的每个元素都用加法运算符反复迭代来得到结果。通过反复使用加法，你把一个数字列表归约成了一个数字。这段代码中有两个参数：

* 总和变量的初始值，在这里是0；
* 将列表中所有元素结合在一起的操作，在这里是+。

要是还能把所有的数字相乘，而不必去复制粘贴这段代码，岂不是很好？这正是reduce操作的用武之地，它对这种重复应用的模式做了抽象。你可以像下面这样对流中所有的元素求和：

```java
int sum = numbers.stream().reduce(0, (a, b) -> a + b);
```

reduce接收两个参数：

* 一个初始值，这里是0
* 一个`BinaryOperator<T>`来将两个元素结合起来产生一个新值，这里我们用的是`lambda (a, b) -> a + b。`

![image-20200414231022136](F:\Java书和视频\笔记\images\java基础\18.png)

#### 4.2 最大值和最小值

```java
Optional<Integer> max = numbers.stream().reduce(Integer::max);
```

```java
Optional<Integer> min = numbers.stream().reduce(Integer::min);
```



### 5、构建流

#### 5.1 由值创建流

可以使用静态方法`Stream.of`，通过显示值创建一个流。它可以接受任意数量的参数。例如，以下代码直接使用`Stream.of`创建了一个字符流，然后将字符串转换为大写，再一个个打印出来：

```java
Stream<String> stream = Stream.of("Java 8 ", "Lambdas ", "In ", "Action");
stream.map(String::toUpperCase).forEach(System.out::println);
```

还可以使用`empty`方法得到一个空流：

```java
Stream<String> emptyStream = Stream.empty();
```

#### 5.2 由数组创建流

可以使用静态方法`Arrays.stream`()从数组创建一个流。它接受一个数组作为参数。例如，可以将一个原始类型int的数组转换成一个`IntStream`：

```java
int[] numbers = {2, 3, 5, 7, 11, 13};
int sum = Arrays.stream(numbers).sum(); ←─总和是41
```

#### 5.3 由文件生成流

`Java`中用于处理文件等`I/O`操作的`NIO API`（非阻塞 I/O）已更新，以便利用`StreamAPI`。`java.nio.file.Files`中的很多静态方法都会返回一个流。例如，一个很有用的方法是`Files.lines`，它会返回一个由指定文件中的各行构成的字符串流。

#### 5.4 由函数生成流：创建无限流

Stream API提供了两个静态方法来从函数生成流：`Stream.iterate`和`Stream.generate`。这两个操作可以创建所谓的无限流：不像从固定集合创建的流那样有固定大小的流。由`iterate`和`generate`产生的流会用给定的函数按需创建值，因此可以无穷无尽地计算下去！一般来说，应该使用`limit(n)`来对这种流加以限制，以避免打印无穷多个值。