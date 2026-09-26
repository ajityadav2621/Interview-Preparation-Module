# When Would You Use Optional, and When Should You Avoid It?

## What is Optional?

`Optional<T>` is a container object which may or may not contain a non-null value. It was introduced in Java 8 as part of the functional programming paradigm to reduce `NullPointerException` and encourage more expressive code.

## When to Use Optional

### 1. Return Type for Methods That Might Not Return a Value

```java
// BAD: Returns null — caller must check
public User findUser(String email) {
    return database.findByEmail(email);  // May return null
}

// Usage:
User user = findUser("alice@example.com");
if (user != null) {  // Null check required
    user.getName();
}

// GOOD: Returns Optional — explicit about the possibility of absence
public Optional<User> findUser(String email) {
    return Optional.ofNullable(database.findByEmail(email));
}

// Usage:
Optional<User> userOpt = findUser("alice@example.com");
userOpt.ifPresent(user -> System.out.println(user.getName()));
```

### 2. Chaining Operations Without Null Checks

```java
// BAD: Multiple null checks
public String getUserCity(User user) {
    if (user != null) {
        if (user.getAddress() != null) {
            if (user.getAddress().getCity() != null) {
                return user.getAddress().getCity().getName();
            }
        }
    }
    return "Unknown";
}

// GOOD: Fluent chaining with Optional
public Optional<String> getUserCity(User user) {
    return Optional.ofNullable(user)
        .map(User::getAddress)
        .map(Address::getCity)
        .map(City::getName)
        .orElse("Unknown");
}
```

### 3. Default Values

```java
// BAD: Verbose null check
String name = user != null ? user.getName() : "Anonymous";

// GOOD: Clean with Optional
String name = Optional.ofNullable(user)
    .map(User::getName)
    .orElse("Anonymous");
```

### 4. Throwing Custom Exceptions

```java
public User findUserOrThrow(String email) {
    return findUser(email)
        .orElseThrow(() -> new UserNotFoundException("User not found: " + email));
}
```

### 5. Filtering and Transforming

```java
Optional<String> result = Optional.ofNullable(input)
    .filter(s -> s.length() > 5)
    .map(String::toUpperCase)
    .orElse("DEFAULT");
```

## When to Avoid Optional

### 1. As a Field in Serializable Classes

```java
// BAD: Optional is not serializable
public class User implements Serializable {
    private Optional<String> email;  // Don't do this!
}

// GOOD: Use nullable field
public class User implements Serializable {
    private String email;  // May be null
}
```

### 2. As a Parameter Type

```java
// BAD: Don't use Optional as a parameter
public void processUser(Optional<User> user) {
    // Awkward API
}

// GOOD: Use overloaded methods or nullable parameters
public void processUser(User user) {
    // user may be null
}

public void processUser(User user, String defaultValue) {
    // Explicit default
}
```

### 3. In Performance-Critical Code

```java
// BAD: Optional adds overhead in tight loops
for (int i = 0; i < 1000000; i++) {
    Optional<String> value = Optional.ofNullable(map.get(i));
    // ...
}

// GOOD: Direct null check for performance
for (int i = 0; i < 1000000; i++) {
    String value = map.get(i);
    if (value != null) {
        // ...
    }
}
```

### 4. As a Return Type for Collections

```java
// BAD: Don't wrap collections in Optional
public Optional<List<User>> findAllUsers() {
    return Optional.ofNullable(userRepository.findAll());
}

// GOOD: Return empty collection instead
public List<User> findAllUsers() {
    return userRepository.findAll();  // Returns empty list, not null
}
```

### 5. In Domain Models

```java
// BAD: Optional in domain model
public class Order {
    private Optional<Customer> customer;  // Don't do this
    private Optional<List<Item>> items;  // Don't do this
}

// GOOD: Use nullable fields or empty collections
public class Order {
    private Customer customer;  // May be null
    private List<Item> items = new ArrayList<>();  // Never null
}
```

## Optional Best Practices

### 1. Always Use Static Factory Methods

```java
Optional<String> empty = Optional.empty();           // Empty Optional
Optional<String> of = Optional.of("value");          // Non-null value
Optional<String> nullable = Optional.ofNullable(value); // May be null
```

### 2. Prefer `ifPresent()` Over `isPresent()` + `get()`

```java
// BAD:
if (optional.isPresent()) {
    System.out.println(optional.get());
}

// GOOD:
optional.ifPresent(System.out::println);
```

### 3. Use `orElse()` vs `orElseGet()` Appropriately

```java
// orElse() — always evaluates the default value
String result1 = optional.orElse(expensiveOperation());

// orElseGet() — only evaluates if Optional is empty
String result2 = optional.orElseGet(() -> expensiveOperation());
```

### 4. Don't Use `get()` Without Checking

```java
// BAD:
String value = optional.get();  // Throws NoSuchElementException if empty

// GOOD:
String value = optional.orElse("default");
// or
String value = optional.orElseThrow(() -> new IllegalStateException());
```

### 5. Chain with `map()` and `flatMap()`

```java
// map() — transforms the value
Optional<String> upper = Optional.ofNullable(name)
    .map(String::toUpperCase);

// flatMap() — for Optional-returning functions
Optional<String> street = Optional.ofNullable(user)
    .flatMap(User::getAddress)  // getAddress returns Optional<Address>
    .map(Address::getStreet);
```

## Common Patterns

### Pattern 1: Safe Navigation

```java
// Instead of:
if (user != null && user.getAddress() != null && user.getAddress().getCity() != null) {
    return user.getAddress().getCity().getName();
}

// Use:
return Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .map(City::getName)
    .orElse("Unknown");
```

### Pattern 2: Default Value

```java
// Instead of:
String name = (user != null) ? user.getName() : "Anonymous";

// Use:
String name = Optional.ofNullable(user)
    .map(User::getName)
    .orElse("Anonymous");
```

### Pattern 3: Exception Throwing

```java
// Instead of:
if (user == null) {
    throw new UserNotFoundException("User not found");
}
return user;

// Use:
return Optional.ofNullable(user)
    .orElseThrow(() -> new UserNotFoundException("User not found"));
```

## Key Takeaway

> Use `Optional` as a **return type** to explicitly signal that a method may not return a value, and to enable fluent, null-safe chaining. **Avoid** using it as a field type, parameter type, or in performance-critical code. The goal is to reduce `NullPointerException` and make code more expressive, not to add unnecessary complexity.
