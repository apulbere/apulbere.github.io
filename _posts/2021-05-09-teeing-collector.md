---
layout: post
title: How to collect more than a single collection or a single scalar
tags: [java-12, collectors, teeing collector]
---

A well-known characteristic of Java _Stream_ is that it can be consumed only once, thus if we want to do some operations that require two passes or more through elements we have to create a new stream or develop a custom collector. Thankfully, Java 12 comes with _Collectors::teeing_ that solves exactly this situation.

Imagine that we need a concise way to get both the min and max values of a stream.

We know how to do that if deal with specialized primitive streams by terminating it with operations like _summarizingInt_:

```java
@Test
void shouldFindMinAndMaxInt() {
    IntSummaryStatistics statistics = IntStream.of(32, 42, 1, 2)
        .summaryStatistics();

    assertEquals(42, statistics.getMax());
    assertEquals(1, statistics.getMin());
}
```
Similarly we have specialized collectors when dealing with objects:
```java
@Test
void shouldFindMinAndMaxProduct() {
    record Product(String label, double price) {}
    
    var products = Stream.of(
            new Product("T-Shirt", 12.99),
            new Product("socks", 5.09),
            new Product("pants", 89.1)
    );

    DoubleSummaryStatistics statistics = products.collect(
        Collectors.summarizingDouble(Product::price)
    );
    
    assertEquals(89.01, statistics.getMax());
    assertEquals(5.09, statistics.getMin());
}
```
The drawback of these collectors is that we don't know which object is the max or min we have just its value. Using _Collectors::maxBy_ and _Collectors::minBy_ will require to pass the stream twice.

Thankfully, the following collector alleviates this problem:

```java
public static <T, R1, R2, R> Collector<T, ?, R> teeing(
    Collector<? super T, ?, R1> downstream1,
    Collector<? super T, ?, R2> downstream2,
    BiFunction<? super R1, ? super R2, R> merger) {
    //...
}
```
_Collectors::teeing_ has three arguments first two are downstream collectors. Every stream element is processed by each of them. The third parameter is a _BiFunction_ that merges two results into the single one, e.g. it can be a function that creates a pair. 

Consider the following record representing a movie:

```java
record Movie(String title, double rating) {}
```

Finding the best and worst movie in a single stream pass is as simple as:

```java
@Test
void shouldFindWorstAndBestMovie() {
    var m1 = new Movie("Groundhog Day", 8);
    var m2 = new Movie("Stop! Or My Mom Will Shoot", 4.4);
    var m3 = new Movie("Forrest Gump", 8.8);

    record MovieStatistics(Movie worst, Movie best) {}

    var ratingComparator = Comparator.comparing(Movie::rating);

    var movieStatistics = Stream.of(m1, m2, m3)
            .collect(Collectors.teeing(
                    Collectors.minBy(ratingComparator),
                    Collectors.maxBy(ratingComparator),
                    (worst, best) -> new MovieStatistics(
                            worst.orElse(null),
                            best.orElse(null)
                    )
            ));

    assertEquals(m2, movieStatistics.worst());
    assertEquals(m3, movieStatistics.best());
}
```

The best part about _teeing_ collector is that we can handle much more complex scenarios by combining multiple collectors downstream. Consider the following enhanced _Movie_ record:

```java
record Movie(
    String title,
    double rating,
    List<Genre> genres
) {}

enum Genre {
    DRAMA,
    //...
}
```

We can find titles of drama movies and their average rating:

```java
@Test
void shouldFindDramasTitlesAndAvgRating() {
    List<Movie> movies = readMoviesFromCSVFile();

    record MovieStatistics(List<String> titles, double avgRating) {}

    var movieStatistics = movies.stream()
            .filter(m -> m.genres().contains(DRAMA))
            .collect(teeing(
                    mapping(Movie::title, toList()),
                    averagingDouble(Movie::rating),
                    MovieStatistics::new
            ));

    var expectedTitles = List.of(
            "Pulp Fiction",
            "You've Got Mail",
            "Forrest Gump"
    );
    assertEquals(expectedTitles.size(), movieStatistics.titles().size());
    assertTrue(expectedTitles.containsAll(movieStatistics.titles()));
    assertEquals(8.13, movieStatistics.avgRating(), /* delta */ 2);
}
```
In the main pipeline we filter movies by genre, then in _teeing_ collector, for one downstream we extract movie titles and collect them in a list. For the second downstream, we produce a scalar - average rating. Finally, for the merger function, we pass a constructor reference that matches types returned by both downstream collectors.

### Conclusion

We can solve most of the problems with one out-of-the-box collector, even when we need to collect more than a collection or a scalar.

Full code can be found [over on GitHub](https://github.com/apulbere/teeing-example).