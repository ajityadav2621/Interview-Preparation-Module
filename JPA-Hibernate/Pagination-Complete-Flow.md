# Pagination in Spring Boot - Complete Flow

## Database Level

### 1. **Database Pagination**
```sql
-- MySQL/MariaDB
SELECT * FROM users 
ORDER BY created_at DESC 
LIMIT 10 OFFSET 20;

-- PostgreSQL
SELECT * FROM users 
ORDER BY created_at DESC 
LIMIT 10 OFFSET 20;

-- Oracle
SELECT * FROM (
  SELECT ROWNUM rn, t.* FROM (
    SELECT * FROM users 
    ORDER BY created_at DESC
  ) t WHERE ROWNUM <= 30
) WHERE rn > 20;
```

### 2. **JPA Pagination**
```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Method with pagination
    Page<User> findByStatus(String status, Pageable pageable);
    
    // Method with sorting
    List<User> findByStatus(String status, Sort sort);
}
```

## Repository Level

### 1. **Pageable Interface**
```java
// Create Pageable object
Pageable pageable = PageRequest.of(pageNumber, pageSize, Sort.by("createdAt").descending());

// Execute query with pagination
Page<User> userPage = userRepository.findByStatus("ACTIVE", pageable);

// Access pagination data
List<User> users = userPage.getContent();
int totalPages = userPage.getTotalPages();
long totalElements = userPage.getTotalElements();
boolean hasNext = userPage.hasNext();
```

### 2. **Custom Repository Implementation**
```java
@Service
public class UserServiceImpl implements UserService {
    
    private final UserRepository userRepository;
    
    public UserServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public Page<User> getAllUsers(Pageable pageable) {
        // Add dynamic sorting and filtering
        Specification<User> spec = createSpecification(pageable);
        return userRepository.findall(spec, pageable);
    }
}
```

## Service Level

### 1. **Service Layer with Pagination**
```java
@Service
public class UserService {
    
    private final UserRepository userRepository;
    private final ModelMapper modelMapper;
    
    public UserService(UserRepository userRepository, ModelMapper modelMapper) {
        this.userRepository = userRepository;
        this.modelMapper = modelMapper;
    }
    
    public Page<UserDTO> getAllUsers(int page, int size, String sortBy, String direction) {
        // Validate and set defaults
        page = Math.max(0, page);
        size = Math.min(100, Math.max(1, size)); // Max 100 per page
        
        // Create Pageable with sorting
        Sort sort = Sort.by(new Sort.Order(Sort.Direction.valueOf(direction.toUpperCase()), sortBy));
        Pageable pageable = PageRequest.of(page, size, sort);
        
        // Execute query
        Page<User> userPage = userRepository.findall(pageable);
        
        // Map to DTO and return
        return userPage.map(user -> modelMapper.map(user, UserDTO.class));
    }
}
```

## Controller Level

### 1. **REST Controller with Pagination**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @GetMapping
    public ResponseEntity<Page<UserDTO>> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "createdAt") String sortBy,
            @RequestParam(defaultValue = "DESC") String direction) {
        
        Page<UserDTO> userPage = userService.getAllUsers(page, size, sortBy, direction);
        
        return ResponseEntity.ok(userPage);
    }
}
```

### 2. **Response with Pagination Metadata**
```json
{
  "content": [
    {"id": 1, "name": "John", "email": "john@example.com"},
    {"id": 2, "name": "Jane", "email": "jane@example.com"}
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 10,
    "sort": {
      "empty": false,
      "sorted": true,
      "unsorted": false
    }
  },
  "totalPages": 5,
  "totalElements": 50,
  "last": false,
  "first": true,
  "size": 10,
  "number": 0,
  "empty": false
}
```

## Complete Flow

### 1. **Request Flow**
```
HTTP GET /api/users?page=1&size=10&sortBy=createdAt&direction=DESC
    ↓
Controller: @RequestParam parameters
    ↓
Service: Creates Pageable object
    ↓
Repository: Executes JPA query with LIMIT/OFFSET
    ↓
Database: Returns paginated results
    ↓
Service: Maps to DTOs
    ↓
Controller: Returns Page<UserDTO> with metadata
```

## Interview Tip
Explain that pagination happens at the database level using LIMIT/OFFSET. Spring Data JPA abstracts this with the Pageable interface. The complete flow is Controller → Service → Repository → Database.