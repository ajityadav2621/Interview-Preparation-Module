# Securing Health Check Endpoints

## Problem
Health check URLs can expose sensitive information to unauthorized users.

## Solutions

### 1. **Security Configuration**
```java
@Configuration
@EnableWebSecurity
public class HealthSecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/actuator/health").permitAll()
                .antMatchers("/actuator/**").hasRole("ACTUATOR")
                .anyRequest().authenticated()
            .and()
            .httpBasic();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 2. **Separate Port for Actuator**
```yaml
management:
  server:
    port: 9090
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

### 3. **Network-Level Security**
- Firewall rules to restrict access
- Internal network only exposure
- VPN access requirements

### 4. **Custom Health Endpoints**
```java
@RestController
public class SecureHealthController {
    
    @GetMapping("/internal/health")
    public ResponseEntity<String> health(@RequestHeader("X-API-Key") String apiKey) {
        if (!isValidApiKey(apiKey)) {
            return ResponseEntity.status(401).build();
        }
        
        return ResponseEntity.ok("UP");
    }
}
```

## Interview Tip
Explain that health checks should be publicly accessible for orchestration, but detailed health information should be secured. Use separate ports and authentication for sensitive endpoints.