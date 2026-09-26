# Rolling Back a Failed Deployment

## Strategies

### 1. **Blue-Green Rollback**
```bash
# Instant rollback to previous version
kubectl patch service app-service -p '{"spec":{"selector":{"app":"app-blue"}}}'
```

### 2. **Canary Rollback**
```bash
# Gradually shift traffic back
kubectl patch virtualservice app -p '{"spec":{"http":[{"route":[{"destination":{"host":"app"},"weight":100},{"destination":{"host":"app-canary"},"weight":0}]}]}}'
```

### 3. **Database Migration Rollback**
```java
// Feature flags for gradual rollback
@FeatureFlag("new_checkout")
public CheckoutResult checkout(Order order) {
    if (featureFlagService.isEnabled("new_checkout")) {
        return newCheckout(order);
    } else {
        return oldCheckout(order);
    }
}
```

## Rollback Checklist
- [ ] Identify affected scope
- [ ] Prepare rollback plan
- [ ] Notify stakeholders
- [ ] Execute rollback
- [ ] Verify functionality
- [ ] Document incident

## Interview Tip
Explain that rollback should be automated and tested. Always have a rollback plan before deploying.