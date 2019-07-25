---
title:  "Kotlin 1.3 inline classes and things you can do with them, part 2"
categories: post
tags:
  - kotlin
  - programming
date: "2019-07-25 16:00:00"
last_modified_at: "2019-07-25 16:00:00"
classes: wide
header:
  og_image: "/assets/images/kotlin.png"
---

[**[Part 1]**](/post/fun-with-inline-classes-part-1/)

## Scopes: comparators and comparables

Comparators are a way of providing custom means of total ordering inherited from Java.
A comparator compares two values using a `compare` method.
Unlinke the `compareTo` convention, however, a comparator is very inconvenient to use in real code, as you need to call the method instead of using the mathematical operators `<`, `<=`, `>` and `>=`.
If only there was a way of converting one convention into another, right?
Something like this:

```kotlin
val comparator: Comparator<T> = ...
with(comparator) {
  val v0: T = ...
  val v1: T = ...

  return v0 <= v1
}
```

This is, however, impossible.
Comparison operators in kotlin expect an operator function `T.compareTo(other: T)` available for the left-hand-side of the comparsion.
This may be added as an extension, which is great.
The problem is that you need to add an extension function **as an extension** to comparator.
Extensions to extensions are not (yet) possible in Kotlin.

What do?
Create another class that has this extension as a member.

```kotlin
class ComparatorScope<T>(val comparator: Comparator<T>) {
  operator fun T.compareTo(that: T) = comparator.compare(this, that)
}
inline fun <T, R> Comparator<T>.scope(body: ComparatorScope.() -> R): R =
    ComparatorScope(this).body()
...
val comparator: Comparator<T> = ...
comparator.scope {
  val v0: T = ...
  val v1: T = ...

  return v0 <= v1
}
```

Hooray!
But here we have the same problem as before: creating an intermediate object that is only needed because of the language limitation.
An object that exists only during a call to one particular function.
Solution? Simple. Make it an inline class instead:

```kotlin
inline class ComparatorScope<T>(val comparator: Comparator<T>) {
  inline operator fun T.compareTo(that: T) = comparator.compare(this, that)
}
```

No intermediate object, no boxing, no overhead.
Getting the best from both worlds.

It may be desirable to add more convenience methods to the scope class:

```kotlin
inline class ComparatorScope<T>(val comparator: Comparator<T>) {
    operator fun T.compareTo(that: T) = comparator.compare(this, that)

    fun Iterable<T>.max() = maxWith(comparator)
    fun Iterable<T>.min() = minWith(comparator)
    fun Collection<T>.sorted() = sortedWith(comparator)
    fun MutableList<T>.sort() = sortWith(comparator)
}
```

This approach may be exploited in any situation you would need an extension function as an extension in some other object, very useful for DSLs built over external APIs.

## Bit masks

Kotlin is very limited when it comes to operating single bits in primitive values.
The bit-by-bit operations are still considered experimental.
However, if you need to implement an algorithm or a data structure that needs to manipulate single bits (a bit set or a trie is a good example), you may need to make your code extra typesafe by distinguishing integers from bit masks, while retaining the performance of bit operations.

As a bonus, you get syntax sugar (unimaginable for integers) like `if(bits[0])` for free!
The solution, as you may already have guessed, is using an inline class.

```kotlin
inline class IntBits
constructor(val data: Int) {
    companion object {}

    inline fun asInt() = data

    inline infix fun shl(bitCount: Int): IntBits = IntBits(data shl bitCount)
    inline infix fun shr(bitCount: Int): IntBits = IntBits(data ushr bitCount)
    inline infix fun and(that: IntBits): IntBits = IntBits(data and that.data)
    inline infix fun andNot(that: IntBits): IntBits = IntBits(data and that.data.inv())
    inline infix fun or(that: IntBits): IntBits = IntBits(data or that.data)
    inline infix fun xor(that: IntBits): IntBits = IntBits(data xor that.data)
    inline fun inv(): IntBits = data.inv().asBits()
    inline fun reverse(): IntBits = Integer.reverse(data).asBits()

    inline val popCount: Int get() = Integer.bitCount(data)
    inline val lowestBitSet get() = IntBits(Integer.lowestOneBit(data))
    inline val highestBitSet get() = IntBits(Integer.highestOneBit(data))
    inline val numberOfLeadingZeros: Int get() = Integer.numberOfLeadingZeros(data)
    inline val numberOfTrailingZeros: Int get() = Integer.numberOfTrailingZeros(data)
    ...
}
```

We call this class `IntBits` because someone may need bit masks of other sizes.
Note that for some functions (like the number of leading zeroes) we had to exploit JVM-specific APIs from java standard library, because Kotlin does not provide these.

Some other convenience methods:

```kotlin
inline operator fun get(index: Int) = data and (1 shl index) != 0
inline fun set(index: Int) = this or (One shl index)
inline fun clear(index: Int) = this andNot (One shl index)
inline fun slice(from: Int = 0, toExclusive: Int = SIZE): IntBits {
    require(from >= 0)
    require(toExclusive >= from)
    require(toExclusive <= SIZE)
    val mask =
            when (val size = toExclusive - from) {
                0 -> return Zero
                SIZE -> return this
                else -> ((1 shl size) - 1) shl from
            }
    return (this and mask.asBits()) shr from
}
```

Note that declaring everything `inline` is not considered a good thing by Kotlin and you even receive a warning.
The impact of inlining these is questionable and you need to benchmark every use-case.

The whole implementation of this class together with some test can be found [here](https://github.com/belyaev-mikhail/kotlin-wheels/blob/master/src/main/kotlin/ru/spbstu/wheels/Bits.kt).

## Non-reified arrays

As another thing inherited from Java, Kotlin's arrays are a little bit weird.
First, you have a generic `Array<>` and you have primitive arrays (although Kotlin itself does not even have a concept of a primitive type).
Second, the arrays are the only generics that are reified: every array *knows* which type it was constructed from.
This is convenient sometimes, but most of the time it's not.

Have you ever read the source code of your favourite data structure implementation?
Most DSs that use arrays use arrays of objects (`Any?` in Kotlin world) instead of real generics.
Why?
Because using an array of some type requires information about this type.
This is bad from type safety standpoint, but it is the only way.

Except it isn't.
As you've already noticed, inline classes are a not-at-all-bad way of adding typesafety without adding overhead in some situations.
This is one of such situations.
Let's design a non-reified generic array class.

```kotlin
@Suppress(Warnings.UNCHECKED_CAST, Warnings.NOTHING_TO_INLINE)
inline class TArray<T>
@Suppress("NON_PUBLIC_PRIMARY_CONSTRUCTOR_OF_INLINE_CLASS")
@PublishedApi
internal constructor(@PublishedApi internal val inner: Array<Any?>) {

    constructor(size: Int) : this(arrayOfNulls(size))

    inline operator fun get(index: Int): T? = inner[index] as T?
    inline operator fun set(index: Int, value: T?) {
        inner[index] = value
    }
    inline val size: Int get() = inner.size
    ...
}
```

We ease our own life here a little bit by making the elements of this array nullable at all times, but it can be designed differently.
If you don't understand what does this `@Suppress("NON_PUBLIC_PRIMARY_CONSTRUCTOR_OF_INLINE_CLASS")` nonsense mean, please refer to [part 1](/post/fun-with-inline-classes-part-1/).

This would also allow us to finally add a half-decent `toString` implementation for the array:

```kotlin
override fun toString(): String {
    if(size == 0) return "[]"
    val sb = StringBuilder("[")
    sb.append(get(0))
    for(i in 1 until size) sb.append(", ").append(get(i))
    sb.append("]")
    return "$sb"
}
```

`equals` and `hashCode` are still out of our reach, though.
It also makes total sense to provide all other operations standard arrays have, but it would be pretty space-consuming.
You can look at the complete implementation [here](https://github.com/belyaev-mikhail/kotlin-wheels/blob/master/src/main/kotlin/ru/spbstu/wheels/TArray.kt).

This array-ish class is a perfect base for a generic implemenation of data structures like `ArrayList` or `ArrayDeque` that would be generic and typesafe not only on the outside, but on the inside as well. In fact, I even have an `ArrayDeque` imlementation in Kotlin for you [right here](https://github.com/belyaev-mikhail/kotlin-wheels/blob/master/src/main/kotlin/ru/spbstu/wheels/Queue.kt). Not a single unsafe cast, everything is inside `TArray` implementation. And at the same time, no overhead over normal array.

## To be continued
