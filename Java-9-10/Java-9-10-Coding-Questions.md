# Java 9–10 – Coding Questions

## Q1: Immutable Collection Usage
Create immutable collections and demonstrate what happens on modification attempts.

```java
public void demonstrateImmutability() {
    List<String> list = List.of("A", "B", "C");
    Set<Integer> set = Set.of(1, 2, 3);
    Map<String, Integer> map = Map.of("a", 1, "b", 2);

    // All throw UnsupportedOperationException
    try { list.add("D"); } catch (UnsupportedOperationException e) {
        System.out.println("List is immutable");
    }
    try { set.add(4); } catch (UnsupportedOperationException e) {
        System.out.println("Set is immutable");
    }
    try { map.put("c", 3); } catch (UnsupportedOperationException e) {
        System.out.println("Map is immutable");
    }
}
```

## Q2: var Type Inference
Use var in various contexts and identify valid vs invalid usages.

```java
public void varDemo() {
    var list = new ArrayList<String>();        // ✅ ArrayList<String>
    var map = new HashMap<String, Integer>();  // ✅ HashMap<String, Integer>
    var stream = IntStream.range(1, 10);       // ✅ IntStream
    var today = LocalDate.now();               // ✅ LocalDate
    
    // for-each with var
    for (var item : list) { System.out.println(item); } // ✅
    
    // lambda with var (Java 10+)
    var func = (var x) -> x * 2;               // ✅ Java 10+
    
    // var in lambda parameters
    var add = (var a, var b) -> a + b;          // ✅ Java 10+
    
    // var in try-with-resources
    try (var reader = Files.newBufferedReader(Paths.get("file.txt"))) {
        // ✅ Java 9+
    }
}
```

## Q3: takeWhile and dropWhile
Implement a pagination helper using takeWhile/dropWhile.

```java
public List<List<Integer>> paginate(List<Integer> items, int pageSize) {
    List<List<Integer>> pages = new ArrayList<>();
    List<Integer> remaining = new ArrayList<>(items);
    
    while (!remaining.isEmpty()) {
        List<Integer> page = remaining.stream()
            .limit(pageSize)
            .collect(Collectors.toList());
        pages.add(page);
        remaining = remaining.stream()
            .dropWhile(i -> page.contains(i))
            .collect(Collectors.toList());
    }
    return pages;
}

// Simpler approach
public List<List<Integer>> paginateSimple(List<Integer> items, int pageSize) {
    return IntStream.range(0, (items.size() + pageSize - 1) / pageSize)
        .mapToObj(i -> items.subList(i * pageSize, 
            Math.min((i + 1) * pageSize, items.size())))
        .collect(Collectors.toList());
}
```

## Q4: Optional Stream
Use Optional.stream() (Java 9) to filter and process optional values.

```java
public List<String> processOptionalValues(List<Optional<String>> optionals) {
    return optionals.stream()
        .flatMap(Optional::stream)  // Stream<String>
        .filter(s -> !s.isEmpty())
        .sorted()
        .collect(Collectors.toList());
}
```

## Q5: Module Information Extraction
Write code to inspect module information at runtime.

```java
public void inspectModule() {
    Module module = String.class.getModule();
    System.out.println("Name: " + module.getName());
    System.out.println("Is named: " + module.isNamed());
    System.out.println("Packages: " + module.getPackages());
    
    // All root modules
    ModuleLayer.boot().modules().forEach(m -> 
        System.out.println("Root: " + m.getName()));
}
```

## Q6: Private Interface Method Refactoring
Refactor a interface to use private methods to reduce code duplication.

```java
interface Validator {
    boolean validate(String input);
    
    default boolean validateEmail(String email) {
        if (email == null) return false;
        if (email.isEmpty()) return false;
        if (!email.contains("@")) return false;
        log("Email validation: " + email);
        return true;
    }
    
    default boolean validatePhone(String phone) {
        if (phone == null) return false;
        if (phone.isEmpty()) return false;
        if (!phone.matches("\\d{10}")) return false;
        log("Phone validation: " + phone);
        return true;
    }
    
    private void log(String message) {
        System.out.println("[VALIDATOR] " + message);
    }
}
```

## Q7: Stream ofNullable
Process a list that may contain null values using ofNullable.

```java
public List<String> processNullable(List<String> inputs) {
    return inputs.stream()
        .flatMap(s -> Stream.ofNullable(s == null ? null : s.trim()))
        .filter(s -> !s.isEmpty())
        .map(String::toUpperCase)
        .collect(Collectors.toList());
}
```

## Q8: Map.of with Large Data
Create a map with more than 10 entries using proper APIs.

```java
public Map<String, Integer> createLargeMap() {
    // Map.of limited to 10 entries
    // Use Map.ofEntries for more
    return Map.ofEntries(
        Map.entry("one", 1), Map.entry("two", 2),
        Map.entry("three", 3), Map.entry("four", 4),
        Map.entry("five", 5), Map.entry("six", 6),
        Map.entry("seven", 7), Map.entry("eight", 8),
        Map.entry("nine", 9), Map.entry("ten", 10),
        Map.entry("eleven", 11), Map.entry("twelve", 12)
    );
}
```
