# Streams vs Traditional Loops

## Core Differences

| Feature | Streams | Traditional Loops |
|---------|---------|-------------------|
| **Style** | Functional/Declarative | Imperative |
| **Performance** | Often slower | Usually faster |
| **Readability** | Concise, fluent | Explicit, verbose |
| **Parallelism** | Easy with parallelStream() | Manual |
| **Debugging** | Harder (lambdas) | Easier (stack traces) |
| **Order** | Maintains encounter order | Explicit control |

## When to Use Streams

### 1. **Complex Data Processing**
```java
// Traditional
List<String> activeUsers = new ArrayList<>();
for (User user : users) {
    if (user.isActive()) {
        activeUsers.add(user.getName());
    }
}
Collections.sort(activeUsers);

// Stream
List<String> activeUsers = users.stream()
    .filter(User::isActive)
    .map(User::getName)
    .sorted()
    .collect(Collectors.toList());
```

### 2. **Parallel Processing**
```java
// Easy parallelism
List<Result> results = largeDataset.parallelStream()
    .map(this::expensiveOperation)
    .collect(Collectors.toList());
```

### 3. **Aggregation Operations**
```java
// Simple aggregations
double averageAge = users.stream()
    .mapToInt(User::getAge)
    .average()
    .orElse(0.0);

Map<String, List<User>> usersByCity = users.stream()
    .collect(Collectors.groupingBy(User::getCity));
```

## When to Use Traditional Loops

### 1. **Performance-Critical Code**
```java
// Faster for simple operations
for (int i = 0; i < largeList.size(); i++) {
    process(largeList.get(i));
}
```

### 2. **Complex Control Flow**
```java
// Traditional loops handle complex logic better
for (int i = 0; i < items.size(); i++) {
    if (someCondition) {
        continue;
    }
    if (anotherCondition) {
        break;
    }
    // Complex processing
}
```

### 3. **Modifying Collections**
```java
// Easier to modify during iteration
for (int i = 0; i < list.size(); i++) {
    if (condition) {
        list.remove(i--); // Handle index carefully
    }
}
```

## Performance Considerations

### Streams Are Good For:
- Small to medium datasets
- Complex transformations
- Readability over performance

### Traditional Loops Are Good For:
- Large datasets
- Simple operations
- Performance-critical sections
- Complex control flow

## Interview Tip
Explain that streams are not always faster. For small datasets, traditional loops may be faster. For large datasets with parallel processing, streams can be faster. Choose based on specific needs.