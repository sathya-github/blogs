# Kotlin Smart Casts: Stop Telling the Compiler What It Already Knows

If you are coming from Java, you know the dance: check whether something is a certain type, then cast it before using it. Kotlin watches that pattern and quietly intervenes.

Smart casts are Kotlin's way of letting the compiler treat a value as a more specific type once it has enough information to prove the move is safe. It is not flashy. It just removes a small but constant tax from everyday code — the kind of friction you stop noticing only because you got used to paying it.

## The Java Pattern

In Java, an `Object` reference usually starts a familiar two-step:

```java
void printLength(Object value) {
    if (value instanceof String) {
        String text = (String) value;
        System.out.println(text.length());
    }
}
```

Modern Java tightened this up with pattern matching for `instanceof`:

```java
void printLength(Object value) {
    if (value instanceof String text) {
        System.out.println(text.length());
    }
}
```

Better. Kotlin solves the same problem, but the idea shows up in many more places across the language.

## The Basic Smart Cast

Here is the Kotlin version:

```kotlin
fun printLength(value: Any) {
    if (value is String) {
        println(value.length)
    }
}
```

Inside the `if`, Kotlin already knows `value` is a `String`. You do not need:

```kotlin
val text = value as String
```

The compiler has done the work for you. That is a smart cast.

## `is` and `!is`

Kotlin uses `is` for type checks and `!is` for the negative form:

```kotlin
fun printUppercase(value: Any) {
    if (value !is String) return

    println(value.uppercase())
}
```

The smart cast does not stop at the closing brace of an `if`. After the early return, the compiler knows anything still running must be a `String`. Smart casts follow control flow, not just blocks.

## Smart Casts and Null Checks

Smart casts are not only about types — Kotlin uses them for null safety too.

```kotlin
fun printLength(name: String?) {
    if (name != null) {
        println(name.length)
    }
}
```

Inside the `if`, `name` is treated as `String`, not `String?`. In Kotlin's type system, those are two different types, and the compiler tracks that difference everywhere. Smart casts are how that nullability tracking stays out of your way.

## Smart Casts with `when`

Smart casts pair naturally with `when`, Kotlin's more capable cousin to `switch`:

```kotlin
fun describe(value: Any): String {
    return when (value) {
        is String -> "A string with length ${value.length}"
        is Int -> "An integer doubled is ${value * 2}"
        is List<*> -> "A list with ${value.size} items"
        else -> "Something else"
    }
}
```

Each branch sees `value` as its checked type — no manual cast in sight. Modern Java has been moving in a similar direction with pattern matching for `switch`, but Kotlin developers have been writing this style for years.

## Combining Conditions

Smart casts compose with logical operators, as long as the logic actually proves the type:

```kotlin
fun printIfNonEmpty(value: Any) {
    if (value is String && value.isNotEmpty()) {
        println(value.uppercase())
    }
}
```

Because `&&` short-circuits left to right, by the time `value.isNotEmpty()` is evaluated, the compiler already knows the type.

`||` works too, but only in the right shape:

```kotlin
fun example(value: Any) {
    if (value !is String || value.isEmpty()) return

    println(value.uppercase())
}
```

After that block exits, `value` must be a non-empty `String`. The compiler is following the flow of your program, not just reading one line at a time.

## Smart Casts After `return`, `throw`, and `continue`

Kotlin understands that some statements end the conversation. `return`, `throw`, `continue`, `break` — anything that exits the current scope or iteration teaches the compiler something about everything below it.

```kotlin
fun printLength(value: Any) {
    if (value !is String) {
        throw IllegalArgumentException("Expected a string")
    }

    println(value.length)
}
```

This works beautifully with the kind of guard clauses Kotlin encourages:

```kotlin
fun process(input: Any?) {
    if (input == null) return
    if (input !is String) return

    println(input.trim())
}
```

By the last line, `input` is known to be a non-null `String`, even though no cast appears anywhere.

## Stable Values vs. Moving Targets

Smart casts only work on values the compiler can trust to stay the same.

A local `val` is the safest case — it cannot be reassigned:

```kotlin
fun printLength(value: Any) {
    val local = value

    if (local is String) {
        println(local.length)
    }
}
```

A local `var` can be smart cast too, as long as nothing changes between the check and the use. Once the value moves, the compiler drops the cast:

```kotlin
fun printLength() {
    var value: Any = "Kotlin"

    if (value is String) {
        value = 42
        // value.length would not compile here
    }
}
```

That is not Kotlin being picky. It is Kotlin refusing to pretend something is safe when it is not.

## When Smart Casts Do Not Work

A common surprise for Java developers: smart casts do not work on mutable properties.

```kotlin
class User(var name: String?)

fun printUserName(user: User) {
    if (user.name != null) {
        println(user.name.length) // does not compile
    }
}
```

Why? Because `name` is a `var`. Between the check and the usage, anything could change it — another method, another thread, a custom getter that recomputes the value on each call. The compiler refuses to lie to you about that.

The fix is to copy into a local `val`:

```kotlin
fun printUserName(user: User) {
    val name = user.name

    if (name != null) {
        println(name.length)
    }
}
```

Now the compiler can trust `name`. This becomes a habit in Kotlin: when you want help from the compiler, hand it stable values.

## Smart Casts vs Explicit Casts

Kotlin still ships with explicit casts when you really need them:

```kotlin
val text = value as String         // throws if it isn't
val text = value as? String        // returns null if it isn't
val length = (value as? String)?.length
```

Smart casts are different — they happen because the compiler proved something from your code, not because you told it to take your word for it.

The rule of thumb:

- Reach for smart casts first.
- Use `as` when you genuinely know more than the compiler.
- Use `as?` when failure is a real possibility you want to handle.

## A Practical Example

Picture handling events from a loosely-typed API:

```kotlin
fun handleEvent(event: Any) {
    when (event) {
        is String -> println("Message: ${event.trim()}")
        is Int -> println("Code: $event")
        is Map<*, *> -> println("Payload keys: ${event.keys}")
        else -> println("Unknown event")
    }
}
```

Not a single manual cast. Each branch just uses the value as the type it has already proven to be. That is the heart of smart casts: your code says what it means, and the compiler quietly carries the knowledge forward.

## Final Thoughts

Smart casts are a small feature that compounds. They are also the thread that ties a lot of Kotlin together: `is`/`!is`, null safety, `when`, control-flow analysis, and the preference for immutable locals all reinforce the same idea. Check the type, check for null, return early when needed, and let the compiler hold onto what it learned.

Java is closing the gap with `instanceof` patterns and `switch` patterns, and modern Java is genuinely nicer than it used to be. But Kotlin built smart casting into the rhythm of the language — and once you have written code where the compiler does the obvious work for you, going back feels like extra typing.
