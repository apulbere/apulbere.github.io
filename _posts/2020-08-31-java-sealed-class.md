---
layout: post
title: An Introduction to Java Sealed Classes and Interfaces
tags: [jep-360, sealed class, sealed interface, java 15, preview feature]
---

In this article, we'll explore what a sealed type in Java is, how to define a class and interface as sealed and finally we'll go through a practical example. 

### Sealed class syntax

To have explicit control over the extensibility of a class we use identifier _sealed_ and then _permits_ to specify the set of classes allowed to extend:

```java

sealed class Advice permits BeforeAdvice, Interceptor, AfterAdvice {

}

non-sealed class BeforeAdvice extends Advice {

}

final class Interceptor extends Advice {

}

sealed class AfterAdvice extends Advice {

}

final class AfterReturningAdvice extends AfterAdvice {

}
```

![Sealed Advice Hierarchy Diagram]({{ site.baseurl }}/images/sealed-advice-diagram.png)

There are some basic rules to keep in mind:

* subclasses are required to specify _non-sealed_ if a class can be extended further, _final_ or _sealed_
* permitted subclasses must directly extend the sealed class
* _permits_ is optional if classes are found in the same source file
* permitted direct subclasses must reside in the same package if the superclass is in an unnamed module
* permitted direct subclasses can reside in different packages only when we have a named module, meaning we declare a _module-info.java_ with appropriate content

### Advantages of using sealed class or interface

Let's talk first about __exhaustiveness__ and why it might come in handy. 

If you watch closely the evolution of OpenJDK, you probably saw a couple of releases ago [switch expression](https://openjdk.java.net/jeps/325){:target="_blank"} and how it doesn't need anymore a _default_ case when dealing with enums:

```java
enum TrafficLight {
    RED,
    YELLOW,
    GREEN
}

String signal(TrafficLight trafficLight) {
    return switch (trafficLight) {
        case RED -> "stop";
        case GREEN -> "go";
    };
}
```

The above code snippet when compiled throws the error message _"switch expression does not cover all possible input values"_, because the values of _TrafficLight_ are known beforehand and the compiler detects that one case is not handled.

When it comes to inheritance and [pattern matching](https://cr.openjdk.java.net/~briangoetz/amber/pattern-match.html){:target="_blank"}, a feature which is due sometime in the next releases, we'll be able to get rid of traditional if-else chain and use a more elegant _switch expression_:

```java
switch (advice) {
    case BeforeAdvice ba -> ba.before();
    case Interceptor i -> i.invoke();
    case AfterAdvice aa -> aa.after();
};
```

instead of

```java
if (advice instanceof BeforeAdvice ba) {
    ba.before();
} else if (advice instanceof Interceptor i) {
    i.invoke();
} else if (advice instanceof AfterAdvice aa) {
    aa.after();
} else {
    throw new IllegalStateException("this is not possible");
}
```

Next advantage is about __restricting the use of a superclass or interface__. Some people already struggled with [similar](https://stackoverflow.com/questions/451182/stopping-inheritance-without-using-final){:target="_blank"} problem in the past and came with rather wacky solutions (e.g. _Do not inherit, please_).

It's especially useful when the API knows how to deal only with a set of predefined subclasses. Moreover, the developer doesn't have to do defensive programming, as we saw in the previous example.

### Restrictions

If an interface is declared as sealed we cannot use it as a lambda expression or create an anonymous class. Like _final_, _sealed_ property and its list of permitted subtypes, are defined in the class file so that it can be enforced at runtime:

```java
public sealed interface Period permits Hour, Day, Week, Month {

    void print();

    public static void main(String[] args) {
        //error: incompatible types: Period is not a functional interface
        Period period1 = () -> System.out.println("lambda period");

        //error: local classes must not extend sealed classes
        Period period2 = new Period() {
            @Override
            public void print() {
                System.out.println("anonymous class");
            }
        };
    }
}
```

### Example

Let's suppose we want to iterate over a list using the well-known head-tail method. Of course, it can be implemented in various ways, but for the sake of example we'll do it with the help of a sealed interface:

```java
import java.util.List;

public sealed interface Sequence<T> permits IterableOps, Nil {

    static <T> Sequence<T> of(T... values) {
        return new IterableOps<>(List.of(values));
    }
}
```
We need _Sequence_ to be sealed because, for this particular example, we'll always deal with only two cases: when there are some elements left in the list and when there are none (_Nil_), and we don't want someone to mess with this hierarchy by "accidentally" implementing the interface.

Next, we define _Nil_ which is a list of nothing.

```java
public final class Nil implements Sequence {
    public static final Nil NIL = new Nil();
}
```

Lastly, we define _IterableOps_ class:

```java
import java.util.Iterator;

public final class IterableOps<T> implements Sequence<T> {
    private final Iterator<T> iterator;

    protected IterableOps(Iterable<T> iterable) {
        this.iterator = iterable.iterator();
    }

    public T head() {
        return iterator.next();
    }

    public Sequence<T> tail() {
        return iterator.hasNext() ? this : Nil.NIL;
    }
}
```

In _main_ we'll create a list of strings and use it in a _for_ loop. Since pattern matching with switch is not yet available, will make use of the next best thing - enhanced _instanceof_ (JEP-305). The stop condition is when we hit _Nil_: 

```java
public static void main(String[] args) {
    for(var seq = Sequence.of("apples", "oranges", "grapes");
        seq instanceof IterableOps<String> itOps;
        seq = itOps.tail()) {

        System.out.println(itOps.head());
    }
}
```

It prints out

```
apples
oranges
grapes
```

### Conclusion 

Without a doubt, starting with Java 15 we'll give more consideration to whatever a class should be open for extension or sealed upon creation. The decision relies on the purpose of the class, be it code reuse, modeling various types that exist in a domain, or defending the integrity of an API.

#### Bibliography

* [OpenJDK. JEP 360: Sealed Classes (Preview)](http://openjdk.java.net/jeps/360){:target="_blank"}
* [Java Language Specification. Version 15](http://cr.openjdk.java.net/~iris/se/15/latestSpec/preview/sealed-classes-jls.html){:target="_blank"}
* [Java 15 Final Release Specification](http://cr.openjdk.java.net/~iris/se/15/latestSpec/java-se-15-annex-3.html){:target="_blank"}
* [Project Amber. Data Classes and Sealed Types for Java](https://cr.openjdk.java.net/~briangoetz/amber/datum.html){:target="_blank"}