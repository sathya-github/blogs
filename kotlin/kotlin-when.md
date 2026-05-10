# Kotlin's `when` Is the New `switch` — And `if`, And `instanceof`

If you are coming from Java, the first time you see Kotlin's `when`, your brain probably files it under "oh, that's their switch statement." That is a fair first read. It is also wrong, in interesting ways.

`when` is a single construct that quietly replaces three things from your Java toolkit: `switch`, the long `if/else if/else if` chain, and `instanceof`-with-cast. Once you internalize what it can do, your Kotlin code starts looking noticeably different from your Java code — usually shorter, often clearer.

This is a quick tour of the parts that matter.

## Statement or expression — your choice

The first thing to know is that `when` is **both** a control-flow statement and a value-producing expression. Same syntax, different uses.

```kotlin
// As a statement — return value discarded
when (status) {
    HttpStatus.OK -> log("success")
    HttpStatus.NOT_FOUND -> log("missing")
    else -> log("other")
}

// As an expression — returns a value you can use
val message = when (status) {
    HttpStatus.OK -> "success"
    HttpStatus.NOT_FOUND -> "missing"
    else -> "other"
}
```

The expression form is the one that quietly changes how you write code. You can use `when` directly as a function body, an assignment target, or a method argument — anywhere a value is expected. A lot of the temporary-variable plumbing that piles up in Java just disappears.

## With a subject, or without

You can write `when` with no subject at all. Each branch then becomes its own boolean condition:

```kotlin
// With a subject
when (temp) {
    in 0..15 -> "cold"
    in 16..25 -> "warm"
    else -> "hot"
}

// Without a subject — branches test unrelated conditions
when {
    temp < 0 -> "freezing"
    temp < 16 && humidity > 80 -> "cold and damp"
    isRaining && windSpeed > 30 -> "stormy"
    else -> "fine"
}
```

This is the form that replaces those long `if/else if/else if` ladders. There is no equivalent in Java's `switch` — `switch` requires a subject and constants. In Kotlin, the same keyword handles both shapes.

A loose rule: in idiomatic Kotlin, **any chain of three or more `if/else if` branches is probably better written as `when`.**

## Range and collection matching with `in`

Branches can match against ranges:

```kotlin
when (httpCode) {
    in 200..299 -> "success"
    in 300..399 -> "redirect"
    in 400..499 -> "client error"
    in 500..599 -> "server error"
    else -> "unknown"
}
```

Or against any collection:

```kotlin
val vowels = setOf('a', 'e', 'i', 'o', 'u')

when (letter) {
    in vowels -> "vowel"
    in 'a'..'z' -> "consonant"
    else -> "not a letter"
}
```

You can negate with `!in`. In Java, the same logic would be a tangle of comparison operators or a `Set.contains` call wrapped in an `if`. Here it reads like English.

## Type matching with smart casts

This is where Kotlin starts to feel like a different language. You can match on type, and the compiler **automatically casts the value** for you inside that branch.

```kotlin
fun describe(value: Any): String = when (value) {
    is Int -> "an integer: ${value + 1}"
    is String -> "a string of length ${value.length}"
    is List<*> -> "a list with ${value.size} elements"
    else -> "something else"
}
```

Inside the `is Int` branch, `value` is treated as an `Int` — you can do arithmetic on it directly. Inside the `is String` branch, you can call `.length`. No explicit cast needed, no `(String) value` boilerplate.

Java got pattern-matching `instanceof` in Java 16, so the gap has narrowed. But Kotlin had this from day one, and the syntax stays uniform whether you are matching on type, value, range, or condition.

## Multiple values per branch

Comma-separate to combine cases — the clean replacement for Java switch fall-through:

```kotlin
when (day) {
    SATURDAY, SUNDAY -> "weekend"
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "weekday"
}
```

No `break` statements. No accidental fall-through. No layout tricks.

## Exhaustiveness with enums and sealed types

When the subject is a closed type — an enum or a sealed class/interface — and `when` is used as an expression, the compiler **enforces** that every variant is handled.

```kotlin
sealed interface PaymentResult {
    data class Success(val transactionId: String) : PaymentResult
    data class Failure(val reason: String) : PaymentResult
    data object Pending : PaymentResult
}

fun describe(result: PaymentResult): String = when (result) {
    is PaymentResult.Success -> "Paid (id: ${result.transactionId})"
    is PaymentResult.Failure -> "Failed: ${result.reason}"
    PaymentResult.Pending -> "Still processing"
}
```

Add a new variant to `PaymentResult` later, and every non-exhaustive `when` becomes a build error pointing you to exactly where you need to handle the new case. This is a structural refactoring aid that Java's `switch` simply does not provide — and once you have it, you stop trusting code reviews to catch missing cases.

## Capturing the subject inline

If the subject is the result of a function call, you can capture it inline with `val` and keep it scoped to the `when` block:

```kotlin
val message = when (val user = fetchUser()) {
    null -> "no user found"
    else -> "hello, ${user.name}"
}
// `user` is not visible outside the when
```

This avoids two patterns from Java: re-calling the function in each branch (wasteful), or declaring a temporary variable that leaks into the surrounding scope (noisy).

## A real-world example

Here is everything composed into one realistic block:

```kotlin
fun categorizeRequest(request: HttpRequest): RequestCategory = when {
    request.method !in setOf("GET", "POST") -> RequestCategory.UNSUPPORTED
    request.path.startsWith("/admin") && !request.isAuthenticated -> RequestCategory.UNAUTHORIZED
    request.path == "/health" -> RequestCategory.HEALTH_CHECK
    request.contentLength > 10_000_000 -> RequestCategory.LARGE_PAYLOAD
    request.headers["X-Internal"] == "true" -> RequestCategory.INTERNAL
    else -> RequestCategory.STANDARD
}
```

A single `when` block expressing what would be 15 to 20 lines of nested `if/else` in Java. Read top to bottom, each branch is one rule, the result type is inferred, and there is no temporary-variable noise.

## When *not* to use `when`

`when`'s flexibility makes it tempting to overuse. Two anti-patterns worth flagging:

- **Two-way branches.** A simple either/or check is still better as `if`/`else`. `when` shines when there are three or more cases or when you want exhaustiveness.
- **Long blocks of unrelated business logic.** If you find yourself writing a 50-line `when` where every branch does its own elaborate thing, that is often a signal to use polymorphism (sealed classes with their own behavior) or to extract helper functions.

Like any sharp tool, the right time to reach for it is when one of its specific strengths — exhaustiveness, expression form, smart casts, range matching — is actually helping.

## Final thoughts

`when` is not Kotlin's `switch`. It is a general-purpose conditional construct that happens to also cover what `switch` does. The mental shift for Java developers is small but real: stop thinking of it as "the switch keyword" and start thinking of it as the default tool for *any* multi-way decision.

Once you do, three patterns from your Java code start to fade out:

- Long `if/else if/else if` ladders → subject-less `when`
- `instanceof` with explicit cast → `is` with smart cast
- `switch` over enums with a defensive `default` → exhaustive `when` with no `else` needed

That is one keyword retiring three tools. Not a bad trade.