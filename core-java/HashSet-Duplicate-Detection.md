# HashSet Duplicate Detection with Bad equals()/hashCode()

## How HashSet Works
- HashSet is backed by HashMap
- It uses hash code to find bucket
- Uses equals() to check for actual duplicates

## The Problem

### Bad Implementation
```java
class BadObject {
    String name;
    
    @Override
    public boolean equals(Object obj) {
        // Implemented correctly
    }
    
    // Forgot to override hashCode()
}

Set<BadObject> set = new HashSet<>();
BadObject obj1 = new BadObject("test");
BadObject obj2 = new BadObject("test");

set.add(obj1);
set.add(obj2); // Both added! Should have been considered duplicate
```

### Why It Fails
- Different hash codes → different buckets
- equals() never called because they're in different buckets
- Both objects added to set

## Proper Implementation
```java
class GoodObject {
    String name;
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        GoodObject other = (GoodObject) obj;
        return Objects.equals(name, other.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name);
    }
}
```

## Interview Tip
Explain that HashSet relies on both hash code (for bucket selection) and equals() (for actual comparison). If either is broken, duplicate detection fails.