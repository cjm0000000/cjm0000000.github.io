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

#### Accessing fields and static variables

In constrast to local variables we have both read and write access to instance fields and static variables from within lambda expressions. This behaviour is well known from anonymous objects.

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
    
#### Accessing Default Interface Methods

Remember the formula example from the first section? Interface `Formula` defines a default method `sqrt` which can be accessed from each formula instance including anonymous objects. This does not work with lambda expressions.

Default methods **cannot** be accessed from within lambda expressions. The following code does not compile:

<?prettify?>
    Formula formula = (a) -> sqrt( a * 100);
    
### Built-in Functional Interfaces

The JDK 1.8 API contains many built-in functional interfaces. Some of them are well known from older versions of Java like `Comparator` or `Runnable`. Those existing interfaces are extended to enable Lambda support via the `@FunctionalInterface` annotation.

But the Java 8 API is also full of new functional interfaces to make your life easier. Some of those new interfaces are well known from the [Google Guava](https://code.google.com/p/guava-libraries/) library. Even if you're familiar with this library you should keep a close eye on how those interfaces are extended by some useful method extensions.

#### Predicates

Predicates are boolean-valued functions of one argument. The interface contains various default methods for composing predicates to complex logical terms (and, or, negate)

<?prettify linenums=1?>
    Predicate<String> predicate = (s) -> s.length() > 0;

    predicate.test("foo");              // true
    predicate.negate().test("foo");     // false

    Predicate<Boolean> nonNull = Objects::nonNull;
    Predicate<Boolean> isNull = Objects::isNull;

    Predicate<String> isEmpty = String::isEmpty;
    Predicate<String> isNotEmpty = isEmpty.negate();
    
#### Functions

Functions accept one argument and produce a result. Default methods can be used to chain multiple functions together (compose, andThen).

<?prettify linenums=1?>
    Function<String, Integer> toInteger = Integer::valueOf;
    Function<String, String> backToString = toInteger.andThen(String::valueOf);

    backToString.apply("123");     // "123"
    
#### Suppliers

Suppliers produce a result of a given generic type. Unlike Functions, Suppliers don't accept arguments.

<?prettify?>
    Supplier<Person> personSupplier = Person::new;
    personSupplier.get();   // new Person
    
#### Consumers

Consumers represents operations to be performed on a single input argument.

<?prettify?>
    Consumer<Person> greeter = (p) -> System.out.println("Hello, " + p.firstName);
    greeter.accept(new Person("Luke", "Skywalker"));
    
#### Comparators

Comparators are well known from older versions of Java. Java 8 adds various default methods to the interface.

<?prettify linenums=1?>
    Comparator<Person> comparator = (p1, p2) -> p1.firstName.compareTo(p2.firstName);

    Person p1 = new Person("John", "Doe");
    Person p2 = new Person("Alice", "Wonderland");

    comparator.compare(p1, p2);             // > 0
    comparator.reversed().compare(p1, p2);  // < 0
    
#### Optionals

Optionals are not functional interfaces, instead it's a nifty utility to prevent `NullPointerException`. It's an important concept for the next section, so let's have a quick look at how Optionals work.

Optional is a simple container for a value which may be null or non-null. Think of a method which may return a non-null result but sometimes return nothing. Instead of returning `null` you return an `Optional` in Java 8.

<?prettify linenums=1?>
    Optional<String> optional = Optional.of("bam");

    optional.isPresent();           // true
    optional.get();                 // "bam"
    optional.orElse("fallback");    // "bam"

    optional.ifPresent((s) -> System.out.println(s.charAt(0)));     // "b"