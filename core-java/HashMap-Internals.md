# How Does HashMap Work Internally?

## Core Concept
HashMap is a hash table-based implementation of the Map interface that stores key-value pairs and allows null keys and null values.

## Internal Structure (Java 8+)

### Node Structure
```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next; // For linked list in case of collisions
}
```

### Array of Buckets
- HashMap maintains an array of Node objects (buckets)
- Default initial capacity: 16 (power of 2)
- Each bucket stores entries that hash to the same index

## How It Works

### 1. **Insertion Process**
```java
public V put(K key, V value) {
    int hash = hash(key); // Apply hash function
    int index = hash & (table.length - 1); // Get bucket index
    
    // If bucket is empty, create new node
    if (table[index] == null) {
        table[index] = new Node<>(hash, key, value, null);
    } else {
        // Handle collision - traverse linked list
        // If key exists, update value
        // Otherwise, add to end of linked list
    }
    
    // If size exceeds threshold, resize
    if (++size > threshold) {
        resize();
    }
}
```

### 2. **Hash Function**
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
- Uses XOR with shifted hash to mix high and low bits
- Improves distribution and reduces collisions

### 3. **Collision Resolution**
- Uses separate chaining (linked list per bucket)
- When multiple keys hash to same bucket, they form a linked list

### 4. **Treeification (Java 8+)**
- When linked list exceeds threshold (8) and table size >= 64
- Converts linked list to balanced BST for O(log n) lookup
- Improves performance for collision-heavy scenarios

### 5. **Resizing**
- When load factor (0.75) is exceeded, table doubles in size
- All entries are rehashed to new positions
- Expensive operation - minimize by sizing correctly

## Performance Characteristics
- **Average Case**: O(1) for get/put
- **Worst Case**: O(n) without treeification, O(log n) with
- **Space**: O(n)

## Important Notes
- Not thread-safe - use ConcurrentHashMap for concurrent access
- Order of iteration is not guaranteed
- Capacity should be power of 2 for efficient modulo operation

## Interview Tip
Explain that HashMap is not synchronized, while Hashtable is synchronized (but legacy). Mention ConcurrentHashMap for thread-safe scenarios.