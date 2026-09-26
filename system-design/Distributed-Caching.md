# Distributed Caching Strategy for Account Balances

## Requirements
- Fast access to frequently queried account balances
- Strong consistency for balance updates
- High availability
- Scalability

## Architecture

### Cache-Aside Pattern
```java
@Service
public class AccountService {
    
    private final Cache<Long, Account> accountCache;
    
    public Account getAccount(Long accountId) {
        // Try cache first
        Account account = accountCache.getIfPresent(accountId);
        
        if (account == null) {
            // Cache miss - load from database
            account = accountRepository.findById(accountId)
                .orElseThrow(() -> new AccountNotFoundException());
            
            // Store in cache
            accountCache.put(accountId, account);
        }
        
        return account;
    }
    
    public void updateBalance(Long accountId, BigDecimal newBalance) {
        // Update database first
        accountRepository.updateBalance(accountId, newBalance);
        
        // Update or invalidate cache
        accountCache.put(accountId, updatedAccount);
        // Or: accountCache.invalidate(accountId);
    }
}
```

### Cache Configuration
```java
// Caffeine cache configuration
Cache<Long, Account> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(5, TimeUnit.MINUTES)
    .expireAfterAccess(10, TimeUnit.MINUTES)
    .recordStats()
    .build();
```

## Consistency Strategies

### 1. **Write-Through Cache**
- Update cache and database together
- Strong consistency
- Slower writes

### 2. **Write-Behind Cache**
- Update cache first
- Async update to database
- Faster writes
- Risk of data loss

### 3. **Cache Invalidation**
- Update database
- Invalidate cache on update
- Next read loads fresh data

## Interview Tip
Explain that caching balances consistency and performance. For financial data, prefer strong consistency with write-through or immediate invalidation.