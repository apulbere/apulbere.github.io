---
layout: post
title: Premature ORM-ification
tags: [orm, architecture, design]
---

ORMs promise to simplify persistence, but in practice, they introduce recurring problems.

In many cases, there’s no clear or well-documented justification for why an ORM was introduced in the first place. Most probably it’s adopted by default, based on assumptions such as the belief that you can switch the database vendor without changing any code or that an ORM somehow “bridges” object-oriented programming and relational databases.

While intended to ease data access, ORMs often end up dictating the entire application architecture. Instead of designing models around business logic, we adapt them to fit ORM limitations, flattening structures, adding unnecessary relationships, or compromising encapsulation just to bypass ORM constraints. The result is a codebase where the data model drives the design and the actual logic comes second.

Since ORMs typically map entire database tables to entities, these entities tend to get passed throughout the application. The issue? You rarely need all the fields, let alone the related entities that are pulled in by default. Projections or custom queries can help, but despite code reviews, overfetching still frequently slips through.

We strive for loose coupling between components for good reason. But under the hood, many ORMs create tightly coupled entity graphs through bidirectional or deeply nested relationships. The model becomes tangled, what seems like a minor change in one part can trigger unpredictable behavior elsewhere. One use case might require a lazy one-to-one mapping, while another needs it eager, both relying on the same entity. The result: conflict and performance degradation.

When debugging performance issues, you’re forced to work at two layers: the ORM abstraction (JPQL, HQL, etc.) and the raw SQL it generates. This doubles the effort and mental overhead.

ORMs claim to offer vendor-agnosticism, but that illusion breaks down the moment you need a database-specific feature or optimization. Suddenly, you're writing native queries or injecting dialect-specific logic, completely undermining the abstraction. Worse, when these two layers collide, you end up manually triggering flushes, refreshes, or other operations to keep them in sync.

Some argue that these issues arise from developers not fully understanding the tool, and sure, there’s truth in that. But ORMs were supposed to simplify development, not require mastery of every quirk just to execute a query properly.