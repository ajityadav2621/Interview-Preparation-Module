# ArrayList vs LinkedList — When Would You Use Each?

## Quick Comparison Table

| Feature | ArrayList | LinkedList |
|---------|-----------|------------|
| **Data Structure** | Resizable array | Doubly linked list |
| **Memory Overhead** | Low (just array + size) | High (each node has 2 pointers) |
| **Random Access (get/set)** | O(1) — direct index | O(n) — must traverse |
| **Insertion at end** | O(1) amortized, O(n) worst (resize) | O(1) — just link new node |
| **Insertion at beginning** | O(n) — shift all elements | O(1) — update head pointer |
| **Insertion at middle** | O(n) — shift elements | O(n) — find position, then O(1) |
| **Deletion** | O(n) — shift elements | O(n) — find node, then O(1) |
| **Iteration** | O(n) — fast (cache-friendly) | O(n) — slower (pointer chasing) |
| **Cache Locality** | Excellent | Poor |
| **Memory Usage** | ~1.5x capacity (over-allocation) | ~3x (data + 2 pointers per node) |

## ArrayList Deep Dive

### How It Works

```java
public class ArrayList<E> extends AbstractList<E> {
    private static final int DEFAULT_CAPACITY = 10;
    private static final Object[] EMPTY_ELEMENTDATA = {};
    private transient Object[] elementData;
    private int size;
}
```

### Resizing Behavior

```java
// When size reaches capacity:
// 1. Create new array (1.5x old capacity + 1)
// 2. Copy all elements to new array
// 3. Discard old array

// newCapacity = oldCapacity + (oldCapacity >> 1)
// e.g., 10 → 15 → 22 → 33 → 49 → 73 → 109 → 163 → 244 → 366
```

### When to Use ArrayList

1. **Random access is needed** — `get(index)` is O(1)
2. **Most operations are at the end** — appending is O(1) amortized
3. **Memory is a concern** — lower overhead per element
4. **Iteration is frequent** — cache-friendly sequential access
5. **Size is known or predictable** — can pre-size to avoid resizing

```java
// Good use case:
List<String> names = new ArrayList<>(1000);  // Pre-sized
for (String name : databaseResults) {
    names.add(name);  // Mostly appending
}
String first = names.get(0);  // Random access needed
```

## LinkedList Deep Dive

### How It Works

```java
public class LinkedList<E> extends AbstractSequentialList<E> {
    transient int size = 0;
    transient Node<E> first;
    transient Node<E> last;

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
        Node(E prev, E element, Node<E> next) { ... }
    }
}
```

### When to Use LinkedList

1. **Frequent insertions/deletions at the beginning or middle** — O(1) once you have the node
2. **Implementing queues or deques** — `LinkedList` implements `Deque`
3. **Unknown size with frequent additions** — no resizing overhead
4. **Using as a stack** — `addFirst()` / `removeFirst()` are O(1)

```java
// Good use case:
Deque<String> queue = new LinkedList<>();
queue.offer("task1");  // O(1)
queue.offer("task2");  // O(1)
String next = queue.poll();  // O(1)
```

## Performance Benchmarks

### Random Access
```java
// ArrayList: O(1)
list.get(500);  // Direct array access

// LinkedList: O(n)
list.get(500);  // Must traverse from head/tail
```

### Insertion at Beginning
```java
// ArrayList: O(n) — shift all elements
list.add(0, "item");  // All elements shift right

// LinkedList: O(1) — just update head
list.add(0, "item");  // New node becomes head
```

### Iteration
```java
// ArrayList: Fast — sequential memory access
for (int i = 0; i < list.size(); i++) {
    list.get(i);  // Direct memory access, cache-friendly
}

// LinkedList: Slower — pointer chasing
for (String s : list) {
    // Each step requires following a pointer
}
```

## Real-World Decision Matrix

| Scenario | Choice | Reason |
|----------|--------|--------|
| Reading by index frequently | ArrayList | O(1) vs O(n) |
| Adding/removing at ends | Either | Both O(1) amortized |
| Adding/removing at beginning | LinkedList | O(1) vs O(n) |
| Adding/removing in middle | LinkedList* | O(1) after finding node |
| Memory-constrained | ArrayList | Lower overhead |
| Queue/Deque operations | LinkedList | Implements Deque |
| Stack operations | LinkedList | O(1) push/pop |
| Large dataset, random access | ArrayList | Cache locality |

*Finding the position in a LinkedList is still O(n), so the overall operation is O(n) for both.

## Common Interview Scenarios

### Scenario 1: Which is faster for `list.add(0, "item")`?
**Answer**: LinkedList — O(1) vs ArrayList's O(n) shift.

### Scenario 2: Which is faster for `list.get(500)`?
**Answer**: ArrayList — O(1) vs LinkedList's O(n) traversal.

### Scenario 3: Which uses less memory for 1000 integers?
**Answer**: ArrayList — LinkedList stores 2 extra pointers per node.

## Best Practices

1. **Default to ArrayList** — it's faster for most use cases due to cache locality
2. **Use LinkedList only when** you need frequent insertions/deletions at known positions
3. **Pre-size ArrayList** when you know the approximate size to avoid resizing
4. **Use `ArrayDeque`** instead of `LinkedList` for stack/queue operations — it's faster and uses less memory
5. **Consider `Collections.nCopies()`** for immutable lists

## Code Example: When LinkedList Shines

```java
// Implementing a sliding window with frequent add/remove at both ends
Deque<Integer> window = new ArrayDeque<>();  // Better than LinkedList
window.offerLast(1);
window.offerLast(2);
window.pollFirst();  // Remove oldest
```

## Key Takeaway

> **ArrayList is the default choice** for most scenarios. **LinkedList** is only better when you need frequent insertions/deletions at the beginning or when using it as a queue/deque. In practice, `ArrayDeque` is often a better choice than `LinkedList` for queue operations.
