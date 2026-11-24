# Java Concurrency --- Core Concepts

# Lesson 10: Immutability and Thread Safety

## 1. What is immutability?

An object is **immutable** when its state cannot be changed after it has
been created.

For example, imagine a `User` object whose name and email never change
after construction.

``` java
public final class User {
    private final String name;
    private final String email;

    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public String getName() {
        return name;
    }

    public String getEmail() {
        return email;
    }
}
```

Notice:

-   The class is `final`, so it cannot be subclassed.
-   The fields are `private` and `final`.
-   There are no setter methods.
-   The constructor sets the fields.

This is a simple example of an immutable class.

## 2. Why is immutability useful in concurrency?

Imagine two threads reading the same object.

If the object cannot change, neither thread has to worry that the other
thread will modify its state.

That means:

-   Threads can safely read the object's state concurrently.
-   There is no race condition caused by changing that state.
-   You often do not need `synchronized` just to read the immutable
    object's fields.

**Important:** Immutability does not automatically solve every
concurrency problem. The object must be designed correctly and safely
made available to other threads.

## 3. How to create an immutable class

A useful starting checklist:

1.  Make the class `final`, unless you have a carefully designed reason
    not to.
2.  Make fields `private final`.
3.  Initialize fields in the constructor.
4.  Do not provide setters.
5.  Do not expose mutable internal objects directly.
6.  If a field refers to a mutable object, make defensive copies when
    needed.

For example, this class is not fully immutable:

``` java
public class Team {
    private final List<String> players;

    public Team(List<String> players) {
        this.players = players;
    }

    public List<String> getPlayers() {
        return players;
    }
}
```

The caller can still change the list after passing it in, or change it
through the getter.

A safer version is:

``` java
public final class Team {
    private final List<String> players;

    public Team(List<String> players) {
        this.players = List.copyOf(players);
    }

    public List<String> getPlayers() {
        return players;
    }
}
```

`List.copyOf()` creates an unmodifiable copy of the list. This protects
the list structure; if the elements themselves are mutable objects, they
may need their own defensive copying too.

## 4. Immutability and thread safety

Thread safety means an object behaves correctly when multiple threads
use it at the same time.

Immutability is one of the easiest ways to make an object safe for
concurrent reading because its state does not change after construction.

Other approaches include:

-   `synchronized`
-   Atomic classes
-   Concurrent collections
-   Carefully designed thread confinement

Use the simplest approach that fits the problem.

## 5. Why does this matter in the Turf project?

Some objects in a booking application represent information that should
not change after creation.

Examples:

-   A booking confirmation snapshot
-   A payment receipt representation
-   A request containing validated booking details
-   A configuration snapshot

Making these objects immutable can reduce accidental changes and make
them easier to share between threads.

However, a booking's live status changes over time. Do not try to make a
changing business entity immutable without considering how those state
transitions will be handled.

## 6. Quick recap

-   Immutable objects cannot change state after construction.
-   Immutability makes concurrent reading much simpler.
-   Use private final fields, constructor initialization, and no
    setters.
-   Protect mutable objects passed into or returned from your class.
-   Immutability is a useful way to achieve thread safety, but it is not
    the only way.

## 7. Check your understanding

1.  What does immutable mean?
2.  Why are immutable objects useful when multiple threads read them?
3.  Why might a `final List` field still allow the list contents to
    change?
4.  Name two other ways to protect shared mutable state.

------------------------------------------------------------------------

# Lesson 11: Safe Publication

## 1. What is publication?

**Publication** means making an object available to other parts of your
application, such as another thread.

For example:

``` java
User user = new User("Sam", "sam@example.com");
sharedUser = user;
```

If another thread can access `sharedUser`, the object has been published
to that thread.

## 2. What does safe publication mean?

Safe publication means making an object available to other threads in a
way that guarantees they can see its correctly initialized state.

Why does this matter?

Imagine one thread creates an object and another thread reads it.
Without a proper coordination mechanism, the second thread is not
guaranteed to observe the object's state as expected.

Safe publication establishes the required visibility and ordering
guarantees.

## 3. Common ways to safely publish an object

### Method 1: Use a volatile reference

``` java
class UserStore {
    private volatile User user;

    public void publish(User newUser) {
        user = newUser;
    }

    public User getUser() {
        return user;
    }
}
```

A write to the volatile reference happens-before a later read of that
same reference.

This safely publishes the object state established before the reference
was written, provided the object is not then modified without additional
coordination.

### Method 2: Use synchronized

``` java
class UserStore {
    private User user;

    public synchronized void publish(User newUser) {
        user = newUser;
    }

    public synchronized User getUser() {
        return user;
    }
}
```

Both methods use the same object's lock. The synchronization provides
the visibility guarantee between publishing and reading.

### Method 3: Use a static initializer

``` java
class AppConfig {
    static final User DEFAULT_USER =
            new User("Sam", "sam@example.com");
}
```

Objects initialized through class initialization are safely published
when the class is initialized.

### Method 4: Use a thread-safe collection

Thread-safe utilities such as `ConcurrentHashMap` provide the required
coordination for their operations.

For example:

``` java
Map<String, User> users = new ConcurrentHashMap<>();

users.put("sam", new User("Sam", "sam@example.com"));
```

A thread-safe collection helps safely share values placed into it
through its documented operations. It does not make later unsynchronized
mutations inside those values safe.

## 4. Does using final fields help?

Yes. Java gives special initialization guarantees to `final` fields when
an object is constructed correctly.

But `final` fields are not a complete replacement for safe publication.

If other threads share an object, use a recognized publication mechanism
such as a volatile reference, synchronization, class initialization, or
a thread-safe collection.

## 5. Safe publication in Spring Boot

Spring commonly manages singleton beans and makes them available to
request-handling threads.

That does not mean every field inside a singleton bean is automatically
thread-safe.

For example:

``` java
@Service
public class BookingService {
    private int requestCount = 0;

    public void book() {
        requestCount++;
    }
}
```

Multiple requests may execute `book()` concurrently. The
`requestCount++` operation is not automatically safe just because Spring
created the service bean.

Safe publication of the bean and thread safety of its mutable fields are
two different concerns.

## 6. Why does this matter in the Turf project?

Suppose your application loads a read-only booking configuration at
startup and shares it with request threads.

You need to ensure that:

1.  The configuration is fully initialized.
2.  It is safely published to the threads that use it.
3.  It is not later modified unsafely.

For actual booking availability and preventing double bookings, safe
publication is not enough. You will still need database transactions,
constraints, and suitable locking because multiple application instances
may handle requests.

## 7. Quick recap

-   Publication means making an object available to other threads.
-   Safe publication guarantees that other threads can see the object's
    properly initialized state.
-   Common techniques include `volatile`, `synchronized`, static
    initialization, and thread-safe collections.
-   `final` fields provide useful initialization guarantees, but do not
    replace every publication requirement.
-   A Spring singleton bean can be safely created and still contain
    fields that are not thread-safe.

## 8. Check your understanding

1.  What does it mean to publish an object?
2.  What problem does safe publication prevent?
3.  Name three ways to safely publish an object.
4.  Does a Spring singleton bean automatically make all its mutable
    fields thread-safe?
5.  Why is safe publication alone insufficient to prevent double
    bookings across multiple application instances?

------------------------------------------------------------------------

## Final connection between Lessons 10 and 11

These lessons solve two related but different problems:

-   **Immutability** helps ensure an object's state cannot be changed
    after construction.
-   **Safe publication** ensures other threads can see an object's
    correctly initialized state.

A well-designed immutable object that is safely published is often
straightforward to share between threads.
