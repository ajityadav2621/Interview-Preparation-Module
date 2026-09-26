# What Happens When Two Keys Have the Same Hashcode?

## Collision vs HashCode Equality
- **Same hashcode** = collision (keys land in same bucket)
- **Same object** = equals() returns true AND same hashcode

## Collision Handling in HashMap

### Scenario 1: Same Hashcode, Different Keys
```java
class Key {
    String name;
    
    @Override
    public int hashCode() {
        return name.hashCode(); // All keys with same name have same hash
    }
    
    @Override
    public boolean equals(Object obj) {
        // Properly compares keys
    }
}

// Both keys hash to same bucket, but are different objects
Key k1 = new Key("John");
Key k2 = new Key("John");
map.put(k1, "value1");
map.put(k2, "value2"); // Different entry, same bucket
```

### Scenario 2: Same Hashcode, Same Keys (equals returns true)
```java
Key k1 = new Key("John");
Key k2 = new Key("John");
map.put(k1, "value1");
map.put(k2, "value2"); // k2.equals(k1) returns true, so value is replaced
```

## Collision Resolution Process

### 1. **Initial Insertion**
- Compute hash → get bucket index
- If bucket empty, create new node
- If bucket occupied, check if keys are equal

### 2. **During Collision**
```java
// When two keys hash to same bucket:
Node<K,V> e = table[i]; // First node in bucket
while (e != null) {
    if (e.hash == hash && (e.key == key || key.equals(e.key))) {
        // Keys are equal - replace value
        e.value = value;
        return oldValue;
    }
    e = e.next; // Move to next node in linked list
}
// If not found, add new node to end of linked list
```

### 3. **Performance Impact**
- Without collisions: O(1) average case
- With collisions: O(n) for linked list traversal
- With treeification: O(log n) when threshold exceeded

## Why HashCode Matters
- Good hash code → even distribution → fewer collisions
- Bad hash code → all keys in same bucket → degraded performance
- HashMap uses hash code to determine bucket, then equals() for actual comparison

## Interview Tip
Explain that hashcode is used for initial bucket selection, but equals() is the final arbiter for key equality. Both must be properly implemented!