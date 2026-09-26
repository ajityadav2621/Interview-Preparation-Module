# Design Patterns Coding Questions

## 1. Singleton Pattern

```java
// Eager initialization
public class SingletonEager {
    private static final SingletonEager INSTANCE = new SingletonEager();
    private SingletonEager() {}
    public static SingletonEager getInstance() { return INSTANCE; }
}

// Lazy initialization with double-checked locking
public class SingletonLazy {
    private volatile static SingletonLazy instance;
    private SingletonLazy() {}
    
    public static SingletonLazy getInstance() {
        if (instance == null) {
            synchronized (SingletonLazy.class) {
                if (instance == null) {
                    instance = new SingletonLazy();
                }
            }
        }
        return instance;
    }
}

// Bill Pugh (inner static class) — best approach
public class SingletonBest {
    private SingletonBest() {}
    
    private static class SingletonHolder {
        private static final SingletonBest INSTANCE = new SingletonBest();
    }
    
    public static SingletonBest getInstance() {
        return SingletonHolder.INSTANCE;
    }
}

// Enum singleton — most robust
public enum SingletonEnum {
    INSTANCE;
    public void doSomething() { /* ... */ }
}
```

---

## 2. Factory Pattern

```java
// Simple Factory
public class ShapeFactory {
    public Shape createShape(String type) {
        switch (type.toLowerCase()) {
            case "circle": return new Circle();
            case "rectangle": return new Rectangle();
            case "square": return new Square();
            default: throw new IllegalArgumentException("Unknown shape: " + type);
        }
    }
}

// Factory Method
abstract class Creator {
    public Product createProduct() {
        Product product = factoryMethod();
        return product;
    }
    protected abstract Product factoryMethod();
}

class ConcreteCreator extends Creator {
    @Override
    protected Product factoryMethod() {
        return new ConcreteProduct();
    }
}

// Abstract Factory
interface AbstractFactory {
    Color getColor(String color);
    Shape getShape(String shape);
}

class ShapeFactory implements AbstractFactory {
    public Shape getShape(String shapeType) { /* ... */ }
    public Color getColor(String color) { return null; }
}

class ColorFactory implements AbstractFactory {
    public Color getColor(String color) { /* ... */ }
    public Shape getShape(String shapeType) { return null; }
}
```

---

## 3. Builder Pattern

```java
public class User {
    private final String firstName;
    private final String lastName;
    private final int age;
    private final String email;
    
    private User(Builder builder) {
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.age = builder.age;
        this.email = builder.email;
    }
    
    public static class Builder {
        private final String firstName;  // Required
        private final String lastName;   // Required
        private int age;                 // Optional
        private String email;            // Optional
        
        public Builder(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }
        
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        
        public Builder email(String email) {
            this.email = email;
            return this;
        }
        
        public User build() {
            return new User(this);
        }
    }
}

// Usage
User user = new User.Builder("John", "Doe")
    .age(30)
    .email("john@example.com")
    .build();
```

---

## 4. Observer Pattern

```java
// Using Java's built-in Observer (deprecated in Java 9)
// Better: Use custom implementation or event bus

public interface Observer<T> {
    void update(T data);
}

public class Subject<T> {
    private List<Observer<T>> observers = new ArrayList<>();
    
    public void addObserver(Observer<T> observer) {
        observers.add(observer);
    }
    
    public void removeObserver(Observer<T> observer) {
        observers.remove(observer);
    }
    
    public void notifyObservers(T data) {
        for (Observer<T> observer : observers) {
            observer.update(data);
        }
    }
}

// Usage
Subject<String> subject = new Subject<>();
subject.addObserver(data -> System.out.println("Received: " + data));
subject.notifyObservers("Hello World");
```

---

## 5. Strategy Pattern

```java
public interface PaymentStrategy {
    void pay(double amount);
}

public class CreditCardStrategy implements PaymentStrategy {
    private String cardNumber;
    public void pay(double amount) {
        System.out.println("Paid " + amount + " with credit card");
    }
}

public class PayPalStrategy implements PaymentStrategy {
    private String email;
    public void pay(double amount) {
        System.out.println("Paid " + amount + " with PayPal");
    }
}

public class ShoppingCart {
    private PaymentStrategy paymentStrategy;
    
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }
    
    public void checkout(double amount) {
        paymentStrategy.pay(amount);
    }
}
```

---

## 6. Decorator Pattern

```java
public interface Coffee {
    double getCost();
    String getDescription();
}

public class SimpleCoffee implements Coffee {
    public double getCost() { return 2.0; }
    public String getDescription() { return "Simple coffee"; }
}

public class MilkDecorator implements Coffee {
    private Coffee coffee;
    public MilkDecorator(Coffee coffee) { this.coffee = coffee; }
    
    public double getCost() { return coffee.getCost() + 0.5; }
    public String getDescription() { return coffee.getDescription() + ", milk"; }
}

public class SugarDecorator implements Coffee {
    private Coffee coffee;
    public SugarDecorator(Coffee coffee) { this.coffee = coffee; }
    
    public double getCost() { return coffee.getCost() + 0.2; }
    public String getDescription() { return coffee.getDescription() + ", sugar"; }
}

// Usage
Coffee coffee = new SugarDecorator(new MilkDecorator(new SimpleCoffee()));
System.out.println(coffee.getDescription());  // "Simple coffee, milk, sugar"
System.out.println(coffee.getCost());         // 2.7
```

---

## 7. Proxy Pattern

```java
// JDK Dynamic Proxy
public class LoggingProxy implements InvocationHandler {
    private Object target;
    
    public LoggingProxy(Object target) {
        this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before: " + method.getName());
        Object result = method.invoke(target, args);
        System.out.println("After: " + method.getName());
        return result;
    }
    
    public static <T> T wrap(T target, Class<T> interfaceType) {
        return (T) Proxy.newProxyInstance(
            interfaceType.getClassLoader(),
            new Class[]{interfaceType},
            new LoggingProxy(target)
        );
    }
}

// Usage
UserService userService = new UserServiceImpl();
UserService proxy = LoggingProxy.wrap(userService, UserService.class);
proxy.getUser(123);  // Logs before and after
```

---

## 8. Adapter Pattern

```java
// Adaptee
public class WeatherService {
    public String getTemperatureInFahrenheit() {
        return "72F";
    }
}

// Target interface
public interface TemperatureProvider {
    String getTemperatureInCelsius();
}

// Adapter
public class WeatherAdapter implements TemperatureProvider {
    private WeatherService weatherService;
    
    public WeatherAdapter(WeatherService weatherService) {
        this.weatherService = weatherService;
    }
    
    @Override
    public String getTemperatureInCelsius() {
        String fahrenheit = weatherService.getTemperatureInFahrenheit();
        // Convert F to C
        int f = Integer.parseInt(fahrenheit.replace("F", ""));
        int c = (f - 32) * 5 / 9;
        return c + "C";
    }
}
```

---

## 9. Template Method Pattern

```java
public abstract class DataProcessor {
    // Template method
    public final void process() {
        readData();
        processData();
        writeData();
    }
    
    protected abstract void readData();
    protected abstract void processData();
    
    protected void writeData() {
        System.out.println("Writing data to database");
    }
}

public class CSVProcessor extends DataProcessor {
    @Override
    protected void readData() {
        System.out.println("Reading CSV file");
    }
    
    @Override
    protected void processData() {
        System.out.println("Processing CSV data");
    }
}
```

---

## 10. Command Pattern

```java
public interface Command {
    void execute();
    void undo();
}

public class LightOnCommand implements Command {
    private Light light;
    
    public LightOnCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.turnOn();
    }
    
    @Override
    public void undo() {
        light.turnOff();
    }
}

public class RemoteControl {
    private Command command;
    
    public void setCommand(Command command) {
        this.command = command;
    }
    
    public void pressButton() {
        command.execute();
    }
}
```

---

## Key Patterns Summary

| Pattern | Category | Purpose |
|---------|----------|---------|
| Singleton | Creational | One instance, global access |
| Factory | Creational | Create objects without specifying class |
| Builder | Creational | Build complex objects step by step |
| Observer | Behavioral | One-to-many dependency |
| Strategy | Behavioral | Select algorithm at runtime |
| Decorator | Structural | Add responsibilities dynamically |
| Proxy | Structural | Control access to an object |
| Adapter | Structural | Make incompatible interfaces work |
| Template Method | Behavioral | Define algorithm skeleton |
| Command | Behavioral | Encapsulate requests as objects |
