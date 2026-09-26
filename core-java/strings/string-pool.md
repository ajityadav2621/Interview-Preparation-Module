# String Pool in Java

## What is the String Pool?

The **String Pool** is a special memory area in the Java heap that stores **string literals** to optimize memory usage and improve performance.

## How the String Pool Works

### String Literals vs. `new String()`

```java
// String literal — stored in the String Pool
String s1 = "hello";

// Another literal — reuses the same object from the pool
String s2 = "hello";

// new String() — creates a new object in the heap (NOT in the pool)
String s3 = new String("hello");

System.out.println(s1 == s2);  // true  — same object in the pool
System.out.println(s1 == s3);  // false — different objects
System.out.println(s1.equals(s3));  // true — same content
```

### How the Pool Works Internally

1. When the JVM encounters a string literal, it first checks the String Pool.
2. If the string already exists in the pool, the reference to the existing object is returned.
3. If not, a new `String` object is created in the pool and its reference is returned.

### `intern()` Method

You can manually add a string to the pool using `intern()`:

```java
String s = new String("hello");  // heap object
String t = s.intern();           // adds "hello" to pool, returns pool reference

String literal = "hello";        // already in pool
System.out.println(t == literal);  // true
```

## Memory Considerations

### Before Java 7

The String Pool was in the **PermGen** (Permanent Generation) space, which had a fixed size. This could cause `OutOfMemoryError` if too many strings were interned.

### Java 7 and Later

The String Pool was moved to the **heap**, which is garbage-collected. This means interned strings can be reclaimed if no longer referenced.

### Java 8 and Later

PermGen was replaced by **Metaspace**. The String Pool remains in the heap.

## Common Interview Scenarios

### Scenario 1: How many objects are created?

```java
String s1 = "hello";
String s2 = "hello";
String s3 = new String("hello");
```

**Answer**: 2 objects — one in the String Pool (`s1` and `s2` share it), one in the heap (`s3`).

### Scenario 2: How many objects with `new String()`?

```java
String s = new String("hello");
```

**Answer**: 2 objects — `"hello"` literal in the pool, and a new `String` object in the heap.

### Scenario 3: `intern()` behavior

```java
String s = new String("hello");
s.intern();
String t = "hello";
System.out.println(s == t);  // false — s still points to heap object
```

```java
String s = new String("hello");
String t = s.intern();
String u = "hello";
System.out.println(t == u);  // true — both point to pool object
```

## Best Practices

1. **Prefer string literals** over `new String()` when possible — they benefit from the pool.
2. **Use `StringBuilder`** for repeated string modifications.
3. **Be cautious with `intern()`** — it can cause memory issues if used excessively.
4. **Use `equals()` for content comparison**, `==` for reference comparison.

## Key Takeaway

The String Pool is a powerful optimization that reduces memory footprint by sharing identical string literals. Understanding it is crucial for:
- Memory optimization
- Correct string comparison
- Avoiding common interview pitfalls
