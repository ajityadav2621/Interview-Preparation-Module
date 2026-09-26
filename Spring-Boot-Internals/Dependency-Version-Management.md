# How Spring Boot Manages Dependency Versions

## BOM (Bill of Materials)

### 1. **spring-boot-dependencies**
- Parent POM defines all dependency versions
- Centralized version management
- No need to specify versions for starter dependencies

### 2. **Dependency Management**
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.1.0</version>
</parent>

<dependencies>
    <!-- No version needed - managed by parent -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Override if needed -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>6.0.0</version>  <!-- Override managed version -->
    </dependency>
</dependencies>
```

### 3. **Version Properties**
```xml
<properties>
    <spring-boot.version>3.1.0</spring-boot.version>
    <java.version>17</java.version>
    
    <!-- Override specific versions -->
    <jackson.version>2.15.0</jackson.version>
</properties>
```

## Interview Tip
Explain that Spring Boot's parent POM provides tested dependency combinations. This prevents version conflicts and ensures compatibility.