---
layout: post
title: "Singleton in Kotlin"
date:   2021-07-06 17:00:00 +0800
categories: kotlin
tags: kotlin singleton design-pattern snippet
---

Singleton is one of the most poupular design pattern.
It ensures there is only one instance of a class.

There are several ways to implement singleton in Kotlin.
Generally, a singleton implementation focuses on followings:
1. How to ensure there is only one (or, in some cases, a fixed number) instance.
2. Thread safe -- ensure the instantiation executed once in multiple threads envieronment.

Sometimes, to develop an application, a singleton object might need the ability to inject the environment into it. For example, in Android, it might need inject context into a singleton to help it access/manage resources and services.

Furthermore, to access/inject the needed environment safely, the ability to choose the timing of initialization would be important. So it might need import lazy initialization. 

To reduce the effort to implement thread-safe, parameterized singleton, the article below collects some singleton implmentations in kotlin.

## Singleton with Object Declaration

The simplest singleton implmentation in kotlin is [object declaration](https://kotlinlang.org/docs/object-declarations.html#object-declarations-overview)

```kotlin
object Singleton {
    init {
        // initializing, will be executed when first access.
    }
    // implement something...
}
```

According to [object declaration](https://kotlinlang.org/docs/object-declarations.html#object-declarations-overview), it support following features natively:

1. Only one instance.
2. Thread-safe
3. Lazy initialization -- initializing when first access.

Although it's simple and easy to use, object declaration have some limitations:
1. No constructor support -- it's hard to be parameterized safely.
2. Easy to misuse and confused with global variables.

There are some suggestions about designing an object declaration singleton:
1. It should be a state-less object. Avoid to store variables in it.
2. No depdendcy with other class/object/environment.

Following is an actual usage about object declaration singleton:
```kotlin
object SafeUnbox {
    @InverseMethod("safeFloat")
    @JvmStatic fun floatToParam(value: Float) : Float {
        return value
    }
    @JvmStatic fun safeFloat(value: Float?) : Float {
        return value ?: run {
            // do something else, for example, log debug information.
            0.0f
        }
    }
}
```
This snippet help android layout element to unbox an nullable float safely.
You can refer to [Data Binding Library/Two-way data binding](https://developer.android.com/topic/libraries/data-binding/two-way#converters) to get more information.

### Variant -- Interface based object declaration
In some situation, it might need fixed number of instances. For example, to provide multiple factories:

```kotlin
intercace Factory<T> {
    fun create(): T
}

object MockFactory: Factory<Int> {
    fun create(): Int
}

object RealFactory: Factory<Int> {
    fun create(): Int
}
```
In this case, make related singletons inherit a common interface would be prefered.
Also, you can use [enum class](/kotlin/2021/07/02/Implement-factory-method-with-enum.html) to implement it with better name space.

---

## Parameterized Singleton 

For singleton which can receive parameters to instantiate, it has following basic form:

```kotlin
class ParameterizedSingleton private constructor(arg: Any) {
    companion object {
        private lateinit var instance: ParameterizedSingleton
        fun getInstance(arg: Any) : ParameterizedSingleton {
            return if(!::instance.isInitialized) {
                ParameterizedSingleton(arg).also {
                    instance = it
                }
            } else {
                instance
            }
        }
    }
}
```
The basic form meets the minimum requirement of original singleton pattern described by famous GoF, but without thread-safe. 

To make this basic form be more reliable in mulit-thread envieronment, it needs other techniques to create the instance. And one of those useful tchniques is [Double-checked locking](https://en.wikipedia.org/wiki/Double-checked_locking):

```kotlin
fun <S> doubleCheckingInit(volatileInst: S?, lock: Any, initBlock: () -> S) =
    volatileInst ?: synchronized(lock) {
        volatileInst ?: initBlock()
    }
```

With the **doublecheckingInit** function, the ParameterizedSingleton could be re-written as following:
```kotlin
class ParameterizedSingleton private constructor(arg: Any) {
    companion object {
        @Volatile
        private var instance: ParameterizedSingleton? = null
        fun getInstance(arg: Any) : ParameterizedSingleton {
            return doubleCheckingInit(instance, this) {
                ParameterizedSingleton(arg).also { instance = it }
            }
        }
    }
}
```
Now,the ParameterizedSingleton meets following requirements:
1. Only one instance.
2. Thread-safe
3. Lazy initialization.
4. Can receive parameters to configure the instantiation.

There are still something need be careful:
1. Try to keep the singleton state-less, or, at least, don't store variables by itself.
2. Realize that the initialization parameter(s) only work at the first instantiation.
3. Furthermore, developer should consider what kind of parameters should/could be used when creating a singleton.

### Variant -- Standard/Shared/Default instance

Sometimes, we want to provide user a default/standard configurated service, which should be singleton; on the other hand, it is also good to allow developer to create and configure their own service.

One of those examples is Apple's [*UserDefaults*](https://developer.apple.com/documentation/foundation/userdefaults). The *UserDefaults* has a class-level variable *standard*, which is a shared defaults object, in additional, it's a singleton. Developers can use the *standard* directly, or create and use his own *UserDefaults* with specified suite name.

It is easy to achieve something like *UserDefaults* with singleton pattern, just makes the constructor to be **public**. That's all.

### Variant -- Extract Singleton to Super Class
Think about two singletons in your app:

```kotlin
class SingletonA private constructor(arg: Any) {
    companion object {
        @Volatile
        private var instance: SingletonA? = null
        fun getInstance(arg: Any) : SingletonA {
            return doubleCheckingInit(instance, this) {
                SingletonA(arg).also { instance = it }
            }
        }
    }
}

class SingletonB private constructor() {
    companion object {
        @Volatile
        private var instance: SingletonB? = null
        fun getInstance() : SingletonB {
            return doubleCheckingInit(instance, this) {
                SingletonB().also { instance = it }
            }
        }
    }
}
```
In previous section, it's already extracting **doubleCheckingInit** to reduce the effort to create singleton, but there are still some similar and duplicate snippets. One of these duplicate is the **volatile** instance. 

In another approach, singleton implmentation might break the single-responsibility principle. So, it would be better to move singleton implementation outside the class itself.

Now, to solve those issue, the SingletonHolder class is introduced[[1](#references)]:
```kotlin
open class SingletonHolder<S> constructor(private val initBlock: () -> S) {
    @Volatile
    protected var defaultInstance: S? = null
    fun default(): S =
        defaultInstance ?: synchronized(this) {
            defaultInstance ?: initBlock().also { defaultInstance = it }
        }
}
```

And we can use the SingletonHolder to create singleton classes:
```kotlin
class SingletonA private constructor(arg: Any) {
    companion object : SingletonHolder<SingletonA> ({
        // singleton init block
        SingletonA(Any) // Also, it can be replaced with builder pattern.
    }) {
        // Other companion object members/methods.
    }
}

class SingletonB private constructor() {
    companion object : SingletonHolder<SingletonB> ({
        // singleton init block
        SingletonB()
    }) {
        // Other companion object members/methods.
    }
}
```
By making companion object inherit to SingletonHolder[[2](#references)], it's simplify and hide the process to create singleton.

## Implement Singleton with Delegated Property

In kotlin, there is another way to implement singleton to out of class -- use delegated progperty[[3](#references)].

```kotlin
class SingletonDelegate<T, V>(private val initBlock: () -> V) : ReadOnlyProperty<T, V> {
    @Volatile
    private var defaultInstance: V? = null

    override fun getValue(thisRef: T, property: KProperty<*>): V {
        return doubleCheckingInit(defaultInstance, this) {
            initBlock().also { defaultInstance = it }
        }
    }
}

```

It's even more easily to create a singleton class by delegated property:
```kotlin
class Singleton private constructor() {
    companion object {
        val default: Singleton by SingletonDelegate {
            Singleton()
        }
        val mock: Singleton by SingletonDelegate {
            Singleton().also {
                //setup mock environment.
            }
        }
    }
}
```
In additionally, the delegated property approach is much easier to create Default/Standard/Shared to support different pre-defined configuration.

Although delegated property can reduce the effort to create singleton, it has an explictly flaw -- it's hard to inject parameters dynamically.

If a singleton depends on some environment variable/object, for example, the context of an android app, it would be better to implement it with **SingletonHolder**.

## Use DI to Manage Singleton
Finally, if you are familiar with dependency injection, it's also good to choose a DI container to help you to manage the singleton.

Below is a example to declare singleton in [Hilt](https://developer.android.com/training/dependency-injection/hilt-android):

```kotlin
// Dependency Inject by Hilt
@Module
@InstallIn(SingletonComponent::class)
class Singleton @Inject constructor() {
}
```

## References: 
1. [How to create a singleton class in kotlin](https://blog.mindorks.com/how-to-create-a-singleton-class-in-kotlin)
2. [A true companion: exploring kotlins companion objects](https://proandroiddev.com/a-true-companion-exploring-kotlins-companion-objects-dbd864c0f7f5)
3. [Kotlin: Delegated Properties](https://kotlinlang.org/docs/delegated-properties.html)
4. [Kotlin: Read only property](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-read-only-property/)