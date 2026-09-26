# How Does HashMap Work Internally?

## Overview

`HashMap` in Java is a hash table-based implementation of the `Map` interface. It stores key-value pairs and provides O(1) average-time complexity for `get()` and `put()` operations.

## Internal Data Structure

### Pre-Java 8

```
HashMap structure:
┌─────────────────────────────────────────────┐
│  Entry[] table (array of buckets)           │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │ bucket0 │ │ bucket1 │ │ bucket2 │ ...   │
│  │  null   │ │  Entry  │ │  null   │       │
│  └─────────┘ └────┬────┘ └─────────┘       │
│                   │                        │
│                   ▼                        │
│              ┌──────────┐                  │
│              │ key="b"  │                  │
│              │ value=2  │                  │
│              │ next ────┼──> null          │
│              └──────────┘                  │
└─────────────────────────────────────────────┘
```

### Java 8+ (with balanced trees)

When a bucket has more than 8 entries, it converts from a linked list to a **balanced binary search tree** (red-black tree) to maintain O(log n) performance.

## Key Components

### 1. The `Entry` (or `Node`) Class

```java
// Simplified version of HashMap.Node
static class Node<K, V> implements Map.Entry<K, V> {
    final int hash;
    final K key;
    V value;
    Node<K, V> next;  // For chaining (linked list)

    Node(int hash, K key, V value, Node<K, V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}
```

### 2. The Table (Array of Buckets)

```java
transient Node<K, V>[] table;  // Array of Node
```

The array size is always a power of 2 (default 16).

### 3. Hash Function

```java
// HashMap's hash() method (simplified)
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

This is a **hash spreading** function — it XORs the high 16 bits with the low 16 bits. This ensures that the hash code is used more uniformly when computing the bucket index.

### 4. Index Calculation

```java
// Simplified index calculation
int index = (table.length - 1) & hash;
```

Since `table.length` is always a power of 2, `(table.length - 1) & hash` is equivalent to `hash % table.length` but faster (bitwise AND instead of modulo).

## How `put()` Works

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, false);
}

// Simplified putVal
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K, V>[] tab; int n, i, cap;
    Node<K, V> p;
    
    // 1. If table is null or empty, resize
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // 2. Calculate index
    i = (n - 1) & hash;
    
    // 3. If bucket is empty, create new node
    if ((p = tab[i]) == null)
        tab[i] = new Node<>(hash, key, value, null);
    else {
        // 4. Bucket has entries — traverse the chain
        Node<K, V> node = p;
        while (true) {
            if (node.hash == hash && Objects.equals(node.key, key)) {
                // Key already exists — update value
                V oldValue = node.value;
                if (!onlyIfAbsent)
                    node.value = value;
                return oldValue;
            }
            if (node.next == null) {
                // End of chain — add new node
                node.next = new Node<>(hash, key, value, null);
                break;
            }
            node = node.next;
        }
    }
    
    // 5. Check if resize is needed (load factor threshold)
    if (++size > threshold)
        resize();
    
    return null;
}
```

## How `get()` Works

```java
public V get(Object key) {
    Node<K, V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

// Simplified getNode
final Node<K, V> getNode(int hash, Object key) {
    Node<K, V>[] tab; Node<K, V> p;
    int n;
    
    if ((tab = table) == null || (n = tab.length) == 0)
        return null;
    
    // 1. Calculate index
    int i = (n - 1) & hash;
    
    // 2. Get the first node in the bucket
    if ((p = tab[i]) == null)
        return null;
    
    // 3. Check if first node matches
    if (p.hash == hash && Objects.equals(p.key, key))
        return p;
    
    // 4. Traverse the chain (or tree)
    if (p.next != null) {
        // If tree, use tree traversal; if list, use linear search
        return p.next.find(hash, key, tab);
    }
    
    return null;
}
```

## Collision Resolution: Separate Chaining

When two keys produce the same hash code (or hash to the same bucket index), they are stored in a **linked list** at that bucket. This is called **separate chaining**.

```
Bucket index 3:
┌──────────┐
│ key="cat" │ ──> ┌──────────┐ ──> ┌──────────┐ ──> null
│ value=1  │     │ key="dog" │     │ key="bat" │
│ hash=963 │     │ value=2  │     │ value=3  │
└──────────┘     │ hash=963 │     │ hash=963 │
                 └──────────┘     └──────────┘
```

## Resizing (Rehashing)

When the number of entries exceeds `capacity * loadFactor` (default load factor = 0.75), the HashMap **resizes** — it doubles the array size and rehashes all entries.

```java
// Default values
static final int DEFAULT_INITIAL_CAPACITY = 16;
static final float DEFAULT_LOAD_FACTOR = 0.75f;
static final int DEFAULT_THRESHOLD = 12;  // 16 * 0.75
```

**Why rehashing is needed**: The bucket index depends on `table.length`, so when the array size changes, all entries must be redistributed.

## Java 8+ Treeification

When a single bucket's linked list exceeds 8 nodes, it's converted to a **red-black tree** (a self-balancing BST). This prevents O(n) worst-case lookup in heavily colliding scenarios.

```java
// Treeify threshold
static final int TREEIFY_THRESHOLD = 8;
// Untreeify threshold (when to convert back to list)
static final int UNTREEIFY_THRESHOLD = 6;
```

## Performance Characteristics

| Operation | Average Case | Worst Case (Java 7) | Worst Case (Java 8+) |
|-----------|-------------|---------------------|----------------------|
| `put()`   | O(1)        | O(n)                | O(log n)             |
| `get()`   | O(1)        | O(n)                | O(log n)             |
| `remove()`| O(1)        | O(n)                | O(log n)             |

## Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| Initial capacity | 16 | Number of buckets |
| Load factor | 0.75 | When to resize |
| Threshold | 12 | capacity * load factor |

## Common Interview Questions

### Q: Why is the array size always a power of 2?
A: So that `(length - 1) & hash` can be used instead of `hash % length`, which is faster.

### Q: What happens during resizing?
A: The array doubles in size, and all entries are rehashed to new bucket positions.

### Q: Why does Java 8 use a tree for collisions?
A: To maintain O(log n) performance even with many collisions, instead of degrading to O(n) with a linked list.

## Related: hashCode() and equals() Contract

See [`hashcode-equals-contract.md`](hashcode-equals-contract.md) for the critical contract that HashMap relies on.
