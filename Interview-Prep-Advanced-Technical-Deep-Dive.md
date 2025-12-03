# 🔥 Advanced Technical Deep-Dive - Interview Preparation
## Shailender Kumar - Supplementary Document

---

## Table of Contents
1. [Technical Deep-Dive: NACH Service Questions](#1-technical-deep-dive-nach-service-questions)
2. [Distributed Transactions - Complete Guide](#2-distributed-transactions---complete-guide)
3. [Concurrency Control - Optimistic vs Pessimistic vs Distributed Locks](#3-concurrency-control---optimistic-vs-pessimistic-vs-distributed-locks)
4. [Better Answer: Most Challenging Technical Problem](#4-better-answer-most-challenging-technical-problem)
5. [Critical & Impactful Tasks from Codebase](#5-critical--impactful-tasks-from-codebase)
6. [Design Patterns - Most Asked in Interviews](#6-design-patterns---most-asked-in-interviews)
7. [SOLID Principles with Examples](#7-solid-principles-with-examples)
8. [Additional Important Interview Questions](#8-additional-important-interview-questions)

---

## 1. Technical Deep-Dive: NACH Service Questions

### Q1: "How did you handle idempotency for webhook processing?"

**Detailed Answer:**

> "Idempotency was critical in the NACH service because Digio sends webhooks for mandate status updates, and any duplicate processing could lead to:
> 1. Duplicate state machine updates
> 2. Incorrect partner notifications
> 3. Data inconsistency
>
> **My implementation approach:**

**Step 1: Unique Identifier Extraction**
```java
// Each Digio callback has a unique mandate ID and event type
public void consumeCallback(DigioCallbackRequestDTO request, TenantEnum tenant, NachTypeEnum nachType) {
    // Extract unique identifiers
    String mandateId = request.getPayload().getUpiMandate().getId();
    String eventType = request.getEvent();  // e.g., "upi_mandate.approved"
    String customerRefId = request.getPayload().getUpiMandate().getCustomerRefNumber();
    
    // Create composite idempotency key
    String idempotencyKey = mandateId + "_" + eventType;
}
```

**Step 2: Idempotency Check Before Processing**
```java
public void consumeCallback(DigioCallbackRequestDTO request, ...) {
    String idempotencyKey = buildIdempotencyKey(request);
    
    // Check if already processed
    Optional<NachApplicationAuditEntity> existingAudit = 
        nachApplicationAuditDBService.findByIdempotencyKey(idempotencyKey);
    
    if (existingAudit.isPresent()) {
        log.info("Duplicate callback received for key: {}, skipping", idempotencyKey);
        return; // Already processed - return success without reprocessing
    }
    
    // Process the callback
    processCallback(request, tenant, nachType);
    
    // Mark as processed
    NachApplicationAuditEntity audit = new NachApplicationAuditEntity();
    audit.setIdempotencyKey(idempotencyKey);
    audit.setProcessedAt(LocalDateTime.now());
    audit.setStatus("PROCESSED");
    nachApplicationAuditDBService.save(audit);
}
```

**Step 3: Database-Level Unique Constraint**
```sql
-- Ensure database-level protection against race conditions
ALTER TABLE nach_application_audit 
ADD CONSTRAINT uk_idempotency_key UNIQUE (idempotency_key);
```

**Step 4: Handling Concurrent Duplicate Requests**
```java
@Transactional
public void processCallback(DigioCallbackRequestDTO request, ...) {
    try {
        // All processing in single transaction
        // If duplicate key violation, transaction rolls back
        nachApplicationAuditDBService.save(audit);
        
        // Process state machine update
        zipCreditIntegrationService.addEntryToZCStateMachine(
            customerRefId, 
            ZcApplicationStageEnum.UPI_MANDATE_SUCCESS
        );
        
    } catch (DataIntegrityViolationException e) {
        // Duplicate key - another thread already processed
        log.info("Concurrent duplicate detected, safe to ignore");
        return;
    }
}
```

**Key Points to Mention:**
- Used composite key (mandateId + eventType) for granular idempotency
- Database unique constraint as last line of defense
- Transaction rollback on duplicate detection
- Logged duplicates for monitoring without failing

---

### Q2: "What happens if Kafka is down when a webhook arrives?"

**Detailed Answer:**

> "This is a critical failure scenario that I designed for carefully. The key principle is **never lose a webhook** even if downstream systems are unavailable.

**My approach used multiple fallback layers:**

**Layer 1: Immediate Acknowledgment Strategy**
```java
@PostMapping("/webhook/digio")
public ResponseEntity<String> handleDigioWebhook(
        @RequestBody DigioCallbackRequestDTO request,
        @RequestHeader("X-Digio-Signature") String signature) {
    
    // Step 1: Validate signature immediately (fast)
    if (!validateHmacSignature(request, signature)) {
        return ResponseEntity.status(401).body("Invalid signature");
    }
    
    // Step 2: Persist to database BEFORE Kafka (guaranteed durability)
    WebhookRecord record = persistWebhookToDatabase(request);
    
    // Step 3: Try to push to Kafka
    try {
        kafkaTemplate.send("nach-webhooks", request).get(5, TimeUnit.SECONDS);
        record.setKafkaStatus("SENT");
    } catch (Exception e) {
        log.error("Kafka unavailable, webhook stored in DB for retry", e);
        record.setKafkaStatus("PENDING");
    }
    
    webhookRepository.save(record);
    
    // Step 4: Return success to Digio (webhook is safely stored)
    return ResponseEntity.ok("Received");
}
```

**Layer 2: Database as Durable Buffer**
```java
@Entity
@Table(name = "webhook_buffer")
public class WebhookRecord {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(columnDefinition = "TEXT")
    private String payload;
    
    @Enumerated(EnumType.STRING)
    private WebhookStatus status;  // PENDING, KAFKA_SENT, PROCESSED, FAILED
    
    private LocalDateTime receivedAt;
    private LocalDateTime lastRetryAt;
    private Integer retryCount;
}
```

**Layer 3: Retry Scheduler (Cron Job)**
```java
@Scheduled(fixedRate = 60000)  // Every minute
public void retryPendingWebhooks() {
    List<WebhookRecord> pending = webhookRepository
        .findByKafkaStatusAndRetryCountLessThan("PENDING", MAX_RETRIES);
    
    for (WebhookRecord record : pending) {
        try {
            kafkaTemplate.send("nach-webhooks", record.getPayload())
                .get(5, TimeUnit.SECONDS);
            record.setKafkaStatus("SENT");
        } catch (Exception e) {
            record.setRetryCount(record.getRetryCount() + 1);
            record.setLastRetryAt(LocalDateTime.now());
            
            if (record.getRetryCount() >= MAX_RETRIES) {
                record.setKafkaStatus("FAILED");
                alertService.sendCriticalAlert("Webhook delivery failed: " + record.getId());
            }
        }
        webhookRepository.save(record);
    }
}
```

**Layer 4: Direct Processing Fallback**
```java
// If Kafka is down for extended period, process directly from DB
@Scheduled(fixedRate = 300000)  // Every 5 minutes
public void processDirectlyIfKafkaDown() {
    if (!isKafkaHealthy()) {
        List<WebhookRecord> unprocessed = webhookRepository
            .findByStatus(WebhookStatus.PENDING);
        
        for (WebhookRecord record : unprocessed) {
            try {
                processWebhookDirectly(record);
                record.setStatus(WebhookStatus.PROCESSED);
            } catch (Exception e) {
                // Will be retried next cycle
            }
            webhookRepository.save(record);
        }
    }
}
```

**Architecture Diagram:**
```
┌─────────┐     ┌─────────────┐     ┌─────────┐     ┌──────────┐
│  Digio  │────▶│  Webhook    │────▶│  Kafka  │────▶│ Consumer │
│         │     │  Endpoint   │     │ (if up) │     │          │
└─────────┘     └──────┬──────┘     └─────────┘     └──────────┘
                       │
                       ▼ (always)
                ┌──────────────┐
                │   Database   │
                │   (Buffer)   │
                └──────┬───────┘
                       │
                       ▼ (if Kafka down)
                ┌──────────────┐
                │ Retry Cron   │
                │ (every 1min) │
                └──────────────┘
```

**Key Points:**
- Database as primary durable storage (never lose webhooks)
- Kafka as optimization for async processing
- Multiple retry mechanisms
- Monitoring and alerting for failures

---

### Q3: "How do you handle mandate status conflicts between Digio and your system?"

**Detailed Answer:**

> "Status conflicts can occur when:
> 1. Webhook arrives out of order
> 2. Our status update failed but Digio's succeeded
> 3. Manual intervention changed status
>
> I implemented a **reconciliation strategy** with conflict resolution rules:

**Strategy 1: Event Timestamp-Based Resolution**
```java
public void handleStatusUpdate(DigioCallbackRequestDTO callback) {
    String mandateId = callback.getPayload().getUpiMandate().getId();
    LocalDateTime eventTimestamp = callback.getTimestamp();
    
    NachApplicationAuditEntity existing = 
        nachApplicationAuditDBService.findByMandateId(mandateId);
    
    // Only process if this event is newer than what we have
    if (existing != null && existing.getLastEventTimestamp() != null) {
        if (eventTimestamp.isBefore(existing.getLastEventTimestamp())) {
            log.warn("Out-of-order event received for mandate: {}, ignoring", mandateId);
            return; // Ignore older events
        }
    }
    
    // Process the status update
    processStatusUpdate(callback);
}
```

**Strategy 2: State Machine Validation**
```java
public enum NachStatus {
    CREATED, PENDING, APPROVED, ACTIVE, PAUSED, CANCELLED, EXPIRED;
    
    // Valid transitions only
    private static final Map<NachStatus, Set<NachStatus>> VALID_TRANSITIONS = Map.of(
        CREATED, Set.of(PENDING, CANCELLED),
        PENDING, Set.of(APPROVED, CANCELLED, EXPIRED),
        APPROVED, Set.of(ACTIVE, CANCELLED),
        ACTIVE, Set.of(PAUSED, CANCELLED, EXPIRED),
        PAUSED, Set.of(ACTIVE, CANCELLED)
    );
    
    public boolean canTransitionTo(NachStatus newStatus) {
        return VALID_TRANSITIONS.getOrDefault(this, Set.of()).contains(newStatus);
    }
}

public void updateStatus(String mandateId, NachStatus newStatus) {
    NachApplicationAuditEntity entity = findByMandateId(mandateId);
    NachStatus currentStatus = entity.getStatus();
    
    if (!currentStatus.canTransitionTo(newStatus)) {
        log.error("Invalid transition: {} -> {} for mandate: {}", 
            currentStatus, newStatus, mandateId);
        
        // Log for reconciliation, don't fail
        createReconciliationRecord(mandateId, currentStatus, newStatus);
        return;
    }
    
    entity.setStatus(newStatus);
    nachApplicationAuditDBService.save(entity);
}
```

**Strategy 3: Periodic Reconciliation Job**
```java
@Scheduled(cron = "0 0 2 * * ?")  // Daily at 2 AM
public void reconcileWithDigio() {
    // Fetch all mandates updated in last 24 hours
    List<NachApplicationAuditEntity> recentMandates = 
        nachApplicationAuditDBService.findUpdatedAfter(LocalDateTime.now().minusDays(1));
    
    for (NachApplicationAuditEntity mandate : recentMandates) {
        try {
            // Call Digio API to get current status
            DigioMandateResponse digioStatus = 
                digioIntegrationService.getMandateStatus(mandate.getMandateId());
            
            if (!mandate.getStatus().equals(digioStatus.getStatus())) {
                // Status mismatch found
                log.warn("Status mismatch for mandate: {}. Ours: {}, Digio: {}", 
                    mandate.getMandateId(), mandate.getStatus(), digioStatus.getStatus());
                
                // Create reconciliation record for manual review
                ReconciliationRecord record = new ReconciliationRecord();
                record.setMandateId(mandate.getMandateId());
                record.setOurStatus(mandate.getStatus());
                record.setDigioStatus(digioStatus.getStatus());
                record.setDetectedAt(LocalDateTime.now());
                reconciliationRepository.save(record);
                
                // Auto-resolve if Digio is authoritative and transition is valid
                if (mandate.getStatus().canTransitionTo(digioStatus.getStatus())) {
                    mandate.setStatus(digioStatus.getStatus());
                    mandate.setReconciled(true);
                    nachApplicationAuditDBService.save(mandate);
                }
            }
        } catch (Exception e) {
            log.error("Reconciliation failed for mandate: {}", mandate.getMandateId(), e);
        }
    }
}
```

**Strategy 4: Conflict Resolution Priority**
```java
public NachStatus resolveConflict(NachStatus ourStatus, NachStatus digioStatus) {
    // Priority order: CANCELLED > EXPIRED > ACTIVE > APPROVED > PENDING > CREATED
    // Terminal states take precedence
    
    if (digioStatus == NachStatus.CANCELLED || ourStatus == NachStatus.CANCELLED) {
        return NachStatus.CANCELLED;
    }
    if (digioStatus == NachStatus.EXPIRED || ourStatus == NachStatus.EXPIRED) {
        return NachStatus.EXPIRED;
    }
    
    // For non-terminal states, Digio is source of truth
    return digioStatus;
}
```

**Key Points:**
- Event timestamp validation for ordering
- State machine prevents invalid transitions
- Daily reconciliation job catches discrepancies
- Digio is treated as source of truth for active mandates
- All conflicts are logged for audit trail

---

## 2. Distributed Transactions - Complete Guide

### Overview: Why Distributed Transactions Are Hard

In microservices, a single business operation often spans multiple services and databases. Traditional ACID transactions don't work across service boundaries.

### Pattern 1: SAGA Pattern (Choreography)

**When to Use:**
- Multiple services need to participate in a transaction
- Eventual consistency is acceptable
- Services are loosely coupled

**Real Example from Our Codebase - Loan Disbursement:**

```java
/**
 * Loan Disbursement SAGA
 * 
 * Steps:
 * 1. Verify KYC → 2. Create Loan in LMS → 3. Process Disbursement → 
 * 4. Update State Machine → 5. Send Notification
 * 
 * If any step fails, compensating transactions are triggered.
 */
@Service
public class LoanDisbursementSaga {
    
    public void executeDisbursement(DisbursementRequest request) {
        String sagaId = UUID.randomUUID().toString();
        SagaContext context = new SagaContext(sagaId);
        
        try {
            // Step 1: Verify KYC
            KycResult kycResult = kycService.verify(request.getApplicationId());
            context.addStep("KYC_VERIFIED", kycResult);
            
            // Step 2: Create Loan in LMS
            LoanCreationResult loanResult = lmsService.createLoan(request);
            context.addStep("LOAN_CREATED", loanResult);
            
            // Step 3: Process Bank Transfer
            TransferResult transferResult = payoutService.initiateTransfer(
                loanResult.getLoanId(), 
                request.getDisbursementAmount()
            );
            context.addStep("TRANSFER_INITIATED", transferResult);
            
            // Step 4: Update State Machine
            stateMachineService.updateStatus(
                request.getApplicationId(), 
                ApplicationStage.DISBURSEMENT_COMPLETED
            );
            context.addStep("STATE_UPDATED", null);
            
            // Step 5: Send Notification
            notificationService.sendDisbursementNotification(request);
            
        } catch (Exception e) {
            // Trigger compensating transactions
            compensate(context, e);
            throw new SagaFailedException("Disbursement failed", e);
        }
    }
    
    private void compensate(SagaContext context, Exception originalError) {
        log.error("SAGA compensation started for: {}", context.getSagaId());
        
        // Compensate in reverse order
        List<SagaStep> steps = context.getCompletedSteps();
        Collections.reverse(steps);
        
        for (SagaStep step : steps) {
            try {
                switch (step.getName()) {
                    case "TRANSFER_INITIATED":
                        // Reverse the bank transfer
                        payoutService.reverseTransfer(step.getResult());
                        break;
                    case "LOAN_CREATED":
                        // Mark loan as cancelled
                        lmsService.cancelLoan(step.getResult().getLoanId());
                        break;
                    case "STATE_UPDATED":
                        // Revert state machine
                        stateMachineService.updateStatus(
                            context.getApplicationId(), 
                            ApplicationStage.DISBURSEMENT_FAILED
                        );
                        break;
                    case "KYC_VERIFIED":
                        // No compensation needed for read operation
                        break;
                }
            } catch (Exception compensationError) {
                // Log compensation failure for manual intervention
                log.error("Compensation failed for step: {}", step.getName(), compensationError);
                alertService.sendCriticalAlert(
                    "SAGA compensation failed: " + context.getSagaId()
                );
            }
        }
    }
}
```

---

### Pattern 2: SAGA Pattern (Orchestration with Kafka)

**When to Use:**
- Need central coordination
- Complex workflows with many steps
- Want visibility into saga state

**Implementation:**

```java
@Service
public class OrchestrationSagaService {
    
    @Autowired
    private KafkaTemplate<String, SagaEvent> kafkaTemplate;
    
    @Autowired
    private SagaStateRepository sagaStateRepository;
    
    public void startLoanSaga(LoanRequest request) {
        // Create saga state
        SagaState saga = new SagaState();
        saga.setSagaId(UUID.randomUUID().toString());
        saga.setCurrentStep("INIT");
        saga.setPayload(objectMapper.writeValueAsString(request));
        sagaStateRepository.save(saga);
        
        // Start first step
        kafkaTemplate.send("saga-commands", new SagaEvent(
            saga.getSagaId(),
            "VERIFY_KYC",
            request
        ));
    }
    
    @KafkaListener(topics = "saga-replies")
    public void handleSagaReply(SagaReply reply) {
        SagaState saga = sagaStateRepository.findById(reply.getSagaId());
        
        if (reply.isSuccess()) {
            // Move to next step
            String nextStep = getNextStep(saga.getCurrentStep());
            saga.setCurrentStep(nextStep);
            saga.addCompletedStep(reply.getStep());
            sagaStateRepository.save(saga);
            
            if (!"COMPLETED".equals(nextStep)) {
                kafkaTemplate.send("saga-commands", new SagaEvent(
                    saga.getSagaId(),
                    nextStep,
                    saga.getPayload()
                ));
            }
        } else {
            // Start compensation
            saga.setStatus("COMPENSATING");
            sagaStateRepository.save(saga);
            triggerCompensation(saga);
        }
    }
}
```

---

### Pattern 3: Two-Phase Commit (2PC)

**When to Use:**
- Strong consistency required
- All participants can support 2PC protocol
- Transaction coordinator can be single point of failure
- **Rarely used in microservices** due to performance overhead

**How it Works:**

```
Phase 1 (Prepare):
Coordinator → All Participants: "Can you commit?"
All Participants → Coordinator: "Yes, prepared" or "No, abort"

Phase 2 (Commit/Rollback):
If all prepared:
    Coordinator → All Participants: "Commit"
Else:
    Coordinator → All Participants: "Rollback"
```

**Simple Implementation (Educational):**

```java
public class TwoPhaseCommitCoordinator {
    
    private List<TransactionParticipant> participants;
    
    public boolean executeTransaction(TransactionRequest request) {
        String transactionId = UUID.randomUUID().toString();
        
        // Phase 1: Prepare
        boolean allPrepared = true;
        for (TransactionParticipant participant : participants) {
            try {
                boolean prepared = participant.prepare(transactionId, request);
                if (!prepared) {
                    allPrepared = false;
                    break;
                }
            } catch (Exception e) {
                allPrepared = false;
                break;
            }
        }
        
        // Phase 2: Commit or Rollback
        if (allPrepared) {
            for (TransactionParticipant participant : participants) {
                participant.commit(transactionId);
            }
            return true;
        } else {
            for (TransactionParticipant participant : participants) {
                participant.rollback(transactionId);
            }
            return false;
        }
    }
}
```

**Why We Don't Use 2PC:**
- Blocking protocol - participants hold locks during prepare phase
- Coordinator is single point of failure
- Network partitions can cause inconsistent state
- Poor performance in distributed systems

---

### Pattern 4: Outbox Pattern

**When to Use:**
- Need atomic database write + event publish
- Exactly-once message delivery required
- Can't afford message loss

**Implementation:**

```java
@Service
public class OutboxPatternService {
    
    @Transactional
    public void createMandateWithEvent(MandateRequest request) {
        // Step 1: Create mandate
        Mandate mandate = new Mandate();
        mandate.setCustomerRefId(request.getCustomerRefId());
        mandate.setAmount(request.getAmount());
        mandate.setStatus(MandateStatus.CREATED);
        mandateRepository.save(mandate);
        
        // Step 2: Write event to outbox table (SAME TRANSACTION)
        OutboxEvent event = new OutboxEvent();
        event.setAggregateId(mandate.getId());
        event.setEventType("MANDATE_CREATED");
        event.setPayload(objectMapper.writeValueAsString(mandate));
        event.setCreatedAt(LocalDateTime.now());
        event.setPublished(false);
        outboxRepository.save(event);
        
        // Transaction commits - both or neither
    }
}

// Separate process publishes events
@Scheduled(fixedRate = 1000)
public void publishOutboxEvents() {
    List<OutboxEvent> unpublished = outboxRepository.findByPublishedFalse();
    
    for (OutboxEvent event : unpublished) {
        try {
            kafkaTemplate.send("mandate-events", event.getPayload())
                .get(5, TimeUnit.SECONDS);
            event.setPublished(true);
            event.setPublishedAt(LocalDateTime.now());
            outboxRepository.save(event);
        } catch (Exception e) {
            log.error("Failed to publish event: {}", event.getId(), e);
        }
    }
}
```

---

### When to Use Which Pattern?

| Pattern | Consistency | Performance | Complexity | Use Case |
|---------|-------------|-------------|------------|----------|
| **SAGA (Choreography)** | Eventual | High | Medium | Loosely coupled services |
| **SAGA (Orchestration)** | Eventual | Medium | High | Complex workflows |
| **2PC** | Strong | Low | High | Legacy systems, XA transactions |
| **Outbox Pattern** | Guaranteed delivery | High | Low | DB + Event publishing |

**Our Usage at PayU:**
- **SAGA Choreography**: Loan disbursement flow
- **Outbox Pattern**: NACH service webhook delivery
- **2PC**: Never (too slow for fintech scale)

---

## 3. Concurrency Control - Optimistic vs Pessimistic vs Distributed Locks

### Optimistic Locking

**When to Use:**
- Low contention (conflicts are rare)
- Read-heavy workloads
- Can handle retry on conflict

**Real Example from Our Codebase:**

```java
// From order-processing-system/InventoryItem.java
@Entity
@Table(name = "inventory_items")
public class InventoryItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_code", nullable = false, unique = true)
    private String productCode;

    @Column(name = "stock_on_hand", nullable = false)
    private int stockOnHand;

    @Column(name = "reserved_quantity", nullable = false)
    private int reservedQuantity;

    @Version  // This enables optimistic locking
    private long version;

    public void reserve(int quantity) {
        ensurePositive(quantity);
        if (getAvailableQuantity() < quantity) {
            throw new InsufficientInventoryException(productCode, quantity, getAvailableQuantity());
        }
        reservedQuantity += quantity;
    }

    public int getAvailableQuantity() {
        return stockOnHand - reservedQuantity;
    }
}
```

**How it Works:**
```sql
-- When updating, Hibernate generates:
UPDATE inventory_items 
SET reserved_quantity = ?, version = version + 1 
WHERE id = ? AND version = ?

-- If version doesn't match, 0 rows updated → OptimisticLockException
```

**Handling Conflicts:**
```java
@Service
public class InventoryService {
    
    @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
    @Transactional
    public void reserveInventory(String productCode, int quantity) {
        InventoryItem item = inventoryRepository.findByProductCode(productCode);
        item.reserve(quantity);
        inventoryRepository.save(item);
    }
    
    @Recover
    public void handleReservationFailure(OptimisticLockException e, 
                                          String productCode, int quantity) {
        log.error("Failed to reserve after 3 retries: {}", productCode);
        throw new InventoryException("Could not reserve inventory");
    }
}
```

---

### Pessimistic Locking

**When to Use:**
- High contention (many updates to same row)
- Short transactions (avoid holding locks too long)
- Can't afford retry overhead

**Implementation:**

```java
@Repository
public interface MandateRepository extends JpaRepository<Mandate, Long> {
    
    // Pessimistic write lock - blocks other readers and writers
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT m FROM Mandate m WHERE m.id = :id")
    Optional<Mandate> findByIdWithLock(@Param("id") Long id);
    
    // Pessimistic read lock - allows other readers, blocks writers
    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT m FROM Mandate m WHERE m.mandateId = :mandateId")
    Optional<Mandate> findByMandateIdWithReadLock(@Param("mandateId") String mandateId);
}

@Service
public class MandateService {
    
    @Transactional
    public void updateMandateStatus(Long id, MandateStatus newStatus) {
        // Lock the row immediately
        Mandate mandate = mandateRepository.findByIdWithLock(id)
            .orElseThrow(() -> new NotFoundException("Mandate not found"));
        
        // No other transaction can modify this row until we commit
        mandate.setStatus(newStatus);
        mandate.setUpdatedAt(LocalDateTime.now());
        mandateRepository.save(mandate);
    }
}
```

**SQL Generated:**
```sql
SELECT * FROM mandate WHERE id = ? FOR UPDATE
```

---

### Distributed Locking with Redis (From ZipCredit Codebase)

**When to Use:**
- Multiple application instances
- Need to coordinate across JVMs
- Database locks aren't sufficient

**Real Implementation from Our Codebase:**

```java
// From dgl-utility/RedisUtility.java
@Component
public class RedisUtility implements CacheUtility {
    private static final Logger logger = LoggerFactory.getLogger(RedisUtility.class);

    @Autowired
    RedissonClient redissonClient;

    @Override
    public boolean tryLock(long timeout, String lockKey) {
        RLock rLock = redissonClient.getLock(lockKey);
        try {
            boolean lockStatus = rLock.tryLock(timeout, TimeUnit.SECONDS);
            if (!lockStatus) {
                logger.info("Redis Lock failed to acquire for lock key {}", lockKey);
                return false;
            }
            logger.info("Redis Lock Acquired for lock key {}", lockKey);
            return true;

        } catch (Exception e) {
            logger.error("Error trying to acquire Redis Lock for key {}: {}", lockKey, e.getMessage());
            return false;
        }
    }

    @Override
    public void releaseLock(String lockKey) {
        RLock rLock = redissonClient.getLock(lockKey);
        try {
            if (rLock != null && rLock.isHeldByCurrentThread()) {
                rLock.unlock();
                logger.info("Redis Lock Released for lock key {}", lockKey);
            } else {
                logger.warn("Redis Lock not held by the current thread or already released for key {}", lockKey);
            }
        } catch (Exception e) {
            logger.error("Error while releasing Redis Lock for key {}: {}", lockKey, e.getMessage());
        }
    }
}
```

**Another Implementation Pattern:**

```java
// From lending-project/base-client/RedisLock.java
@RequiredArgsConstructor
public class RedisLock implements Lock {
    private final RedisTemplate redisTemplate;
    
    @Override
    public Boolean lock(String lockName) {
        // SETNX - Set if Not Exists (atomic operation)
        return redisTemplate.opsForValue().setIfAbsent(lockName, "locked");
    }

    @Override
    public Boolean lock(String lockName, long duration, TimeUnit timeUnit) {
        // SETNX with TTL to prevent deadlocks
        return redisTemplate.opsForValue().setIfAbsent(lockName, "locked", duration, timeUnit);
    }

    @Override
    public Boolean releaseLock(String lockName) {
        return redisTemplate.delete(lockName);
    }
}
```

**Usage in ZipCredit for Application Processing:**

```java
// From ZCVersion4ServiceImpl.java
public Response processApplication(ApplicationRequest request) {
    String lockKey = "app_lock:" + request.getApplicationId();
    
    // Determine lock type based on configuration
    CacheType cacheType = applicationValidator.isRedisLockFlagEnable(tenantId) 
        ? CacheType.REDIS 
        : CacheType.LOCAL;
    
    CacheUtility cacheUtility = cacheUtilityFactory.getCacheUtility(cacheType);
    
    try {
        // Try to acquire lock with timeout
        boolean lockAcquired = cacheUtility.tryLock(30, lockKey);
        
        if (!lockAcquired) {
            log.warn("Could not acquire lock for application: {}", request.getApplicationId());
            throw new ConcurrentModificationException("Application is being processed");
        }
        
        // Process application (only one instance can do this at a time)
        return doProcessApplication(request);
        
    } finally {
        // Always release lock
        cacheUtility.releaseLock(lockKey);
    }
}
```

---

### Comparison Table

| Aspect | Optimistic | Pessimistic | Redis Distributed |
|--------|------------|-------------|-------------------|
| **Lock Time** | At commit | At read | Explicit acquire/release |
| **Blocking** | No | Yes | No (fails fast) |
| **Scalability** | High | Medium | High |
| **Use Case** | Low contention | High contention | Multi-instance coordination |
| **Failure Mode** | Exception on conflict | Timeout waiting | TTL expiry |
| **Our Usage** | Inventory updates | Bank account updates | Application processing |

---

## 4. Better Answer: Most Challenging Technical Problem

### Option 1: Multi-Tenant NACH Data Migration

> **Situation:**
> "When we onboarded new lending partners to the NACH service, we faced a critical challenge. These partners had existing mandates created through a legacy system that needed to be migrated to our new service without disrupting active EMI collections."

> **Task:**
> "I was tasked with designing and executing a zero-downtime data migration for 50,000+ active mandates across 5 partners while ensuring:
> 1. No duplicate mandate registrations
> 2. No missed EMI collections during migration
> 3. Complete audit trail for compliance"

> **Action:**
> 
> **1. Dual-Write Strategy:**
> ```java
> @Service
> public class MigrationService {
>     
>     // During migration, write to both old and new system
>     public void createMandate(MandateRequest request) {
>         if (isMigrationPhase()) {
>             // Write to legacy system
>             legacyService.createMandate(request);
>             // Write to new system
>             nachService.createMandate(request);
>             // Verify consistency
>             verifyConsistency(request.getMandateId());
>         } else {
>             nachService.createMandate(request);
>         }
>     }
> }
> ```
>
> **2. Batch Migration with Checkpointing:**
> ```java
> @Scheduled(cron = "0 0 2 * * ?")  // 2 AM daily
> public void migrateBatch() {
>     Long lastProcessedId = checkpointRepository.getLastProcessedId();
>     List<LegacyMandate> batch = legacyRepository.findByIdGreaterThan(lastProcessedId, 1000);
>     
>     for (LegacyMandate legacy : batch) {
>         try {
>             NachApplicationAuditEntity newMandate = mapper.convert(legacy);
>             nachService.save(newMandate);
>             checkpointRepository.updateLastProcessedId(legacy.getId());
>         } catch (DuplicateKeyException e) {
>             // Already migrated, skip
>             log.info("Mandate already exists: {}", legacy.getMandateId());
>         }
>     }
> }
> ```
>
> **3. Reconciliation Job:**
> ```java
> // Daily reconciliation to ensure data consistency
> @Scheduled(cron = "0 0 6 * * ?")
> public void reconcile() {
>     List<String> mismatches = new ArrayList<>();
>     List<LegacyMandate> allLegacy = legacyRepository.findAll();
>     
>     for (LegacyMandate legacy : allLegacy) {
>         NachApplicationAuditEntity newMandate = 
>             nachService.findByMandateId(legacy.getMandateId());
>         
>         if (!isEqual(legacy, newMandate)) {
>             mismatches.add(legacy.getMandateId());
>         }
>     }
>     
>     if (!mismatches.isEmpty()) {
>         alertService.sendReconciliationReport(mismatches);
>     }
> }
> ```

> **Result:**
> - Migrated 50,000+ mandates across 5 partners
> - Zero EMI collection failures during migration
> - 100% data consistency verified through reconciliation
> - Migration completed in 2 weeks vs estimated 6 weeks

---

### Option 2: Race Condition in Webhook Processing

> **Situation:**
> "In production, we discovered that 3% of NACH webhooks were failing with 'mandate not found' errors. After investigation, I found a race condition: Digio's webhooks were arriving before our mandate creation transaction was committed."

> **Task:**
> "Fix the race condition without:
> 1. Losing any webhooks
> 2. Introducing significant latency
> 3. Requiring Digio to change their behavior"

> **Action:**
>
> **1. Root Cause Analysis:**
> ```
> Timeline of Race Condition:
> T0: Our API receives create mandate request
> T1: We call Digio API to create mandate
> T2: Digio creates mandate and sends webhook (immediately)
> T3: Webhook arrives at our system
> T4: We try to find mandate → NOT FOUND (transaction still in progress)
> T5: Our create mandate transaction commits
> ```
>
> **2. Solution - Webhook Buffer with Delayed Processing:**
> ```java
> @PostMapping("/webhook/digio")
> public ResponseEntity<String> handleWebhook(@RequestBody String payload) {
>     // Step 1: Immediately persist to buffer (fast)
>     WebhookBuffer buffer = new WebhookBuffer();
>     buffer.setPayload(payload);
>     buffer.setReceivedAt(Instant.now());
>     buffer.setStatus(BufferStatus.PENDING);
>     webhookBufferRepository.save(buffer);
>     
>     // Step 2: Return success immediately
>     return ResponseEntity.ok("Received");
> }
> 
> // Step 3: Process with delay
> @Scheduled(fixedRate = 2000)  // Every 2 seconds
> public void processBufferedWebhooks() {
>     // Only process webhooks older than 3 seconds
>     Instant cutoff = Instant.now().minusSeconds(3);
>     List<WebhookBuffer> ready = webhookBufferRepository
>         .findByStatusAndReceivedAtBefore(BufferStatus.PENDING, cutoff);
>     
>     for (WebhookBuffer buffer : ready) {
>         processWithRetry(buffer);
>     }
> }
> 
> private void processWithRetry(WebhookBuffer buffer) {
>     for (int attempt = 1; attempt <= 3; attempt++) {
>         try {
>             nachService.consumeCallback(parsePayload(buffer));
>             buffer.setStatus(BufferStatus.PROCESSED);
>             webhookBufferRepository.save(buffer);
>             return;
>         } catch (MandateNotFoundException e) {
>             if (attempt < 3) {
>                 Thread.sleep(attempt * 1000);  // Exponential backoff
>             }
>         }
>     }
>     // After 3 attempts, move to DLQ
>     buffer.setStatus(BufferStatus.FAILED);
>     webhookBufferRepository.save(buffer);
> }
> ```
>
> **3. Added Idempotency:**
> ```java
> public void consumeCallback(DigioCallbackRequestDTO request) {
>     String idempotencyKey = request.getMandateId() + "_" + request.getEvent();
>     
>     // Check if already processed
>     if (processedEventRepository.existsByIdempotencyKey(idempotencyKey)) {
>         log.info("Duplicate webhook, skipping: {}", idempotencyKey);
>         return;
>     }
>     
>     // Process...
> }
> ```

> **Result:**
> - Webhook failure rate dropped from 3% to 0.01%
> - Average processing delay: 3.5 seconds (acceptable for async)
> - Zero data loss
> - No changes required from Digio

---

## 5. Critical & Impactful Tasks from Codebase

### Task 1: Insurance Vendor Factory Pattern (InsureX)

**What I Built:**
```java
// From insure-x/InsuranceVendorFactory.java
@Component
@Slf4j
public class InsuranceVendorFactory {

    @Autowired
    private IciciInsuranceVendorImpl iciciInsuranceVendor;

    @Autowired
    private AckoInsuranceVendorImpl ackoInsuranceVendor;

    public InsuranceVendor getInsuranceVendor(String vendorCode) throws InsureXException {
        if ("ICICI".equalsIgnoreCase(vendorCode)) {
            return iciciInsuranceVendor;
        } else if ("ACKO".equalsIgnoreCase(vendorCode)) {
            return ackoInsuranceVendor;
        } else {
            log.error("Invalid vendor code");
            return null;
        }
    }
}
```

**Impact:**
- Multi-vendor support without code changes
- Each vendor has its own implementation
- Easy to add new vendors (just add new implementation)
- Reduced integration time from 3 weeks to 1 week for new vendors

---

### Task 2: NACH Callback Strategy Pattern

**What I Built:**
```java
// From dls-nach-service/DigioCallbackServiceFactory.java
@Slf4j
@Component
public class DigioCallbackServiceFactory {

    private final EnumMap<NachTypeEnum, DigioCallbackService> strategyMap;

    public DigioCallbackServiceFactory(List<DigioCallbackService> strategies) {
        this.strategyMap = new EnumMap<>(NachTypeEnum.class);
        for (DigioCallbackService strategy : strategies) {
            this.strategyMap.put(strategy.getNachTye(), strategy);
        }
    }

    public DigioCallbackService getStrategy(NachTypeEnum type) throws CallbackNachAPIFlowException {
        return Optional.ofNullable(strategyMap.get(type))
            .orElseThrow(() -> new CallbackNachAPIFlowException(
                NachServiceErrors.CALLBACK_CONSUMTION_NOT_SUPPORTED_FOR_NACH_TYPE,
                ApiEndpointEnum.NACH_CALLBACK, 
                MDCUtility.get(MiscConstants.MDC_REQUEST_ID)
            ));
    }
}
```

**Impact:**
- Clean separation of UPI, API, and Physical NACH processing
- Auto-registration of strategies via Spring DI
- Type-safe with EnumMap
- Easy to add new NACH types

---

### Task 3: State Machine for Loan Applications

**What I Built:**
```java
// Comprehensive state machine with 275+ states
public enum ApplicationStage {
    // Application lifecycle
    CREATED,
    APPLICANT_DETAIL_UPDATED,
    COMPANY_DETAIL_UPDATED,
    
    // Eligibility
    SOFT_ELIGIBILITY_IN_PROGRESS,
    SOFT_ELIGIBILITY_APPROVED,
    SOFT_ELIGIBILITY_DECLINED,
    
    // KYC States
    CKYC_PULLED,
    CKYC_FAILED,
    OKYC_OTP_SENT,
    OKYC_OTP_ACCEPTED,
    VKYC_INITIATED,
    VKYC_MANUAL_APPROVED,
    
    // NACH States
    UPI_MANDATE_GENERATED,
    UPI_MANDATE_SUCCESS,
    UPI_MANDATE_FAILED,
    API_NACH_REGISTERED,
    
    // Document States
    SANCTION_GENERATED,
    SANCTION_SIGNED,
    KFS_SIGNED,
    
    // Final States
    LOAN_DISBURSED,
    LOAN_CLOSED,
    LOAN_REJECTED,
    // ... 275+ total states
}
```

**Impact:**
- Complete visibility into application lifecycle
- Invalid transitions prevented by design
- Partner-specific status mapping
- Audit trail for compliance

---

### Task 4: Webhook Retry System

**What I Built:**
```java
// From dglSchedulers/WebhookSchedulerJob.java
public void retryFailedWebhooks() {
    List<CallbackLogBean> failedCallbacks = callbackLogService
        .findByIsCompleted("false");
    
    for (CallbackLogBean callbackLog : failedCallbacks) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        HttpEntity<String> entity = new HttpEntity<>(callbackLog.getRequest_body(), headers);
        
        try {
            ResponseEntity<String> response = restTemplate.exchange(
                callbackLog.getUrl(), 
                HttpMethod.POST, 
                entity, 
                String.class
            );
            callbackLog.setCallback_status(response.getStatusCode().value());
            callbackLog.setResponse_body(response.getBody());
            callbackLog.setIs_completed("true");
            
        } catch (HttpStatusCodeException e) {
            callbackLog.setCounter(callbackLog.getCounter() + 1);
            
            if (maxRetryReached.test(callbackLog.getCounter(), maxRetryCount)) {
                callbackLog.setIs_completed("true");
                callbackLog.setResponse_body(e.getResponseBodyAsString());
            }
        } finally {
            callbackLogService.updateCallbackBean(callbackLog);
        }
    }
}

BiPredicate<Integer, Integer> maxRetryReached = (counter, maxCount) -> counter >= maxCount;
```

**Impact:**
- Guaranteed webhook delivery to partners
- Configurable retry count per partner
- Audit trail of all attempts
- Reduced partner complaints by 80%

---

## 6. Design Patterns - Most Asked in Interviews

### 1. Factory Pattern

**Purpose:** Create objects without specifying exact class

**Our Implementation:**
```java
// InsuranceVendorFactory.java
public InsuranceVendor getInsuranceVendor(String vendorCode) {
    switch (vendorCode.toUpperCase()) {
        case "ICICI": return iciciInsuranceVendor;
        case "ACKO": return ackoInsuranceVendor;
        default: throw new UnsupportedVendorException(vendorCode);
    }
}
```

**When to Use:**
- Multiple implementations of same interface
- Object creation logic is complex
- Client shouldn't know concrete classes

---

### 2. Strategy Pattern

**Purpose:** Define family of algorithms, make them interchangeable

**Our Implementation:**
```java
// Interface
public interface DigioCallbackService {
    NachTypeEnum getNachType();
    ApiStatusEnum consumeCallback(DigioCallbackRequestDTO request, TenantEnum tenant);
    ApiStatusEnum sendPartnerCallback(DigioCallbackRequestDTO request, TenantEnum tenant);
}

// UPI Implementation
@Service
public class DigioUpiMandateCallbackService implements DigioCallbackService {
    @Override
    public NachTypeEnum getNachType() {
        return NachTypeEnum.UPI;
    }
    
    @Override
    public ApiStatusEnum consumeCallback(...) {
        // UPI-specific logic
    }
}

// API Implementation
@Service
public class DigioApiNachCallbackService implements DigioCallbackService {
    @Override
    public NachTypeEnum getNachType() {
        return NachTypeEnum.API;
    }
    
    @Override
    public ApiStatusEnum consumeCallback(...) {
        // API NACH-specific logic
    }
}

// Context (Factory)
@Component
public class DigioCallbackServiceFactory {
    private final EnumMap<NachTypeEnum, DigioCallbackService> strategyMap;
    
    public DigioCallbackServiceFactory(List<DigioCallbackService> strategies) {
        this.strategyMap = new EnumMap<>(NachTypeEnum.class);
        strategies.forEach(s -> strategyMap.put(s.getNachType(), s));
    }
    
    public DigioCallbackService getStrategy(NachTypeEnum type) {
        return Optional.ofNullable(strategyMap.get(type))
            .orElseThrow(() -> new UnsupportedOperationException("No strategy for: " + type));
    }
}
```

**When to Use:**
- Multiple algorithms for same task
- Algorithm selection at runtime
- Avoid large switch/if-else blocks

---

### 3. Builder Pattern

**Purpose:** Construct complex objects step by step

**Implementation:**
```java
@Builder
@Data
public class MandateRequest {
    private String customerRefId;
    private String bankAccountNumber;
    private String ifscCode;
    private BigDecimal amount;
    private LocalDate startDate;
    private LocalDate endDate;
    private NachTypeEnum nachType;
    private TenantEnum tenant;
}

// Usage
MandateRequest request = MandateRequest.builder()
    .customerRefId("CUST123")
    .bankAccountNumber("1234567890")
    .ifscCode("HDFC0001234")
    .amount(new BigDecimal("10000"))
    .startDate(LocalDate.now())
    .endDate(LocalDate.now().plusYears(5))
    .nachType(NachTypeEnum.UPI)
    .tenant(TenantEnum.GPAY)
    .build();
```

**When to Use:**
- Objects with many optional parameters
- Immutable objects
- Fluent API desired

---

### 4. Singleton Pattern

**Purpose:** Ensure only one instance exists

**Spring Implementation:**
```java
@Service  // Singleton by default in Spring
public class ConfigService {
    
    private Map<String, String> configCache;
    
    @PostConstruct
    public void init() {
        configCache = loadConfigFromDatabase();
    }
    
    public String getConfig(String key) {
        return configCache.get(key);
    }
}
```

**Traditional Implementation:**
```java
public class DatabaseConnection {
    private static volatile DatabaseConnection instance;
    private Connection connection;
    
    private DatabaseConnection() {
        // Private constructor
        this.connection = createConnection();
    }
    
    public static DatabaseConnection getInstance() {
        if (instance == null) {
            synchronized (DatabaseConnection.class) {
                if (instance == null) {
                    instance = new DatabaseConnection();
                }
            }
        }
        return instance;
    }
}
```

**When to Use:**
- Database connection pools
- Configuration managers
- Logger instances
- Cache managers

---

### 5. Observer Pattern

**Purpose:** One-to-many dependency, notify on state change

**Spring Events Implementation:**
```java
// Event
public class LoanDisbursedEvent extends ApplicationEvent {
    private final String loanId;
    private final BigDecimal amount;
    
    public LoanDisbursedEvent(Object source, String loanId, BigDecimal amount) {
        super(source);
        this.loanId = loanId;
        this.amount = amount;
    }
}

// Publisher
@Service
public class LoanService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public void disburseLoan(LoanRequest request) {
        // Disburse loan...
        
        // Publish event
        eventPublisher.publishEvent(
            new LoanDisbursedEvent(this, loan.getId(), loan.getAmount())
        );
    }
}

// Observers
@Component
public class NotificationObserver {
    
    @EventListener
    public void onLoanDisbursed(LoanDisbursedEvent event) {
        sendSmsNotification(event.getLoanId());
        sendEmailNotification(event.getLoanId());
    }
}

@Component
public class AnalyticsObserver {
    
    @EventListener
    public void onLoanDisbursed(LoanDisbursedEvent event) {
        trackDisbursementMetrics(event.getLoanId(), event.getAmount());
    }
}
```

**When to Use:**
- Decoupled event notification
- Multiple components need to react to state changes
- Audit logging
- Analytics tracking

---

### 6. Decorator Pattern

**Purpose:** Add behavior dynamically without modifying original

**Implementation:**
```java
// Base interface
public interface LoanProcessor {
    LoanResponse process(LoanRequest request);
}

// Concrete implementation
@Service
public class BasicLoanProcessor implements LoanProcessor {
    public LoanResponse process(LoanRequest request) {
        // Basic processing
        return new LoanResponse(request.getId(), LoanStatus.PROCESSED);
    }
}

// Decorator - adds logging
@Service
public class LoggingLoanProcessor implements LoanProcessor {
    
    private final LoanProcessor delegate;
    
    public LoggingLoanProcessor(@Qualifier("basicLoanProcessor") LoanProcessor delegate) {
        this.delegate = delegate;
    }
    
    public LoanResponse process(LoanRequest request) {
        log.info("Processing loan: {}", request.getId());
        long start = System.currentTimeMillis();
        
        LoanResponse response = delegate.process(request);
        
        log.info("Loan processed in {}ms: {}", System.currentTimeMillis() - start, request.getId());
        return response;
    }
}

// Decorator - adds validation
@Service
public class ValidatingLoanProcessor implements LoanProcessor {
    
    private final LoanProcessor delegate;
    
    public LoanResponse process(LoanRequest request) {
        validateRequest(request);
        return delegate.process(request);
    }
    
    private void validateRequest(LoanRequest request) {
        if (request.getAmount() == null || request.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new ValidationException("Invalid loan amount");
        }
    }
}
```

**When to Use:**
- Add behavior without modifying existing code
- Combine behaviors flexibly
- Alternative to subclassing

---

### 7. Template Method Pattern

**Purpose:** Define skeleton of algorithm, let subclasses fill in steps

**Implementation:**
```java
// Abstract template
public abstract class AbstractWebhookProcessor {
    
    // Template method - defines the algorithm
    public final void processWebhook(WebhookRequest request) {
        validateSignature(request);
        WebhookPayload payload = parsePayload(request);
        processPayload(payload);
        sendAcknowledgement(request);
        logProcessing(request);
    }
    
    // Common implementation
    private void validateSignature(WebhookRequest request) {
        // HMAC validation
    }
    
    // Abstract - subclasses implement
    protected abstract WebhookPayload parsePayload(WebhookRequest request);
    protected abstract void processPayload(WebhookPayload payload);
    
    // Hook - optional override
    protected void logProcessing(WebhookRequest request) {
        log.info("Processed webhook: {}", request.getId());
    }
    
    private void sendAcknowledgement(WebhookRequest request) {
        // Send OK response
    }
}

// Concrete implementation for Digio
@Service
public class DigioWebhookProcessor extends AbstractWebhookProcessor {
    
    @Override
    protected WebhookPayload parsePayload(WebhookRequest request) {
        return objectMapper.readValue(request.getBody(), DigioPayload.class);
    }
    
    @Override
    protected void processPayload(WebhookPayload payload) {
        // Digio-specific processing
        nachService.updateMandateStatus(payload);
    }
}
```

**When to Use:**
- Common algorithm with varying steps
- Enforce sequence of operations
- Avoid code duplication

---

## 7. SOLID Principles with Examples

### S - Single Responsibility Principle

**Definition:** A class should have only one reason to change.

**Bad Example:**
```java
public class LoanService {
    public void createLoan(LoanRequest request) { ... }
    public void sendEmailNotification(String email) { ... }  // Wrong!
    public void generatePdfDocument(Loan loan) { ... }  // Wrong!
    public void calculateEMI(BigDecimal principal, int tenure) { ... }
}
```

**Good Example:**
```java
// Each class has ONE responsibility
@Service
public class LoanService {
    public Loan createLoan(LoanRequest request) { ... }
}

@Service
public class NotificationService {
    public void sendEmailNotification(String email, String content) { ... }
}

@Service
public class DocumentService {
    public byte[] generatePdfDocument(Loan loan) { ... }
}

@Service
public class EMICalculatorService {
    public BigDecimal calculateEMI(BigDecimal principal, int tenure, BigDecimal rate) { ... }
}
```

---

### O - Open/Closed Principle

**Definition:** Open for extension, closed for modification.

**Bad Example:**
```java
public class PaymentProcessor {
    public void process(Payment payment) {
        if (payment.getType().equals("CREDIT_CARD")) {
            processCreditCard(payment);
        } else if (payment.getType().equals("UPI")) {
            processUPI(payment);
        } else if (payment.getType().equals("NETBANKING")) {
            processNetbanking(payment);
        }
        // Adding new payment type requires modifying this class!
    }
}
```

**Good Example (Using Strategy Pattern):**
```java
public interface PaymentStrategy {
    void process(Payment payment);
    String getPaymentType();
}

@Service
public class CreditCardPaymentStrategy implements PaymentStrategy {
    public void process(Payment payment) { /* Credit card logic */ }
    public String getPaymentType() { return "CREDIT_CARD"; }
}

@Service
public class UPIPaymentStrategy implements PaymentStrategy {
    public void process(Payment payment) { /* UPI logic */ }
    public String getPaymentType() { return "UPI"; }
}

// Adding new payment type = just add new class, no modification needed!
@Service
public class WalletPaymentStrategy implements PaymentStrategy {
    public void process(Payment payment) { /* Wallet logic */ }
    public String getPaymentType() { return "WALLET"; }
}

@Service
public class PaymentProcessor {
    private final Map<String, PaymentStrategy> strategies;
    
    public PaymentProcessor(List<PaymentStrategy> strategyList) {
        strategies = strategyList.stream()
            .collect(Collectors.toMap(PaymentStrategy::getPaymentType, s -> s));
    }
    
    public void process(Payment payment) {
        strategies.get(payment.getType()).process(payment);
    }
}
```

---

### L - Liskov Substitution Principle

**Definition:** Subtypes must be substitutable for their base types.

**Bad Example:**
```java
public class Bird {
    public void fly() {
        System.out.println("Flying...");
    }
}

public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins can't fly!");  // Violates LSP!
    }
}
```

**Good Example:**
```java
public interface Bird {
    void eat();
    void sleep();
}

public interface FlyingBird extends Bird {
    void fly();
}

public class Sparrow implements FlyingBird {
    public void eat() { ... }
    public void sleep() { ... }
    public void fly() { System.out.println("Flying..."); }
}

public class Penguin implements Bird {
    public void eat() { ... }
    public void sleep() { ... }
    // No fly method - Penguin is not a FlyingBird
}
```

---

### I - Interface Segregation Principle

**Definition:** Clients shouldn't be forced to depend on interfaces they don't use.

**Bad Example:**
```java
public interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
    void writeCode();
    void reviewCode();
}

// Robot doesn't eat or sleep!
public class Robot implements Worker {
    public void work() { ... }
    public void eat() { throw new UnsupportedOperationException(); }
    public void sleep() { throw new UnsupportedOperationException(); }
    // ...
}
```

**Good Example:**
```java
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public interface Developer extends Workable {
    void writeCode();
    void reviewCode();
}

// Human developer
public class HumanDeveloper implements Developer, Eatable, Sleepable {
    public void work() { ... }
    public void eat() { ... }
    public void sleep() { ... }
    public void writeCode() { ... }
    public void reviewCode() { ... }
}

// Robot only implements what it can do
public class RobotDeveloper implements Developer {
    public void work() { ... }
    public void writeCode() { ... }
    public void reviewCode() { ... }
}
```

---

### D - Dependency Inversion Principle

**Definition:** Depend on abstractions, not concretions.

**Bad Example:**
```java
public class LoanService {
    private MySQLLoanRepository repository = new MySQLLoanRepository();  // Bad!
    private SmtpEmailService emailService = new SmtpEmailService();  // Bad!
    
    public void createLoan(LoanRequest request) {
        Loan loan = buildLoan(request);
        repository.save(loan);
        emailService.sendEmail(loan.getCustomerEmail(), "Loan Created");
    }
}
```

**Good Example:**
```java
// Depend on abstractions
public interface LoanRepository {
    void save(Loan loan);
    Optional<Loan> findById(String id);
}

public interface EmailService {
    void sendEmail(String to, String subject, String body);
}

@Service
public class LoanService {
    
    private final LoanRepository repository;  // Interface
    private final EmailService emailService;  // Interface
    
    // Constructor injection
    public LoanService(LoanRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }
    
    public void createLoan(LoanRequest request) {
        Loan loan = buildLoan(request);
        repository.save(loan);
        emailService.sendEmail(loan.getCustomerEmail(), "Loan Created", "...");
    }
}

// Implementations
@Repository
public class MySQLLoanRepository implements LoanRepository { ... }

@Repository
public class PostgresLoanRepository implements LoanRepository { ... }

@Service
public class SmtpEmailService implements EmailService { ... }

@Service
public class SendGridEmailService implements EmailService { ... }
```

---

## 8. Additional Important Interview Questions

### Q1: "How do you ensure API backward compatibility?"

**Answer:**
> "I follow several practices:
>
> **1. API Versioning:**
> ```java
> @RestController
> @RequestMapping("/api/v1/loans")
> public class LoanControllerV1 { ... }
> 
> @RestController
> @RequestMapping("/api/v2/loans")
> public class LoanControllerV2 { ... }
> ```
>
> **2. Additive Changes Only:**
> - Add new fields as optional
> - Never remove existing fields
> - Never change field types
>
> **3. Deprecation Strategy:**
> ```java
> @Deprecated
> @GetMapping("/loans/{id}")
> public LoanResponseV1 getLoanV1(@PathVariable String id) {
>     // Old implementation
> }
> 
> @GetMapping("/v2/loans/{id}")
> public LoanResponseV2 getLoanV2(@PathVariable String id) {
>     // New implementation
> }
> ```
>
> **4. Feature Flags:**
> - Roll out changes gradually
> - Easy rollback if issues arise

---

### Q2: "How do you handle secrets management?"

**Answer:**
> "We use multiple layers:
>
> **1. Environment Variables:**
> ```yaml
> spring:
>   datasource:
>     password: ${DB_PASSWORD}
> ```
>
> **2. AWS Secrets Manager:**
> ```java
> @Service
> public class SecretsService {
>     @Autowired
>     private AWSSecretsManager secretsManager;
>     
>     public String getSecret(String secretName) {
>         GetSecretValueRequest request = new GetSecretValueRequest()
>             .withSecretId(secretName);
>         return secretsManager.getSecretValue(request).getSecretString();
>     }
> }
> ```
>
> **3. Never in Code:**
> - No hardcoded secrets
> - .gitignore for .env files
> - Rotate secrets regularly

---

### Q3: "How do you handle database schema changes in production?"

**Answer:**
> "Using Flyway with safe practices:
>
> **1. Backward Compatible Migrations:**
> ```sql
> -- V1: Add column (nullable first)
> ALTER TABLE loans ADD COLUMN new_field VARCHAR(255);
> 
> -- V2: After code handles null, make NOT NULL
> UPDATE loans SET new_field = 'default' WHERE new_field IS NULL;
> ALTER TABLE loans ALTER COLUMN new_field SET NOT NULL;
> ```
>
> **2. Two-Phase Deployment:**
> - Deploy code that handles both old and new schema
> - Run migration
> - Remove old schema handling in next deploy
>
> **3. Index Creation:**
> ```sql
> -- Use CONCURRENTLY to avoid blocking
> CREATE INDEX CONCURRENTLY idx_loans_status ON loans(status);
> ```

---

### Q4: "How do you debug production issues?"

**Answer:**
> "Structured approach:
>
> **1. Logging with Context:**
> ```java
> // Using MDC for request tracing
> MDC.put("requestId", request.getRequestId());
> MDC.put("userId", request.getUserId());
> log.info("Processing loan application");
> ```
>
> **2. Distributed Tracing:**
> - Correlation IDs across services
> - Jaeger/Zipkin for trace visualization
>
> **3. Metrics and Alerts:**
> - Grafana dashboards for latency/error rate
> - PagerDuty alerts for critical issues
>
> **4. Log Aggregation:**
> - ELK stack for centralized logs
> - Query by requestId to trace full flow

---

### Q5: "What's your approach to code reviews?"

**Answer:**
> "I focus on:
>
> **1. Functionality:**
> - Does it solve the problem?
> - Edge cases handled?
>
> **2. Readability:**
> - Clear naming
> - Comments for complex logic
>
> **3. Performance:**
> - N+1 queries
> - Proper indexing
>
> **4. Security:**
> - Input validation
> - SQL injection prevention
>
> **5. Testing:**
> - Unit tests for business logic
> - Integration tests for critical paths
>
> **I also:**
> - Ask questions instead of making demands
> - Suggest improvements, not just problems
> - Appreciate good code patterns I see

---

### Q6: "How do you estimate tasks?"

**Answer:**
> "I use a structured approach:
>
> **1. Break Down:**
> - Split into smallest deliverable units
> - Identify dependencies
>
> **2. Consider:**
> - Development time
> - Testing time (unit + integration)
> - Code review cycles
> - Deployment and verification
>
> **3. Add Buffer:**
> - 20% for unknowns
> - 50% for external dependencies
>
> **4. Communicate:**
> - Share assumptions
> - Flag risks early
> - Update if timeline changes

---

*Document prepared for Shailender Kumar's Senior Software Engineer interviews*
*Supplementary to main preparation document*
*Last updated: December 2024*

