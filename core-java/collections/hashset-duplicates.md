# Why Can HashSet Detect Duplicate Objects Incorrectly if equals() and hashCode() Are Implemented Badly?

## How HashSet Works Internally

`HashSet` is backed by a `HashMap` internally. When you add an element to a `HashSet`, it:

1. Calls `hashCode()` on the element to find the bucket
2. If the bucket is empty, adds the element
3. If the bucket has elements, calls `equals()` to check for duplicates
4. If `equals()` returns `true` for any existing element, the new element is **not added** (duplicate)

```java
// Simplified HashSet.add()
public boolean add(E e) {
    return map.put(e, PRESENT) == null;  // Uses HashMap internally
}
```

## The Problem: Broken hashCode() and equals()

### Scenario 1: Overriding equals() but NOT hashCode()

```java
public class Person {
    private String name;
    private int age;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Person)) return false;
        Person person = (Person) o;
        return age == person.age && Objects.equals(name, person.name);
    }
    // hashCode() NOT overridden — uses default Object.hashCode()
}

// What happens:
Set<Person> set = new HashSet<>();
Person p1 = new Person("Alice", 30);
Person p2 = new Person("Alice", 30);

System.out.println(p1.equals(p2));  // true — same content
System.out.println(p1.hashCode());  // e.g., 123456789
System.out.println(p2.hashCode());  // e.g., 987654321 — DIFFERENT!

set.add(p1);  // Goes to bucket based on p1.hashCode()
set.add(p2);  // Goes to DIFFERENT bucket based on p2.hashCode()
              // equals() is never called because they're in different buckets!

System.out.println(set.size());  // 2 — should be 1!
```

**Result**: Two "equal" objects are stored as duplicates because they end up in different buckets.

### Scenario 2: hashCode() Returns a Constant

```java
public class BadKey {
    private String value;

    @Override
    public int hashCode() {
        return 42;  // Always returns the same hash code
    }

    @Override
    public boolean equals(Object o) {
        // Proper equals implementation
    }
}

// What happens:
Set<BadKey> set = new HashSet<>();
set.add(new BadKey("a"));
set.add(new BadKey("b"));
set.add(new BadKey("c"));

// All go to the same bucket! HashSet degrades to a linked list.
// Performance: O(n) for add/contains instead of O(1)
```

**Result**: All elements end up in the same bucket, degrading performance to O(n).

### Scenario 3: Inconsistent hashCode() (Mutable Fields)

```java
public class MutableKey {
    private String name;  // Mutable field

    @Override
    public int hashCode() {
        return name.hashCode();  // BAD: depends on mutable field
    }

    @Override
    public boolean equals(Object o) {
        // ...
    }

    public void setName(String name) {
        this.name = name;
    }
}

// What happens:
Set<MutableKey> set = new HashSet<>();
MutableKey key = new MutableKey();
key.setName("Alice");
set.add(key);  // hashCode = "Alice".hashCode()

key.setName("Bob");  // hashCode changes!
set.contains(key);  // false! — key is now in the wrong bucket
set.remove(key);   // false! — can't find the key
```

**Result**: The object becomes "lost" in the HashSet — it can't be found, removed, or detected as a duplicate.

### Scenario 4: Inconsistent equals() (Not Symmetric)

```java
public class BadEquals {
    private String name;

    @Override
    public boolean equals(Object o) {
        if (o instanceof BadEquals) {
            BadEquals other = (BadEquals) o;
            return name.equals(other.name);
        }
        if (o instanceof String) {
            return name.equals(o);
        }
        return false;
    }

    @Override
    public int hashCode() {
        return name.hashCode();
    }
}

// What happens:
Set<BadEquals> set = new HashSet<>();
BadEquals key = new BadEquals("test");
set.add(key);

System.out.println(set.contains(key));           // true
System.out.println(set.contains("test"));        // true — but "test" is a String!
// The String "test" and BadEquals "test" have the same hashCode
// but equals() is not symmetric: key.equals("test") is true, but "test".equals(key) is false
```

**Result**: Inconsistent behavior — the set may contain both the `BadEquals` object and the `String` "test" even though they're considered "equal" by one direction of `equals()`.

## The Correct Way

```java
public class Person {
    private final String name;  // final — immutable
    private final int age;      // final — immutable

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Person)) return false;
        Person person = (Person) o;
        return age == person.age && Objects.equals(name, person.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);  // Consistent with equals
    }
}
```

## Key Rules for HashSet Keys

1. **Always override both `hashCode()` and `equals()`** — never just one
2. **Use the same fields in both methods** — if a field is used in `equals()`, it must be used in `hashCode()`
3. **Use immutable fields** — mutable fields can change the hash code after insertion
4. **Ensure consistency** — `hashCode()` must return the same value for the lifetime of the object
5. **Equal objects must have equal hash codes** — this is the fundamental contract

## Common Interview Question

**Q**: What happens if you put a `Person` object into a `HashSet`, then change the `Person`'s name (which is used in `hashCode()`)?

**A**: The object becomes "lost" — it's still in the set, but `contains()` and `remove()` can't find it because the hash code changed, putting it in the wrong bucket.

## Related

- See [`hashcode-equals-contract.md`](hashcode-equals-contract.md) for the full contract
- See [`hashmap-internals.md`](hashmap-internals.md) for how HashMap uses hash codes
