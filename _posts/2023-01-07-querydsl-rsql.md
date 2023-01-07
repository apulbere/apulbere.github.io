---
layout: post
title: REST Search API with QueryDSL
tags: [querydsl, rsql, rest]
---

One of the most common features in a typical web application is the search functionality. The tricky thing is that we have to provide it via REST API and make it quite dynamic in terms of supported operators and filtrable fields. In the following paragraphs, we'll go through what QueryDSL offers and how a lightweight extension could help. 

Out of the box, QueryDSL comes with [Web Support](https://docs.spring.io/spring-data/commons/docs/current/reference/html/#core.web.type-safe). It boils down to having a Predicate parameter in REST controller method. The bad news is that you cannot do much of the customization. Imagine the ubiquitous example of Pet Shop, where we have to search by nickname using contains or exact match. The natural thing that comes to mind is having in a query parameter the search field, operator and value:
```
/pets?nickname=like:Br
```
or 
```
/pets?nickname=Britney
```
The above example is possible if you extend `QuerydslBinderCustomizer` parse the parameter value, extract the operator and invoke the appropriate expression. But it is not possible if you have other data type than String. QueryDSL first looks at the Q type field and converts the value, only after that we have it in the customizer. For example, the following `/pets?birthdate=between:2000-01-01,2023-01-01` will fail because the value `between:2000-01-01,2023-01-01` cannot be parsed as `LocalDate` and we have no (correct) way to fix that.

### Presenting RSQL-QueryDSL library

The philosophy behind the library is to be just a bridge between REST and QueryDSL. Things like parsing fields and operators, conversion of values, and building predicates are already well handled by Spring and QueryDSL.

So let's jump straight to the example and see how it works in the Pet Shop web application.

First of all we add the dependency in our POM:

```xml
<dependency>
   <groupId>io.github.apulbere</groupId>
   <artifactId>rsql-querydsl</artifactId>
   <version>1.0</version>
</dependency>
```

Next, we define a DTO containing all fields that we want to use in the search:

```java
@Setter
@Getter
public class PetCriteria {
    LongCriteria id = LongCriteria.empty();
    LocalDateCriteria born = LocalDateCriteria.empty();
    StringCriteria petType = StringCriteria.empty();
    StringCriteria nickname = StringCriteria.empty();
}
```

And finally, we use the above DTO as a parameter in the controller method. Spring will automatically map request parameters to the DTO:

```java
@GetMapping("/pets")
List<PetRecord> search(PetCriteria criteria, Pageable page) {
    var predicate = criteria.id.match(pet.id)
            .and(criteria.born.match(pet.birthdate))
            .and(criteria.nickname.match(pet.name))
            .and(criteria.petType.match(pet.type));
    return petRepository.findAll(predicate, page)
            .stream()
            .map(petMapper::map)
            .toList();
}
```
Now we can make a request using the parameter names matching the ones from DTO and operators natural to their types. For example, it makes sense to have `LIKE` with `String` but not with `Long`:

```
/pets?nickname.like=Br
```

The above request produces the following SQL:
```sql
select
    p1_0.id,
    p1_0.birthdate,
    p1_0.name,
    p1_0.type 
from
    pet p1_0 
where
    lower(p1_0.name) like ? offset ? rows fetch first ? rows only
```

Full code can be found over on [GitHub](https://github.com/apulbere/pet-shop-rsql-querydsl).