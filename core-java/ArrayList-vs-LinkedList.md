# ArrayList vs LinkedList

## Core Differences

| Feature | ArrayList | LinkedList |
|---------|-----------|------------|
| **Internal Structure** | Dynamic array | Doubly linked list |
| **Memory Usage** | Contiguous memory | Non-contiguous, extra memory for pointers |
| **Get by Index** | O(1) | O(n) |
| **Add at End** | O(1) amortized | O(1) |
| **Add at Beginning** | O(n) | O(1) |
| **Add in Middle** | O(n) | O(n) to find position, O(1) to insert |
| **Remove** | O(n) | O(n) to find, O(1) to remove |
| **Memory Overhead** | Less (only array) | More (each node has prev/next pointers) |

## When to Use Each

### Use ArrayList When:
- **Frequent indexing operations** - get(i) is O(1)
- **Mostly append operations** - add at end is efficient
- **Memory efficiency is important** - less overhead
- **Better cache locality** - elements are contiguous in memory
- **Iterating with index** - for loops with random access

```java
// Good for reading data
List<String> users = new ArrayList<>();
for (int i = 0; i < users.size(); i++) {
    String user = users.get(i); // Fast O(1)
}
```

### Use LinkedList When:
- **Frequent insertions/deletions at beginning or middle**
- **Queue/Deque operations** - add/remove from both ends
- **No random access needed** - sequential access only
- **Size changes frequently** - no resizing overhead

```java
// Good for task queue
Queue<String> taskQueue = new LinkedList<>();
taskQueue.offer("task1"); // Fast O(1)
taskQueue.poll(); // Fast O(1)
```

## Performance Considerations

### ArrayList Performance
- **Best for**: read-heavy workloads, indexed access
- **Worst for**: frequent insertions at beginning
- **Memory**: contiguous, better cache performance

### LinkedList Performance
- **Best for**: frequent modifications at both ends
- **Worst for**: random access, memory overhead
- **Memory**: scattered, poor cache performance

## Interview Tip
Always ask: "What operations will be most frequent?" The answer determines the choice. Mention that for most use cases, ArrayList is preferred unless you specifically need LinkedList's insertion advantages.