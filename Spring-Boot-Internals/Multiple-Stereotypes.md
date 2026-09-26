# What Happens When @Controller, @Service, @Repository Are on Same Class?

## Component Scanning Behavior

### 1. **Detection**
- Spring scans for stereotype annotations
- Multiple annotations on same class are all detected
- Bean is registered once with multiple aliases

### 2. **Bean Registration**
```java
@Controller
@Service
@Component
public class UserService {
    // Bean registered as:
    // - "userController" (from @Controller)
    // - "userService" (from @Service) 
    // - "userService" (from @Component, default)
}
```

### 3. **Potential Issues**
- **Ambiguous bean definitions**: Multiple bean names
- **AOP confusion**: Different proxies for different roles
- **Testing complexity**: Hard to mock specific roles

## Best Practice
- Use one stereotype annotation per class
- `@Controller` for MVC controllers
- `@Service` for business logic
- `@Repository` for data access

## Interview Tip
Explain that while technically allowed, it's bad practice. Each annotation serves a specific purpose and enables proper AOP and testing.