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

---

# 7. What About `optional = false`?

This is **not related to LAZY or EAGER**.

```java
optional = false
```

means:

> A `UserProfile` must have a `UserAuth`.

In other words, the relationship cannot be absent.

---

# 8. What About `nullable = false`?

This is also separate from LAZY/EAGER.

```java
@JoinColumn(
    name = "user_auth_id",
    nullable = false
)
```

means:

> The `user_auth_id` column in the database cannot contain NULL.

So the database enforces that the relationship must have a value.

---

# 9. What About `unique = true`?

Again, this is separate from fetch type.

```java
@JoinColumn(
    name = "user_auth_id",
    nullable = false,
    unique = true
)
```

`unique = true` means the same `user_auth_id` cannot be used by multiple `UserProfile` rows.

This helps enforce the **one-to-one** relationship at the database level.

Conceptually:

```text
UserProfile              UserAuth

Profile A  ───────────→  Auth A
Profile B  ───────────→  Auth B
Profile C  ───────────→  Auth C
```

You cannot have:

```text
Profile A  ───────────→  Auth A
Profile B  ───────────→  Auth A   ❌
```

because `user_auth_id` is unique.

---

# 10. Understanding Your Full Annotation

Your code:

```java
@OneToOne(
    fetch = FetchType.LAZY,   // WHEN to load UserAuth
    optional = false          // WHETHER the relationship may be absent
)
@JoinColumn(
    name = "user_auth_id",
    nullable = false,         // DB column cannot be NULL
    unique = true             // One UserAuth ↔ One UserProfile
)
private UserAuth userAuth;
```

Each setting has a different responsibility:

| Setting | What it controls |
|---|---|
| `fetch = LAZY` | **When** `UserAuth` is loaded |
| `fetch = EAGER` | Load `UserAuth` immediately |
| `optional = false` | Relationship must exist |
| `nullable = false` | Database column cannot be NULL |
| `unique = true` | Prevent multiple profiles from using the same `UserAuth` |

---

# 11. The Easy Rule to Remember

Think about a restaurant:

| Fetch Type | Meaning |
|---|---|
| `LAZY` | 🍽️ Bring it **when I ask for it** |
| `EAGER` | 🍽️ Bring it **right now** |

A useful general rule:

> **Use LAZY when you don't always need the related entity.**

Then explicitly fetch the related data when a particular operation actually needs it.

---

# Quick Summary

```text
LAZY
 ↓
Load the main entity first
 ↓
Load the relationship only when accessed
```

```text
EAGER
 ↓
Load the main entity
 +
Load the relationship immediately
```

For your `UserProfile → UserAuth` relationship:

```java
@OneToOne(fetch = FetchType.LAZY, optional = false)
@JoinColumn(name = "user_auth_id", nullable = false, unique = true)
private UserAuth userAuth;
```

means:

> `UserProfile` must have a `UserAuth`, the database requires a `user_auth_id`, each `UserAuth` can belong to only one profile, and Hibernate should load `UserAuth` only when it is actually needed.
