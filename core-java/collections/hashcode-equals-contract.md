# hashCode() and equals() Contract

## The Contract

The `hashCode()` and `equals()` methods are defined in `java.lang.Object` and have a strict contract that must be followed when overridden. This contract is critical for the correct functioning of hash-based collections like `HashMap`, `HashSet`, and `Hashtable`.

## The Rules

### Rule 1: Consistency of equals()
- `x.equals(x)` must return `true` (reflexive)
- `x.equals(y)` must return the same value as `y.equals(x)` (symmetric)
- `x.equals(y)` and `y.equals(z)` return `true` → `x.equals(z)` must return `true` (transitive)
- Multiple calls to `x.equals(y)` must return the same value (consistent)
- `x.equals(null)` must return `false`

### Rule 2: hashCode() Consistency
- Multiple calls to `x.hashCode()` must return the same value (consistent)
- If `x.equals(y)` is `true`, then `x.hashCode()` must equal `y.hashCode()`
- If `x.equals(y)` is `false`, `x.hashCode()` **may or may not** be equal (but should ideally differ for performance)

## Why the Contract Matters

### The Critical Rule: Equal Objects Must Have Equal HashCodes

```java
// If this contract is violated, HashMap breaks:
MyKey key1 = new MyKey("name", 1);
MyKey key2 = new MyKey("name", 1);

// If key1.equals(key2) is true but key1.hashCode() != key2.hashCode()
// Then HashMap.put(key1, "value") and HashMap.get(key2) will fail!
```

### How HashMap Uses This Contract

```java
// Simplified HashMap.get()
public V get(Object key) {
    int hash = hash(key.hashCode());       // Step 1: Get hash code
    int index = (table.length - 1) & hash; // Step 2: Find bucket
    // Step 3: Search bucket using equals()
    // ...
}
```

If two equal objects have different hash codes, they'll be placed in different buckets, and `equals()` will never be called to find the match.

## What Happens When Two Keys Have the Same HashCode?

This is called a **hash collision**. It's perfectly normal and expected — the contract only requires that equal objects have equal hash codes, not that unequal objects have different hash codes.

### Collision Handling: Separate Chaining

```java
// When two keys hash to the same bucket:
// Key1: hash=100, index=4
// Key2: hash=200, index=4 (same bucket!)

// HashMap stores them in a linked list at bucket[4]:
// bucket[4] -> Node(key1, value1) -> Node(key2, value2) -> null
```

When you call `get(key2)`:
1. Compute `hash(key2)` → 200
2. Compute `index = (table.length - 1) & 200` → 4
3. Go to `bucket[4]`
4. Traverse the linked list, calling `key2.equals(node.key)` on each node
5. Find the match and return the value

### Performance Impact

- **No collision**: O(1) lookup
- **Collision (linked list)**: O(n) lookup in the worst case for that bucket
- **Java 8+ (tree)**: O(log n) lookup when a bucket has > 8 entries

## Common Mistakes

### Mistake 1: Overriding equals() but not hashCode()

```java
public class Person {
    private String name;
    private int age;

    @Override
    public boolean equals(Object o) {
        // ... proper equals implementation
    }
    // hashCode() NOT overridden — uses default Object.hashCode()
}

// Problem:
Person p1 = new Person("Alice", 30);
Person p2 = new Person("Alice", 30);
System.out.println(p1.equals(p2));  // true
System.out.println(p1.hashCode() == p2.hashCode());  // false!

// HashMap behavior:
Map<Person, String> map = new HashMap<>();
map.put(p1, "Engineer");
System.out.println(map.get(p2));  // null! — can't find p2 because hash differs
```

### Mistake 2: Using Mutable Fields in hashCode()

```java
public class BadKey {
    private String name;  // mutable

    @Override
    public int hashCode() {
        return name.hashCode();  // BAD: name can change!
    }
}

BadKey key = new BadKey("Alice");
map.put(key, "value");
key.setName("Bob");  // hashCode changes!
map.get(key);  // null! — entry is now in the wrong bucket
```

### Mistake 3: Inconsistent equals() (not symmetric)

```java
public class BadEquals {
    private String name;
    private List<String> aliases;

    @Override
    public boolean equals(Object o) {
        if (o instanceof BadEquals) {
            BadEquals other = (BadEquals) o;
            return name.equals(other.name) || aliases.contains(other.name);
        }
        return false;
    }
    // This is NOT symmetric!
}
```

## Best Practices for Implementation

### Using IDE Generation

Most IDEs can generate `hashCode()` and `equals()` automatically:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Person person = (Person) o;
    return age == person.age && Objects.equals(name, person.name);
}

@Override
public int hashCode() {
    return Objects.hash(name, age);
}
```

### Using `java.util.Objects`

```java
@Override
public int hashCode() {
    return Objects.hash(name, age);
}
```

### Using `java.util.Arrays` for arrays

```java
public class WithArray {
    private int[] values;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof WithArray)) return false;
        WithArray that = (WithArray) o;
        return Arrays.equals(values, that.values);
    }

    @Override
    public int hashCode() {
        return Arrays.hashCode(values);
    }
}
```

## The Golden Rule

> **If two objects are equal according to `equals()`, they MUST have the same `hashCode()`.**

The reverse is NOT required — two objects with the same `hashCode()` may or may not be equal.

## Related Topics

- See [`hashset-duplicates.md`](hashset-duplicates.md) for how `HashSet` relies on this contract
- See [`hashmap-internals.md`](hashmap-internals.md) for how HashMap uses hash codes internally
