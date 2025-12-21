# ACID Transactions

ACID properties are the foundation of reliable database transactions. They ensure that database operations are processed reliably and maintain data integrity, even in the face of errors, power failures, or other unforeseen circumstances.

## What are ACID Properties?

ACID is an acronym that stands for:
- **Atomicity**: All operations in a transaction succeed or all fail
- **Consistency**: Database remains in a consistent state before and after transaction
- **Isolation**: Concurrent transactions don't interfere with each other
- **Durability**: Once committed, changes are permanent and survive failures

## Atomicity

### Definition
Atomicity ensures that a transaction is treated as a single, indivisible unit of work. Either all operations within the transaction succeed, or none of them do.

### How It Works
- **All-or-Nothing**: No partial transaction results
- **Rollback**: Failed operations are completely undone
- **Commit**: All operations completed successfully

### Example Scenario
```sql
-- Transfer $100 from account A to account B
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';
COMMIT;
```

**Without Atomicity (Bad):**
- Account A debited $100
- System crashes before crediting Account B
- Money disappears!

**With Atomicity (Good):**
- Both operations succeed, or both are rolled back
- Data integrity maintained

### Implementation
- **Write-Ahead Logging (WAL)**: Log changes before applying to database
- **Undo Logs**: Record how to undo operations
- **Two-Phase Commit**: Coordinated commit across multiple resources

## Consistency

### Definition
Consistency ensures that a transaction brings the database from one valid state to another valid state, maintaining all defined rules and constraints.

### Types of Consistency

#### Database Consistency
- **Referential Integrity**: Foreign key constraints maintained
- **Domain Constraints**: Data type and value constraints enforced
- **Business Rules**: Application-specific rules upheld

#### Transaction Consistency
- **Pre-condition**: Database valid before transaction
- **Post-condition**: Database valid after transaction
- **Invariant Preservation**: Business rules maintained throughout

### Example
```sql
-- Bank account constraints
CREATE TABLE accounts (
    account_id VARCHAR PRIMARY KEY,
    balance DECIMAL(10,2) CHECK (balance >= 0),
    account_type VARCHAR NOT NULL
);

-- Transfer must maintain balance >= 0
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
-- Constraint violation if balance goes negative
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';
COMMIT;
```

### Consistency vs ACID Consistency
ACID consistency is about database constraints, not the CAP theorem's consistency (which is about data visibility across nodes).

## Isolation

### Definition
Isolation ensures that concurrent execution of transactions produces the same result as if they were executed serially (one after another).

### Why Isolation Matters
- **Race Conditions**: Concurrent access to same data
- **Dirty Reads**: Reading uncommitted changes
- **Lost Updates**: One transaction overwrites another's changes
- **Inconsistent Analysis**: Reading inconsistent intermediate states

### Isolation Levels

#### Read Uncommitted (Lowest Isolation)
- **Transactions see changes from other uncommitted transactions**
- **Problems**: Dirty reads, lost updates, inconsistent analysis
- **Performance**: Highest performance
- **Use Case**: Rarely used, mostly for bulk operations

**Example Problems:**
```sql
-- Transaction 1: Transfer money
UPDATE accounts SET balance = 500 WHERE account_id = 'A';

-- Transaction 2: Read balance (sees uncommitted change)
SELECT balance FROM accounts WHERE account_id = 'A'; -- Returns 500

-- Transaction 1 rolls back
ROLLBACK;

-- Transaction 2 has "dirty read" - saw data that never really existed
```

#### Read Committed
- **Transactions only see committed changes from other transactions**
- **Problems**: Non-repeatable reads, phantom reads
- **Performance**: Good balance of performance and consistency
- **Use Case**: Most common isolation level (default in many databases)

**Example Problems:**
```sql
-- Transaction 1: Read balance
SELECT balance FROM accounts WHERE account_id = 'A'; -- Returns 1000

-- Transaction 2: Update and commit
UPDATE accounts SET balance = 900 WHERE account_id = 'A';
COMMIT;

-- Transaction 1: Read balance again
SELECT balance FROM accounts WHERE account_id = 'A'; -- Returns 900 (non-repeatable read)
```

#### Repeatable Read
- **Transactions see consistent snapshot throughout execution**
- **Problems**: Phantom reads
- **Performance**: Lower than Read Committed
- **Use Case**: Financial applications, complex business logic

**Example Problems:**
```sql
-- Transaction 1: Count accounts with balance > 1000
SELECT COUNT(*) FROM accounts WHERE balance > 1000; -- Returns 5

-- Transaction 2: Insert new account and commit
INSERT INTO accounts (account_id, balance) VALUES ('C', 1500);
COMMIT;

-- Transaction 1: Count again
SELECT COUNT(*) FROM accounts WHERE balance > 1000; -- Returns 6 (phantom read)
```

#### Serializable (Highest Isolation)
- **Transactions completely isolated, as if executed serially**
- **Problems**: None (by definition)
- **Performance**: Lowest performance
- **Use Case**: High-stakes operations requiring absolute consistency

### Isolation Level Comparison

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|----------------|------------|-------------------|--------------|-------------|
| Read Uncommitted | ❌ Allowed | ❌ Allowed | ❌ Allowed | Highest |
| Read Committed | ✅ Prevented | ❌ Allowed | ❌ Allowed | High |
| Repeatable Read | ✅ Prevented | ✅ Prevented | ❌ Allowed | Medium |
| Serializable | ✅ Prevented | ✅ Prevented | ✅ Prevented | Lowest |

## Durability

### Definition
Durability guarantees that once a transaction is committed, its changes are permanent and will survive any subsequent failures (system crashes, power outages, etc.).

### How Durability Works

#### Write-Ahead Logging (WAL)
- **Log First**: Changes logged before writing to database
- **Sequential Writes**: Logs written sequentially for performance
- **Recovery**: Logs replayed to restore database after crash

#### Checkpointing
- **Periodic Saves**: Database periodically saves state to disk
- **Log Truncation**: Old logs discarded after successful checkpoints
- **Recovery Optimization**: Reduces recovery time

#### Force-Write Policy
- **Immediate Writes**: Critical data written immediately to disk
- **Group Commits**: Multiple transactions committed together
- **Battery-Backed Cache**: Protects against power failures

### Durability Levels

#### Synchronous Durability
- **Immediate Persistence**: Changes written to disk before commit returns
- **Highest Safety**: Guaranteed durability
- **Performance Cost**: Slower commits

#### Asynchronous Durability
- **Delayed Writes**: Changes written to disk asynchronously
- **Better Performance**: Faster commits
- **Risk**: Recent commits lost on crash

## ACID in Practice

### Banking Transfer Example
```sql
-- Complete ACID transaction
BEGIN TRANSACTION;

-- Atomicity: Both operations or none
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';

-- Consistency: Balance constraints maintained
-- (CHECK constraints, foreign keys, etc.)

COMMIT; -- Durability: Changes permanent

-- Isolation: Concurrent transfers don't interfere
```

### What Happens on Failure?

#### System Crash During Transaction
- **Before Commit**: Changes rolled back using undo logs
- **After Commit**: Changes replayed from WAL during recovery
- **Partial Commits**: Two-phase commit ensures all-or-nothing

#### Network Failure
- **Connection Lost**: Transaction automatically rolled back
- **Client Timeout**: Application handles retry logic
- **Distributed Transactions**: Two-phase commit coordinates

## ACID Implementation in Different Databases

### PostgreSQL
```sql
-- Explicit transaction with isolation level
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';
COMMIT;
```

### MySQL (InnoDB)
```sql
-- Autocommit off for explicit transactions
SET autocommit = 0;
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';
COMMIT;
```

### MongoDB
```javascript
// Single document atomicity (not multi-document)
db.accounts.updateOne(
  { _id: accountAId, balance: { $gte: 100 } },
  { $inc: { balance: -100 } }
);

db.accounts.updateOne(
  { _id: accountBId },
  { $inc: { balance: 100 } }
);

// For multi-document: Use transactions (MongoDB 4.0+)
const session = db.getMongo().startSession();
session.startTransaction();
try {
  db.accounts.updateOne(
    { _id: accountAId },
    { $inc: { balance: -100 } },
    { session }
  );
  db.accounts.updateOne(
    { _id: accountBId },
    { $inc: { balance: 100 } },
    { session }
  );
  session.commitTransaction();
} catch (error) {
  session.abortTransaction();
}
```

### Redis
```bash
# Redis transactions (not fully ACID)
MULTI
SET user:balance:A 900
SET user:balance:B 1100
EXEC
```

## ACID Trade-offs

### Performance vs Consistency
- **Higher Isolation**: Slower performance, better consistency
- **Lower Isolation**: Faster performance, potential inconsistencies
- **Tuning**: Choose appropriate level for use case

### Complexity vs Reliability
- **Full ACID**: Complex implementation, maximum reliability
- **Relaxed ACID**: Simpler systems, acceptable inconsistency
- **BASE**: High availability, eventual consistency

## ACID in Java Applications

### Spring Framework Transaction Management
```java
@Service
public class TransferService {
    
    @Autowired
    private AccountRepository accountRepository;
    
    @Transactional(isolation = Isolation.SERIALIZABLE, 
                   propagation = Propagation.REQUIRED)
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        Account fromAccount = accountRepository.findById(fromAccountId)
            .orElseThrow(() -> new AccountNotFoundException(fromAccountId));
        
        Account toAccount = accountRepository.findById(toAccountId)
            .orElseThrow(() -> new AccountNotFoundException(toAccountId));
        
        if (fromAccount.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }
        
        fromAccount.setBalance(fromAccount.getBalance().subtract(amount));
        toAccount.setBalance(toAccount.getBalance().add(amount));
        
        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);
    }
}
```

### Programmatic Transaction Management
```java
@Service
public class TransferService {
    
    @Autowired
    private PlatformTransactionManager transactionManager;
    
    @Autowired
    private AccountRepository accountRepository;
    
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        DefaultTransactionDefinition def = new DefaultTransactionDefinition();
        def.setIsolationLevel(TransactionDefinition.ISOLATION_SERIALIZABLE);
        def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        
        TransactionStatus status = transactionManager.getTransaction(def);
        
        try {
            Account fromAccount = accountRepository.findById(fromAccountId).get();
            Account toAccount = accountRepository.findById(toAccountId).get();
            
            fromAccount.setBalance(fromAccount.getBalance().subtract(amount));
            toAccount.setBalance(toAccount.getBalance().add(amount));
            
            accountRepository.save(fromAccount);
            accountRepository.save(toAccount);
            
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

### Transaction Propagation
```java
@Service
public class OrderService {
    
    @Autowired
    private PaymentService paymentService;
    @Autowired
    private InventoryService inventoryService;
    
    @Transactional
    public void placeOrder(Order order) {
        // Create order (participates in existing transaction)
        orderRepository.save(order);
        
        // Call payment service (joins existing transaction)
        paymentService.processPayment(order);
        
        // Call inventory service (joins existing transaction)  
        inventoryService.updateInventory(order);
    }
}

@Service
public class PaymentService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processPayment(Order order) {
        // This creates a new transaction
        // If it fails, order is still saved
        paymentRepository.save(new Payment(order));
    }
}
```

## Common ACID Challenges

### Long-Running Transactions
**Problem:** Transactions holding locks for extended periods
**Solutions:**
- Break into smaller transactions
- Use optimistic locking
- Implement compensating actions

### Deadlocks
**Problem:** Two transactions waiting for each other's locks
**Solutions:**
- Consistent lock ordering
- Timeout-based deadlock detection
- Deadlock prevention algorithms

### Distributed Transactions
**Problem:** ACID across multiple databases
**Solutions:**
- Two-phase commit protocol
- Saga pattern for microservices
- Eventual consistency with compensation

## Monitoring ACID Transactions

### Transaction Metrics
- **Transaction Rate**: Transactions per second
- **Transaction Duration**: Average, percentiles
- **Rollback Rate**: Percentage of transactions that fail
- **Lock Wait Time**: Time spent waiting for locks

### Database-Specific Monitoring
```sql
-- PostgreSQL: Active transactions
SELECT * FROM pg_stat_activity WHERE state = 'active';

-- MySQL: InnoDB transaction info
SHOW ENGINE INNODB STATUS;

-- Transaction isolation level
SELECT @@tx_isolation;
```

## ACID Best Practices

### Transaction Design
- **Keep Short**: Minimize time locks are held
- **Avoid User Interaction**: No user input during transactions
- **Use Appropriate Isolation**: Don't over-isolate unnecessarily
- **Handle Deadlocks**: Implement retry logic

### Error Handling
- **Rollback on Errors**: Don't leave partial transactions
- **Custom Exceptions**: Clear error messages
- **Logging**: Log transaction start/end/failures
- **Monitoring**: Alert on high rollback rates

### Performance Optimization
- **Connection Pooling**: Reuse database connections
- **Batch Operations**: Group multiple operations
- **Read Replicas**: Offload reads from writes
- **Async Processing**: Move non-critical work outside transactions

## Conclusion

ACID properties are fundamental to reliable database operations. They ensure data integrity and consistency in the face of failures and concurrent access.

**Key Takeaways:**
- **Atomicity**: All-or-nothing transaction execution
- **Consistency**: Database constraints always maintained
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed changes survive failures

**Practical Considerations:**
- Choose appropriate isolation levels for performance vs consistency needs
- Design transactions to be as short as possible
- Implement proper error handling and rollback logic
- Monitor transaction performance and deadlock occurrences

Understanding ACID properties is essential for designing reliable, consistent database applications that maintain data integrity under all circumstances.
