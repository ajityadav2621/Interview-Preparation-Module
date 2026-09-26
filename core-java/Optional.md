# Optional - When to Use and When to Avoid

## What is Optional?
Optional is a container object that may or may not contain a non-null value. It's designed to make null handling more explicit.

## When to Use Optional

### 1. **Return Values**
```java
// Good - explicit about possible absence
public Optional<User> getUserById(String id) {
    // ...
    return Optional.ofNullable(user);
}

// Caller must handle absence
Optional<User> user = getUserById("123");
user.ifPresent(u -> System.out.println(u.getName()));
```

### 2. **Method Chaining**
```java
// Avoids null pointer exceptions
String name = getUserById(id)
    .map(User::getName)
    .orElse("Unknown");
```

### 3. **API Design**
- Make it clear when a value might be absent
- Force callers to consider the empty case

## When to Avoid Optional

### 1. **Performance-Critical Code**
- Optional creates new objects
- Adds overhead in hot paths

### 2. **Fields and Parameters**
```java
// BAD - unnecessary wrapping
private Optional<String> name;

// GOOD - use null or default values
private String name = "";
```

### 3. **Simple Cases**
```java
// Overkill for simple null checks
if (user != null) {
    return user.getName();
}
return "Unknown";

// vs
return Optional.ofNullable(user)
    .map(User::getName)
    .orElse("Unknown");
```

### 4. **Serialization**
- Optional is not serializable
- Use fields instead

## Best Practices

### Good Usage
```java
// Return value
public Optional<User> findUser(String id) {
    return repository.findById(id);
}

// Method parameters (rarely)
public void process(Optional<String> optName) {
    optName.ifPresent(name -> System.out.println(name));
}
```

### Bad Usage
```java
// Avoid in fields
class User {
    private Optional<String> email; // Bad
}

// Avoid in method parameters (usually)
public void saveUser(Optional<User> user) { // Bad
    user.orElseThrow(() -> new IllegalArgumentException());
}
```

## Interview Tip
Explain that Optional is a tool for API design, not a replacement for null checks. It makes code more readable but adds overhead.