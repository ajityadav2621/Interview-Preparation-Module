# Java 8 – Coding Questions

## Q1: Filter and Transform List
Given a list of strings, return a list of uppercase strings that have length > 3, sorted alphabetically.

```java
public List<String> filterAndTransform(List<String> input) {
    return input.stream()
        .filter(s -> s.length() > 3)
        .map(String::toUpperCase)
        .sorted()
        .collect(Collectors.toList());
}
```

## Q2: Find Duplicates Using Streams
Find all duplicate elements in a list using the Stream API.

```java
public List<Integer> findDuplicates(List<Integer> numbers) {
    return numbers.stream()
        .collect(Collectors.groupingBy(i -> i, Collectors.counting()))
        .entrySet().stream()
        .filter(e -> e.getValue() > 1)
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());
}
```

## Q3: Convert List to Map
Convert a list of Employee objects to a Map using employee ID as key and name as value.

```java
public Map<Long, String> toMap(List<Employee> employees) {
    return employees.stream()
        .collect(Collectors.toMap(Employee::getId, Employee::getName));
}

// Handling duplicates:
public Map<Long, String> toMapWithMerge(List<Employee> employees) {
    return employees.stream()
        .collect(Collectors.toMap(
            Employee::getId,
            Employee::getName,
            (existing, replacement) -> existing + ", " + replacement
        ));
}
```

## Q4: Lazy Evaluation Demo
Demonstrate lazy evaluation in Streams — intermediate operations are not executed until terminal.

```java
public void demonstrateLazyEvaluation() {
    List<String> result = Stream.of("apple", "banana", "cherry", "date")
        .filter(s -> {
            System.out.println("Filtering: " + s);
            return s.length() > 5;
        })
        .map(s -> {
            System.out.println("Mapping: " + s);
            return s.toUpperCase();
        })
        .collect(Collectors.toList());
    // Only prints for elements that pass the filter
}
```

## Q5: Parallel Stream Word Count
Count words in a large text using parallel streams.

```java
public long countWords(String text) {
    return Arrays.stream(text.split("\\s+"))
        .parallel()
        .filter(word -> !word.isEmpty())
        .count();
}
```

## Q6: Date Formatter
Format a given LocalDate into multiple formats using DateTimeFormatter.

```java
public Map<String, String> formatDate(LocalDate date) {
    Map<String, String> formats = new LinkedHashMap<>();
    formats.put("ISO", date.format(DateTimeFormatter.ISO_DATE));
    formats.put("US", date.format(DateTimeFormatter.ofPattern("MM/dd/yyyy")));
    formats.put("EU", date.format(DateTimeFormatter.ofPattern("dd.MM.yyyy")));
    formats.put("Custom", date.format(DateTimeFormatter.ofPattern("yyyy年MM月dd日")));
    return formats;
}
```

## Q7: CompletableFuture Timeout
Complete a task with a timeout using CompletableFuture.

```java
public String executeWithTimeout(Callable<String> task, long timeoutSeconds) {
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        try { return task.call(); }
        catch (Exception e) { throw new RuntimeException(e); }
    });
    try {
        return future.get(timeoutSeconds, TimeUnit.SECONDS);
    } catch (TimeoutException e) {
        future.cancel(true);
        throw new RuntimeException("Task timed out");
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

## Q8: Group Anagrams
Group anagrams together from a list of strings using Stream API.

```java
public List<List<String>> groupAnagrams(List<String> words) {
    return words.stream()
        .collect(Collectors.groupingBy(
            word -> word.chars()
                .sorted()
                .collect(StringBuilder::new, StringBuilder::appendCodePoint, StringBuilder::append)
                .toString()
        ))
        .values().stream()
        .collect(Collectors.toList());
}
```

## Q9: Optional-based Safe Division
Implement a safe division method that returns Optional to avoid division by zero.

```java
public Optional<Double> safeDivide(double numerator, double denominator) {
    if (denominator == 0) return Optional.empty();
    return Optional.of(numerator / denominator);
}

// Usage
Optional<Double> result = safeDivide(10, 2);
double value = result.orElse(0.0);
double valueOrThrow = result.orElseThrow(() -> new ArithmeticException("Division by zero"));
```

## Q10: Flatten and Filter Matrix
Given a 2D array (matrix), flatten it and filter elements greater than a threshold using Streams.

```java
public List<Integer> flattenAndFilter(int[][] matrix, int threshold) {
    return Arrays.stream(matrix)
        .flatMapToInt(Arrays::stream)
        .filter(n -> n > threshold)
        .boxed()
        .collect(Collectors.toList());
}
```
