# Performance & Reliability Interview Questions

## Q25. What Deployment Strategies Do You Prefer for Critical Banking Applications and Why?

### Why Banking Demands Special Care

Banking applications have unique constraints:
- **Zero downtime** — 24/7 customer access
- **Regulatory compliance** — every change auditable
- **Data integrity** — no transaction lost or duplicated
- **Rollback capability** — must recover quickly from bad releases
- **Risk-averse culture** — minimize blast radius

### Recommended Strategy: Tiered Approach

```
┌─────────────────────────────────────────────────┐
│         Internal Tools / Non-Critical           │
│         → Recreate / Rolling Update             │
├─────────────────────────────────────────────────┤
│         Customer-Facing (Standard)              │
│         → Rolling Update + Auto-Rollback        │
├─────────────────────────────────────────────────┤
│         Money-Movement (Critical)                │
│         → Blue-Green + Canary + Feature Flags   │
├─────────────────────────────────────────────────┤
│         Core Banking (Most Critical)             │
│         → Blue-Green + Manual Approval +        │
│           Change Advisory Board (CAB)            │
└─────────────────────────────────────────────────┘
```

### Strategy 1: Blue-Green Deployment

```
                    ┌─────────────────┐
    Load Balancer → │  Blue (current) │ ← 100% traffic
                    │  v1.0           │
                    └─────────────────┘
                    
                    ┌─────────────────┐
                    │ Green (new)     │ ← 0% traffic (testing)
                    │ v1.1            │
                    └─────────────────┘

After validation:
                    
                    ┌─────────────────┐
                    │  Blue (idle)    │ ← rollback ready
                    │  v1.0           │
                    └─────────────────┘
                    
                    ┌─────────────────┐
    Load Balancer → │ Green (active)  │ ← 100% traffic
                    │ v1.1            │
                    └─────────────────┘
```

**Implementation:**
```yaml
# Kubernetes service switches selector to point to green
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment-service
    version: green    # ← change from "blue" to "green"
  ports:
    - port: 80
```

**Pros:**
- ✅ Instant rollback (switch selector back)
- ✅ Zero downtime
- ✅ Test green before traffic

**Cons:**
- ❌ 2x resource cost during deployment
- ❌ Database migrations tricky (need backward compat)

### Strategy 2: Canary Deployment

```
Step 1: Deploy v2 to 5% of pods
Step 2: Route 5% traffic to v2, monitor metrics for 30 min
Step 3: If good → 25% → 50% → 100%
Step 4: If bad → automatic rollback
```

**With Flagger/Argo Rollouts:**
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: payment-service
spec:
  provider: kubernetes
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  progressDeadlineSeconds: 600
  metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 30s
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 30s
  analysis:
    interval: 30s
    threshold: 5       # max failed checks
    maxWeight: 50      # max traffic to canary
    stepWeight: 5      # increment per step
    steps:
      - setWeight: 5
      - pause: 60s
      - setWeight: 25
      - pause: 120s
      - setWeight: 50
```

**Pros:**
- ✅ Real production traffic validates
- ✅ Limited blast radius
- ✅ Automatic rollback

**Cons:**
- ❌ Slower rollout
- ❌ Need good metrics

### Strategy 3: Feature Flags (Dark Launch)

```java
@RestController
public class PaymentController {
    
    @PostMapping("/transfer")
    public TransferResponse transfer(@RequestBody TransferRequest req) {
        if (featureFlags.isEnabled("new-transfer-flow", userContext(req))) {
            return newTransferFlow(req);      // New implementation
        } else {
            return oldTransferFlow(req);      // Old, safe
        }
    }
}
```

**Flag Types:**
- **Release flag** — short-lived, removed after rollout
- **Experiment flag** — A/B testing
- **Ops flag** — kill switch for ops to disable
- **Permission flag** — based on user role

**Pros:**
- ✅ Deploy code without exposing feature
- ✅ Enable/disable instantly
- ✅ Test in production
- ✅ Per-user/percentage rollout

**Cons:**
- ❌ Code complexity (both paths exist)
- ❌ Flag debt if not cleaned up

### Strategy 4: Database Migration — Expand-Migrate-Contract

The biggest risk in banking deploys is **schema changes**.

```
Phase 1: EXPAND — Add new schema (backward compat)
   ALTER TABLE accounts ADD COLUMN new_balance DECIMAL(19,4);
   -- Old code ignores new column

Phase 2: MIGRATE — Backfill & dual-write
   UPDATE accounts SET new_balance = balance;
   -- Code writes to BOTH columns

Phase 3: CONTRACT — Remove old schema
   -- Code reads from new_balance only
   ALTER TABLE accounts DROP COLUMN balance;
```

**Tooling:** Flyway, Liquibase

```sql
-- V1__expand.sql
ALTER TABLE accounts ADD COLUMN new_balance DECIMAL(19,4);

-- V2__migrate.sql
UPDATE accounts SET new_balance = balance WHERE new_balance IS NULL;

-- V3__contract.sql (run AFTER all instances use new_balance)
ALTER TABLE accounts DROP COLUMN balance;
```

### Combining Strategies

For **banking critical path**:
```
1. Code deployed with feature flag DISABLED
2. Canary 5% → 25% → 100% (traffic to NEW CODE)
3. Feature flag still OFF — no behavior change
4. Validate metrics stable
5. Enable feature flag for 1% of users
6. Gradually increase flag rollout
7. Remove flag once 100%
```

### Pre-Deployment Checklist

- [ ] **All tests pass** (unit, integration, contract)
- [ ] **Performance benchmarks** within SLO
- [ ] **Security scan** (SAST, DAST, dependency check)
- [ ] **Database migration** tested in staging
- [ ] **Rollback plan** documented
- [ ] **Runbook** updated
- [ ] **On-call rotation** aware
- [ ] **Stakeholders** notified
- [ ] **Maintenance window** scheduled (if needed)
- [ ] **Monitoring dashboards** ready

### Rollback Strategy

```bash
# Kubernetes
kubectl rollout undo deployment/payment-service
kubectl rollout undo deployment/payment-service --to-revision=5

# Helm
helm rollback payment-service 1

# Database — forward-only, so rollback = new migration to undo
# (Avoid destructive changes)
```

### Database Rollback Considerations

**Forward-only migrations** (best practice):
- Add columns → never drop in same release
- Rename via new column + dual-write + drop later
- Never delete data — soft delete with `deleted_at`

```java
// Safe rename pattern
@Entity
@Table(name = "accounts")
public class Account {
    @Column(name = "new_balance")  // new column
    BigDecimal newBalance;
    
    @Column(name = "balance")       // old column (still present)
    @Deprecated
    BigDecimal oldBalance;
}
```

### Specific Banking Concerns

**1. Idempotency During Deployment**
- Deploys may cause requests to be re-routed mid-flight
- Use idempotency keys + state machine

**2. Database Connection Migration**
- Drain connections gracefully
- New pods wait for old to release

**3. Kafka Consumer Re-balancing**
- Use cooperative-sticky assignor
- Allows rebalancing without stop-the-world

**4. Caches**
- Warm cache before serving traffic (readiness probe)
- New nodes don't crash on cold cache

**5. Scheduled Jobs**
- Quartz / Spring Scheduler — only one node runs job
- Use ShedLock for distributed locking

### My Preferred Approach for Banking

```
┌──────────────────────────────────────────────────┐
│  1. Feature Flag (release hidden behind flag)   │
│  2. Canary (5% → 25% → 100% traffic)            │
│  3. Blue-Green (for DB changes only)            │
│  4. Auto-rollback on metric degradation         │
│  5. Manual approval for money-movement changes  │
│  6. Forward-only DB migrations                  │
│  7. Comprehensive runbook + on-call rotation    │
└──────────────────────────────────────────────────┘
```

### Why Not "Just Rolling Update"?

Rolling update is fine for **internal tools** but **risky for banking**:
- ❌ Mixed versions during rollout (incompatibilities)
- ❌ Rollback not instant
- ❌ Hard to test new version under real load
- ❌ No kill switch if metrics degrade

### Key Takeaways

- **Tiered deployment strategy** — risk-based
- **Blue-Green** for instant rollback
- **Canary** for validation with real traffic
- **Feature flags** for safe dark launches
- **Forward-only DB migrations** — never destructive
- **Auto-rollback** on metric degradation
- **Pre-deployment checklist** is non-negotiable
- **On-call team** aware of every change
- **Audit trail** of every deployment
