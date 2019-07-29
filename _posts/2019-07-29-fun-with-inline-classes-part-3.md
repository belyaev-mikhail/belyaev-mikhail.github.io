---
title:  "Kotlin 1.3 inline classes and things you can do with them, part 3"
categories: post
tags:
  - kotlin
  - programming
date: "2019-07-29 15:00:00"
last_modified_at: "2019-07-29 15:00:00"
classes: wide
---

[**[Part 1]**](/post/fun-with-inline-classes-part-1/)  
[**[Part 2]**](/post/fun-with-inline-classes-part-2/)  

This is where things get a little bit weird, but stay with me.

## Scopes: temporary-free DSLs

One of the selling features of Kotlin is the rich applicability of the language to build DSLs with the introduction of [type-safe builders](https://kotlinlang.org/docs/reference/type-safe-builders.html).
They are awesome, but have a simple drawback we are already familiar with in this series of posts: they need introduction of intermediate objects for each new context, and that is a little bit annoying.
Applying inline classes to this problem, however, seems impossible, because type-safe builders operate using a set of *nested* context objects that hold their own data in a tree-like manner (see the KHTML library as an example).
Except it is not.

Let's build a simple DSL for printing out stuff to an `Appendable` (which is the interface implemented by both output writers and string builders, so, not a bad choice) with structured indentation.
What I want to achieve here is something like this:

```kotlin
val sb = StringBuilder()
prettyPrintTo(sb) {
    appendln("fun foo() {")
    indent {
        appendln("val x = 2")
        appendln("val y = 3")
        appendln("run {")
        indent(4) {
            appendln("println(\"x = \$x\")")
        }
        appendln("}")
        appendln("return x * y")
    }
    appendln("}")
}

println(sb)
```

and this should print

```kotlin
fun foo() {
    val x = 2
    val y = 3
    run {
        println("x = $x")
    }
    return x * y
}
```

to stdout.

This is an overly simple DSL with just two contexts (non-indented and indented), but one of them may be arbitrarily nested.
It is pretty straightforward to implement these two to build the DSL, but what I want to achieve here is getting rid of **all** the useless intermediate objects that this code may produce.

The top-level context is easy.

```kotlin
inline class AppendScope(val appendable: Appendable) {
    inline fun indent(indent: Int = 4, body: IndentScope.() -> Unit) {
        IndentScope(indent).body()
    }
    inline fun appendln(value: CharSequence) = appendable.appendln(value)
}
```

The `appendable` property should probably not be public, but it introduces unnecessary complications which I don't want to address here.
Now we need to implemented `IndentScope`, which should hold at least:

- The `appendable` to write to;
- Current level of indentation.

So, two values at least, not applicable for an inline class.
But do we really need to duplicate the `appendable`?
Actually, no, it is already available in the `prettyPrintTo` scope, we just need a way to get to it.
And the way is pretty simple.

```kotlin
inline class IndentScope(val indent: Int = 4) {
    inline fun AppendScope.appendln(value: CharSequence) = appendable.append(" ".repeat(indent)).appendln(value)

    inline fun indent(indent: Int = 4, body: IndentScope.() -> Unit) {
        IndentScope(this.indent + indent).body()
    }
}
```

See what we did here?
We made `appendln` both a member of `IndentScope` and an extension to `AppendScope`, giving it access to both.
Now we can get away with only one field and make `IndentScope` an `inline` class.
We also introduce our own `indent` to make nested indentation possible.
This approach does not scale to more than two receivers, though.

Aaand... it does not work:

```kotlin
fun foo() {
val x = 2
val y = 3
run {
println("x = $x")
}
return x * y
}
```

Why?
Because there are two `appendln`s available inside the indentation scope, and one of them is a direct member of `AppendScope`, while we want to call an extension.
Overload resolution in Kotlin is a pretty complex process, but one rule works in nearly all cases: members win over extensions.
The fix, however, is simple: make the second `appendln` an extension, too.

```kotlin
inline class AppendScope(val appendable: Appendable) {
    inline fun indent(indent: Int = 4, body: IndentScope.() -> Unit) {
        IndentScope(indent).body()
    }
}
inline fun AppendScope.appendln(value: CharSequence) = appendable.appendln(value)
```

Does it work? Oh yeah, it does.

```kotlin
fun foo() {
    val x = 2
    val y = 3
    run {
        println("x = $x")
    }
    return x * y
}
```

Just as planned.
Using decompilation to Java in IntelliJ IDEA gives us the following code:

```java
public static final void main() {
   StringBuilder sb = new StringBuilder();
   int $i$f$prettyPrintTo = false;
   Appendable $this$prettyPrintTo = AppendScope.constructor-impl((Appendable)sb);
   int var3 = false;
   CharSequence value$iv = (CharSequence)"fun foo() {";
   int $i$f$indent = false;
   Appendable var10000 = $this$prettyPrintTo.append(value$iv);
   Intrinsics.checkExpressionValueIsNotNull(var10000, "append(value)");
   StringsKt.appendln(var10000);
   int indent$iv = 4;
   $i$f$indent = false;
   int $this$indent = IndentScope.constructor-impl(indent$iv);
   int var9 = false;
   CharSequence value$iv = (CharSequence)"val x = 2";
   int var13 = false;
   var10000 = $this$prettyPrintTo.append((CharSequence)StringsKt.repeat((CharSequence)" ", $this$indent));
   Intrinsics.checkExpressionValueIsNotNull(var10000, "appendable.append(\" \".repeat(indent))");
   Appendable var14 = var10000;
   var10000 = var14.append(value$iv);
   Intrinsics.checkExpressionValueIsNotNull(var10000, "append(value)");
   StringsKt.appendln(var10000);
   value$iv = (CharSequence)"val y = 3";
   var13 = false;
   var10000 = $this$prettyPrintTo.append((CharSequence)StringsKt.repeat((CharSequence)" ", $this$indent));
   Intrinsics.checkExpressionValueIsNotNull(var10000, "appendable.append(\" \".repeat(indent))");
   var14 = var10000;
   var10000 = var14.append(value$iv);
   Intrinsics.checkExpressionValueIsNotNull(var10000, "append(value)");
   StringsKt.appendln(var10000);
   value$iv = (CharSequence)"run {";
   var13 = false;
   var10000 = $this$prettyPrintTo.append((CharSequence)StringsKt.repeat((CharSequence)" ", $this$indent));
   Intrinsics.checkExpressionValueIsNotNull(var10000, "appendable.append(\" \".repeat(indent))");
   var14 = var10000;
   var10000 = var14.append(value$iv);
   Intrinsics.checkExpressionValueIsNotNull(var10000, "append(value)");
   StringsKt.appendln(var10000);
   int indent$iv = 4;
   int $i$f$indent = false;
   int $this$indent = IndentScope.constructor-impl($this$indent + indent$iv);
   int var23 = false;
   CharSequence value$iv = (CharSequence)"println(\"x = $x\")";
   int var18 = false;
   var10000 = $this$prettyPrintTo.append((CharSequence)StringsKt.repeat((CharSequence)" ", $this$indent));
   Intrinsics.checkExpressionValueIsNotNull(var10000, "appendable.append(\" \".repeat(indent))");
   Appendable var19 = var10000;
   var10000 = var19.append(value$iv);
   Intrinsics.checkExpressionValueIsNotNull(var10000, "append(value)");
   StringsKt.appendln(var10000);
   value$iv = (CharSequence)"}";
   var13 = false;
   var10000 = $this$prettyPrintTo.append((CharSequence)StringsKt.repeat((CharSequence)" ", $this$indent));
   Intrinsics.checkExpressionValueIsNotNull(var10000, "appendable.append(\" \".repeat(indent))");
   var14 = var10000;
   var10000 = var14.append(value$iv);
   Intrinsics.checkExpressionValueIsNotNull(var10000, "append(value)");
   StringsKt.appendln(var10000);
   value$iv = (CharSequence)"return x * y";
   var13 = false;
   var10000 = $this$prettyPrintTo.append((CharSequence)StringsKt.repeat((CharSequence)" ", $this$indent));
   Intrinsics.checkExpressionValueIsNotNull(var10000, "appendable.append(\" \".repeat(indent))");
   var14 = var10000;
   var10000 = var14.append(value$iv);
   Intrinsics.checkExpressionValueIsNotNull(var10000, "append(value)");
   StringsKt.appendln(var10000);
   value$iv = (CharSequence)"}";
   $i$f$indent = false;
   var10000 = $this$prettyPrintTo.append(value$iv);
   Intrinsics.checkExpressionValueIsNotNull(var10000, "append(value)");
   StringsKt.appendln(var10000);
   System.out.println(sb);
}
```

The code could be cleaner, but there are no temporary objects created.
We just build a small DSL without any temporaries involved.

A gist with the whole example can be found [here](https://gist.github.com/belyaev-mikhail/ae1209fa177c757e23a8915a5c57d760).

## Sum types (tagged unions)

Sum types (or eithers, or variants) are found in many functional languages as the first key component of *algebraic data types* (the second being product types).
They are useful to represent values that are *either* of type `A` or type `B` (of course, more types may be introduced, but we will stop on 2).
Sum types are similar to *union types*, another type theory concept representing a choice, but are usually distinguished by the fact that sum types are *tagged*, meaning they contain some runtime information on which type is actually stored inside.
Implementing sum types is rather straightforward, but, you guessed it, introduces overhead through boxing.
I happen to have a working (generated) implementation in [my ktuples library](https://github.com/belyaev-mikhail/ktuples/blob/master/src/main/resources/kotlin/ru/spbstu/ktuples/Variants.kt.template).

In this section we'll try to implement a hybrid approach: a sum type of two types that does not have a tag, but rather uses the types themselves as tags, using the `is` operator.
You may find the class itself pretty similar to `Option` from [part 1](/post/fun-with-inline-classes-part-1/).

```kotlin
inline class Either<out A, out B>
@Suppress("NON_PUBLIC_PRIMARY_CONSTRUCTOR_OF_INLINE_CLASS")
@PublishedApi
internal constructor(@PublishedApi internal val unsafeValue: Any?) {
    companion object {
        inline fun <A> left(value: A): Either<A, Nothing> = Either(value)

        inline fun <B> right(value: B): Either<Nothing, B> = Either(value)
    }
}
```

For an explanation on all this annotation mess, please refer to [part 1](/post/fun-with-inline-classes-part-1/).
What we did here is basically introducing an inline class that can hold anything (`Any?` in Kotlin terms) and then restricting it to contain only values of type `A` or `B` by the only functions that can construct these values.

Let's add some more functions into the mix.

```kotlin
val <T> Either<T, T>.value: T
    get() = unsafeValue as T

inline fun <A> Either<A, *>.asLeft(): A = value as A

inline fun <B> Either<*, B>.asRight(): B = value as B
```

First, we need a property so that `Either<String, String>.value` would be a `String`, not `Any?` (that would be very unfortunate).
Second, we need a way to force getting left-hand or right-hand type from the value.

Now for the checking part.

```kotlin
inline fun <reified A> Either<A, *>.isLeft(): Boolean = value is A
inline fun <reified B> Either<*, B>.isRight(): Boolean = value is B
inline fun <R, reified T : R> Either<T, *>.leftOr(other: () -> R): R =
        if (unsafeValue is T) unsafeValue else other()
inline fun <R, reified T : R> Either<*, T>.rightOr(other: () -> R): R =
        if (unsafeValue is T) unsafeValue else other()
inline fun <reified A, B, R> Either<A, B>.visit(onLeft: (A) -> R, onRight: (B) -> R) =
        when {
            isLeft() -> onLeft(asLeft())
            else -> onRight(asRight())
        }
```

This brings us to the problem with this class: it just doesn't work if the types it uses are not *runtime-available* (say, not `Int` or `String` or `Parrot`, but some type parameter of some function) because you cannot use `is` in this situation.
Another problem is that for a left-valued `Either<String, CharSequence>` both `isRight()` and `isLeft()` would return `true`: the price to pay for using classes themselves as tags.

One may ask why are the definitions for `leftOr`/`rightOr` so weird, introducing two type parameters where you could get away with one?
Well, if there was only one parameter, it would very easy to infer it to `Any` or `Any?`, making the `is` check always return `true` for anything.
Pay close attention to type inference when lambdas are involved.

You can find the whole code with some additional methods and tests [here](https://github.com/belyaev-mikhail/kotlin-wheels/blob/master/src/main/kotlin/ru/spbstu/wheels/Either.kt).

## A word on arrays

As already mentioned in the first part, any of the types introduced in this series are auto-boxed when placed into arrays, which limits their usage quite a lot, especially for building data structures.
There is, however, an escape root, which is not pleasant, but works.

You need to introduce a new array type for every inline class you bring in.
So, for `Option<T>` you need to make an `OptionArray<T>`.
For `Either<A, B>` you need to make an `EitherArray<A, B>`.
And so on.

You can find these two classes [here](https://github.com/belyaev-mikhail/kotlin-wheels/blob/master/src/main/kotlin/ru/spbstu/wheels/OptionArray.kt) and [here](https://github.com/belyaev-mikhail/kotlin-wheels/blob/master/src/main/kotlin/ru/spbstu/wheels/EitherArray.kt) to get the basic understanding on how this works.

## Summing up

Inline classes may seem a questionable language feature at best, with the main goal being programmer-driven optimization and, as we all know, **premature optimization is the root of all evil** in programming.
They do, however, introduce an opportunity to optimize where you find a need to.
Together with inline functions, you can make your code never use any temporary objects that exist only to make your code cleaner, which are pretty common in JVM world.

But remember: inlining is neither a universal optimization technique nor a magical pill, no matter if we speak abount inlining code or data.
Prove that it helps with proper benchmarking and only then apply.

Kudos for reading.
