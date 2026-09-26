# Secure and Scalable Authentication for Banking APIs

## Authentication Strategies

### 1. **OAuth 2.0 + JWT**
```java
// JWT Token Structure
{
  "sub": "user123",
  "authorities": ["READ_ACCOUNTS", "TRANSFER_MONEY"],
  "exp": 1640995200
}

// Resource server configuration
/security/config:
- Validate JWT signature
- Extract authorities from token
- Enable method-level security
```

### 2. **API Gateway Authentication**
- Validate tokens at gateway
- Forward authenticated user info
- Rate limiting per client

## Authorization Patterns

### 1. **Role-Based Access Control (RBAC)**
```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(String userId) {
    // Only admin can delete users
}
```

### 2. **Attribute-Based Access Control (ABAC)**
```java
@PreAuthorize("hasPermission(#user, 'READ') and #user.orgId == authentication.principal.orgId")
public Account getAccount(User user) {
    // Dynamic based on user attributes
}
```

## Security Best Practices

### 1. **Token Management**
- Short-lived access tokens (15-60 minutes)
- Refresh tokens for session renewal
- Token revocation support

### 2. **HTTPS Everywhere**
- TLS 1.2+ for all communications
- HSTS headers
- Certificate pinning for mobile apps

### 3. **Rate Limiting**
```java
// Per-client rate limiting
bucket4j:
  bandwidth: 1000 requests per minute
  burst: 100 requests
```

## Interview Tip
Explain that banking APIs need defense in depth. Mention that OAuth 2.0 with JWT is the standard, but consider additional layers like mTLS for inter-service communication.