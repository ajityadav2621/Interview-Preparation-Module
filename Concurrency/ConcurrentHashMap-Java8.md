# ConcurrentHashMap Internals (Java 8+)

## Evolution from Java 7 to Java 8

### Java 7: Segment-based
- Array of segments (default 16)
- Each segment is a separate HashMap with its own lock
- Lock striping for better concurrency
- Multiple threads can operate on different segments simultaneously

### Java 8+: CAS + Synchronized
- No segments - uses synchronized blocks on individual nodes
- Uses CAS (Compare-And-Swap) operations for updates
- Better scalability and performance
- Treeified buckets for collision handling

## Key Features

### 1. **CAS Operations**
```java
// Atomic operations without locking
public V putIfAbsent(K key, V value) {
    while (true) {
        Node<K,V> tab[] = table;
        Node<K,V> p = tabAt(tab, i);
        if (p == null) {
            if (casTabAt(tab, i, null, new Node<>(hash, key, value))) {
                break;
            }
        }
        // Handle collision
    }
}
```

### 2. **Treeification**
- When linked list exceeds threshold (8) and table size >= 64
- Converts linked list to balanced BST for O(log n) lookup
- Improves performance for collision-heavy scenarios

### 3. **Size Counting**
- Uses LongAdder for concurrent size tracking
- More accurate than simple counter under contention

## Performance Characteristics
- **Average Case**: O(1) for get/put
- **Worst Case**: O(log n) with treeification
- **Thread Safety**: Better than Hashtable (which locks entire map)

## Interview Tip
Explain that ConcurrentHashMap uses optimistic reading with CAS for writes. Mention that it's not just "synchronized HashMap" - it uses more sophisticated concurrency control.