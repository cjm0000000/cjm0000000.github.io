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

Filter接受一个predicate来筛选数据流中的所有元素。该操作是 _中间的_ ，使我们可以在结果上调用另一个流的操作（`forEach`）。ForEach接受一个consumer为每个在过滤流中的元素执行。ForEach是一个终端操作。它是`void`的，所以我们不能调用另一个流操作。

<?prettify linenums=1?>
    stringCollection
        .stream()
        .filter((s) -> s.startsWith("a"))
        .forEach(System.out::println);

    // "aaa2", "aaa1"

#### Sorted

Sorted是一个 _中间的_ 操作，它返回流的排序视图。元素是按自然顺序排序的，除非你传入一个自定义的`Comparator`。

<?prettify linenums=1?>
    stringCollection
        .stream()
        .sorted()
        .filter((s) -> s.startsWith("a"))
        .forEach(System.out::println);

    // "aaa1", "aaa2"

请记住，`sorted`仅仅是创建了流的排序视图而无需操控备份集合的顺序。`stringCollection`的顺序是没有改变的：

<?prettify?>
    System.out.println(stringCollection);
    // ddd2, aaa2, bbb1, aaa1, bbb3, ccc, bbb2, ddd1

#### Map

_中间_ 操作`map`通过给定的函数将每个元素转换到另一个对象。下面的例子把每个字符串转换成大写字符串。但是你也可以使用`map`把每个对象转换成另一种类型。所产生的数据流的泛型类型取决于你传递给`map`的函数的泛型类型。

<?prettify linenums=1?>
    stringCollection
        .stream()
        .map(String::toUpperCase)
        .sorted((a, b) -> b.compareTo(a))
        .forEach(System.out::println);

    // "DDD2", "DDD1", "CCC", "BBB3", "BBB2", "AAA2", "AAA1"

#### Match

大量的匹配操作，可用于检查某个predicate是否匹配流。所有的这些操作都是最终的并且返回一个布尔结果。

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

Count是一个以`long`类型返回流中元素的数目的 _最终的_ 操作。

<?prettify linenums=1?>
    long startsWithB =
        stringCollection
            .stream()
            .filter((s) -> s.startsWith("b"))
            .count();

    System.out.println(startsWithB);    // 3

#### Reduce

该 _终端_ 操作通过给定的函数在流的元素上执行一个reduction操作。其结果是一个`Optional`，其保持着合并后的值。

<?prettify linenums=1?>
    Optional<String> reduced =
        stringCollection
            .stream()
            .sorted()
            .reduce((s1, s2) -> s1 + "#" + s2);

    reduced.ifPresent(System.out::println);
    // "aaa1#aaa2#bbb1#bbb2#bbb3#ccc#ddd1#ddd2"

### 并行Streams

如上文提到的，数据流可以是串行的或并行的。操作在顺序流中都在一个单独的线程中执行，而操作在并行流中是在多个线程中并发地执行的。

下面的例子演示了使用并行数据流来提高性能是多么容易。

首先我们创建一个唯一元素的大列表：

<?prettify linenums=1?>
    int max = 1000000;
    List<String> values = new ArrayList<>(max);
    for (int i = 0; i < max; i++) {
        UUID uuid = UUID.randomUUID();
        values.add(uuid.toString());
    }

现在我们来测量排序这个集合流所花费的时间。

#### 顺序排序

<?prettify linenums=1?>
    long t0 = System.nanoTime();

    long count = values.stream().sorted().count();
    System.out.println(count);

    long t1 = System.nanoTime();

    long millis = TimeUnit.NANOSECONDS.toMillis(t1 - t0);
    System.out.println(String.format("sequential sort took: %d ms", millis));

    // sequential sort took: 899 ms

#### 并行排序

<?prettify linenums=1?>
    long t0 = System.nanoTime();

    long count = values.parallelStream().sorted().count();
    System.out.println(count);

    long t1 = System.nanoTime();

    long millis = TimeUnit.NANOSECONDS.toMillis(t1 - t0);
    System.out.println(String.format("parallel sort took: %d ms", millis));

    // parallel sort took: 472 ms

正如你可以看到这两个代码片段几乎是相同的，但是并行排序大约快了50%。你只需要把`stream()` 改成 `parallelStream()`。

### Map

正如已经提到的map不支持流。取而代之的时map现在支持做通用任务的各种新的和有用的方法。

<?prettify linenums=1?>
    Map<Integer, String> map = new HashMap<>();

    for (int i = 0; i < 10; i++) {
        map.putIfAbsent(i, "val" + i);
    }

    map.forEach((id, val) -> System.out.println(val));

上面的代码应该是不解自明的：`putIfAbsent`阻止我们编写额外代码做null检查；`forEach` 接受一个消费者对map的每一个值进行操作。

这个例子说明如何通过定制函数在map上计算代码：

<?prettify linenums=1?>
    map.computeIfPresent(3, (num, val) -> val + num);
    map.get(3);             // val33

    map.computeIfPresent(9, (num, val) -> null);
    map.containsKey(9);     // false

    map.computeIfAbsent(23, num -> "val" + num);
    map.containsKey(23);    // true

    map.computeIfAbsent(3, num -> "bam");
    map.get(3);             // val33

接下来，我们将学习如何对给定的键删除条目，只有当它的当前映射到给定的值：

<?prettify linenums=1?>
    map.remove(3, "val3");
    map.get(3);             // val33

    map.remove(3, "val33");
    map.get(3);             // null

另一个有用的方法：

<?prettify?>
    map.getOrDefault(42, "not found");  // not found

合并map的条目是很容易的：

<?prettify linenums=1?>
    map.merge(9, "val9", (value, newValue) -> value.concat(newValue));
    map.get(9);             // val9

    map.merge(9, "concat", (value, newValue) -> value.concat(newValue));
    map.get(9);             // val9concat

合并要么把键/值放入map，如果该键不存在任何条目，或者合并函数将被调用用来改变现有的值。

### Date API

Java 8 contains a brand new date and time API under the package `java.time`. The new Date API is comparable with the [Joda-Time](http://www.joda.org/joda-time/) library, however it's [not the same](http://blog.joda.org/2009/11/why-jsr-310-isn-joda-time_4941.html). The following examples cover the most important parts of this new API.

#### Clock

Clock provides access to the current date and time. Clocks are aware of a timezone and may be used instead of `System.currentTimeMillis()` to retrieve the current milliseconds. Such an instantaneous point on the time-line is also represented by the class `Instant`. Instants can be used to create legacy `java.util.Date` objects.

<?prettify linenums=1?>
    Clock clock = Clock.systemDefaultZone();
    long millis = clock.millis();

    Instant instant = clock.instant();
    Date legacyDate = Date.from(instant);   // legacy java.util.Date

#### Timezones

Timezones are represented by a `ZoneId`. They can easily be accessed via static factory methods. Timezones define the offsets which are important to convert between instants and local dates and times.

<?prettify linenums=1?>
    System.out.println(ZoneId.getAvailableZoneIds());
    // prints all available timezone ids

    ZoneId zone1 = ZoneId.of("Europe/Berlin");
    ZoneId zone2 = ZoneId.of("Brazil/East");
    System.out.println(zone1.getRules());
    System.out.println(zone2.getRules());

    // ZoneRules[currentStandardOffset=+01:00]
    // ZoneRules[currentStandardOffset=-03:00]

#### LocalTime

LocalTime represents a time without a timezone, e.g. 10pm or 17:30:15. The following example creates two local times for the timezones defined above. Then we compare both times and calculate the difference in hours and minutes between both times.

<?prettify linenums=1?>
    LocalTime now1 = LocalTime.now(zone1);
    LocalTime now2 = LocalTime.now(zone2);

    System.out.println(now1.isBefore(now2));  // false

    long hoursBetween = ChronoUnit.HOURS.between(now1, now2);
    long minutesBetween = ChronoUnit.MINUTES.between(now1, now2);

    System.out.println(hoursBetween);       // -3
    System.out.println(minutesBetween);     // -239

LocalTime comes with various factory method to simplify the creation of new instances, including parsing of time strings.

<?prettify linenums=1?>
    LocalTime late = LocalTime.of(23, 59, 59);
    System.out.println(late);       // 23:59:59

    DateTimeFormatter germanFormatter = DateTimeFormatter
        .ofLocalizedTime(FormatStyle.SHORT).withLocale(Locale.GERMAN);

    LocalTime leetTime = LocalTime.parse("13:37", germanFormatter);
    System.out.println(leetTime);   // 13:37

#### LocalDate

LocalDate represents a distinct date, e.g. 2014-03-11. It's immutable and works exactly analog to LocalTime. The sample demonstrates how to calculate new dates by adding or substracting days, months or years. Keep in mind that each manipulation returns a new instance.

<?prettify linenums=1?>
    LocalDate today = LocalDate.now();
    LocalDate tomorrow = today.plus(1, ChronoUnit.DAYS);
    LocalDate yesterday = tomorrow.minusDays(2);
    
    LocalDate independenceDay = LocalDate.of(2014, Month.JULY, 4);
    DayOfWeek dayOfWeek = independenceDay.getDayOfWeek();
    System.out.println(dayOfWeek);    // FRIDAY

Parsing a LocalDate from a string is just as simple as parsing a LocalTime:

<?prettify linenums=1?>
    DateTimeFormatter germanFormatter = DateTimeFormatter
            .ofLocalizedDate(FormatStyle.MEDIUM).withLocale(Locale.GERMAN);

    LocalDate xmas = LocalDate.parse("24.12.2014", germanFormatter);
    System.out.println(xmas);   // 2014-12-24

#### LocalDateTime

LocalDateTime represents a date-time. It combines date and time as seen in the above sections into one instance. `LocalDateTime` is immutable and works similar to LocalTime and LocalDate. We can utilize methods for retrieving certain fields from a date-time:

<?prettify linenums=1?>
    LocalDateTime sylvester = LocalDateTime.of(2014, Month.DECEMBER, 31, 23, 59, 59);

    DayOfWeek dayOfWeek = sylvester.getDayOfWeek();
    System.out.println(dayOfWeek);      // WEDNESDAY

    Month month = sylvester.getMonth();
    System.out.println(month);          // DECEMBER

    long minuteOfDay = sylvester.getLong(ChronoField.MINUTE_OF_DAY);
    System.out.println(minuteOfDay);    // 1439

With the additional information of a timezone it can be converted to an instant. Instants can easily be converted to legacy dates of type java.util.Date.

<?prettify linenums=1?>
    Instant instant = sylvester.atZone(ZoneId.systemDefault()).toInstant();

    Date legacyDate = Date.from(instant);
    System.out.println(legacyDate);     // Wed Dec 31 23:59:59 CET 2014


Formatting date-times works just like formatting dates or times. Instead of using pre-defined formats we can create formatters from custom patterns.

<?prettify linenums=1?>
    DateTimeFormatter formatter = DateTimeFormatter.ofPattern("MMM dd, yyyy - HH:mm");

    LocalDateTime parsed = LocalDateTime.parse("Nov 03, 2014 - 07:13", formatter);
    String string = formatter.format(parsed);
    System.out.println(string);     // Nov 03, 2014 - 07:13

Unlike `java.text.NumberFormat` the new `DateTimeFormatter` is immutable and **thread-safe**.

For details on the pattern syntax read [here](http://download.java.net/jdk8/docs/api/java/time/format/DateTimeFormatter.html).

### Annotations

Annotations in Java 8 are repeatable. Let's dive directly into an example to figure that out.

First, we define a wrapper annotation which holds an array of the actual annotations:

<?prettify linenums=1?>
    @interface Hints {
        Hint[] value();
    }

    @Repeatable(Hints.class)
    @interface Hint {
        String value();
    }

Java 8 enables us to use multiple annotations of the same type by declaring the annotation `@Repeatable`.

Variant 1: Using the container annotation (old school)

<?prettify linenums=1?>
    @Hints({@Hint("hint1"), @Hint("hint2")})
    class Person {}

Variant 2: Using repeatable annotations (new school)

<?prettify linenums=1?>
    @Hint("hint1")
    @Hint("hint2")
    class Person {}

Using variant 2 the java compiler implicitly sets up the @Hints annotation under the hood. That's important for reading annotation informations via reflection.

<?prettify linenums=1?>
    Hint hint = Person.class.getAnnotation(Hint.class);
    System.out.println(hint);                   // null

    Hints hints1 = Person.class.getAnnotation(Hints.class);
    System.out.println(hints1.value().length);  // 2

    Hint[] hints2 = Person.class.getAnnotationsByType(Hint.class);
    System.out.println(hints2.length);          // 2

Although we never declared the @Hints annotation on the Person class, it's still readable via getAnnotation(Hints.class). However, the more convenient method is getAnnotationsByType which grants direct access to all annotated @Hint annotations.

Furthermore the usage of annotations in Java 8 is expanded to two new targets:

<?prettify linenums=1?>
    @Target({ElementType.TYPE_PARAMETER, ElementType.TYPE_USE})
    @interface MyAnnotation {}

### That's it

My programming guide to Java 8 ends here. If you want to learn more about all the new classes and features of the JDK 8 API, just read my [follow up article](http://winterbe.com/posts/2014/03/29/jdk8-api-explorer/). It helps you figuring out all the new classes and hidden gems of JDK 8, like `Arrays.parallelSort`, `StampedLock` and `CompletableFuture` - just to name a few.

I recently published a [Java 8 Nashorn Tutorial](http://winterbe.com/posts/2014/04/05/java8-nashorn-tutorial/). The Nashorn Javascript Engine enables you to run javascript code natively on the JVM.

I hope this guide was helpful to you and you enjoyed reading it. The full source code of the tutorial samples is [hosted on GitHub](https://github.com/winterbe/java8-tutorial). Feel free to [fork the repository](https://github.com/winterbe/java8-tutorial/fork) or send me your feedback via [Twitter](https://twitter.com/benontherun).