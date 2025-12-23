---
layout: post
title: "CrOp 0.1.3: Adding Result Counts and a Look at API Trade-offs"
tags: [rsql, java, rest, jpa]
---

I'm happy to share that I've added a new feature to [CrOp](https://github.com/apulbere/crop) and also moved it to a new namespace.

In my [previous articles](https://adrian.md/2024/07/14/crop/), I explained what problem it solves in a typical REST service. At the time, I was still missing a way to count the total result. This is a must-have in a paginated query. [`@Query`](https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/Query.html) from Spring Data JPA, for example, has a separate attribute for that - `countQuery`.

Seems trivial now, as any problem in retrospect, but at the time I was getting `SqmRoot not yet resolved to TableGroup` error. And that was because I was using the wrong root to apply predicates for count. 
Storing predicates as a function that accepts another root allowed me to reapply them for count.

Below is an example of count. Note that it is the same query as for select, but a different execution method at the end:

```java
@GetMapping("/pets/count")
Long count(PetCriteriaOperator searchCriteria) {
    return cropService.create(Pet.class, searchCriteria)
            .match(Pet_.id, PetCriteriaOperator::getId)
            .match(Pet_.name, PetCriteriaOperator::getNickname)
            .match(Pet_.birthdate, PetCriteriaOperator::getBirthdate)
            .match(Pet_.price, PetCriteriaOperator::getPrice)
            .join(Pet_.features)
                .match(PetFeature_.feature, PetCriteriaOperator::getFeatures)
            .endJoin()
            .match(Pet_.active, PetCriteriaOperator::getActive)
            .join(Pet_.petType)
                .match(PetType_.code, PetCriteriaOperator::getType)
                .join(PetType_.petCategory)
                    .match(PetCategory_.code, PetCriteriaOperator::getCategory)
                .endJoin()
            .endJoin()
            .getCount();
}
```

About the namespace, I did it for simple economic reasons - did not want to renew the old domain `com.apulbere`, so I published instead under `md.adrian`. The library can now be found at the coordinates:

```xml
<dependency>
    <groupId>md.adrian</groupId>
    <artifactId>crop</artifactId>
    <version>0.1.3</version>
</dependency>
```

This project was more than just trying to solve a problem. It made me think very carefully about the design choices of an API, just a few of them:

* Was it a correct choice to use the Metamodel API? 

Example `Pet_.name` in `.match(Pet_.name, PetCriteriaOperator::getNickname)`. Perhaps using strings and forcing the JPA provider to perform the Metamodel lookup at runtime could offer more flexibility, but less type safety. At the same time, in the examples I provided, I used a statically generated metamodel, but not everyone is a fan such tools. It's always about compromises.

* How should I test it? 

While I am currently using a demo project for integration testing, the long-term goal is to move tests closer to the source code. That means bringing a specific JPA provider and testing through it.

* Does the API have enough flexibility for further enhancements and extensibility? 

Let's say I need to add customizable validation for input parameters without breaking the existing contract. Here, I should carefully consider what the core idea and domain of the library are, and whatever needs to be added is a part of it.

* And perhaps the most challenging one: Does the API offer too much? 

Currently, once you define a filter, you automatically get every operator for that data type. For example, a string filter exposes `like` alongside the basics `eq`, `neq`, and `in`. This can be a problem if your database isn't optimized for full-text search. I mention this because using a tool in the wrong context doesn't make the tool itself flawed - it just means it's being pushed beyond its intended limits.

The primary goal was to power a front-end data grid, and the library handles that job perfectly. Of course, every design comes with trade-offs, and I’ve tried to be as transparent as possible about those here.