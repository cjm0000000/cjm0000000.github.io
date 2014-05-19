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

### Method and Constructor References

The above example code can be further simplified by utilizing static method references:

    <?prettify linenums=1?>
    Converter<String, Integer> converter = Integer::valueOf;
    Integer converted = converter.convert("123");
    System.out.println(converted);   // 123
    
ava 8 enables you to pass references of methods or constructors via the :: keyword. The above example shows how to reference a static method. But we can also reference object methods:

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
    
Let's see how the `::` keyword works for constructors. First we define an example bean with different constructors:

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
    
Next we specify a person factory interface to be used for creating new persons:

    <?prettify linenums=1?>
    interface PersonFactory<P extends Person> {
        P create(String firstName, String lastName);
    }
    
Instead of implementing the factory manually, we glue everything together via constructor references:

    <?prettify linenums=1?>
    PersonFactory<Person> personFactory = Person::new;
    Person person = personFactory.create("Peter", "Parker");

We create a reference to the Person constructor via `Person::new`. The Java compiler automatically chooses the right constructor by matching the signature of `PersonFactory.create`.
