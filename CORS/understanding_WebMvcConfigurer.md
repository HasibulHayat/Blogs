<br>
    
# WebMvcConfigurer in Spring Boot — Detailed Beginner Guide

<br>

## 1. What is WebMvcConfigurer?

`WebMvcConfigurer` is a **Java interface provided by Spring**.

Its purpose is to give you a way to **customize Spring MVC**.

Think of it like this:

```text
Spring MVC
    |
    |-- already has default behavior
    |
    ↓
WebMvcConfigurer
    |
    |-- gives you ways to customize that behavior
    |
    ├── CORS
    ├── Interceptors
    ├── Resource handling
    ├── View controllers
    ├── Path matching
    └── Other MVC behavior
```

You are normally **not building Spring MVC yourself** when you use this interface.

You are saying:

> ### "Spring, keep your normal MVC behavior, but I want to customize these particular parts."

<br>
<br>


# 2. First: what is Spring MVC?

<br>

Spring MVC is the part of Spring that handles web requests.

For example:

```http
GET /api/v1/units
```

A simplified flow is:

```text
HTTP Request
     |
     ↓
Spring MVC
     |
     ↓
Find the correct controller
     |
     ↓
Controller method
     |
     ↓
Service
     |
     ↓
Response
```

Spring MVC already knows how to do many things, such as:

- Map URLs to controller methods
- Handle HTTP methods
- Read request bodies
- Convert JSON
- Write JSON responses
- Work with interceptors
- Handle CORS configuration
- Match request paths
- Integrate with exception handling

`WebMvcConfigurer` gives you customization points for some of these areas.

<br>
<br>


# 3. What is an interface in Java?

<br>

An interface is a **contract** that a class can implement.

For example:

```java
public interface Animal {

    void makeSound();
}
```

A class can implement it:

```java
public class Dog implements Animal {

    @Override
    public void makeSound() {
        System.out.println("Woof");
    }
}
```

The interface says:

> ### "A class implementing me must provide the required behavior."

> ### But `WebMvcConfigurer` has an important feature: many of its methods are **default methods**.

<br>
<br>


# 4. What is a default method?

<br>

Java interfaces can contain methods with the `default` keyword.

For example:

```java
public interface MyInterface {

    default void sayHello() {
        System.out.println("Hello");
    }
}
```

A class can implement the interface without writing `sayHello()`:

```java
public class MyClass implements MyInterface {
}
```

It gets the default implementation.

Or the class can override it:

```java
public class MyClass implements MyInterface {

    @Override
    public void sayHello() {
        System.out.println("Hello from MyClass");
    }
}
```

This is the important idea behind `WebMvcConfigurer`.

<br>
<br>


# 5. WebMvcConfigurer uses default methods

<br>

Conceptually, the interface contains methods like:

```java
public interface WebMvcConfigurer {

    default void addCorsMappings(CorsRegistry registry) {
    }

    default void addInterceptors(InterceptorRegistry registry) {
    }

    default void addResourceHandlers(ResourceHandlerRegistry registry) {
    }

    default void addViewControllers(ViewControllerRegistry registry) {
    }

    default void configurePathMatch(PathMatchConfigurer configurer) {
    }

    // Other methods...
}
```

This is a **simplified representation**, not the complete source code.

The important part is:

```java
default
```

Because the methods have defaults, you don't have to implement all of them.

You can override only the method you need.

<br>
<br>


# 6. Your WebConfig class

<br>

A typical configuration class looks like:

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
}
```

There are two important parts.

## `@Configuration`

```java
@Configuration
```

This tells Spring:

> "This class contains configuration for my application."

Spring discovers this class during application startup.


## `implements WebMvcConfigurer`

```java
implements WebMvcConfigurer
```

This tells Java:

> "My WebConfig class implements the WebMvcConfigurer interface."

It also allows your class to provide Spring MVC customizations.

<br>
<br>


# 7. The CORS method

<br>

One commonly used method is:

```java
addCorsMappings(CorsRegistry registry)
```

For example:

```java
@Override
public void addCorsMappings(CorsRegistry registry) {

    registry.addMapping("/**")
            .allowedOrigins("https://example.com")
            .allowedMethods(
                    "GET",
                    "POST",
                    "PUT",
                    "DELETE",
                    "OPTIONS"
            )
            .allowedHeaders("*");
}
```

This means:

> "Spring MVC, here are my CORS rules."

<br>
<br>


# 8. What is CorsRegistry?

<br>

Look at:

```java
public void addCorsMappings(CorsRegistry registry)
```

The parameter:

```java
CorsRegistry registry
```

is an object supplied by Spring.

Think of it as a **place where you register your CORS rules**.

You can tell it:

```text
Which URLs?
Which origins?
Which methods?
Which headers?
```

For example:

```java
registry.addMapping("/**")
```

means:

> Apply this CORS configuration to all matching paths.

Then:

```java
.allowedOrigins("https://example.com")
```

means:

> Allow requests from this origin.

Then:

```java
.allowedMethods("GET", "POST")
```

means:

> Allow these HTTP methods.

<br>
<br>


# 9. Another WebMvcConfigurer method: interceptors

<br>

You can customize interceptors:

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {

    registry.addInterceptor(myInterceptor);
}
```

An interceptor can run around requests.

Conceptually:

```text
Request
   |
   ↓
Interceptor
   |
   ↓
Controller
   |
   ↓
Response
```

This can be useful for request-related processing, logging, and other cross-cutting behavior.

<br>
<br>


# 10. Resource handlers

<br>

Another customization method is:

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {

    registry.addResourceHandler("/images/**")
            .addResourceLocations("classpath:/images/");
}
```

This can tell Spring MVC how to serve certain resources.

Conceptually:

```text
Browser requests:

/images/logo.png

        ↓

Spring MVC

        ↓

Look in:

classpath:/images/
```

<br>
<br>


# 11. View controllers

<br>

You can also configure simple view mappings:

```java
@Override
public void addViewControllers(ViewControllerRegistry registry) {

    registry.addViewController("/home")
            .setViewName("home");
}
```

This can map a URL directly to a view without requiring a full controller method.

Conceptually:

```text
/home
  ↓
"home" view
```

<br>
<br>


# 12. Path matching

<br>

Another customization point is:

```java
@Override
public void configurePathMatch(PathMatchConfigurer configurer) {

    // customize path matching
}
```

This allows customization of how Spring MVC matches incoming URLs to controller mappings.

For example:

```text
Incoming request:

GET /api/v1/units/123

        ↓

Spring MVC determines:

Which controller method handles this?
```

Path matching configuration can influence that behavior.

<br>
<br>


# 13. What happens when the application starts?

<br>

A simplified startup process looks like:

```text
Spring Boot starts
       |
       ↓
Spring creates application context
       |
       ↓
Spring discovers @Configuration classes
       |
       ↓
Spring finds WebConfig
       |
       ↓
Spring sees:
implements WebMvcConfigurer
       |
       ↓
Spring uses the configuration methods
       |
       ↓
MVC is configured
       |
       ↓
Application starts
```

So your `WebConfig` is primarily part of **application configuration/setup**.

It isn't something that your frontend directly calls.

<br>
<br>


# 14. The whole thing in one picture

<br>

```text
                    Spring Boot
                        |
                        ↓
                   Spring MVC
                        |
              Default MVC behavior
                        |
                        |
             WebMvcConfigurer
                        |
             "Customize this..."
                        |
        ┌───────────────┼────────────────┐
        ↓               ↓                ↓
       CORS        Interceptors      Resources
        |
        ↓
addCorsMappings()
        |
        ↓
CorsRegistry
        |
        ├── addMapping()
        ├── allowedOrigins()
        ├── allowedMethods()
        └── allowedHeaders()
```

<br>
<br>


# 15. Key things to remember

<br>

### `WebMvcConfigurer`

A Spring interface used to **customize Spring MVC**.

### `implements`

```java
implements WebMvcConfigurer
```

means your class implements the interface and can provide MVC customizations.

### `@Override`

```java
@Override
```

means you're overriding a method provided by the interface.

### `default`

Many methods in the interface have default implementations, so you don't need to implement everything.

### `addCorsMappings()`

A customization method specifically for configuring CORS.

### `CorsRegistry`

The object you use to register CORS rules.

<br>
<br>


# 16. The simplest mental model

<br>

Remember this:

```text
Spring MVC
    ↓
Already works with defaults
    ↓
WebMvcConfigurer
    ↓
"I want to customize something."
    ↓
Override only the method you need
```

For your CORS case:

```text
WebMvcConfigurer
       ↓
addCorsMappings()
       ↓
CorsRegistry
       ↓
CORS rules
       ↓
Spring MVC uses those rules
       ↓
Browser enforces the cross-origin policy
```

<br>

## Final takeaway

The most important sentence is:

> **`WebMvcConfigurer` is a Spring interface that lets your application customize Spring MVC without replacing Spring MVC's built-in behavior.**
