<br>

# JPA Fetch Type — LAZY vs EAGER

<br>
<br>

## The Basic Idea

In JPA/Hibernate, `fetch` controls **when a related entity is loaded from the database**.

Think of it like this:

> **LAZY = "Give it to me only when I ask."**  
> **EAGER = "Give it to me immediately."**

<br>

## Example Scenario

```java
@OneToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "user_auth_id", nullable = false, unique = true)
private UserAuth userAuth;
```

Let's say, `UserProfile` has a relationship with `UserAuth`:

```text
UserProfile
    |
    | user_auth_id
    ↓
UserAuth
```

<br>

### 1. LAZY Loading

```java
fetch = FetchType.LAZY
```

LAZY means:

> When I load `UserProfile`, don't immediately load `UserAuth`.

For example:

```java
UserProfile profile = userProfileRepository.findById(id);
```

Conceptually, Hibernate first loads:

```sql
SELECT * FROM user_profile WHERE id = ?
```

It does **not** immediately need to load `UserAuth`.

Then later, if you do:

```java
profile.getUserAuth().getEmail();
```

Hibernate realizes that you actually need `UserAuth` and loads it:

```sql
SELECT * FROM user_auth WHERE id = ?
```

<br>

### 2. EAGER Loading

If you use:

```java
@OneToOne(fetch = FetchType.EAGER)
```

you're telling Hibernate:

> Whenever you load `UserProfile`, load `UserAuth` too.

So:

```java
UserProfile profile = userProfileRepository.findById(id);
```

means Hibernate needs both:

```text
UserProfile
+
UserAuth
```

<br>


## Why LAZY Is Usually Preferred

Imagine your `UserProfile` eventually has:

```java
private UserAuth userAuth;
private Client client;
private Address address;
private Department department;
```

With EAGER relationships, loading one profile can cause Hibernate to load a lot of related data that you may not even need.

For example, you might only need:

```java
profile.getFirstName();
profile.getLastName();
```

But EAGER relationships could cause related entities to be loaded too.

Conceptually:

```text
UserProfile
 ├── UserAuth
 ├── Client
 ├── Address
 └── Department
```

With LAZY:

```text
UserProfile
```

gets loaded first.

Only when you actually access a relationship does Hibernate load that related data.

<br>

## LAZY Does NOT Mean "Never Load"

This is very important.

**LAZY does not mean:**

> "Hibernate will never load `UserAuth`."

It means:

> "Hibernate will wait until you actually need `UserAuth`."

For example:

```java
UserProfile profile = repository.findById(id);
```

At this point:

```text
UserProfile ✅ loaded
UserAuth    ⏳ not loaded
```

Then:

```java
profile.getUserAuth().getEmail();
```

Now:

```text
UserProfile ✅ loaded
UserAuth    ✅ loaded
```

So LAZY means **load later when needed**, not **never load**.

<br>

## The Common LAZY Problem: LazyInitializationException

One common issue with LAZY loading is:

```text
LazyInitializationException
```

This can happen when Hibernate needs to load the relationship but the Hibernate session/transaction has already ended.

For example:

```java
@Transactional
public UserProfile getProfile(UUID id) {

    UserProfile profile = repository.findById(id).orElseThrow();

    return profile;
}
```

After the transaction/session ends, you might later do:

```java
profile.getUserAuth().getEmail();
```

Hibernate may no longer have an active session with which to load `UserAuth`.

That can result in:

```text
LazyInitializationException
```

### How is this normally handled?

In a Spring Boot application, common approaches include:

- Proper service-layer transaction boundaries
- `JOIN FETCH`
- `@EntityGraph`
- DTO/projection queries
- Explicitly loading the required relationship inside the transaction

Usually, it is better to solve the specific data-loading requirement rather than simply changing everything to EAGER.

<br>
<br>


