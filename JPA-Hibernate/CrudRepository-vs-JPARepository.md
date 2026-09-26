# CrudRepository vs JpaSpecificationExecutor vs JpaRepository

## CrudRepository (Spring Data)
- **Purpose**: Basic CRUD operations
- **Methods**: save, findAll, findById, count, delete
- **Interface**: `org.springframework.data.repository.CrudRepository`
- **Features**: Generic CRUD operations only

```java
public interface UserRepository extends CrudRepository<User, Long> {
    // Only basic CRUD methods available
}
```

## JPA Repository (Spring Data JPA)
- **Purpose**: JPA-specific repository
- **Extends**: `CrudRepository` AND `JpaSpecificationExecutor`
- **Methods**: All CRUD methods + JPA-specific methods
- **Features**: Query methods, pagination, sorting, specifications

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Additional JPA-specific methods
    List<User> findByEmail(String email);
    
    // Pagination and sorting
    Page<User> findByAge(int age, Pageable pageable);
}
```

## Key Differences

| Feature | CrudRepository | JpaRepository |
|---------|---------------|---------------|
| **Methods** | Basic CRUD | CRUD + JPA-specific |
| **Pagination** | No | Yes |
| **Sorting** | No | Yes |
| **Batch Operations** | No | Yes |
| **Flush/Clear** | No | Yes |
| **Specifications** | No | Yes |

## When to Use Each

### Use CrudRepository When:
- **Simple CRUD operations** needed
- **Non-JPA databases** (Mongo, Cassandra)
- **Minimal repository functionality**

### Use JpaRepository When:
- **JPA databases** (Hibernate, EclipseLink)
- **Complex queries** with pagination/sorting
- **Batch operations** needed
- **Advanced JPA features** required

## Interview Tip
Explain that CrudRepository is the base interface, while JpaRepository extends it with JPA-specific features. Most Spring applications use JpaRepository.