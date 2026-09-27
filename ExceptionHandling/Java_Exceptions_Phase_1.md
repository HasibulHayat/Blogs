<br>

# Java Exceptions — Phase 1

<br>
<br>

This note combines following lessons:

1. What an exception is, why exceptions happen, and the Java exception hierarchy.
2. Checked vs unchecked exceptions and common Java exceptions.

<br>

## What Is an Exception?

An **exception is a problem that happens while your program is running and interrupts the normal flow of the program.**

```java
int result = 10 / 0;
```

Java cannot divide a number by zero, so it throws:

```text
ArithmeticException
```

Simple definition:

> An exception is an unexpected condition that interrupts the normal execution of your program.

<br>

## Why do exceptions happen?

Exceptions happen when something occurs that the program cannot continue with normally.

<br>

### Dividing by zero

```java
int result = 10 / 0;
```

### Using something that is null

```java
String name = null;

name.length();
```

### Accessing an invalid array position

```java
int[] numbers = {10, 20, 30};

System.out.println(numbers[5]);
```

There is no position `5`, so Java throws an exception.

<br>

## Exception vs Error

Java has:

```text
Exception
Error
```

Both are related to `Throwable`, but they represent different kinds of problems.

<br>

### Exception

Usually a problem that an application can reasonably deal with.

Examples:

```text
Invalid input
Missing resource
Bad argument
Database-related problem
User not found
```

### Error

Usually a more serious problem involving the JVM/runtime or environment.

Examples:

```text
OutOfMemoryError
StackOverflowError
```

Simple mental model:

```text
Exception
    ↓
Application-level problem
    ↓
Often something we can handle

Error
    ↓
Serious JVM/runtime problem
    ↓
Usually not something we try to recover from
```

<br>

## The Exception Hierarchy

At the top is:

```text
Throwable
```

Simplified:

```text
              Throwable
              /                    /                 Exception      Error
```

More detail:

```text
Throwable
│
├── Exception
│   │
│   ├── RuntimeException
│   │   ├── NullPointerException
│   │   ├── IllegalArgumentException
│   │   ├── IllegalStateException
│   │   └── ...
│   │
│   └── Other exceptions
│
└── Error
    ├── OutOfMemoryError
    ├── StackOverflowError
    └── ...
```

You do not need to memorize the entire hierarchy. Understand the relationships.

<br>

## What is `Throwable`?

`Throwable` is the top-level class for things that can be thrown by Java.

```text
Throwable
├── Exception
└── Error
```

Both `Exception` and `Error` are descendants of `Throwable`.

Although Java technically allows:

```java
try {
    // something
} catch (Throwable t) {
    // handle
}
```

you generally should **not catch `Throwable` in normal application code**.

<br>

## Catch Specific Exceptions When Possible

Compare:

```java
catch (UserNotFoundException e) {
    ...
}
```

with:

```java
catch (Exception e) {
    ...
}
```

The first says:

> "I specifically know how to handle a user-not-found situation."

The second says:

> "Almost any exception that reaches here, I'll handle the same way."

For real backend code, understanding the exact problem is important.

<br>

## Checked vs Unchecked

The easiest way to remember the difference:

> **Checked exception = Java forces you to deal with it.**

> **Unchecked exception = Java doesn't force you to deal with it.**

<br>

# Checked Exceptions

A checked exception is an exception that the compiler requires you to explicitly handle or declare.

Example:

```java
FileReader reader = new FileReader("data.txt");
```

Opening a file can fail because the file may not exist.

You can handle it:

```java
try {
    FileReader reader = new FileReader("data.txt");
} catch (FileNotFoundException e) {
    System.out.println("File doesn't exist");
}
```

Or declare it:

```java
public void readFile() throws FileNotFoundException {
    FileReader reader = new FileReader("data.txt");
}
```

Otherwise, the code will not compile.

<br>

# Why are they called "checked"?

Because the **compiler checks your code**.

Java sees an operation that can throw a checked exception and essentially says:

> "This can fail. What are you going to do about it?"

You must either:

### Handle it

```java
try {
    ...
} catch (FileNotFoundException e) {
    ...
}
```

or:

### Declare it

```java
throws FileNotFoundException
```

<br>

## Unchecked Exceptions

Unchecked exceptions are generally subclasses of:

```java
RuntimeException
```

Examples:

```text
RuntimeException
├── NullPointerException
├── IllegalArgumentException
├── IllegalStateException
└── IndexOutOfBoundsException
```

Java does not force you to catch these.

For example:

```java
String name = null;

name.length();
```

produces:

```text
NullPointerException
```

but your code can still compile without a `try/catch`.

That's why it is **unchecked**.

<br>

## Why doesn't Java force us to catch RuntimeExceptions?

Many RuntimeExceptions indicate programming mistakes.

For example:

```java
String name = null;

name.length();
```

Instead of always writing:

```java
try {
    name.length();
} catch (NullPointerException e) {
    ...
}
```

you normally fix the underlying problem.

For example:

```java
String name = getName();

if (name != null) {
    name.length();
}
```

<br>

## The Important Hierarchy

```text
Throwable
│
├── Exception
│   │
│   ├── Checked exceptions
│   │
│   └── RuntimeException
│       │
│       ├── NullPointerException
│       ├── IllegalArgumentException
│       ├── IllegalStateException
│       └── IndexOutOfBoundsException
│
└── Error
```

Key distinction:

```text
Exception
├── Checked exceptions
└── RuntimeException
      ↑
      Unchecked exceptions
```

Easy rule:

```text
Checked
   ↓
Compiler says:
"You MUST handle or declare this."

Unchecked
   ↓
Compiler says:
"You don't HAVE to handle this."
```

<br>

# Part 3 — Common Java Exceptions

## 16. `NullPointerException`

Short name:

```text
NPE
```

It happens when you try to use something that is `null` as if it were an actual object.

```java
String name = null;

System.out.println(name.length());
```

Here:

```text
name
 ↓
null
```

Then:

```java
name.length()
```

Java says:

> "You told me to call `length()` on an object, but there is no object."

So:

```text
NullPointerException
```

### Easy analogy

Imagine you have a box, but the box doesn't exist.

Then you say:

> "Open the box."

Java says:

> "What box?"

---

## 17. Another NPE example

```java
User user = null;

user.getEmail();
```

You get:

```text
NullPointerException
```

because:

```text
user → null
```

and you're trying to call `getEmail()` on nothing.

---

# 18. `IllegalArgumentException`

This means:

> **You passed an invalid argument to a method.**

Example:

```java
public void setAge(int age) {

    if (age < 0) {
        throw new IllegalArgumentException(
            "Age cannot be negative"
        );
    }
}
```

Then:

```java
setAge(-10);
```

The method says:

> "You gave me an argument that isn't valid."

So:

```text
IllegalArgumentException
```

### Easy analogy

A vending machine accepts:

```text
$1
$2
$5
```

You give it a rock.

It says:

> "That's not a valid input."

---

# 19. `IllegalStateException`

This means:

> **The object or application is currently in an invalid state for the operation you're trying to perform.**

Example:

```java
public class Payment {

    private boolean completed;

    public void refund() {

        if (!completed) {
            throw new IllegalStateException(
                "Cannot refund an incomplete payment"
            );
        }

        // refund...
    }
}
```

Suppose:

```text
Payment
   ↓
completed = false
```

Then:

```java
payment.refund();
```

The problem isn't the argument.

The problem is:

> "This object isn't currently in the right state for this operation."

So:

```text
IllegalStateException
```

---

# 20. IllegalArgumentException vs IllegalStateException

### IllegalArgumentException

Problem with **what you gave me**.

Example:

```java
setAge(-5);
```

```text
Argument → What YOU gave me
```

### IllegalStateException

Problem with **the current state**.

Example:

```java
payment.refund();
```

when the payment isn't completed.

```text
State → What I currently am
```

Easy memory trick:

> **Argument = input problem.**
>
> **State = current-condition problem.**

---

# 21. `IndexOutOfBoundsException`

This happens when you try to access something outside its valid range.

```java
List<String> names = List.of(
    "Hasibul",
    "Rahim",
    "Karim"
);

System.out.println(names.get(10));
```

The list has:

```text
0 → Hasibul
1 → Rahim
2 → Karim
```

There is no position `10`.

So Java throws:

```text
IndexOutOfBoundsException
```

For arrays, you may specifically see:

```text
ArrayIndexOutOfBoundsException
```

---

# 22. `NumberFormatException`

This happens when Java tries to convert text into a number, but the text isn't a valid number.

```java
int age = Integer.parseInt("hello");
```

Java asks:

> "How am I supposed to convert `hello` into an integer?"

So:

```text
NumberFormatException
```

But:

```java
int age = Integer.parseInt("25");
```

works:

```text
"25" → 25
```

---

# 23. `ClassCastException`

This happens when you try to treat an object as an incompatible type.

```java
Object value = "Hello";

Integer number = (Integer) value;
```

But:

```text
value = String
```

not:

```text
Integer
```

So Java throws:

```text
ClassCastException
```

---

# 24. `ArithmeticException`

Example:

```java
int result = 10 / 0;
```

Java throws:

```text
ArithmeticException
```

This is another `RuntimeException`.

---

# 25. `NoSuchElementException`

You may encounter this with Java collections and `Optional`.

Example:

```java
Optional<User> user = Optional.empty();

User result = user.get();
```

There is no user inside the `Optional`.

So:

```text
NoSuchElementException
```

In Spring Boot code, you'll often see:

```java
repository.findById(id)
    .orElseThrow(...);
```

This is generally preferable because you can control the exception that gets thrown.

For example:

```java
User user = repository.findById(id)
    .orElseThrow(() ->
        new UserNotFoundException("User not found")
    );
```

This produces a meaningful application exception instead of an accidental `NoSuchElementException`.

---

# 26. Common Exception Reference

| Exception | Simple meaning | Example |
|---|---|---|
| `NullPointerException` | You're using `null` like an object | `user.getName()` when `user == null` |
| `IllegalArgumentException` | You gave a method an invalid argument | `setAge(-5)` |
| `IllegalStateException` | Object is in the wrong state | Refund before payment completes |
| `IndexOutOfBoundsException` | Invalid list/index position | `list.get(10)` when size is 3 |
| `NumberFormatException` | Text can't be converted to a number | `Integer.parseInt("hello")` |
| `ClassCastException` | Invalid type conversion | Casting `String` to `Integer` |
| `ArithmeticException` | Invalid arithmetic operation | `10 / 0` |
| `NoSuchElementException` | Expected an element but none exists | `Optional.empty().get()` |

---

# 27. Notice Something Important

Most of these common exceptions are:

```text
RuntimeException
│
├── NullPointerException
├── IllegalArgumentException
├── IllegalStateException
├── IndexOutOfBoundsException
├── NumberFormatException
├── ClassCastException
├── ArithmeticException
└── NoSuchElementException
```

That's why they are **unchecked exceptions**.

Java doesn't force you to write `try/catch` around them.

---

# 28. What Should You Do When You Encounter an Exception?

Don't immediately think:

> "I need to catch it."

Instead ask:

> **"Why did this happen?"**

For example:

```java
user.getEmail();
```

causes:

```text
NullPointerException
```

Don't automatically do:

```java
try {
    user.getEmail();
} catch (NullPointerException e) {
    ...
}
```

Instead ask:

> "Why is `user` null?"

Maybe the correct solution is:

```java
User user = repository.findById(id)
    .orElseThrow(() ->
        new UserNotFoundException("User not found")
    );
```

Now the application has a meaningful problem:

```text
UserNotFoundException
```

rather than an accidental:

```text
NullPointerException
```

This mindset is very important in backend development.

---

# 29. The Big Picture

```text
                    Throwable
                   /                           /                     Exception            Error
             |
       RuntimeException
        /      |              /       |              ↓        ↓         ↓
    Null     Illegal    Illegal
  Pointer    Argument   State
 Exception   Exception  Exception
```

And:

```text
Checked Exception
    ↓
Compiler requires handling/declaring it

Unchecked Exception
    ↓
Usually RuntimeException
    ↓
Compiler doesn't require handling
```

---

# 30. The Four Most Important Ideas

You don't need to memorize every exception right now.

Remember these:

### ① Exception

> Something went wrong while the program was running.

### ② Checked exception

> Java's compiler forces you to handle or declare it.

### ③ Unchecked exception

> Usually a `RuntimeException`; Java doesn't force you to handle it.

### ④ Common exceptions have meaning

```text
NullPointerException
→ "You're using null."

IllegalArgumentException
→ "You gave me a bad argument."

IllegalStateException
→ "I'm currently in the wrong state."

IndexOutOfBoundsException
→ "That position doesn't exist."
```

---

# Quick Mental Model

Think of exceptions as messages from your program:

```text
NullPointerException
    → "There is nothing here."

IllegalArgumentException
    → "You gave me something invalid."

IllegalStateException
    → "I'm not ready/in the right state for this."

IndexOutOfBoundsException
    → "That position doesn't exist."

NumberFormatException
    → "This text isn't a valid number."

ClassCastException
    → "This object isn't that type."

ArithmeticException
    → "That mathematical operation isn't valid."

NoSuchElementException
    → "You expected something, but there isn't anything there."
```

---

# Phase 1 Complete

The foundation is:

```text
What is an exception?
        ↓
Why exceptions happen
        ↓
Exception vs Error
        ↓
Throwable
        ↓
Exception
        ↓
RuntimeException
        ↓
Checked vs Unchecked
        ↓
Common exceptions
```

The next phase is **Phase 2 — Handling Exceptions in Java**:

```text
try
 ↓
catch
 ↓
finally

throw
 ↓
throws

Exception propagation
 ↓
Custom exceptions
```

This will prepare you for Spring Boot's `@ExceptionHandler`, `@RestControllerAdvice`, and eventually the exception handling implemented in the Woodland backend.
