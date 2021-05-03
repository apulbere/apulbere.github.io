---
layout: post
title: How to Organize the Code in the Ports and Adapters Architecture
tags: [java, hexagonal architecture, maven modules]
---

In this article, we'll have a look at how to separate an application into different modules so that it can be developed and tested in isolation of any external dependencies. As a result, we'll obtain loosely coupled components where things like database and communication protocol can easily be replaced without any changes in the domain logic.

First of all, let's introduce a user story so that the domain logic of the example will be clear from the start:

<table>
    <tbody>
        <tr>
            <td>
                <b>User Story</b>
                <br><br>
                <i>As a</i> client of the bookshop
                <br>
                <i>I want</i> to be able to look up books by title
                <br>
                <i>So that</i> I can choose the one I like
            </td>
        </tr>
        <tr>
            <td>
                <b>Acceptance Criteria</b>
                <br><br>
                <i>Given</i> a collection of books
                <br>
                <i>When</i> the user types a title
                <br>
                <i>Then</i> all books matching that title will be returned
            </td>
        </tr>
    </tbody>
</table>

Is natural that the first module we create, _bookshop-domain_, is the one that holds domain DTOs and secondary ports (secondary, because the application itself triggers the calls to other systems). 

As per the bookshop example, we'll need one _record_ to hold all book's fields:

```java
public record Book(UUID id, String title, List<String> authors) {

}
```
 And another record for search criteria:
```java
public record BookSearchCriteria(String title) {

}
```
The secondary port _BookRepository_ is an interface that declares a method for obtaining all the shop's books:

```java
public interface BookRepository {

    List<Book> findAll();

}
```
Note that all the classes we just defined are not tied in any way to any framework's specific classes, interfaces, or annotations. For example, _Book_ is annotated neither with _javax.persistence_ nor with _Jackson_ annotations, although our application will persist data and expose REST endpoints.

The second module, _bookshop-core_, as the previous one, is also part of domain logic. It will hold a service that does exactly what was asked from us in the user story:

```java
@AllArgsConstructor
public class BookService {

    private final BookRepository bookRepository;

    public List<Book> findAll(BookSearchCriteria criteria) {
        return bookRepository.findAll()
                .stream()
                .filter(matches(criteria))
                .collect(toList());
    }

    private Predicate<Book> matches(BookSearchCriteria criteria) {
        return book -> book.title()
                .toLowerCase()
                .contains(criteria.title().toLowerCase());
    }
}
```
_BookService_ can also be called _port_ in the hexagonal architecture as it is the class that exposes and implements the domain logic. Creating an interface first for the port it's not always necessary, especially if the interface and implementation reside in the same module.

The next module we want to develop is _bookshop-inmem-persistence_ which will hold the following adapter:
```java
public class InMemBooksRepository implements BookRepository {

    private final List<Book> books;

    public InMemBooksRepository(List<Book> books) {
        this.books = List.copyOf(books);
    }

    @Override
    public List<Book> findAll() {
        return books;
    }
}
```
For the sake of simplicity, we did an in-memory implementation, but normally this module will hold all entities, JPA repositories or DAOs, and converters from entities to _bookshop-domain_ DTOs.

Finally, the last module, _bookshop-rest_, exposes the endpoint and invokes the port to our domain logic:

```java
@RestController
@AllArgsConstructor
public class BooksController implements BooksApi {

    private final BookService bookService;
    private final DomainMapper domainMapper;

    @Override
    public ResponseEntity<List<Book>> listBooks(String title) {
        var criteria = new BookSearchCriteria(title);
        var books = bookService.findAll(criteria)
                .stream()
                .map(domainMapper::map)
                .collect(toList());

        return ResponseEntity.ok(books);
    }
}
```

Notice that we convert the DTO returned by the service to another DTO, generated from an OpenAPI specification.

### Conclusion

We have seen examples of ports and adapters in hexagonal architecture, and how to properly organize the code in a decoupled fashion.

Full code can be found [over on GitHub](https://github.com/apulbere/bookshop)