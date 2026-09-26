# Java Core Concepts - OOP, Strings, and Memory

## 1. String Interning and Memory

### String Pool
Special memory area in JVM for String literals. Same strings share the same memory.

```java
String s1 = "Hello"; // Pool
String s2 = "Hello"; // Same reference as s1
String s3 = new String("Hello"); // Heap (different object)

System.out.println(s1 == s2); // true (same pool reference)
System.out.println(s1 == s3); // false (different objects)
```

### Interning
```java
String s4 = s3.intern(); // Returns pooled version
System.out.println(s1 == s4); // true
```

## 2. Final, Finally, and finalize()

### Final Keyword
```java
final int x = 10; // Cannot be changed
final class MyClass {} // Cannot be extended
public final void method() {} // Cannot be overridden
```

### Finally Block
```java
try {
    // Code that might throw exception
} catch (Exception e) {
    // Handle exception
} finally {
    // Always executes (cleanup code)
    // Even if exception occurs or return statement
}
```

### finalize() Method (Deprecated)
```java
@Override
protected void finalize() throws Throwable {
    // Called before GC (deprecated in Java 9+)
    // Don't rely on this for cleanup
}
```

## 3. OOP Principles

### Encapsulation
```java
public class BankAccount {
    private double balance; // Hidden data
    
    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }
    
    public double getBalance() {
        return balance; // Controlled access
    }
}
```

### Inheritance
```java
class Animal {
    void makeSound() {
        System.out.println("Animal sound");
    }
}

class Dog extends Animal {
    @Override
    void makeSound() {
        System.out.println("Woof!");
    }
    
    void fetch() {
        System.out.println("Dog fetching");
    }
}
```

### Polymorphism
```java
// Same method, different behavior
Animal myDog = new Dog();
myDog.makeSound(); // "Woof!" (runtime polymorphism)

// Method overloading (compile-time)
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }
}
```

### Abstraction
```java
// Abstract class
abstract class Shape {
    abstract double area(); // No implementation
}

class Circle extends Shape {
    double radius;
    
    @Override
    double area() {
        return Math.PI * radius * radius;
    }
}

// Interface
interface Drivable {
    void start();
    void stop();
}

class Car implements Drivable {
    public void start() { /* ... */ }
    public void stop() { /* ... */ }
}
```

## 4. this and super Keywords

### this Keyword
```java
class Person {
    String name;
    
    Person(String name) {
        this.name = name; // Distinguish instance variable from parameter
    }
    
    void introduce() {
        System.out.println("Hi, I'm " + this.name);
    }
    
    Person createCopy() {
        return new Person(this.name); // Pass current object
    }
}
```

### super Keyword
```java
class Animal {
    Animal(String type) {
        System.out.println("Animal: " + type);
    }
}

class Dog extends Animal {
    Dog() {
        super("Dog"); // Call parent constructor
    }
    
    void speak() {
        super.speak(); // Call parent method (if exists)
    }
}
```

## 5. Static vs Instance

### Static Members
```java
class Counter {
    static int count = 0; // Shared across all instances
    
    Counter() {
        Counter.count++;
    }
    
    static void printCount() {
        System.out.println("Count: " + count);
    }
}
```

### Instance Members
```java
class Student {
    String name; // Each instance has its own
    int rollNo;
    
    void display() {
        System.out.println(name + " " + rollNo);
    }
}
```

## Core Coding Questions

### Q1: Reverse a String
```java
public static String reverse(String str) {
    return new StringBuilder(str).reverse().toString();
}

// Manual implementation
public static String reverseManual(String str) {
    char[] chars = str.toCharArray();
    StringBuilder result = new StringBuilder();
    for (int i = chars.length - 1; i >= 0; i--) {
        result.append(chars[i]);
    }
    return result.toString();
}
```

### Q2: Check if String is Palindrome
```java
public static boolean isPalindrome(String str) {
    String cleaned = str.replaceAll("[^a-zA-Z0-9]", "").toLowerCase();
    return cleaned.equals(new StringBuilder(cleaned).reverse().toString());
}
```

### Q3: Find Duplicate Characters
```java
public static void printDuplicates(String str) {
    Map<Character, Integer> count = new HashMap<>();
    for (char c : str.toCharArray()) {
        count.put(c, count.getOrDefault(c, 0) + 1);
    }
    
    for (Map.Entry<Character, Integer> entry : count.entrySet()) {
        if (entry.getValue() > 1) {
            System.out.println(entry.getKey() + ": " + entry.getValue());
        }
    }
}
```

### Q4: Fibonacci Series
```java
public static int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// Optimized with memoization
public static int fibonacciMemo(int n) {
    int[] memo = new int[n + 1];
    memo[0] = 0; memo[1] = 1;
    for (int i = 2; i <= n; i++) {
        memo[i] = memo[i - 1] + memo[i - 2];
    }
    return memo[n];
}
```

### Q5: Check if Array Contains Duplicate
```java
public static boolean hasDuplicates(int[] arr) {
    Set<Integer> set = new HashSet<>();
    for (int num : arr) {
        if (!set.add(num)) {
            return true; // Duplicate found
        }
    }
    return false;
}
```