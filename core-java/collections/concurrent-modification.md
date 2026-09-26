# What Happens if You Modify a Collection While Iterating Over It?

## The Problem: ConcurrentModificationException

When you modify a collection (add, remove, or update elements) while iterating over it using an enhanced for-loop or an iterator, Java throws a `ConcurrentModificationException`.

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));

// This throws ConcurrentModificationException
for (String s : list) {
    if (s.equals("b")) {
        list.remove(s);  // Modifying the collection during iteration!
    }
}
```

## Why This Happens: Fail-Fast Iterators

Most Java collections (ArrayList, HashMap, HashSet, etc.) use **fail-fast iterators**. These iterators check for structural modifications (changes to the collection's size or internal structure) during iteration.

### How Fail-Fast Works

```java
// Simplified ArrayList iterator
private class Itr implements Iterator<E> {
    private int expectedModCount = modCount;  // Snapshot of modification count

    public E next() {
        if (modCount != expectedModCount)  // Check if collection was modified
            throw new ConcurrentModificationException();
        // ... return next element
    }
}
```

- `modCount` is a field in the collection that tracks the number of structural modifications
- When you create an iterator, it captures the current `modCount` as `expectedModCount`
- Before each `next()` call, the iterator checks if `modCount` has changed
- If it has, the iterator throws `ConcurrentModificationException`

## Common Scenarios That Trigger the Exception

### Scenario 1: Removing During Iteration

```java
List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));

// WRONG: Throws ConcurrentModificationException
for (Integer n : numbers) {
    if (n % 2 == 0) {
        numbers.remove(n);
    }
}
```

### Scenario 2: Adding During Iteration

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b"));

// WRONG: Throws ConcurrentModificationException
for (String s : list) {
    list.add("new");
}
```

### Scenario 3: Modifying a Map During Iteration

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);

// WRONG: Throws ConcurrentModificationException
for (String key : map.keySet()) {
    if (map.get(key) > 1) {
        map.remove(key);
    }
}
```

## Solutions

### Solution 1: Use Iterator.remove()

```java
List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));

// CORRECT: Use iterator's remove() method
Iterator<Integer> it = numbers.iterator();
while (it.hasNext()) {
    Integer n = it.next();
    if (n % 2 == 0) {
        it.remove();  // Safe — iterator knows about the modification
    }
}
```

### Solution 2: Use removeIf() (Java 8+)

```java
List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));

// CORRECT: Use removeIf
numbers.removeIf(n -> n % 2 == 0);
```

### Solution 3: Collect Items to Remove, Then Remove After Iteration

```java
List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
List<Integer> toRemove = new ArrayList<>();

for (Integer n : numbers) {
    if (n % 2 == 0) {
        toRemove.add(n);
    }
}
numbers.removeAll(toRemove);
```

### Solution 4: Use a Copy for Iteration

```java
List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));

// Iterate over a copy
for (Integer n : new ArrayList<>(numbers)) {
    if (n % 2 == 0) {
        numbers.remove(n);
    }
}
```

### Solution 5: Use Java 8+ Streams

```java
List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));

// Filter using streams
List<Integer> filtered = numbers.stream()
    .filter(n -> n % 2 != 0)
    .collect(Collectors.toList());
```

### Solution 6: Use Concurrent Collections

```java
// For maps
Map<String, Integer> map = new ConcurrentHashMap<>();

// For lists
List<String> list = new CopyOnWriteArrayList<>();

// These don't throw ConcurrentModificationException
```

## Map Iteration Solutions

### Using Iterator

```java
Map<String, Integer> map = new HashMap<>();
Iterator<Map.Entry<String, Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    Map.Entry<String, Integer> entry = it.next();
    if (entry.getValue() > 1) {
        it.remove();  // Safe removal
    }
}
```

### Using Java 8+ removeIf on values

```java
map.values().removeIf(v -> v > 1);
```

### Using ConcurrentHashMap (no exception)

```java
Map<String, Integer> map = new ConcurrentHashMap<>();
for (String key : map.keySet()) {
    if (map.get(key) > 1) {
        map.remove(key);  // Safe — no ConcurrentModificationException
    }
}
```

## CopyOnWriteArrayList — A Special Case

`CopyOnWriteArrayList` is a thread-safe variant where **all mutative operations** are implemented by creating a new copy of the underlying array.

```java
List<String> list = new CopyOnWriteArrayList<>();
list.add("a");
list.add("b");

// Safe to modify during iteration — iterator works on a snapshot
for (String s : list) {
    list.add("new");  // No exception!
}
```

**Trade-off**: Write operations are expensive (O(n) copy), but reads and iterations are lock-free and fast.

## Key Takeaways

1. **Fail-fast iterators** detect concurrent modification and throw `ConcurrentModificationException`
2. **Use `Iterator.remove()`** for safe removal during iteration
3. **Use `removeIf()`** for conditional removal (Java 8+)
4. **Use concurrent collections** (`ConcurrentHashMap`, `CopyOnWriteArrayList`) when you need to modify during iteration
5. **Collect-then-remove** pattern works for simple cases
6. **Streams** provide a functional approach to filtering

## Related

- See [`concurrent-hashmap.md`](concurrent-hashmap.md) for thread-safe map alternatives
