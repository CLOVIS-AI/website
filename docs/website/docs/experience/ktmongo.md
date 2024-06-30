# KtMongo

!!! note ""
    View the [repository](https://github.com/4sh/ktmongo).

The KMongo Kotlin driver for MongoDB was started in 2016 to add Kotlin-idiomatic DSLs on top of the official Java driver. The existence of KMongo is one of the greatest reasons of the popularity of MongoDB in the Kotlin ecosystem.

In 2023, the MongoDB team released an official Kotlin driver, and the development of KMongo stopped. However, the official driver lacks the DSL that made KMongo popular in the first place—existing projects are faced with the difficult choice of continuing to use the unmaintained KMongo driver, or switching to the official driver and rewriting most of their database layer with a worse syntax.

4SH has many production projects based on MongoDB, and we didn't like the situation.

## KtMongo

KtMongo is a (currently prototypal) re-imagining of KMongo, using modern Kotlin and MongoDB features to make usage even simpler and safer. In particular, KtMongo eliminates layers of abstractions in the internal implementation to make query building faster, and to avoid mistakes due to incorrect operator usage.

KtMongo is developed the Task Force programme, under which employees can obtain a few days of budget to prototype a project that could become useful to the company in the future. 

## My involvement

I am leading and developing the KtMongo project, helped by two experienced Kotlin and MongoDB developers.
