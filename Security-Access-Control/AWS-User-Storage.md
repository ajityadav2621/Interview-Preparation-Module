# Storing User Details and Passwords in AWS

## AWS Services for User Management

### 1. **AWS Cognito**
- **Purpose**: Fully managed user authentication
- **Features**: Sign-up, sign-in, social login, MFA
- **Use Case**: Application user management without backend complexity

```java
// Using AWS SDK
/software.amazon.awssdk.services.cognitoidp.CognitoIdpClient

// User authentication flow
AuthenticateUserRequest request = AuthenticateUserRequest.builder()
    .clientId(clientId)
    .username(username)
    .password(password)
    .build();
```

### 2. **AWS RDS/DynamoDB with Encryption**
- **RDS**: Relational database with encryption at rest
- **DynamoDB**: NoSQL with built-in encryption
- **Encryption**: AWS KMS for key management

```java
// Password hashing with BCrypt
@Component
public class PasswordEncoder {
    
    private final BCryptPasswordEncoder encoder = new BCryptPasswordEncoder();
    
    public String encode(String rawPassword) {
        return encoder.encode(rawPassword);
    }
    
    public boolean matches(String rawPassword, String encodedPassword) {
        return encoder.matches(rawPassword, encodedPassword);
    }
}
```

### 3. **AWS Secrets Manager**
- **Purpose**: Securely store secrets
- **Use Case**: Database credentials, API keys
- **Integration**: Automatic rotation

```java
// Retrieve secret from AWS
@Value("${aws.secret.name}")
private String secretName;

public String getSecret() {
    GetSecretValueRequest request = GetSecretValueRequest.builder()
        .secretId(secretName)
        .build();
    
    GetSecretValueResponse response = secretsClient.getSecretValue(request);
    return response.secretString();
}
```

## Best Practices

### 1. **Password Security**
- **Hash passwords**: Never store plain text
- **Use BCrypt**: Strong hashing algorithm
- **Salt passwords**: Automatic with BCrypt
- **Implement lockout**: After failed attempts

### 2. **Data Encryption**
- **At rest**: AWS KMS encryption
- **In transit**: TLS/HTTPS
- **Field-level**: Encrypt sensitive fields

### 3. **Access Control**
- **IAM roles**: Least privilege principle
- **VPC**: Network isolation
- **Security groups**: Firewall rules

## Interview Tip
Explain that AWS Cognito is the recommended service for user management. For custom implementations, use RDS/DynamoDB with proper encryption and password hashing.