# Handling Eventual Consistency

## What is Eventual Consistency?
Updates to data will eventually propagate to all replicas, but temporary inconsistencies may exist.

## Strategies

### 1. **Read Your Writes**
```java
// Ensure user sees their own updates immediately
public User getUser(Long userId, boolean isOwner) {
    if (isOwner) {
        // Read from primary for owner
        return primaryRepository.findById(userId);
    } else {
        // Read from replica for others
        return replicaRepository.findById(userId);
    }
}
```

### 2. **Version Vectors**
```java
// Track version for conflict detection
class VersionedData<T> {
    T data;
    Map<String, Integer> version; // Vector clock
}

// Detect and resolve conflicts
public void merge(VersionedData<T> local, VersionedData<T> remote) {
    if (local.version.equals(remote.version)) {
        return; // Same version
    }
    
    // Resolve based on business rules
    if (isNewer(local.version, remote.version)) {
        // Use local version
    } else {
        // Use remote version
    }
}
```

### 3. **Conflict Resolution**
```java
// Last-write-wins (LWW)
@Version
private Long version;

// Custom resolution
public void resolve(Order a, Order b) {
    if (a.getTimestamp().isAfter(b.getTimestamp())) {
        return a;
    } else if (b.getTimestamp().isAfter(a.getTimestamp())) {
        return b;
    } else {
        // Same timestamp - merge fields
        return merge(a, b);
    }
}
```

## Interview Tip
Explain that eventual consistency is acceptable for many systems but not for financial data. Choose consistency model based on business requirements.