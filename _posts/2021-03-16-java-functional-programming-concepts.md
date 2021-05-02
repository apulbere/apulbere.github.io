---
layout: post
title: Functional Programming Concepts in Java
tags: [java 16, functional programming]
---

With the rise of lambda expression and newer features, there is an increasing necessity to review some of the functional programming concepts and how are they applied in Java.

### Currying

Currying is the process of turning a function with multiple arguments into a function with fewer arguments. For example, we can turn an addition between three numbers into a single argument function:

```java
Function<Integer, Function<Integer, Function<Integer, Integer>>> abc = a -> b -> c -> a + b + c;

assert abc.apply(1).apply(2).apply(3) == 6;
```

Which is fun, but is it really useful? Definitely! Sometimes we don't want to repeat the same argument and cannot overload the method (to simulate default parameter):

```java
String buildUrl(String protocol, String base) {
    return protocol + "://" + base;
}
//...

Function<String, String> secureUrlFun = base -> buildUrl("https", base);
Function<String, String> unsecureUrlFun = base -> buildUrl("http", base);

assert secureUrlFun.apply("adrian.md").equals("https://adrian.md");
assert unsecureUrlFun.apply("adrian.md").equals("http://adrian.md");
```

### Higher-order function

```java
Function<String, String> buildSafeUrl() {
    return base -> buildUrl("https", base);
}
```

In the previous example, `buildUrl` is also called first-order while `buildSafeUrl` - higher-order function. The definition of later one says that it should have at least one function as an argument or return a function. This opens a whole new world of possibilities, like **lazy evaluation**, code reuse, and function composition.

Let's imagine we have the following structure of records / classes:

```java
record Plug(String type) {

}

record Engine(Plug plug) {

}

record Car(Engine engine) {
}
```
And we are given the classic boring task of extracting `type` from a `car` object. The problem is that we might have a null at any level. 

A naive engineer could slap an `Optional` and call it a day:

```java
String plugType = Optional.of(new Car(null))
                .map(Car::engine)
                .map(Engine::plug)
                .map(Plug::type)
                .orElse(null);

assert plugType == null;
```

Spoiler alert, it's wrong to use `Optional`. The only valid usage of `Optional` is as a return value in a method as confirmed by API note from JavaDoc. Not to mention that we didn't create disposable wrapper objects for null check before Java 8 and there is no reason whatsoever to do these days.

We can solve this task by creating a null safe (higher-order) function:

```java
interface NullSafeFunction<T, R> extends Function<T, R> {

    default <V> NullSafeFunction<T, V> andThen(Function<? super R, ? extends V> after) {
        return (T t) -> {
            R result = apply(t);
            if(result != null) {
                return after.apply(result);
            }
            return null;
        };
    }

    static <T> NullSafeFunction<T, T> identity() {
        return t -> t;
    }
}

//...

String nullPlugType = NullSafeFunction.<Car>identity()
        .andThen(Car::engine)
        .andThen(Engine::plug)
        .andThen(Plug::type)
        .apply(new Car(null));

assert nullPlugType == null;

String sparkPlugType = NullSafeFunction.<Car>identity()
        .andThen(Car::engine)
        .andThen(Engine::plug)
        .andThen(Plug::type)
        .apply(new Car(new Engine(new Plug("spark plug"))));

assert sparkPlugType.equals("spark plug");
```
Breaking down the example, we can see that `NullSafeFunction::andThen` returns a composed function that executes the caller first and then the argument.

_To Be Continued ..._