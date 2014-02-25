---
layout: post
title: "java之Unsafe类学习"
description: ""
category: Java
tags: [native]
---
{% include JB/setup %}
最近在阅读Java并发组件源码，发现很多方法的底层都是涉及到一个叫`Unsafe`的类，为了弄清楚底层的原理，特意学习了下`sun.misc.Unsafe`类。

#### 如何获取Unsafe实例

    private static final Unsafe unsafe = Unsafe.getUnsafe();
    
源码里面很多像上面这样的初始化片段，如果直接copy过来放到自己的类里面用的话，发现无法实例化，会抛出一个`SecurityException`异常：

    Caused by: java.lang.SecurityException: Unsafe
    
原因是`getUnsafe`方法的源码只接受受信任的代码调用这个方法，代码如下：

    public static Unsafe getUnsafe() {
        Class cc = sun.reflect.Reflection.getCallerClass(2);
        if (cc.getClassLoader() != null)
            throw new SecurityException("Unsafe");
        return theUnsafe;
    }

从代码可以看出，当类加载器为空的时候，才能调用`getUnsafe`方法，也就是只能由`BootstrapClassLoader`加载的类才能调用`getUnsafe`方法。

从网上搜寻到一种方法获取`Unsafe`的实例，代码如下：

    Field uField = Unsafe.class.getDeclaredField("theUnsafe");
    uField.setAccessible(true);
    Unsafe unsafe = (Unsafe) uField.get(null);

#### Singleton模式的破解

`Unsafe`类有个方法可以实现初始化类：`allocateInstance`方法，下面给出代码测试：

    public class ClassTemplate {
      private int a;

      public ClassTemplate() {
        this.a = 10;
      }

      public String toString() {
        return "[" + this.a + "]";
      }
      
      public static void main(String[] args) throws InstantiationException, IllegalAccessException{
        // method 1
        ClassTemplate ct1 = new ClassTemplate();
        System.out.println(ct1);
        // method 2
        ClassTemplate ct2 = ClassTemplate.class.newInstance();
        System.out.println(ct2);
        // method 3
        ClassTemplate ct3 = (ClassTemplate) UnsafeClass.getUnsafe().allocateInstance(ClassTemplate.class);
        System.out.println(ct3);
      }
    }

运行这段代码的输出结果是这样的：

    [10]
    [10]
    [0]

用`allocateInstance`初始化的类，*看起来没有执行构造器里面的初始化代码*

我们修改下代码，在默认构造器上加一个入参，修改以后的代码片段如下：

    public ClassTemplate(int temp) {
        this.a = 10;
      }
    
    // method 1
    ClassTemplate ct1 = new ClassTemplate(1);
     
修改以后再次运行代码，这次报异常`java.lang.InstantiationException`，原因是通过反射初始化类的前提是要有一个不带参数的默认构造器。

再次修改代码，把下面这段注释掉：

    // method 2
    //ClassTemplate ct2 = ClassTemplate.class.newInstance();
    //System.out.println(ct2);
    
保存运行代码，这次没有报错，运行结果如下：

    [10]
    [0]
    
由此我猜测：*`allocateInstance`可能会通过某种途径初始化对象，即使类没有默认构造器*

这个结论有待查看`Unsafe`源码验证。