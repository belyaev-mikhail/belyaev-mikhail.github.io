---
title:  "Kotlin 1.3 inline classes and things you can do with them, part 1"
categories: post
tags:
  - kotlin
  - programming
date: "2019-07-24 16:06:00"
last_modified_at: "2019-07-24 16:06:00"
classes: wide
---

Kotlin 1.3.* is here for quite some time and some experimental features are here with it.
One of those features, deeply built into the core of the language, are inline classes.
Note that the usage of inline classes in user code is currently considered experimental, so restrictions may apply.

## What are inline classes?

Inline classes are a special kind of classes that may be _inlined_ in the resulting code, similar to how inline functions work.
Inline classes are very similar to data classes, although even more restrictive:

- An inline class may have only one constructor-defined property;
- It may not (currently) override `equals` or `hashCode` implementations;
- It may not (currently) have init blocks;
- The primary constructor (currently) must be public;
- There are other limitations we'll cover when we stumble upon them.

The idea behind inline classes is similar to some other features of other languages (see haskell newtypes, for example): at runtime, there is no inline class object, only the property it contains, reducing the boxing-unboxing burden and cost of additional indirection.
Plus, when dealing with primitive platform types, it may even remove boxing entirely, making it even more efficient.

What exactly do we mean by boxing here?
Creating an intemediate, useless object whose whole purpose is to hold a reference to another value.
It introduces unnecessary indirection when accessing values, slows things down (especially if you use it a lot) and is overall feared in the JVM world.

## Do they really work like that?

Well, it's complicated.
For starters, the boxing is still there.
At any point the compiler may choose to box your value into an inline class object.
It must do so when working with them as generics, for example, when putting these into collections or arrays.
No magic there.

There are also some peculiar cases even without generics.
Let's try implementing an inline class and look at the bytecode it generates for the JVM:

```kotlin
inline class BetterArray<T>(val array: Array<Any?>) {
    override fun toString(): String {
        return array.joinToString(prefix = "[", postfix = "]")
    }
}
fun main(args: Array<String>) {
    val ba = BetterArray<Int>(arrayOf(1,2,3))

    println(ba) // box(ba) // conversion to kotlin.Any?
    println(ba.toString()) // BetterArray$Erased.toString(ba) // inlined/static
    println("$ba") // String.valueOf(box(ba)) // WHY???
}
```

See the problem?
If, at some point, your inline class is converted to a generic or to `kotlin.Any`, it is auto-boxed.
As you can see with the interpolation (`"$ba"`) example, it may happen out-of-the-blue.
Let's just hope these child issues will be fixed soon and move on.

## Why inline classes?

The standard library has two main usages for this feature: unsigned integer types and the `kotlin.Result` generic.
Unsigned integer types are wrapped around their signed counterparts, redefining some operations through overloading, without an explicit overhead of such conversion.
`kotlin.Result` is essentially similar to the `Try` generic of Scala, wrapping either a value or an exception that happened computing the value.
Unlike `Try`, however, it does not box the value in the more common non-exceptional case, making it potentitally much more efficient to use this type instead of the alternative.

## Are there more potential usages?

Certainly!
Let's start with a simple one.

### Non-boxing optionals

Kotlin is generally antagonistic to stuff like `scala.Option` or `java.Optional` to represent an absense of value due to its clever handling of nullable types (`T?`).
Really, why use a library-based construct when you can just use `null` instead?

There are, however, use-cases where such values are needed.

- Library code where `null` is a valid value, not an absense of one. Java made mistakes in this part when designing its `Map` containers API long before Kotlin was even conceived.
- Code that is too prone to errors due to auto-conversion between non-nullables and nullables. Better be safe than sorry.
- A personal choice to be more specific with your values. It's all about choice after all, isn't it?

Of course, you can easily roll your own class to represent an optional, but there you have two chairs to choose to sit on:

- Boxing. See `scala.Option`, which always boxes its values. Paying a price of a significant overhead (when you design your whole API around it) is too much for some people.
- No boxing, but not allowing `null` as a value. See `java.Optional`. Even kotlin itself has a similar problem with `lateinit` properties: they essentially represent the same pattern and cannot be `null`.

With inline classes, you can avoid both of these pitfalls and have a nice non-boxing api for optionals (modulo current limitations given above):

```kotlin
inline class Option<out T>
@Suppress("NON_PUBLIC_PRIMARY_CONSTRUCTOR_OF_INLINE_CLASS")
@PublishedApi
internal constructor(@PublishedApi internal val unsafeValue: Any?) {
    companion object {
        internal val NOVALUE = Any()
        @Suppress(Warnings.DEPRECATION)
        private val EMPTY = Option<Nothing>(NOVALUE)

        @Suppress(Warnings.NOTHING_TO_INLINE, Warnings.DEPRECATION)
        inline fun <T> just(value: T) = Option<T>(value)

        fun <T> empty(): Option<T> = EMPTY

        fun <T> ofNullable(value: T) = value?.let(::just) ?: empty()
    }

    fun isEmpty() = NOVALUE === unsafeValue
    fun isNotEmpty() = NOVALUE !== unsafeValue

    fun getOrNull(): T? = getOrElse { null }
    fun get(): T = getOrElse { throw IllegalStateException("Option.empty().get()") }

    override fun toString(): String = when {
        isEmpty() -> "Option.empty()"
        else -> "Option.just(${get()})"
    }
}
```

The main idea is to introduce an effectively private value to represent nothingness different from `null`, called NOVALUE here.
It is hidden from user code, so accidental wrong usage should not be an issue.
Checking for absense becomes a trivial reference-equality check, as efficient as a `null` check would be.
This is a typical way of dealing with the problem of "valid `null` value" in java libraries.

What is this `@Supress("SOME_CONFUSING_STUFF_IN_CAPS")` thingie, you must ask?
Well, it seems that kotlin team itself is not happy with the single-public-primary-constructor limitation and peeled a hole in the compiler to lift it for its own classes.
Seems like we can do it, too (do not forbid it, please).

Obviously, we cannot live with a public primary constructor, 'cos it breaks the whole idea of type safety for our class: one can put a `String` into an `Option<Int>` **and we don't want that**.

What is this `@PublishedApi` nonsense then?
It's a special annotation you give to kotlin entities that are internal from the language standpoint, but can be accessed by public inline entities.
We want to make everything we want inline for this guy, so we need for both the constructor and the property.

You may find the whole class implemented in my experimental kotlin stuff library [here](https://github.com/belyaev-mikhail/kotlin-wheels/blob/master/src/main/kotlin/ru/spbstu/wheels/Option.kt).

Let's put this little guy to use (the code for `flatMap` may be found with the link above):

```kotlin
val o2 = Option.just(2)
val o3 = Option.just(3)
val oe = Option.empty<Int>()
// o2 + o3
assertEquals(Option.just(5), o2.flatMap { v2 -> o3.flatMap { v3 -> Option.just(v2 + v3) } })
// o2 + o3 * oe
assertEquals(Option.empty(),
        o2.flatMap { v2 -> o3.flatMap { v3 -> oe.flatMap { ve -> Option.just(v2 + v3 * ve) } } }
)
```

What `flatMap` does here is called *coalescing*: combining several values, some of which may be absent, together to form a resulting value.
For people familiar with functional programming this is clearly related to `Option` being a *monad* (something we will not be speaking about in this post).

#### How to use an optional?

`var` of type `Option` has proven to be a curiously good replacement for `lateinit var` in my code for cases when the value absolutely should be able to represent `null`.
I've also used it to work with `Map`s that are notoriously bad at deciding whether `null` means it has a value or not.
In normal life, please stick to using `null`s as the language authors intended you to do.

Unfortunately, we still cannot put such a value in a collection or array unless we don't care about boxing and inefficiencies it brings with itself.
We'll try to mitigate this restriction somewhat in the next posts.

### To be continued
