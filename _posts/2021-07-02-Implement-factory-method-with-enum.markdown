---
layout: tag_page
title:  "Implement factory method with enum"
date:   2021-07-02 13:47:00 +0800
categories: kotlin
tags: kotlin enum design-pattern snippet
---

Traditionally, to implement a factory method in Kotlin, it is always need a manipulator to choose what factory to use.

In Kotlin, it needs write manipulator with *when* statement.

Futhermore, if developer needs a generic factory method, it always implements the manipulator by a generic inline function with 'reified' keyword.

```kotlin
inline fun <reified T: IProudct> create(): T {
    return when (T::class) {
        ProductA::class -> createProductA() 
        ProudctB::class -> createProductB() 
        else -> throw Exception("No such product")
    }
}

val product_a: ProductA = create()
val product_b: ProductB = create()
```

The *reified* keyword provide type information at **run-time**, that the manipulator can use it to decide which product should be created.

Also, it always need an *else* to process unexpected situation.

But, with latest version of Kotlin, there has another way to implement a generic and manipulator-free factory method.

Following snippet uses Enum class with anonymous classes to implement it:

```kotlin
// This is abstract product
interface Product {
    val lcid: String
}

// Followings are concrete products
class ProductUS : Product {
    override val lcid = "en-us"
}

class ProductSpain: Product {
    override val lcid = "es-es"
}

// This is a generic creator, has a factory method create()
interface Factory<T> {
    fun create() : T
}

// This is the concreate creator, which is implemented by enum class with anonymous classes.
// This decalare provides a static and non-manipulator implementation in Kotlin. 
enum class ProductFactory: Factory<Product> {
    US {
        override fun create() = ProductUS()
    }
    Spain {
        override fun create() = ProudctSpain()
    }  
}

// Use cases
val product_us = ProductFactory.US.create()
val product_es = ProductFactory.Spain.create()
```

This implmentation has following pros:
1. Sometimes factory method combine with singleton, declare with enum guarantees the factory has only one instance.
2. Don't need implement manipulator.
3. The type information determined at compile time. It should be faster than traditional implementaion with *reified*.
