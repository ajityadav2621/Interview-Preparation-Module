# Modifying Collection While Iterating

## The Problem
Modifying a collection while iterating over it can lead to unpredictable behavior, including ConcurrentModificationException.

## What Happens

### Fail-Fast Iterators (ArrayList, HashMap, etc.)
```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));

// BAD - ConcurrentModificationException
for (String item : list) {
    if (item.equals("B")) {
        list.remove(item); // Throws ConcurrentModificationException
    }
}
```

### Why It Happens
- Iterators use a `modCount` field to track modifications
- When iterator is created, it records the current modCount
- Any structural modification increments modCount
- Iterator checks modCount on each next() call
- If modCount changed, throws ConcurrentModificationException

### Proper Ways to Modify During Iteration

#### 1. Use Iterator's Remove Method
```java
Iterator<String> iterator = list.iterator();
while (iterator.hasNext()) {
    String item = iterator.next();
    if (item.equals("B")) {
        iterator.remove(); // Safe removal
    }
}
```

#### 2. Use For-Index Loop (ArrayList)
```java
for (int i = list.size() - 1; i >= 0; i--) {
    if (list.get(i).equals("B")) {
        list.remove(i); // Safe for ArrayList
    }
}
```

#### 3. Collect and Remove Afterwards
```java
List<String> toRemove = new ArrayList<>();
for (String item : list) {
    if (item.equals("B")) {
        toRemove.add(item);
    }
}
list.removeAll(toRemove);
```

#### 4. Use RemoveIf (Java 8+)
```java
list.removeIf(item -> item.equals("B"));
```

## Collections with Different Behavior

### ConcurrentHashMap
- Weakly consistent iterator
- May or may not reflect concurrent modifications
- No ConcurrentModificationException

### CopyOnWriteArrayList
- Creates new copy on modification
- Safe for concurrent iteration and modification
- Expensive for frequent modifications

## Interview Tip
Explain that this is a design choice for data integrity. Mention that modern Java provides better alternatives like removeIf and Stream API.