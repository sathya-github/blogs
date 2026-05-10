# Unchecked and Unbothered: How Kotlin Rethinks Java's Exception Handling

If you've spent any time in a Java codebase, you've probably caught yourself writing `throws Exception` just to make the compiler stop complaining — or worse, swallowing errors in an empty catch block and hoping nothing goes wrong. Kotlin runs on the same JVM, uses the same `Throwable` hierarchy, and even shares the `try`/`catch`/`finally`/`throw` keywords. But its approach to exception handling is fundamentally different — and the differences reveal a quietly clever piece of compiler trickery.

First the surface. Then the wiring.

## The headline: no checked exceptions

Kotlin has no checked exceptions. None. Every exception is effectively unchecked.

```java
// Java
public String readFile(String path) throws IOException {
    return Files.readString(Paths.get(path));
}
```

```kotlin
// Kotlin
fun readFile(path: String): String {
    return Files.readString(Paths.get(path))  // no throws clause needed
}
```

The reasoning is the same argument made by Anders Hejlsberg, Bruce Eckel, and others years ago: checked exceptions don't scale. They tend to produce either `throws Exception` everywhere (defeating the purpose) or empty catch blocks (worse). They also break down badly with lambdas and higher-order functions, which Kotlin uses heavily.

The trade-off: you lose the compiler nudge to handle I/O failures, JDBC errors, and the like. You're back to documentation and discipline.

## try/catch is an expression

This is the syntactic feature you'll use most often:

```kotlin
val number: Int = try {
    input.toInt()
} catch (e: NumberFormatException) {
    0
}
```

In Java you'd need a mutable variable initialized before the try, which is awkward and prevents `final`/`val`. The expression form composes nicely with Kotlin's general "everything is an expression" philosophy.

## Resource management with `use`

Java's try-with-resources:

```java
try (BufferedReader br = new BufferedReader(new FileReader(path))) {
    return br.readLine();
}
```

Kotlin's equivalent is the `use` extension function on `Closeable`:

```kotlin
BufferedReader(FileReader(path)).use { br ->
    br.readLine()
}
```

`use` calls `close()` on completion or exception. Functionally identical, but it's a library function rather than language syntax. The advantage: you can write your own `use`-like extensions for any cleanup pattern.

## The `Nothing` type for "this always throws"

```kotlin
fun fail(message: String): Nothing {
    throw IllegalStateException(message)
}

val name: String = user.name ?: fail("name required")
// compiler knows fail() never returns, so `name` is non-null String
```

`Nothing` is the bottom type — a subtype of every type. Java has no equivalent; you'd return `void` or declare `throws`, and the compiler still wouldn't help with flow analysis the way Kotlin does.

## Standard library precondition helpers

Instead of writing the throw yourself:

```kotlin
require(age >= 0) { "age must be non-negative, was $age" }     // IllegalArgumentException
check(state == State.READY) { "wrong state: $state" }         // IllegalStateException
val name = requireNotNull(user.name) { "name was null" }
error("unreachable")                                          // IllegalStateException, returns Nothing
```

The lazy lambda for the message is a nice touch — the string isn't constructed unless the check fails.

## `runCatching` and `Result<T>`

```kotlin
val result: Result<User> = runCatching {
    fetchUser(id)
}

result
    .onSuccess { user -> println(user.name) }
    .onFailure { e -> log.error("failed", e) }
```

Closer in spirit to Rust's `Result` or Scala's `Try`. Java has nothing built-in like this; you'd reach for Vavr or roll your own. Use it carefully — `runCatching` catches all `Throwable`, which can hide bugs. Idiomatic Kotlin still uses try/catch for most cases; `runCatching` shines at API boundaries.

## Null safety eliminates a whole category of exceptions

Kotlin's biggest practical impact on exception handling isn't in the syntax — it's that NullPointerExceptions largely vanish from day-to-day code. The type system distinguishes `String` from `String?`, and you have `?.`, `?:`, `let`, and (the dangerous one) `!!`. In Java you spend real effort defending against null. In Kotlin, most of that moves to compile time.

## But wait — how does this even work?

Here's where it gets interesting. If Kotlin doesn't enforce checked exceptions, what happens when Kotlin code calls Java code declared with a `throws` clause? Is there some hidden runtime catching them all? Some clever wrapping?

**The answer: nothing handles them. They just propagate.**

There's no built-in mechanism in Kotlin that "handles all checked exceptions." Kotlin simply ignores the distinction between checked and unchecked at the language level. The exception still propagates up the stack at runtime exactly like it would in Java — it just doesn't require any syntactic acknowledgment.

```kotlin
fun loadConfig(): String {
    return Files.readString(Paths.get("config.json"))  // throws IOException
}

fun main() {
    val config = loadConfig()  // if IOException is thrown, JVM unwinds the stack
    println(config)
}
```

If `readString` throws, the JVM unwinds the stack and — if nothing catches it — the thread dies with a stack trace, just like an uncaught `RuntimeException` would in Java.

### Why this works: checked-ness is a compiler fiction

Here's the key insight: **the JVM has no concept of checked exceptions.** None. At the bytecode level, `IOException` and `RuntimeException` are indistinguishable in how they propagate. Both are just `Throwable` subclasses that unwind the stack identically.

The "checked" property exists *only* in the Java compiler. `javac` reads the `throws` clause from a method's metadata and refuses to compile callers that don't either catch the exception or declare it themselves. That's it. It's a compile-time enforcement layer on top of a runtime that doesn't care.

When you compile this Java method:

```java
public String readFile(String path) throws IOException { ... }
```

The `throws IOException` is recorded in the class file as an `Exceptions` attribute on the method. `javac` reads that attribute when compiling callers and enforces the rule. The JVM itself doesn't read it for execution purposes — it's purely informational metadata.

Kotlin's compiler reads the same metadata but chooses to ignore it for enforcement. It treats every Java method as if it had no `throws` clause. The bytecode Kotlin generates is identical to what it would generate for an unchecked exception — no special wrapping, no translation, no handler installation.

### What happens at the call site

When Kotlin compiles `Files.readString(...)`, it emits a normal `invokestatic` (or `invokevirtual`) bytecode instruction. There's no try/catch wrapper inserted. There's no `Exceptions` attribute added to the calling Kotlin method. The exception just flows up the stack until something catches it — or reaches the thread's uncaught exception handler and terminates the thread.

### Practical consequences

**You can still catch them.** Just because Kotlin doesn't force you to doesn't mean you can't:

```kotlin
try {
    Files.readString(Paths.get("config.json"))
} catch (e: IOException) {
    // works fine
}
```

**Uncaught checked exceptions kill the thread.** Same as any uncaught exception. No safety net.

**Documentation becomes load-bearing.** Since the compiler won't tell you which Java APIs throw what, you rely on Javadocs and IDE hints. IntelliJ does surface declared exceptions of Java methods at Kotlin call sites, which helps.

**The reverse direction needs `@Throws`.** When Java calls *Kotlin* code, the Java compiler still enforces checked exceptions based on bytecode metadata. If your Kotlin function throws `IOException` and you want Java callers to catch it without an "unreachable catch block" error, annotate it:

```kotlin
@Throws(IOException::class)
fun readConfig(): String { ... }
```

This adds the `Exceptions` attribute to the bytecode so Java's compiler sees it.

## The takeaway

Kotlin's "handling" of checked exceptions is to pretend they don't exist at the language level and let the JVM do what it was always going to do anyway. The whole checked-exception system was a Java-compiler convention from the start, and Kotlin opted out of the convention.

The mental shift for a Java developer: stop thinking about *which exceptions to declare*, and start thinking about *which failures are part of your function's contract* (use `require`/`check` or sealed result types) versus *which are exceptional and should propagate*. The compiler won't push you anymore, so document failure modes in KDoc and let unchecked exceptions fly for genuinely exceptional cases.

For domain errors that callers should reasonably handle, many Kotlin codebases prefer sealed class hierarchies over exceptions entirely:

```kotlin
sealed class FetchResult {
    data class Success(val user: User) : FetchResult()
    data class Error(val message: String) : FetchResult()
}
```

Exceptions become the "didn't expect this" channel, not a control-flow mechanism. It turns out that giving up the safety net forces you to think more carefully about which errors actually matter — which, in the end, might be the whole point.