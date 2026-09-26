# Why Must equals() and hashCode() Follow a Contract?

## The Contract

### 1. **Consistency**
- During an object's lifetime, hashCode() can return different values only if the object is modified
- If you override equals(), you MUST override hashCode()

### 2. **Equality implies equal hash codes**
- If a.equals(b) returns true, then a.hashCode() == b.hashCode() MUST be true

### 3. **Equal hash codes don't imply equality**
- Different objects can have the same hash code (collision)

## Why This Matters

### HashMap Example
```java
class BadKey {
    String name;
    
    @Override
    public boolean equals(Object obj) {
        // Implemented correctly
    }
    
    // forgot to override hashCode()
}

HashMap<BadKey, String> map = new HashMap<>();
BadKey k1 = new BadKey("test");
BadKey k2 = new BadKey("test");

map.put(k1, "value1");
map.get(k2); // Returns null! Because k2.hashCode() != k1.hashCode()
```

### HashSet Example
```java
Set<BadKey> set = new HashSet<>();
BadKey k1 = new BadKey("test");
BadKey k2 = new BadKey("test");

set.add(k1);
set.add(k2); // Both added! Should have been considered duplicate
```

## Proper Implementation

### Good Example
```java
class GoodKey {
    String name;
    int age;
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        GoodKey other = (GoodKey) obj;
        return age == other.age && Objects.equals(name, other.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

## Interview Tip
Mention that IDEs can auto-generate these methods correctly. Also explain that if you override equals(), you must override hashCode() to maintain the contract.