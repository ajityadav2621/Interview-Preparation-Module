# Why is String Immutable in Java?

## Core Concept
String objects cannot be modified after they are created. Once a String is instantiated, its internal character array cannot be changed.

## Why Strings Are Immutable

### 1. **Security**
- Prevents unauthorized modification of string values
- Critical for security-sensitive operations (file paths, passwords, URLs)
- Example: If String were mutable, changing a file path could compromise security

### 2. **String Pool Optimization**
- Immutable strings can be cached and reused
- Multiple variables can share the same String object in memory
- Saves memory and improves performance

### 3. **Thread Safety**
- Immutable objects are inherently thread-safe
- No synchronization needed when sharing strings across threads
- Eliminates race conditions

### 4. **Hash Code Caching**
- The hash code is computed once and cached
- Essential for efficient use as HashMap keys
- If mutable, hash code could change, breaking hash-based collections

### 5. **Performance**
- No defensive copying needed when passing strings around
- String constants can be shared without copying

## Internal Implementation
```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    private final char value[]; // Internal array is final
    private int hash; // Cached hash code
    
    // All methods return new String objects, never modify this
    public String toUpperCase() {
        return new String(...); // Creates new instance
    }
}
```

## When Would Mutable Strings Be Problematic?
- Security: Path traversal attacks
- Caching: Cache keys changing unexpectedly
- Concurrency: Data races in multi-threaded environments
- HashMap: Keys changing after insertion

## Interview Tip
Mention that while String is final and its internal array is private/final, you can still use reflection to modify it (though you shouldn't!). This shows deeper understanding.