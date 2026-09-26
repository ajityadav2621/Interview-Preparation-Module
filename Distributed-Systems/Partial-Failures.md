# Handling Partial Failures

## Strategies

### 1. **Graceful Degradation**
```java
// Return partial results when possible
public List<User> getUsers() {
    List<User> users = new ArrayList<>();
    
    // Try primary source
    try {
        users.addAll(userService.getFromDatabase());
    } catch (Exception ex) {
        log.warn("Database unavailable, using cache");
        users.addAll(userService.getFromCache());
    }
    
    // Try to enhance with additional data
    try {
        enhanceWithPreferences(users);
    } catch (Exception ex) {
        log.warn("Preferences service unavailable");
    }
    
    return users;
}
```

### 2. **Fallback Responses**
```java
@CircuitBreaker(name = "recommendation", fallbackMethod = "defaultRecommendations")
public List<Product> getRecommendations(User user) {
    return recommendationService.getFor(user);
}

public List<Product> defaultRecommendations(User user, Exception ex) {
    return Arrays.asList(popularProduct, newReleaseProduct);
}
```

### 3. **Compensation Transactions**
```java
// If one operation fails, compensate previous ones
public void processOrder(Order order) {
    try {
        inventoryService.reserve(order.getItems());
        paymentService.charge(order.getPayment());
        shippingService.createShipment(order);
    } catch (Exception ex) {
        // Compensate
        inventoryService.release(order.getItems());
        paymentService.refund(order.getPayment());
        throw new OrderProcessingException("Order failed", ex);
    }
}
```

## Interview Tip
Explain that partial failures are normal in distributed systems. The goal is to provide the best possible user experience despite failures.