---
layout: post
title: The Definitive Guide to Java Sealed Classes and Interfaces
tags: [java, jep-360, sealed class, sealed interface, java 15, preview feature]
---

Java 15 introduces for the first time sealed classes and interfaces in order to restrict which other classes or interfaces may extend or implement them.

It comes as preview feature, though, which means that it is suspect to changes and can be activated only by providing the `enable-preview` command-line flag:

```bash
javac --enable-preview --source 15 com/j15/App.java
java --enable-preview com/j15/App 
```

### Sealed class syntax

Java 15 introduces a new keyword, ```non-sealed``` (yes, now we have [hyphenated keywords](https://openjdk.java.net/jeps/8223002){:target="_blank"}) and two restricted identifiers: ```sealed``` and ```permits```. Restricted because they are not allowed in some contexts.

So, in order to have explicit control over the extensibility of a class we use identifier ```sealed``` and then ```permits``` to specify the set of classes allowed to extend:

```java

sealed class Advice permits BeforeAdvice, Interceptor, AfterAdvice {}

non-sealed class BeforeAdvice extends Advice {}

final class Interceptor extends Advice {}

sealed class AfterAdvice extends Advice {}

final class AfterReturningAdvice extends AfterAdvice {}

```

![Sealed Advice Hierarchy Diagram]({{ site.baseurl }}/images/sealed-advice-diagram.png)

There are some basic rules to keep in mind:

* subclasses are required to specify ```non-sealed``` if class can be extended further, ```final``` or ```sealed```
* permited subclasses must directly extend the sealed class
* ```permits``` is optional if classes are found in the same source file
* permitted direct subclasses must reside in the same package if superclass is in unnamed module
* permitted direct subclasses can reside in different packages only when we have a named module, meaning we declare a ```module-info.java``` with appropriate content

### Advantages of using sealed class or interface

Let's talk first about __exhaustiveness__ and why it might come in handy. 

If you watch closely the evolution of OpenJDK, you probably saw a couple of releases ago [switch expression](https://openjdk.java.net/jeps/325){:target="_blank"} and how it doesn't need anymore a ```default``` case when dealing with enums:

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

The above code snippet when compiled throws the error message ```"switch expression does not cover all possible input values"```, because the values of ```TrafficLight``` are known beforehand and compiler detects that one case is not handled.

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

It's especially useful when the API knows how to deal only with a set of predefined subclasses. Moreover, the developer doesn't have to do defensive programming, like we saw in previous example.

### Restrictions

If an interface is declared as sealed we cannot use it as lambda expression or create an anonymous class. Like `final`, `sealed` property and its list of permitted subtypes, are defined in the classfile so that it can be enforced at runtime:

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

Also, be aware that you cannot mock a sealed type using Mockito framework (so far).

### Example

Let's suppose we want to iterate over a list using the well known head-tail method. Of course it can be implemented in various ways, but for the sake of example we'll do it with the help of a sealed interface:

```java
import java.util.List;

public sealed interface Sequence<T> permits IterableOps, Nil {

    static <T> Sequence<T> of(T... values) {
        return new IterableOps<>(List.of(values));
    }
}
```
We need `Sequence` to be sealed because, for this particular example, we'll always deal with only two cases: when there are some elements left in the list and when there are none (`Nil`), and we don't want someone to mess with this hierarchy by accidentally implementing the interface.

Next, we define `Nil` which is basically a list of nothing.

```java
public final class Nil implements Sequence {
    public static final Nil NIL = new Nil();
}
```

Lastly, we define `IterableOps` class:

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
In `main` we'll create a list of strings and use it in a `for` loop. Sadly, pattern matching with switch is not yet available, so will make use of JEP 305. The stop condition is when we hit `Nil`: 

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

Without a doubt, starting with Java 15 we'll give more consideration to whatever a class should be left open to extension or sealed upon creation. The decision relies on the purpose of the class, be it code reuse, modeling various types that exist in a domain or defending the integrity of an API.

#### Bibliography

* [OpenJDK. JEP 360: Sealed Classes (Preview)](http://openjdk.java.net/jeps/360){:target="_blank"}
* [Java Language Specification. Version 15](http://cr.openjdk.java.net/~iris/se/15/latestSpec/preview/sealed-classes-jls.html){:target="_blank"}
* [Java 15 Final Release Specification](http://cr.openjdk.java.net/~iris/se/15/latestSpec/java-se-15-annex-3.html){:target="_blank"}
* [Project Amber. Data Classes and Sealed Types for Java](https://cr.openjdk.java.net/~briangoetz/amber/datum.html){:target="_blank"}