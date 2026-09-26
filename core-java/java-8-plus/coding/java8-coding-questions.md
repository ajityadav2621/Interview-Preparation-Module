# Java 8+ Coding Questions

## 1. Filter and Collect with Streams

```java
// Problem: Given a list of employees, find all employees in the "Engineering" department
// who earn more than $80,000, and return their names sorted alphabetically.

public class EmployeeFilter {
    static class Employee {
        String name;
        String department;
        double salary;

        Employee(String name, String department, double salary) {
            this.name = name;
            this.department = department;
            this.salary = salary;
        }
    }

    public static List<String> filterEmployees(List<Employee> employees) {
        return employees.stream()
            .filter(e -> e.department.equals("Engineering"))
            .filter(e -> e.salary > 80000)
            .map(e -> e.name)
            .sorted()
            .collect(Collectors.toList());
    }
}
```

---

## 2. Group By with Collectors

```java
// Problem: Group employees by department and count them

public static Map<String, Long> countByDepartment(List<Employee> employees) {
    return employees.stream()
        .collect(Collectors.groupingBy(
            e -> e.department,
            Collectors.counting()
        ));
}

// Problem: Group employees by department and get average salary

public static Map<String, Double> averageSalaryByDepartment(List<Employee> employees) {
    return employees.stream()
        .collect(Collectors.groupingBy(
            e -> e.department,
            Collectors.averagingDouble(e -> e.salary)
        ));
}

// Problem: Group employees by department, then by salary range

public static Map<String, Map<String, List<Employee>>> groupByDeptAndSalary(
        List<Employee> employees) {
    return employees.stream()
        .collect(Collectors.groupingBy(
            e -> e.department,
            Collectors.groupingBy(e -> {
                if (e.salary < 50000) return "Low";
                if (e.salary < 100000) return "Medium";
                return "High";
            })
        ));
}
```

---

## 3. Partitioning with Collectors

```java
// Problem: Partition employees into senior (salary > 100k) and junior

public static Map<Boolean, List<Employee>> partitionBySalary(List<Employee> employees) {
    return employees.stream()
        .collect(Collectors.partitioningBy(e -> e.salary > 100000));
}
```

---

## 4. Find First Non-Repeating Character

```java
public static Optional<Character> firstNonRepeating(String str) {
    Map<Character, Long> charCount = str.chars()
        .mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(
            c -> c,
            Collectors.counting()
        ));

    return str.chars()
        .mapToObj(c -> (char) c)
        .filter(c -> charCount.get(c) == 1)
        .findFirst();
}
```

---

## 5. Flatten Nested Lists

```java
// Problem: Given a list of lists, flatten into a single list
public static List<Integer> flatten(List<List<Integer>> nested) {
    return nested.stream()
        .flatMap(List::stream)
        .collect(Collectors.toList());
}

// Problem: Given a list of sentences, get all unique words
public static List<String> uniqueWords(List<String> sentences) {
    return sentences.stream()
        .flatMap(s -> Arrays.stream(s.split("\\s+")))
        .map(String::toLowerCase)
        .distinct()
        .sorted()
        .collect(Collectors.toList());
}
```

---

## 6. Custom Collector

```java
// Problem: Join names with a delimiter, prefix, and suffix
public static String joinNames(List<Employee> employees) {
    return employees.stream()
        .map(e -> e.name)
        .collect(Collectors.joining(", ", "[", "]"));
}

// Problem: Collect to a Map with merge function
public static Map<String, Employee> toMap(List<Employee> employees) {
    return employees.stream()
        .collect(Collectors.toMap(
            e -> e.name,
            e -> e,
            (existing, replacement) -> replacement  // Merge function for duplicates
        ));
}
```

---

## 7. Optional Chaining

```java
// Problem: Safely get the city name from a user, handling nulls at each level

public static String getUserCity(User user) {
    return Optional.ofNullable(user)
        .map(User::getAddress)
        .map(Address::getCity)
        .map(City::getName)
        .orElse("Unknown");
}

// Problem: Get the first active user's email, or throw an exception
public static String getFirstActiveUserEmail(List<User> users) {
    return users.stream()
        .filter(User::isActive)
        .findFirst()
        .map(User::getEmail)
        .orElseThrow(() -> new IllegalStateException("No active users"));
}
```

---

## 8. CompletableFuture Chain

```java
// Problem: Fetch user, then fetch their orders, then calculate total
public static CompletableFuture<Double> calculateTotal(String userId) {
    return fetchUser(userId)
        .thenCompose(user -> fetchOrders(user.getId()))
        .thenApply(orders -> orders.stream()
            .mapToDouble(Order::getAmount)
            .sum());
}

private static CompletableFuture<User> fetchUser(String userId) {
    return CompletableFuture.supplyAsync(() -> 
        database.findUser(userId));
}

private static CompletableFuture<List<Order>> fetchOrders(String userId) {
    return CompletableFuture.supplyAsync(() -> 
        orderService.getOrders(userId));
}
```

---

## 9. Parallel Stream Processing

```java
// Problem: Process a large list of numbers in parallel
public static long countPrimes(List<Integer> numbers) {
    return numbers.parallelStream()
        .filter(PrimeChecker::isPrime)
        .count();
}

// Problem: Sum all numbers in parallel
public static long parallelSum(int[] numbers) {
    return Arrays.stream(numbers)
        .parallel()
        .mapToLong(Long::valueOf)
        .sum();
}
```

---

## 10. Method References and Functional Interfaces

```java
// Problem: Sort employees by name, then by salary
public static List<Employee> sortEmployees(List<Employee> employees) {
    return employees.stream()
        .sorted(Comparator.comparing(Employee::getName)
            .thenComparing(Employee::getSalary))
        .collect(Collectors.toList());
}

// Problem: Convert list to map using method reference
public static Map<String, Double> nameToSalary(List<Employee> employees) {
    return employees.stream()
        .collect(Collectors.toMap(
            Employee::getName,
            Employee::getSalary,
            (existing, replacement) -> replacement
        ));
}
```

---

## 11. Reduce Operations

```java
// Problem: Find the employee with the highest salary
public static Optional<Employee> highestPaid(List<Employee> employees) {
    return employees.stream()
        .reduce((e1, e2) -> e1.salary > e2.salary ? e1 : e2);
}

// Problem: Concatenate all names
public static String concatenateNames(List<Employee> employees) {
    return employees.stream()
        .map(e -> e.name)
        .reduce("", (a, b) -> a + b + ", ");
}

// Problem: Sum all salaries
public static double totalSalary(List<Employee> employees) {
    return employees.stream()
        .mapToDouble(e -> e.salary)
        .sum();
}
```

---

## 12. Custom Predicate Composition

```java
// Problem: Filter employees with complex conditions
public static List<Employee> filterComplex(List<Employee> employees) {
    Predicate<Employee> isEngineer = e -> e.department.equals("Engineering");
    Predicate<Employee> isSenior = e -> e.salary > 100000;
    Predicate<Employee> isFromNYC = e -> e.location.equals("NYC");

    return employees.stream()
        .filter(isEngineer.and(isSenior).or(isFromNYC))
        .collect(Collectors.toList());
}
```

---

## Key Patterns

| Pattern | Stream Operation |
|---------|-----------------|
| Filter | `filter()` |
| Transform | `map()` |
| Flatten | `flatMap()` |
| Aggregate | `reduce()`, `collect()` |
| Group | `Collectors.groupingBy()` |
| Partition | `Collectors.partitioningBy()` |
| Join | `Collectors.joining()` |
| Count | `count()` |
| Find | `findFirst()`, `findAny()` |
| Match | `anyMatch()`, `allMatch()`, `noneMatch()` |
| Sort | `sorted()` |
| Distinct | `distinct()` |
| Limit | `limit()` |
| Skip | `skip()` |
