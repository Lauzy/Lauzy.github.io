---
title: Java注解，安卓IOC
date: 2017-05-09 22:43:44
tags: Android 框架
toc: false
category: Android
---


## Java注解(Annotation)

Java注解，指的是代码里边的特殊标记，可以在编译、运行时被读取，并执行相应的处理。Annotation可用于修饰包、类、构造器、方法、变量等。

### Annotation 类型

#### 基本Annotation

Java中5个基本的注解分别为：
- @Override  ————  用来限定子类重写父类的方法。
- @Deprecated  ————  标记已经过时的方法。
- @SuppressWarnings  ————  抑制编译器的警告。
- @SafeVarargs  ————  Java7 抑制“堆污染”警告，可变参数与泛型类一起使用会出现类型安全警告，若开发人员确信不会出现问题，可用此注解进行声明。
- @FunctionalInterface  ————  Java8 函数式接口，检测指定某个接口中只有一个抽象方法。

#### 元Annotation

元Annotation是用来修饰其他注解定义，即注解其他注解。
Java中有6种元注解，其中@Native注解一般用于给IDE工具做提示用。下边具体介绍其他几种注解。

1、@Retention：指定本修饰的注解可以保留多长时间。包含了一个RetentionPolicy类的value值，所以需指定该value的值。

```java 
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Retention {
    RetentionPolicy value();
}
```
- RetentionPolicy.CLASS  ————  默认值，编译器将注解记录在字节码文件中，程序运行时，JVM不保留注解信息。
- RetentionPolicy.RUNTIME  ————  编译器将注解记录在字节码文件中，程序运行时，JVM可以获得注解信息，可通过反射获取该注解的信息。
- RetentionPolicy.SOURCE  ————  注解只保留在源代码中，编译器直接丢弃。

2、@Target：指定被修饰的注解能用于哪些程序元素。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target {
    ElementType[] value();
}
```

- ElementType.ANNOTATION_TYPE: 修饰Annotation。
- ElementType.TYPE: 修饰类、接口（包括注解类型）、枚举。
- ElementType.FIELD: 修饰成员变量。
- ElementType.METHOD: 修饰方法定义。
- ElementType.PARAMETER: 修饰参数定义。
- ElementType.CONSTRUCTOR: 修饰构造方法。
- ElementType.LOCAL_VARIABLE: 修饰局部变量。
- ElementType.PACKAGE: 修饰包定义。

3、@Documented：被该注解修饰的注解会被javadoc工具提取成文档。如果一个注解由@Documented修饰，则使用该注解的程序api文档中会包含该注解的说明。

4、@Inherited: 此注解修饰的注解具有继承性。若@XXX被@Inherited修饰，则使用@XXX注解的类具有继承性，其子类自动被@XXX修饰。