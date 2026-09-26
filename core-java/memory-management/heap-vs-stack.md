# Heap vs Stack — What Is Stored Where?

## Overview

The JVM divides memory into two main areas: the **Heap** and the **Stack**. Understanding what goes where is crucial for debugging memory issues, optimizing performance, and passing interviews.

## The Stack (Thread Stack)

### What is Stored in the Stack?

1. **Method frames** — one per method invocation
2. **Local variables** — primitive types and object references
3. **Method parameters**
4. **Return address** — where to return after method completes
5. **Operand stack** — temporary storage for intermediate calculations

### Stack Characteristics

| Property | Description |
|----------|-------------|
| **Per-thread** | Each thread has its own stack |
| **LIFO** | Last In, First Out — methods are pushed/popped |
| **Fixed size** | Size is determined at thread creation |
| **Fast access** | Very fast — just pointer arithmetic |
| **Automatic cleanup** | Entire frame is removed when method returns |
| **StackOverflowError** | Thrown when stack is exhausted (too deep recursion) |

### Stack Frame Structure

```
┌─────────────────────────────────┐
│  Frame for method: calculate()  │
├─────────────────────────────────┤
│  Local Variables:               │
│    int x = 10                   │
│    String s = "hello"           │
│    Object obj = 0x1234abcd      │
├─────────────────────────────────┤
│  Operand Stack:                 │
│    [10]                         │
│    [20]                         │
│    [30]  ← result of 10 + 20    │
├─────────────────────────────────┤
│  Return Address: 0x7f8b4c005000 │
└─────────────────────────────────┘
```

### Example: Stack in Action

```java
public class StackExample {
    public static void main(String[] args) {
        int a = 10;                    // Stored in main()'s frame
        String s = "hello";          // Reference stored in main()'s frame
        Person p = new Person("Bob"); // Reference in main()'s frame, object in heap
        
        int result = add(a, 5);      // Pushes add() frame onto stack
        System.out.println(result);  // Pops add() frame
    }

    public static int add(int x, int y) {
        int sum = x + y;  // x, y, sum stored in add()'s frame
        return sum;       // Returns to main(), add()'s frame is popped
    }
}
```

### Stack Memory Layout

```
Stack (grows downward):
┌─────────────────────────┐ ← Stack top
│  main() frame            │
│    a = 10                │
│    s = "hello"           │
│    p = 0x1234abcd        │
├─────────────────────────┤
│  add() frame            │
│    x = 10                │
│    y = 5                 │
│    sum = 15              │
├─────────────────────────┤
│  println() frame        │
│    ...                   │
└─────────────────────────┘ ← Stack bottom
```

## The Heap

### What is Stored in the Heap?

1. **All objects** — `new` creates objects in the heap
2. **Arrays** — all arrays are objects in the heap
3. **String literals** — stored in the String Pool (part of heap)
4. **Static variables** — stored in the Method Area (part of heap in Java 8+)
5. **Class metadata** — class definitions, method bytecodes

### Heap Characteristics

| Property | Description |
|----------|-------------|
| **Shared** | One heap per JVM, shared by all threads |
| **Managed by GC** | Garbage collector reclaims unused objects |
| **Larger** | Typically much larger than stack |
| **Slower access** | Requires dereferencing pointers |
| **OutOfMemoryError** | Thrown when heap is exhausted |
| **Complex structure** | Divided into generations (Young, Old, Metaspace) |

### Heap Structure (Java 8+)

```
┌─────────────────────────────────────────────┐
│  Metaspace (class metadata)                 │
│  - Class definitions                        │
│  - Method bytecodes                         │
│  - Static variables                         │
├─────────────────────────────────────────────┤
│  Young Generation (Eden + Survivor)        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │  Eden    │ │ Survivor1│ │ Survivor2│    │
│  │  (8:1)   │ │  (1:1)   │ │  (1:1)   │    │
│  └──────────┘ └──────────┘ └──────────┘    │
├─────────────────────────────────────────────┤
│  Old Generation (Tenured Space)              │
│  - Long-lived objects                       │
│  - Objects that survived multiple GCs       │
└─────────────────────────────────────────────┘
```

### Example: Heap in Action

```java
public class HeapExample {
    public static void main(String[] args) {
        // Objects created in heap:
        Person p1 = new Person("Alice");  // Object in heap, reference in stack
        Person p2 = new Person("Bob");    // Object in heap, reference in stack
        
        int[] numbers = {1, 2, 3, 4, 5};  // Array object in heap
        
        String s1 = "hello";              // String literal in String Pool (heap)
        String s2 = new String("world");  // String object in heap
        
        p1.setFriend(p2);  // p1's object in heap now references p2's object
    }
}
```

## Key Differences

| Aspect | Stack | Heap |
|--------|-------|------|
| **Scope** | Per-thread | Per-JVM (shared) |
| **Lifespan** | Method execution | Object lifetime (until GC) |
| **Size** | Small (typically 512KB - 2MB) | Large (typically 1GB+) |
| **Access Speed** | Very fast | Slower (GC overhead) |
| **Error** | StackOverflowError | OutOfMemoryError |
| **Content** | Method frames, local vars | Objects, arrays, class metadata |
| **Management** | Automatic (push/pop) | Garbage collection |
| **Growth** | Fixed at thread creation | Dynamic (can grow/shrink) |

## What Goes Where?

### Stack Only
- Primitive local variables (`int x = 10`)
- Object references (`Person p = new Person()`)
- Method parameters
- Return addresses

### Heap Only
- Objects (`new Person()`)
- Arrays (`new int[100]`)
- String literals (`"hello"`)
- Static variables
- Class metadata

### Both
- Object references are stored in the stack, but the objects themselves are in the heap

```java
public class Example {
    private static int staticVar = 100;  // Heap (Method Area)
    
    public void method() {
        int localVar = 10;              // Stack
        String str = "hello";           // str ref: Stack, "hello": Heap (String Pool)
        Person p = new Person("Bob");   // p ref: Stack, Person object: Heap
    }
}
```

## Common Interview Questions

### Q: What happens when the stack is full?
A: `StackOverflowError` — typically caused by infinite recursion.

### Q: What happens when the heap is full?
A: `OutOfMemoryError: Java heap space` — the GC can't reclaim enough memory.

### Q: Can two threads access the same object?
A: Yes — the object is in the heap (shared), but each thread has its own reference in its stack.

### Q: Where is the String Pool?
A: In the heap (specifically, in the Method Area / Metaspace in Java 8+).

## Key Takeaway

> **Stack** = per-thread, fast, stores method frames and local variables (including object references). **Heap** = shared, managed by GC, stores all objects and arrays. Understanding this distinction is essential for debugging memory issues, optimizing performance, and writing efficient Java code.
