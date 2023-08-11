---
layout: post
title: Unit Testing Anti-Patterns  
tags: [ut, unit testing, anti-patterns]
---

Writing unit tests might be challenging, especially when dealing with an ongoing project with already established hard-to-break anti-patterns.

I will go through some of the most common pitfalls I've encountered quite often.

### Verifying the stubs
Let's suppose we write the following piece of code:

```java
@AllArgsConstructor
public class UserCreationService {
    private final UserFactory factory;
    private final UserRepository repository;
    private final NotificationService notificationService;

    public void create(UserCreateRequest request) {
        var user = factory.createEntity(request);
        user = repository.save(user);
        notificationService.notify(USER_CREATED, user.getId());
    }
}
```
And then create a test for the happy path:
```java
@Test
void userCreatedSuccessfully() {
    var request = new UserCreateRequest();
    var userEntity = new UserEntity(randomUUID());
    when(factory.createEntity(request)).thenReturn(userEntity);
    when(repository.save(userEntity)).then(returnsFirstArg());

    userCreationService.create(request);

    verify(repository).save(userEntity);
    verify(notificationService).notify(USER_CREATED, userEntity.getId());
}
```
The first `verify` is redundant. Some engineers keep verifying every single stub resulting in noise and duplicated code. Every time you modify the stub you also have to do the same for `verify` step. More than that, if the stub is broken, then verify will not be executed, and if the stub passes then `verify` will always pass.
The issue is worse if the test has an `@AfterEach` with `verifyNoMoreInteractions` and `strictness = Strictness.WARN`. It forces you to write `verify` even if you don't want to. 

Verifying the stubs should be rather an exception than a regular thing.

### Test data constants
When setting up the test module for a project, often the second thing to do after adding the testing framework is to create a common test data class.
In no time this class is filled up with lots of constants shared across dozen of tests.

Let's start enumerating the problems:
1. Constants are misused. A constant like `public static final String ID = "550e8400..."` is set as user id, market id, and so on. There is no relation whatever between all these entities, it's misused most probably because fits the data type and was automatically imported by IDE. What if the market id should be UUID while the user id is the username?
2. Mutable objects are declared as constants. This is the best recipe for "Why do the tests fail on Jenkins?".

The alternatives to test data constants are factories and abstract factories. If you need a user id then declare a method in `UserFactory` that creates that id. This way it's easy to change the value, make it a constant or randomly generated.

### Tests know too much
Using reflection to set fields, invoke a method, a constructor, change access modifier, etc. - all this done for testing purposes is always a sure sign of poorly written tests. The worst part is if you go and alter the source code for the sake of a test.
Doing any of the above tricks is a sign that the code requires refactoring.

---

In conclusion, testing should show how good the code you wrote is. And one thing that many forget is that tests should adhere to the same programming principle as the base code.