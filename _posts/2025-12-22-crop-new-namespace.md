---
layout: post
title: CrOp 0.1.3 Release
tags: [rsql, java, rest, jpa]
---

I'm happy to share that I've added a new feature to [CrOp](https://github.com/apulbere/crop) and also moved it to a new namespace.

In my [previous articles](https://adrian.md/2024/07/14/crop/), I explained what problem it solves in a typical REST service. At the time, I was still missing a way to count the total result. This is a must-have in a paginated query. [`@Query`](https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/Query.html) from Spring Data JPA, for example, has a separate attribute for that - `countQuery`.

Seems trivial now, as any problem in retrospect, but at the time I was getting `SqmRoot not yet resolved to TableGroup`. And this was because I was using the wrong root to apply predicates for count. 
Storing predicates as a function that accepts another root allowed me to reapply them for count.

Below is an example of count, note that it is the same query as for select, but a different execution method at the end:

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
