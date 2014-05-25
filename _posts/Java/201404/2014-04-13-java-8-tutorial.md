---
layout: post
title: "Java 8教程"
description: ""
category: Java8
tags: [Tutorial]
---
{% include JB/setup %}
本文译自[Java 8 Tutorial](http://winterbe.com/posts/2014/03/16/java-8-tutorial/)。

<?prettify ?>
	“Java还是没有死去——人们开始明白这一点。”

欢迎来到我对[Java 8](https://jdk8.java.net/)的介绍。本教程逐步指导你贯通所有新语言特性。通过简短的代码示例，你将学会如何使用默认接口方法，lambda表达式，方法引用和可重复的注解。在文章的最后，你将熟悉像流，函数式接口，map扩展和新的日期API等最新的[API](http://download.java.net/jdk8/docs/api/)改变。

没有墙的文字 —— 只是一堆注释的代码片段。享受吧！

### 接口的默认方法

Java 8 使我们能够利用`default`关键字添加非抽象方法实现到接口中。此功能也称为**扩展方法**。这是我们的第一个例子：

<?prettify linenums=1?>
    interface Formula {
       double calculate(int a);

       default double sqrt(int a) {
          return Math.sqrt(a);
       }
    }

除了抽象方法`calculate`，接口`Formula`也定义了默认方法`sqrt`。具体的类只需要实现抽象方法`calculate`。默认方法`sqrt`可以使用现成的。

<?prettify linenums=1?>
    Formula formula = new Formula() {
        @Override
        public double calculate(int a) {
            return sqrt(a * 100);
        }
    };

	formula.calculate(100);     // 100.0
	formula.sqrt(16);           // 4.0

formula作为一个匿名对象实现。代码是相当冗长：这样一个简单的`sqrt(a * 100)`计算需要6行代码。正如我们将在下一节看到的，在Java 8中有更好的方式实现单一方法的对象。

### Lambda 表达式

让我们从一个关于在Java的先前版本中如何将一个字符串列表排序的简单例子开始:

<?prettify linenums=1?>
    List<String> names = Arrays.asList("peter", "anna", "mike", "xenia");

    Collections.sort(names, new Comparator<String>() {
        @Override
        public int compare(String a, String b) {
            return b.compareTo(a);
        }
	});

静态工具方法`Collections.sort`接受一个列表和一个比较器以便对给定的列表中的元素排序。你经常会发现自己创建匿名比较器并且将他们传递给排序方法。

Java 8配备了更短的语法，**lambda表达式**，取代成天创建匿名对象：

<?prettify linenums=1?>
    Collections.sort(names, (String a, String b) -> {
        return b.compareTo(a);
    });
	
正如你所看到的，代码更短，更易于阅读。但它可以更短：

<?prettify?>
    Collections.sort(names, (String a, String b) -> b.compareTo(a));

对于一行的方法体可以跳过这两个大括号{}和return关键字。但它可以更短：

<?prettify?>
    Collections.sort(names, (a, b) -> b.compareTo(a));

Java编译器是知道参数类型的，所以你可以跳过它们。让我们更深入地研究lambda表达式如何在自然环境下使用。

### 功能接口

lambda表达式是如何融入Java的类型系统的？每个lambda对应于一个由接口指定的类型。所谓的功能接口必须**只包含一个抽象方法**声明。该类型的每个lambda表达式将被匹配到这个抽象方法。由于**default**方法不是抽象的，你可以自由地添加**default**方法到你的功能接口。

我们可以使用任意的接口作为lambda表达式，只要该接口只包含一个抽象方法。为了确保你的接口满足要求，你应该添加`@FunctionalInterface`注解。编译器知道此注解，一旦你尝试添加第二个抽象方法声明到这个接口，会抛出一个编译错误。

例子:

<?prettify linenums=1?>
    @FunctionalInterface
    interface Converter<F, T> {
        T convert(F from);
    }

<?prettify linenums=1?>
    Converter<String, Integer> converter = (from) -> Integer.valueOf(from);
    Integer converted = converter.convert("123");
    System.out.println(converted);    // 123

记住如果@FunctionalInterface注解被忽略了，代码还是有效的。

### 方法和构造器引用

上面的示例代码可以利用静态方法引用进一步简化：

<?prettify linenums=1?>
    Converter<String, Integer> converter = Integer::valueOf;
    Integer converted = converter.convert("123");
    System.out.println(converted);   // 123

Java 8使你能够通过`::`关键字传递方法或者构造器的引用。上面的例子演示了如何引用一个静态方法。但是我们也可以引用对象的方法：

<?prettify linenums=1?>
    class Something {
        String startsWith(String s) {
            return String.valueOf(s.charAt(0));
        }
    }
    
<?prettify linenums=1?>
    Something something = new Something();
    Converter<String, String> converter = something::startsWith;
    String converted = converter.convert("Java");
    System.out.println(converted);    // "J"

让我们看看`::`关键字是如何为构造器工作的。首先我们用不同构造器定义一个例子bean：

<?prettify linenums=1?>
    class Person {
        String firstName;
        String lastName;

        Person() {}

        Person(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }
    }

接下去我们指定一个person工厂用于创建新的人：

<?prettify linenums=1?>
    interface PersonFactory<P extends Person> {
        P create(String firstName, String lastName);
    }

我们通过构造器引用把所有的事情胶合起来，取代手工实现工厂：

<?prettify linenums=1?>
    PersonFactory<Person> personFactory = Person::new;
    Person person = personFactory.create("Peter", "Parker");

我们通过`Person::new`创建了一个到Person构造器的引用。Java编译器会自动匹配`PersonFactory.create`的签名选择合适的构造器。

### Lambda作用域

从lambda表达式访问外部变量的作用域和匿名对象很相似。你可以从局部外部作用域以及实例字段和静态变量访问final变量。

#### 访问局部变量

我们可以从lambda表达式的外部作用域读取final局部变量：

<?prettify linenums=1?>
    inal int num = 1;
    Converter<Integer, String> stringConverter = (from) -> String.valueOf(from + num);

    stringConverter.convert(2);     // 3

但是和匿名对象不同的是num变量不需要声明成final。这个代码也是有效的：

<?prettify linenums=1?>
    int num = 1;
    Converter<Integer, String> stringConverter = (from) -> String.valueOf(from + num);

    stringConverter.convert(2);     // 3

然而为了让代码通过编译，`num`变量必须被隐式声明为final。下面的代码**不能**编译：

<?prettify linenums=1?>
    int num = 1;
    Converter<Integer, String> stringConverter = (from) -> String.valueOf(from + num);
    num = 3;

从lambda表达式内部写入`num`也被禁止。

#### 访问字段和静态变量

和局部变量相比，我们都能从lambda表达式内部读写访问实例字段和静态变量。此行为在匿名对象中众所周知。

<?prettify linenums=1?>
    class Lambda4 {
        static int outerStaticNum;
        int outerNum;

        void testScopes() {
            Converter<Integer, String> stringConverter1 = (from) -> {
                outerNum = 23;
                return String.valueOf(from);
            };

            Converter<Integer, String> stringConverter2 = (from) -> {
                outerStaticNum = 72;
                return String.valueOf(from);
            };
        }
    }
    
#### 访问默认接口方法

还记得第一部分的formula例子吗？`Formula`接口定义了一个`sqrt`方法，它可以从formula的每个实例包括匿名对象来访问。这在lambda表达式中行不通。

默认方法**不能**从lambda表达式中访问。下面的代码无法编译：

<?prettify?>
    Formula formula = (a) -> sqrt( a * 100);
    
### 内置功能接口

在JDK1.8的API中包含许多内置功能接口。其中有些在老版Java中众所周知，比如 `Comparator` 或`Runnable`。这些现有的接口通过`@FunctionalInterface`注解的扩展支持lambda表达式。

但是Java8的API也充满了新的功能接口，使你开发更轻松。其中一些新的接口从[Google Guava](https://code.google.com/p/guava-libraries/)库中众所周知。即使你熟悉这个库，你也应该密切关注这些接口是如何由一些有用的方法扩展延伸的。

#### Predicate接口

Predicate是带一个参数返回布尔值的函数。该接口包含各种默认的方法组成predicates的复杂逻辑条件（and, or, negate）

<?prettify linenums=1?>
    Predicate<String> predicate = (s) -> s.length() > 0;

    predicate.test("foo");              // true
    predicate.negate().test("foo");     // false

    Predicate<Boolean> nonNull = Objects::nonNull;
    Predicate<Boolean> isNull = Objects::isNull;

    Predicate<String> isEmpty = String::isEmpty;
    Predicate<String> isNotEmpty = isEmpty.negate();
    
#### Function接口

Functions接受一个参数并产生一个结果。默认方法可以用来将多个函数连成一体（compose, andThen）。

<?prettify linenums=1?>
    Function<String, Integer> toInteger = Integer::valueOf;
    Function<String, String> backToString = toInteger.andThen(String::valueOf);

    backToString.apply("123");     // "123"
    
#### Supplier接口

Supplier产生一个给定的泛型类型的结果。和Functions不同的是Suppliers不接受参数。

<?prettify?>
    Supplier<Person> personSupplier = Person::new;
    personSupplier.get();   // new Person
    
#### Consumer接口

Consumers表示要在单一输入参数中执行操作。

<?prettify?>
    Consumer<Person> greeter = (p) -> System.out.println("Hello, " + p.firstName);
    greeter.accept(new Person("Luke", "Skywalker"));
    
#### Comparator接口

Comparators在老版本Java中众所周知。Java 8添加许多默认方法到这个接口。

<?prettify linenums=1?>
    Comparator<Person> comparator = (p1, p2) -> p1.firstName.compareTo(p2.firstName);

    Person p1 = new Person("John", "Doe");
    Person p2 = new Person("Alice", "Wonderland");

    comparator.compare(p1, p2);             // > 0
    comparator.reversed().compare(p1, p2);  // < 0
    
#### Optional接口

Optionals不是功能接口，相反，它是防止`NullPointerException`异常的极好工具。它是下一个部分的重要概念，所以让我们快速浏览下Optional是如何工作的。

Optional是一个存放一个值的简单容器，它可以为空或者不为空。想象一个方法，它可能返回一个非空结果，但是有时什么都不返回。在Java 8中你可以返回一个`Optional`而不是返回`null`。

<?prettify linenums=1?>
    Optional<String> optional = Optional.of("bam");

    optional.isPresent();           // true
    optional.get();                 // "bam"
    optional.orElse("fallback");    // "bam"

    optional.ifPresent((s) -> System.out.println(s.charAt(0)));     // "b"

### Streams

一个`java.util.Stream`表示在其上的一个或多个操作可以被执行的元素序列。流操作或是中间的或是终端的。当终端操作返回一个特定类型的结果时，中间操作返回流本身，所以你现在可以链入多个方法调用。流由源创建，比如像lists或者sets的一个`java.util.Collection`（不支持maps）。流操作可以被顺序或并行执行。

首先让我们来看看顺序流是如何工作的。首先我们从一个字符串列表创建一个简单的源：

<?prettify linenums=1?>
    List<String> stringCollection = new ArrayList<>();
    stringCollection.add("ddd2");
    stringCollection.add("aaa2");
    stringCollection.add("bbb1");
    stringCollection.add("aaa1");
    stringCollection.add("bbb3");
    stringCollection.add("ccc");
    stringCollection.add("bbb2");
    stringCollection.add("ddd1");

集合在Java8中被扩展了，所以你可以简单地通过调用`Collection.stream()`或`Collection.parallelStream()`来创建流。以下各节介绍最常见的流操作。

#### Filter

Filter accepts a predicate to filter all elements of the stream. This operation is _intermediate_ which enables us to call another stream operation (`forEach`) on the result. ForEach accepts a consumer to be executed for each element in the filtered stream. ForEach is a terminal operation. It's `void`, so we cannot call another stream operation.

<?prettify linenums=1?>
    stringCollection
        .stream()
        .filter((s) -> s.startsWith("a"))
        .forEach(System.out::println);

    // "aaa2", "aaa1"

#### Sorted

Sorted is an _intermediate_ operation which returns a sorted view of the stream. The elements are sorted in natural order unless you pass a custom `Comparator`.

<?prettify linenums=1?>
    stringCollection
        .stream()
        .sorted()
        .filter((s) -> s.startsWith("a"))
        .forEach(System.out::println);

    // "aaa1", "aaa2"

Keep in mind that `sorted` does only create a sorted view of the stream without manipulating the ordering of the backed collection. The ordering of `stringCollection` is untouched:

<?prettify?>
    System.out.println(stringCollection);
    // ddd2, aaa2, bbb1, aaa1, bbb3, ccc, bbb2, ddd1

#### Map

The _intermediate_ operation `map` converts each element into another object via the given function. The following example converts each string into an upper-cased string. But you can also use `map` to transform each object into another type. The generic type of the resulting stream depends on the generic type of the function you pass to `map`.

<?prettify linenums=1?>
    stringCollection
        .stream()
        .map(String::toUpperCase)
        .sorted((a, b) -> b.compareTo(a))
        .forEach(System.out::println);

    // "DDD2", "DDD1", "CCC", "BBB3", "BBB2", "AAA2", "AAA1"

#### Match

Various matching operations can be used to check whether a certain predicate matches the stream. All of those operations are terminal and return a boolean result.

<?prettify linenums=1?>
    boolean anyStartsWithA = stringCollection.stream()
            .anyMatch((s) -> s.startsWith("a"));

    System.out.println(anyStartsWithA);      // true

    boolean allStartsWithA = stringCollection.stream()
            .allMatch((s) -> s.startsWith("a"));

    System.out.println(allStartsWithA);      // false

    boolean noneStartsWithZ = stringCollection.stream()
            .noneMatch((s) -> s.startsWith("z"));

    System.out.println(noneStartsWithZ);      // true

#### Count

Count is a _terminal_ operation returning the number of elements in the stream as a `long`.

<?prettify linenums=1?>
    long startsWithB =
        stringCollection
            .stream()
            .filter((s) -> s.startsWith("b"))
            .count();

    System.out.println(startsWithB);    // 3

#### Reduce

This _terminal_ operation performs a reduction on the elements of the stream with the given function. The result is an `Optional` holding the reduced value.

<?prettify linenums=1?>
    Optional<String> reduced =
        stringCollection
            .stream()
            .sorted()
            .reduce((s1, s2) -> s1 + "#" + s2);

    reduced.ifPresent(System.out::println);
    // "aaa1#aaa2#bbb1#bbb2#bbb3#ccc#ddd1#ddd2"

### Parallel Streams

As mentioned above streams can be either sequential or parallel. Operations on sequential streams are performed on a single thread while operations on parallel streams are performed concurrent on multiple threads.

The following example demonstrates how easy it is to increase the performance by using parallel streams.

First we create a large list of unique elements:

<?prettify linenums=1?>
    int max = 1000000;
    List<String> values = new ArrayList<>(max);
    for (int i = 0; i < max; i++) {
        UUID uuid = UUID.randomUUID();
        values.add(uuid.toString());
    }

Now we measure the time it takes to sort a stream of this collection.

#### Sequential Sort

<?prettify linenums=1?>
    long t0 = System.nanoTime();

    long count = values.stream().sorted().count();
    System.out.println(count);

    long t1 = System.nanoTime();

    long millis = TimeUnit.NANOSECONDS.toMillis(t1 - t0);
    System.out.println(String.format("sequential sort took: %d ms", millis));

    // sequential sort took: 899 ms

#### Parallel Sort

<?prettify linenums=1?>
    long t0 = System.nanoTime();

    long count = values.parallelStream().sorted().count();
    System.out.println(count);

    long t1 = System.nanoTime();

    long millis = TimeUnit.NANOSECONDS.toMillis(t1 - t0);
    System.out.println(String.format("parallel sort took: %d ms", millis));

    // parallel sort took: 472 ms

As you can see both code snippets are almost identical but the parallel sort is roughly 50% faster. All you have to do is change `stream()` to `parallelStream()`.

### Map

As already mentioned maps don't support streams. Instead maps now support various new and useful methods for doing common tasks.

<?prettify linenums=1?>
    Map<Integer, String> map = new HashMap<>();

    for (int i = 0; i < 10; i++) {
        map.putIfAbsent(i, "val" + i);
    }

    map.forEach((id, val) -> System.out.println(val));

The above code should be self-explaining: `putIfAbsent` prevents us from writing additional if null checks; `forEach` accepts a consumer to perform operations for each value of the map.

This example shows how to compute code on the map by utilizing functions:

<?prettify linenums=1?>
    map.computeIfPresent(3, (num, val) -> val + num);
    map.get(3);             // val33

    map.computeIfPresent(9, (num, val) -> null);
    map.containsKey(9);     // false

    map.computeIfAbsent(23, num -> "val" + num);
    map.containsKey(23);    // true

    map.computeIfAbsent(3, num -> "bam");
    map.get(3);             // val33

Next, we learn how to remove entries for a a given key, only if it's currently mapped to a given value:

<?prettify linenums=1?>
    map.remove(3, "val3");
    map.get(3);             // val33

    map.remove(3, "val33");
    map.get(3);             // null

Another helpful method:

<?prettify?>
    map.getOrDefault(42, "not found");  // not found

Merging entries of a map is quite easy:

<?prettify linenums=1?>
    map.merge(9, "val9", (value, newValue) -> value.concat(newValue));
    map.get(9);             // val9

    map.merge(9, "concat", (value, newValue) -> value.concat(newValue));
    map.get(9);             // val9concat

Merge either put the key/value into the map if no entry for the key exists, or the merging function will be called to change the existing value.