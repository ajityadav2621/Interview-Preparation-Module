# Heap vs Stack - What is Stored Where?

## Core Differences

| Feature | Stack | Heap |
|---------|-------|------|
| **Storage** | Primitive types, object references | Objects, arrays, instance variables |
| **Allocation** | Automatic (LIFO) | Manual (garbage collected) |
| **Access Speed** | Very fast | Slower |
| **Size** | Limited (typically 1-8MB) | Large (limited by RAM) |
| **Lifespan** | Method invocation | Application lifetime |
| **Thread Safety** | Thread-specific | Shared across threads |

## What's Stored Where

### Stack Memory
```java
public class Example {
    private int x = 10; // Heap (instance variable)
    private String name; // Heap (reference)
    
    public void method(int param) { // param on stack
        int localVar = 20; // Stack
        Object obj = new Object(); // obj reference on stack, Object on heap
    }
}
```

### Heap Memory
- Object instances
- Arrays
- Instance variables
- Static variables
- String literals (interned strings)

## Memory Layout
```
Stack Frame:
+---------------------+
| Local Variables     | <- Primitive types, references
| Operand Stack       | <- Intermediate computation results
| Frame References    | <- References to other objects
+---------------------+

Heap:
+---------------------+
| Object 1            |
| Object 2            |
| Object 3            |
| ...                 |
+---------------------+
```

## Interview Tip
Explain that each thread has its own stack, but all threads share the heap. Stack overflow occurs when recursion is too deep, while heap overflow occurs when too many objects are created.