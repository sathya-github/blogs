# Enums in Kotlin: Small, Useful, and More Powerful Than They Look

If you are coming from Java, Kotlin's enums will feel familiar right away.

An enum represents a fixed set of possible values: directions, statuses, roles, modes, colors, and so on. Kotlin enums start simple, but they can also hold data and behavior when you need them to.

Let's build up from the basics.

## The Simplest Enum

Here is a basic Kotlin enum:

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST
}
```

You can use it like this:

```kotlin
val direction = Direction.NORTH
```

An enum is useful when a value should only be one of a known set of choices. Instead of passing strings like `"north"` or `"south"`, you get type safety:

```kotlin
fun move(direction: Direction) {
    println("Moving $direction")
}
```

Now callers cannot accidentally pass `"nroth"` or `"up-ish"`.

## Using Enums with `when`

Enums work beautifully with Kotlin's `when` expression.

```kotlin
fun describe(direction: Direction): String {
    return when (direction) {
        Direction.NORTH -> "Going up"
        Direction.SOUTH -> "Going down"
        Direction.EAST -> "Going right"
        Direction.WEST -> "Going left"
    }
}
```

One especially nice Kotlin feature: if your `when` covers every enum value, you do not need an `else`.

That means if you later add another enum value:

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST,
    CENTER
}
```

Kotlin can warn you that your `when` is no longer exhaustive. That is a small but very useful safety net.

## Enum Properties

Like Java enums, Kotlin enums can have properties.

```kotlin
enum class HttpStatus(val code: Int) {
    OK(200),
    NOT_FOUND(404),
    INTERNAL_SERVER_ERROR(500)
}
```

Now each enum value carries data:

```kotlin
val status = HttpStatus.NOT_FOUND

println(status.code) // 404
```

This is useful when each constant has a stable associated value: an HTTP code, display label, database code, priority level, and so on.

You can also add multiple properties:

```kotlin
enum class Priority(val label: String, val weight: Int) {
    LOW("Low", 1),
    MEDIUM("Medium", 2),
    HIGH("High", 3)
}
```

## Enum Functions

Enums can also have functions.

```kotlin
enum class HttpStatus(val code: Int) {
    OK(200),
    NOT_FOUND(404),
    INTERNAL_SERVER_ERROR(500);

    fun isError(): Boolean {
        return code >= 400
    }
}
```

Usage:

```kotlin
val status = HttpStatus.INTERNAL_SERVER_ERROR

if (status.isError()) {
    println("Something went wrong")
}
```

Notice the semicolon after the enum constants:

```kotlin
INTERNAL_SERVER_ERROR(500);
```

In Kotlin, if an enum has members after the constants, such as functions or properties, you separate the constants from the rest of the class body with a semicolon.

## Overriding Behavior Per Enum Value

Sometimes each enum value needs its own behavior. Kotlin supports that too.

```kotlin
enum class Operation {
    ADD {
        override fun apply(a: Int, b: Int): Int = a + b
    },
    SUBTRACT {
        override fun apply(a: Int, b: Int): Int = a - b
    },
    MULTIPLY {
        override fun apply(a: Int, b: Int): Int = a * b
    };

    abstract fun apply(a: Int, b: Int): Int
}
```

Usage:

```kotlin
val result = Operation.ADD.apply(2, 3)

println(result) // 5
```

This is powerful, but use it with some restraint. If the behavior gets large or complicated, it may be easier to move that logic into regular classes or functions.

## Useful Built-In Enum Features

Every Kotlin enum value has a `name` and `ordinal`.

```kotlin
val direction = Direction.NORTH

println(direction.name)    // NORTH
println(direction.ordinal) // 0
```

A quick warning for Java developers: avoid relying on `ordinal` for storage, APIs, or database values. If you reorder the enum constants, the ordinal changes.

Prefer an explicit property instead:

```kotlin
enum class Role(val code: String) {
    ADMIN("admin"),
    EDITOR("editor"),
    VIEWER("viewer")
}
```

Kotlin also gives you `entries`, which returns all enum values:

```kotlin
for (direction in Direction.entries) {
    println(direction)
}
```

You may also see older Kotlin code using `values()`:

```kotlin
Direction.values()
```

In modern Kotlin, prefer `entries`.

To look up an enum by name:

```kotlin
val direction = Direction.valueOf("NORTH")
```

Be careful: `valueOf` throws an exception if the name does not match exactly.

For safer parsing, you can write:

```kotlin
fun parseDirection(value: String): Direction? {
    return Direction.entries.find { it.name == value.uppercase() }
}
```

## Enums Can Implement Interfaces

Enums can implement interfaces, which is useful when different enum types share a common contract.

```kotlin
interface Displayable {
    val displayName: String
}

enum class AccountType(override val displayName: String) : Displayable {
    FREE("Free"),
    PRO("Pro"),
    ENTERPRISE("Enterprise")
}
```

Now anything that expects a `Displayable` can work with `AccountType`.

```kotlin
fun printDisplayName(item: Displayable) {
    println(item.displayName)
}
```

## A Practical Example

Here is a small enum you might actually use in an app:

```kotlin
enum class SubscriptionPlan(
    val displayName: String,
    val monthlyPrice: Int
) {
    FREE("Free", 0),
    PLUS("Plus", 10),
    PRO("Pro", 25);

    fun isPaid(): Boolean = monthlyPrice > 0
}
```

And a function using it:

```kotlin
fun messageFor(plan: SubscriptionPlan): String {
    return when (plan) {
        SubscriptionPlan.FREE -> "You are on the free plan."
        SubscriptionPlan.PLUS -> "Thanks for subscribing to Plus."
        SubscriptionPlan.PRO -> "Welcome, power user."
    }
}
```

This gives you:

- a fixed set of valid plans
- readable names in code
- associated data
- behavior close to the data
- exhaustive `when` handling

That is a lot of mileage from a small language feature.

## Final Thoughts

Kotlin enums are familiar if you know Java, but they fit especially nicely with Kotlin's type system and `when` expressions.

Start with simple enum constants. Add properties when each value needs data. Add functions when the behavior belongs naturally with the enum. Use per-value overrides only when the behavior is still small and clear.

For many everyday cases, enums are exactly the right tool: simple, readable, and type-safe.
