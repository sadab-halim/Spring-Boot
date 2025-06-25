# Transactional Annotation

## Introduction
### Critical Section
- Code segment, where shared resources are being accessed and modified.
- When multiple request try to access this critical section, Data inconsistency can happen.
- Its solution is usage of **Transaction**, It helps to achieve ACID property.


- **A**tomicity: ensures all operations within a transaction are completed successfully. If any operation fails, the entire transaction will be rollbacked.
- **C**onsistency: ensures that db state before and after the transactions should be consistent only.
- **I**solation: ensures that even if multiple transactions are running in parallel, they do not interfere with each other. 
- **D**urability: ensures that committed transaction will never be lost despite system failure or crash.

**BEGIN_TRANSACTION:**
- Debit from A
- Credit to B
- If all success:
  - COMMIT
- else
  - ROLLBACK
- END_TRANSACTION

In Spring Boot, we can use `@Transactional` annotation.
And for that:
1. We need to add below dependency in pom.xml (based on db we are using, suppose we are using RELATIONAL DB) <br>
   Spring Boot Data JPA (Java Persistence API): helps us to interact with Relational Databases without 
   ```
   <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
   </dependency>
   ```

2. [Optional] Activate, Transaction Management by using `@EnableTransactionManagement` in main class. (Spring Boot generally auto configures it, so we don't need to specially add it.)
   ```java
   @SpringBootApplication
   @EnableTransactionManagement
   public class SpringbootApplication {
        public static void main(String args[]) {
            SpringApplication.run(SpringBootApplication.class, );
        }
   }
   ```

`@Transactional` annotation can be applied at the method level and at the class level.

### `@Transactional` at Class Level
- Transaction applied to all public methods
- ```java
    @Component
    @Transactional
    public class CarService {
        public void updateCar() {
            //this method will get executed within a transaction
        }
        
        public void updateBulkCars() {
            //this method will get executed within a transaction
        }
  
        private void helperMethod() {
            //this method will not get affected by Transactional annotation
        } 
    }
   ```

### `@Transactional` at Method Level
- Transaction is applied to a particular method only.
- ```java
  @Component
  public class CarService() {
    @Transactional
    public void updateCar() {
        //this method will get executed within a transaction
    }
  
    public void updateBulkCurs() {
        //this method will NOT be executed within a transaction
    }
  }
  ```

### Transaction Management in Spring Boot uses AOP
1. Uses Pointcut expression to search for method, which has `@Transactional` annotation like: `@within(org.springframework.transaction.annotation.Transactional)`
2. Once Pointcut expression matches, it runs an "Around" type Advice.
    - Advice is: **invokeWithinTransaction** method present in the **TransactionalInterceptor**

#### Internal Working of `@Transactional`
- Inside **TransactionalInterceptor** below method is present 👇
  - ```java
    @Nullable
    protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass, final InvocationCallback invocation) throws Throwable {
        /*
          -------- Some code in between --------
        */
        //BEGIN_TRANSACTION 👇
        TransactionInfo txnInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);
        Object retVal;
        try {
          //This is an around advice: Invoke the next interceptor in the chain.
          //This will normally result in a target object being invoked.
          retVal = invocation.proceedWithInvocation(); // --> YOUR TASK
        } catch (Throwable ex) {
          //target invocation exception
          completeTransactionAfterThrowing(txnInfo, ex); //Any Failure, ROLLBACK will occur
          throw ex;
        } finally {
          cleanupTransactionAfterThrowing(txInfo, ex);
        }
        
        if(retVal != null && txAttr != null) {
            TransactionStatus status = txInfo.getTransactionStatus();
            if(status != null) {
                if (retVal instanceof Future<?> future && future.isDone()) {
                    try {
                        future.get();
                    } catch (ExecutionException ex) {
                        if (txAttr.rollbackOn(ex.getCause())) {
                            status.setRollbackOnly();
                        }
                    } catch (InterruptedException ex) {
                        Thread.currentThread().interrupt();
                    }
                }    
                else if (vavrPresent && VavrDelegate.isVavrTry(retVal)) {
                    //set rollback-only in case of Vavr failure matching our rollback rules...
                    retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
                }
            }
        }
        //All Success, COMMIT the transaction 👇
        commitTransactionAfterReturning(txnInfo);
    }
    ```

### More Transactional Concepts
- Transaction Manager
  - Programmatic
  - Declarative
- Propagation
  - REQUIRED
  - REQUIRED_NEW
  - SUPPORTS
  - NOT_SUPPORTED
  - MANDATORY
  - NEVER
  - NESTED
- Isolation Level
  - READ_UNCOMMITTED
  - READ_COMMITTED
  - REPEATABLE_READ
  - SERIALIZABLE
- Configure Transaction Timeout
- What is Read Only Transaction

## Hierarchy of Transaction Manager
_pic need to be added here_

## Transaction Management
### Declarative
Transaction Management through Annotation

```java
@Component
public class User {
    @Transactional
    public void updateUser() {
        System.out.println("UPDATE QUERY TO update the user db values.");
    }
}
```

Here, based on underlying DataSource used like JDBC or JPA etc. Spring Boot will choose appropriate Transaction Manager.

```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("org.h2.Driver");
        dataSource.setUrl("jdbc:h2:mem:testdb");
        dataSource.setUsername("sa");
        dataSource.setPassword("");
        return dataSource;
    }
    
    @Bean
    public PlatformTransactionManager userTransactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

```java
@Component
public class UserDeclarative {
    @Transactional(transactionManager = "userTransactionManager") 
    public void updateUserProgrammatic() {
        //SOME DB OPERATIONS
        System.out.println("Insert Query Ran");
        System.out.println("Update Query Ran");
    }
}
```

### Programmatic
- Transaction Management through code
- Flexible but difficult to maintain

```java
@Component
public class User {
    @Transactional //👈 will cause an issue, because External API Call is being made
    public void updateUser() {
        //1. Update DB
        //2. External API Call
        //3. Update DB
    }
}
```

- There are two ways to implement Transaction by Programmatic:
#### Approach 1
```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("org.h2.Driver");
        dataSource.setUrl("jdbc:h2:mem:testdb");
        dataSource.setUsername("sa");
        dataSource.setPassword("");
        return dataSource;
    }

    @Bean
    public PlatformTransactionManager userTransactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

```java
@Component
public class UserProgrammaticApproach1 {
    //Step 1👇
    PlatformTransactionManager userTransactionManager;
    
    UserProgrammaticApproach1(PlatformTransactionManager userTransactionManager) {
        this.userTransactionManager = userTransactionManager;    
    }
    
    public void updateUserProgrammatic() {
        TransactionStatus status = userTransactionManager.getTransaction(null);
        try {
            //SOME INITIAL SET OF DB OPERATIONS
            System.out.println("Insert Query Run1");
            System.out.println("Update Query Run1");
            userTransactionManager.commit(status);
        } catch (Exception e) {
            userTransactionManager.rollback(status);
        }
    }
}
```

#### Approach 2 (Better Way)
Usage of TransactionTemplate

```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("org.h2.Driver");
        dataSource.setUrl("jdbc:h2:mem:testdb");
        dataSource.setUsername("sa");
        dataSource.setPassword("");
        return dataSource;
    }

    @Bean
    public PlatformTransactionManager userTransactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
    
    @Bean
    public TransactionTemplate transactionTemplate(PlatformTransactionManager userTransactionManager) {
        return new TransactionTemplate(userTransactionManager);
    }
}
```

```java
@Component
public class UserProgrammaticApproach2 {
    TransactionTemplate transactionTemplate;
    
    public UserProgrammaticApproach2(TransactionTemplate transactionTemplate) {
        this.transactionTemplate = transactionTemplate;
    }
    
    public void updateUserProgrammatic() {
        TransactionCallback<TransactionStatus> dbOperationsTask = (TransactionStatus status) -> {
            System.out.println("Insert Query Ran");
            System.out.println("Update Query Ran");
            return status;
        };
        TransactionStatus status = transactionTemplate.execute(dbOperationsTask);
    }
}
```

## Propagation
When we try to create a new Transaction, it first checks the PROPAGATION value set, and this tell whether we have to create new transaction or not.

### REQUIRED (default propagation)
```java
@Transactional(propagation = Propagation.REQUIRED)
/*
   if(parent txn present)
      use it
   else 
      create new transaction
*/
```

### REQUIRES_NEW
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
/*
   if(parent txn present)
      suspend the parent txn
      create a new txn and once finished
      resume the parent txn
   else
      create new transaction and execute the method     
*/
```

### SUPPORTS
```java
@Transactional(propagation = Propagation.SUPPORTS)
/*
   if(parent txn present)
      use it
   else
      execute the method without any transaction      
*/
```

### NOT_SUPPORTED
```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
/*
   if(parent txn present)
      suspend the parent txn
      execute the method without any transaction
      resume the parent txn
   else
      execute the method without any transaction      
*/     
```

### MANDATORY
```java
@Transactional(propagation = Propagation.MANDATORY)
/*
   if(parent txn present)
      use it
   else
      throw exception     
*/
```

### NEVER
```java
@Transactional(propagation = Propagation.MANDATORY)
/*
   if(parent txn present)
      throw exception
   else
      execute the method without any transaction      
*/
```

## Isolation Level
- It tells, how the changes made by one transaction are visible to other transactions running in parallel.
- Default isolation level, depends upon the DB which we are using.
- Like most Relational Databases uses READ_COMMITTED as default isolation, but it depends upon DB to DB.
```java
@Transactional(propagation = Propagation.REQUIRED, isolation = Isolation.READ_COMMITTED)
public void updateUser() {
    //some operations here
}
```

| Isolation Level | Dirty Read Possible | Non-Repeatable Read Possible | Phantom Read Possible |
|-----------------|---------------------|------------------------------|-----------------------|
| READ_UNCOMMITTED | Yes | Yes | Yes |
| READ_COMMITTED | No | Yes | Yes |
| REPEATABLE_COMMITTED | No | No | Yes |
| SERIALIZABLE | No | No | No |

Concurrency: High - Low (Order Wise)

### Dirty Read Problem
Transaction A reads the un-committed data of other transaction and if other transaction get ROLLED BACK, the un-commited data which is read by Transaction A is known as Dirty Read.

| Time | Transaction A | Transaction B                                | DB Status |
|------|---------------|----------------------------------------------|-----------|
| T1 | BEGIN_TRANSACTION | BEGIN_TRANSACTION                            | ID: 123 <br> Status: free |
| T2 | | Update Row <br> ID: 123 <br> Status = booked | ID: 123 <br> Status: booked (Not Committed by Transaction B yet) |
| T3 | Read Row <br> ID: 123 (Got Status = Booked) |                                              | ID: 123 <br> Status: Booked (Not Committed by Transaction B yet) |
| T4 | | Rollback                                     | TD: 123 <br> Status: Free (Uncommitted change of Transaction B got Rolled Back) |

### Non-Repeatable Read Problem
If suppose Transaction A, reads the same row several times and there is a chance that it get different value, then its known as Non-Repeatable Read Problem.

| Time | Transaction A                        | DB                        |
|------|--------------------------------------|---------------------------|
| T1   | BEGIN_TRANSACTION                    | ID: 1 <br> Status: Free   |
| T2   | Read Row ID: 1 (reads status: Free)  | ID: 1 <br> Status: Free   |
| T3   |                                      | ID: 1 <br> Status: Booked |
| T4   | Read Row ID: 1 (read status: Booked) | ID: 1 <br> Status: Free   |
| T5   | COMMIT | |

### Phantom Read Problem
If suppose Transaction A, executed same query several times but there is a change that rows returned are different. Then, its known as **Phantom Read Problem**.

| Time | Transaction A                                                        | DB                                                                      |
|------|----------------------------------------------------------------------|-------------------------------------------------------------------------|
| T1   | BEGIN_TRANSACTION                                                    | ID: 1, Status: Free <br> ID: 3, Status: Booked                          |
| T2   | Read Row where ID > 0 and ID < 5 (reads 2 rows ID: 1 and ID: 3)      | ID: 1, Status: Free <br> ID: 3, Status: Booked                          |                       |
| T3   |                                                                      | ID: 1, Status: Free <br> ID: 2, Status: Free <br> ID: 3, Status: Booked |
| T4   | Read Row where ID > 0 and ID < 5 (reads 3 rows, ID:1, ID:2 and ID:3) | ID: 1, Status: Free <br> ID: 2, Status: Free <br> ID: 3, Status: Booked |                                                |
| T5   | COMMIT                                                               |                                                                         |

### DB Locking Types
Locking makes sure that, no other transaction updates the locked rows.

| Lock Type | Another Shared Lock | Another Exclusive Lock |
|-----------|---------------------|------------------------|
| Have Shared Lock | Yes | No |
| Have Exclusive Lock | No | No |

- Shared Lock (S) also known as **READ LOCK**
- Exclusive Lock(X) also known as **WRITE LOCK**

## Isolation Level and Locking Strategy

| Isolation Level | Locking Strategy |
|-----------------|------------------|
| Read Uncommitted | **Read:** No Lock acquired <br> **Write:** No Lock acquired |
| Read Committed | **Read:** Shared lock acquired and released as soon as read is done. <br> **Write:** Exclusive lock acquired and kept till the end of transaction. |
| Repeatable Read | **Read:** Shared lock acquired and released only at the end of the transaction. <br> **Write:** Exclusive lock acquired and released only at the end of the transaction. |
| Serializable | Same as repeatable read locking strategy + apply range lock and lock is release only at the end of the transaction. |

----

# NEW

----

## Transaction Management
#### What is a Transaction in the Context of Databases?
A **Database Transaction** represents a single logical unit of work that comprises one or more database operations (e.g., inserts, updates, deletes). The fundamental principle behind a transaction is that all operations within it must either succeed completely or fail completely. There's no partial success. This "all or nothing" principle is crucial for maintaining data consistency.

Database transactions are governed by the **ACID properties**:
- **Atomicity**: Guarantees that all operations within a transaction are treated as a single, indivisible unit. If any part of the transaction fails, the entire transaction is rolled back, leaving the database in its original state.
- **Consistency**: Ensures that a transaction brings the database from one valid state to another. It preserves predefined rules and constraints (e.g., uniqueness, foreign key relationships).
- **Isolation**: Guarantees that concurrent transactions do not interfere with each other. The intermediate state of a transaction is not visible to other transactions until it is committed. This prevents issues like dirty reads, non-repeatable reads, and phantom reads.
- **Durability**: Ensures that once a transaction has been committed, its changes are permanent and will survive system failures (e.g., power outages, crashes).

#### How Spring Boot Abstracts and Manages Transactions
Spring Boot, leveraging the Spring Framework's powerful abstraction, simplifies transaction management through the `PlatformTransactionManager` interface. This interface acts as a central strategy interface for managing different transaction technologies (e.g., JDBC, JPA, Hibernate, JTA). Spring provides various concrete implementations of `PlatformTransactionManager` out-of-the-box, such as `DataSourceTransactionManager` for JDBC, `JpaTransactionManager` for JPA, and `HibernateTransactionManager` for Hibernate.

Spring Boot's auto-configuration capabilities typically detect your chosen data access technology (e.g., JPA with Hibernate) and automatically configure the appropriate `PlatformTransactionManager` bean, making setup seamless.

## Declarative Transaction Management with `@Transactional`
The `@Transactional` annotation can be applied at either the class level or the method level:
- **Class-level**: Applying `@Transactional` to a class means that all public methods within that class will execute within a transaction.
- **Method-level**: Applying `@Transactional` to a specific method means only that method will execute within a transaction. This overrides any class-level `@Transactional` settings for that particular method.

#### How it simplifies transaction boundaries
When a method annotated with `@Transactional` is invoked, Spring's AOP (Aspect-Oriented Programming) proxy intercepts the call. Before the method execution, the proxy initiates a transaction. If the method completes successfully, the transaction is committed. If an unhandled runtime exception occurs (or a checked exception if configured), the transaction is rolled back

### Important Configuration Attributes of `@Transactional`
The @Transactional annotation offers several attributes that allow fine-grained control over transaction behavior:

#### 1. `propagation`:
Defines how transactional methods behave when they are called within an existing transaction.
- `REQUIRED` **(Default)**: If a transaction already exists, the method joins it. Otherwise, a new transaction is created. This is the most common and generally suitable propagation level.
- `REQUIRES_NEW`: Always creates a new, independent transaction. If an existing transaction is present, it is suspended until the new transaction completes. Useful for operations that must execute independently regardless of the calling transaction's success or failure (e.g., logging an audit trail).
- `NESTED`: If a transaction already exists, the method executes within a nested transaction (a savepoint). If the nested transaction fails, only its changes are rolled back to the savepoint, not the entire outer transaction. If no transaction exists, it behaves like `REQUIRED`. Requires a JDBC 3.0 driver or higher.
- `MANDATORY`: Requires an existing transaction. If no transaction is found, an `IllegalTransactionStateException` is thrown. Useful when a method absolutely must run within a transaction
- `SUPPORTS`: Supports an existing transaction, but does not create a new one if none exists. The method will run non-transactionally if no transaction is in progress.
- `NOT_SUPPORTED`: Executes the method non-transactionally. If a transaction is in progress, it is suspended until the method completes. Useful for operations that should explicitly not be part of a transaction (e.g., sending an email).
- `NEVER`: Executes the method non-transactionally. If a transaction is in progress, an `IllegalTransactionStateException` is thrown. Useful for operations that should never run within a transaction.

#### 2. `isolation`:
Specifies the isolation level of the transaction, controlling how concurrent transactions interact with each other. Lower isolation levels offer higher concurrency but can introduce concurrency issues. Higher isolation levels reduce concurrency issues but can impact performance.
- `READ_UNCOMMITTED`: The lowest isolation level. Allows dirty reads (reading uncommitted changes made by other transactions). Not recommended for most applications due to severe data consistency issues.
- `READ_COMMITTED` **(Default for most databases like PostgreSQL, SQL Server)**: Prevents dirty reads. A transaction can only read data that has been committed by other transactions. However, it can suffer from non-repeatable reads (reading the same data twice and getting different values if another transaction commits changes between reads) and phantom reads (new rows appearing in a result set during repeated queries).
- `REPEATABLE_READ` **(Default for MySQL InnoDB)**: Prevents dirty reads and non-repeatable reads. Once a transaction reads a row, it guarantees that subsequent reads of the same row within the same transaction will yield the same value. However, it can still suffer from phantom reads (new rows being inserted by other transactions).
- `SERIALIZABLE`: The highest isolation level. Prevents dirty reads, non-repeatable reads, and phantom reads. Transactions execute in a completely isolated manner, as if they were executed serially. This offers the strongest consistency but significantly reduces concurrency and can lead to performance bottlenecks.

#### 3. `rollbackFor`:
Specifies an array of `Throwable` classes. If an exception of any of these types (or their subclasses) is thrown during the transaction, the transaction will be rolled back. By default, `@Transactional` rolls back only on `RuntimeException` and `Error`.

#### 4. `noRollbackFor`:
Specifies an array of `Throwable` classes. If an exception of any of these types (or their subclasses) is thrown, the transaction will not be rolled back. This is useful for specific checked exceptions that you don't want to trigger a rollback.

#### 5. `readOnly`:
A  boolean flag (default `false`). If set to `true`, the transaction is optimized for read-only operations. This can lead to performance improvements as the underlying database might apply specific optimizations (e.g., not acquiring write locks). It's crucial not to perform write operations within a read-only transaction, as this can lead to unexpected behavior or exceptions.

### Real-World Code Examples
#### Scenario 1: Rolling back on a Custom Checked Exception
By default, `@Transactional` rolls back only on `RuntimeException` and `Error`. If you have a custom checked exception that should trigger a rollback, you need to configure `rollbackFor`.
```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

// Custom checked exception
class InsufficientFundsException extends Exception {
    public InsufficientFundsException(String message) {
        super(message);
    }
}

@Service
public class AccountService {

    // Assume AccountRepository handles database interactions
    private final AccountRepository accountRepository;

    public AccountService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Transactional(rollbackFor = InsufficientFundsException.class)
    public void transferFunds(Long fromAccountId, Long toAccountId, double amount) throws InsufficientFundsException {
        // Retrieve accounts
        Account fromAccount = accountRepository.findById(fromAccountId)
                                 .orElseThrow(() -> new RuntimeException("Sender account not found"));
        Account toAccount = accountRepository.findById(toAccountId)
                               .orElseThrow(() -> new RuntimeException("Receiver account not found"));

        // Check for sufficient funds
        if (fromAccount.getBalance() < amount) {
            throw new InsufficientFundsException("Insufficient funds in account: " + fromAccountId);
        }

        // Deduct from sender and add to receiver
        fromAccount.setBalance(fromAccount.getBalance() - amount);
        toAccount.setBalance(toAccount.getBalance() + amount);

        // Save updated accounts (implicitly within the transaction)
        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);

        System.out.println("Funds transferred successfully.");
    }
}

// Dummy Account and AccountRepository for demonstration
class Account {
    private Long id;
    private double balance;

    public Account(Long id, double balance) {
        this.id = id;
        this.balance = balance;
    }

    public Long getId() { return id; }
    public double getBalance() { return balance; }
    public void setBalance(double balance) { this.balance = balance; }
}

interface AccountRepository {
    java.util.Optional<Account> findById(Long id);
    Account save(Account account);
}
```

In this example, if `InsufficientFundsException` is thrown, the entire `transferFunds` transaction will be rolled back, ensuring that neither account is partially updated.

#### Scenario 2: Handling Nested Transactions with `REQUIRES_NEW` for Auditing
Imagine an audit log that should always be persisted, even if the main transaction fails.
```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final AuditService auditService; // Inject AuditService

    public OrderService(OrderRepository orderRepository, AuditService auditService) {
        this.orderRepository = orderRepository;
        this.auditService = auditService;
    }

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        System.out.println("Order placed: " + order.getId());

        // Log the order placement, even if subsequent operations fail
        auditService.logOrderPlacement(order.getId(), "Order placed successfully.");

        // Simulate a failure in another part of the order processing
        // if (order.getTotalAmount() > 1000) {
        //     throw new RuntimeException("Order too large, simulating failure.");
        // }
    }
}

@Service
class AuditService {

    // Assume AuditRepository for saving audit logs
    private final AuditRepository auditRepository;

    public AuditService(AuditRepository auditRepository) {
        this.auditRepository = auditRepository;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrderPlacement(Long orderId, String message) {
        AuditLog auditLog = new AuditLog(orderId, message, java.time.LocalDateTime.now());
        auditRepository.save(auditLog);
        System.out.println("Audit log saved successfully for order: " + orderId);
    }
}

// Dummy Order, OrderRepository, AuditLog, AuditRepository
class Order {
    private Long id;
    private double totalAmount;

    public Order(Long id, double totalAmount) {
        this.id = id;
        this.totalAmount = totalAmount;
    }

    public Long getId() { return id; }
    public double getTotalAmount() { return totalAmount; }
}

interface OrderRepository {
    Order save(Order order);
}

class AuditLog {
    private Long orderId;
    private String message;
    private java.time.LocalDateTime timestamp;

    public AuditLog(Long orderId, String message, java.time.LocalDateTime timestamp) {
        this.orderId = orderId;
        this.message = message;
        this.timestamp = timestamp;
    }
}

interface AuditRepository {
    AuditLog save(AuditLog auditLog);
}
```

In this setup, if `placeOrder` encounters a `RuntimeException` after `auditService.logOrderPlacement` is called, the `placeOrder` transaction will be rolled back, but the `logOrderPlacement` transaction (which uses `REQUIRES_NEW`) will have already committed its changes, ensuring the audit trail persists.

#### Scenario 3: Optimizing Read-Only Queries
For methods that only retrieve data and do not modify it, marking them as `readOnly = true` can provide performance benefits.
```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
public class ProductService {

    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Transactional(readOnly = true)
    public List<Product> getAllProducts() {
        return productRepository.findAll();
    }

    @Transactional(readOnly = true)
    public Product getProductById(Long productId) {
        return productRepository.findById(productId)
                                .orElseThrow(() -> new RuntimeException("Product not found"));
    }

    @Transactional
    public Product saveProduct(Product product) {
        return productRepository.save(product);
    }
}

// Dummy Product and ProductRepository
class Product {
    private Long id;
    private String name;
    private double price;

    public Product(Long id, String name, double price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public double getPrice() { return price; }
}

interface ProductRepository {
    List<Product> findAll();
    java.util.Optional<Product> findById(Long id);
    Product save(Product product);
}
```

By marking `getAllProducts()` and `getProductById()` as read-only, Spring can suggest to the underlying database driver and ORM (like JPA) to use read-optimized connections or avoid acquiring write locks, improving concurrency and potentially performance.

### Limitations of Spring's Proxy-Based AOP Model
Spring's `@Transactional` annotation works by creating dynamic proxies around the annotated beans. When you call a method on a Spring-managed bean, you're actually calling the proxy, which then intercepts the call and applies transactional logic. This proxy-based approach has a few important limitations:

#### 1. Self-Invocation (Internal Method Calls):
If a transactional method within the same object calls another transactional method within the same object, the transaction will not be applied to the internally called method. This is because the internal call bypasses the proxy. The call is made directly on `this`, not on the proxied instance
```java
@Service
public class MyService {

    @Transactional // Transactional method A
    public void methodA() {
        // ... some logic ...
        methodB(); // Self-invocation: methodB's @Transactional will be ignored
        // ... some more logic ...
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW) // Transactional method B
    public void methodB() {
        // This method will NOT run in a new transaction if called from methodA directly.
        // It will inherit methodA's transaction or run non-transactionally if methodA is not transactional.
    }
}
```

**Workaround**: To make internal calls transactional, you need to inject the `MyService` itself into `MyService` (though this can lead to circular dependency issues if not careful, and is generally not recommended) or obtain a proxy to the current object using `AopContext.currentProxy()`. The cleanest solution is often to refactor the logic into separate service classes.

#### 2. Private Methods:
`@Transactional` annotations on private methods are ignored by Spring's AOP proxy. Proxies can only intercept public method calls. Therefore, never apply @Transactional to private methods

#### 3. Final Methods/Classes:
While not strictly a limitation for `@Transactional`, if you are using CGLIB proxies (which Spring defaults to when no interface is implemented), `final` methods cannot be overridden. If a class is `final`, it cannot be proxied by CGLIB. This typically isn't an issue with `@Transactional` on typical service classes, as Spring usually relies on JDK dynamic proxies if an interface is present, or CGLIB otherwise. However, it's a good practice to avoid `final` on service methods that might be subject to AOP.

---

## Programmatic Transaction Management
While declarative transaction management is preferred for its simplicity, there are scenarios where programmatic transaction management offers finer-grained control. This approach involves explicitly managing transaction boundaries using Spring's `PlatformTransactionManager` or `TransactionTemplate`.

#### When to use programmatic transaction management:
- When you need to manage transactions across multiple methods that are not part of a single logical unit of work defined by an annotation.
- When transactional behavior needs to be conditional based on complex runtime logic.
- When integrating with legacy code or third-party libraries that require explicit transaction control.

#### Using `PlatformTransactionManager` Directly
The `PlatformTransactionManager` interface provides methods like `getTransaction()`, `commit()`, and `rollback()`.
```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.DefaultTransactionDefinition;

@Service
public class ManualTransactionService {

    private final PlatformTransactionManager transactionManager;
    private final ProductRepository productRepository;

    public ManualTransactionService(PlatformTransactionManager transactionManager, ProductRepository productRepository) {
        this.transactionManager = transactionManager;
        this.productRepository = productRepository;
    }

    public void processProductsConditionally(List<Product> products, boolean shouldCommit) {
        DefaultTransactionDefinition def = new DefaultTransactionDefinition();
        def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        TransactionStatus status = transactionManager.getTransaction(def);

        try {
            for (Product product : products) {
                // Perform some business logic
                product.setPrice(product.getPrice() * 1.1); // Increase price by 10%
                productRepository.save(product);
                System.out.println("Processed product: " + product.getName());
            }

            if (shouldCommit) {
                transactionManager.commit(status);
                System.out.println("Transaction committed.");
            } else {
                transactionManager.rollback(status);
                System.out.println("Transaction rolled back (conditional).");
            }

        } catch (Exception e) {
            transactionManager.rollback(status);
            System.err.println("Transaction rolled back due to error: " + e.getMessage());
            throw e; // Re-throw to propagate the exception
        }
    }
}
```

This approach is more verbose and requires manual exception handling for rollback, making it prone to errors if not handled carefully.

#### Using `TransactionTemplate`
`TransactionTemplate` is a more convenient and idiomatic way to perform programmatic transaction management in Spring. It handles the boilerplate code for acquiring, committing, and rolling back transactions, similar to how `@Transactional` works but within a programmatic block.
```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.TransactionCallbackWithoutResult;
import org.springframework.transaction.support.TransactionTemplate;

@Service
public class TemplateTransactionService {

    private final TransactionTemplate transactionTemplate;
    private final ProductRepository productRepository;

    public TemplateTransactionService(TransactionTemplate transactionTemplate, ProductRepository productRepository) {
        this.transactionTemplate = transactionTemplate;
        this.productRepository = productRepository;
    }

    public void updateProductsAndLog(List<Product> products) {
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus status) {
                try {
                    for (Product product : products) {
                        product.setName(product.getName() + " - Updated");
                        productRepository.save(product);
                        System.out.println("Updating product: " + product.getName());

                        // Simulate a condition where we might want to roll back
                        if (product.getPrice() > 1000) {
                            System.out.println("Price too high, setting rollback only.");
                            status.setRollbackOnly(); // Mark transaction for rollback
                            // No exception thrown here, but transaction will roll back on commit attempt
                        }
                    }

                    // Simulate a scenario where an exception occurs only after some updates
                    // if (products.size() > 2) {
                    //    throw new RuntimeException("Simulating error after 2 products.");
                    // }

                } catch (Exception e) {
                    System.err.println("Exception caught during product update, setting rollback only: " + e.getMessage());
                    status.setRollbackOnly(); // Mark for rollback if an exception occurs
                    throw e; // Re-throw to propagate
                }
            }
        });
        System.out.println("Transaction block completed (either committed or rolled back).");
    }

    // Example with a return value
    public int createAndCount(String productName) {
        return transactionTemplate.execute(status -> {
            Product newProduct = new Product(null, productName, 50.0);
            productRepository.save(newProduct);
            System.out.println("Created new product: " + productName);
            return 1; // Return a value from the transaction
        });
    }
}
```

The `TransactionTemplate` is generally preferred over direct `PlatformTransactionManager` usage for programmatic transactions as it reduces boilerplate. The `execute` method takes a `TransactionCallback` (or `TransactionCallbackWithoutResult` if no return value is needed) where you can define your transactional logic. Crucially, `status.setRollbackOnly()` can be called to explicitly mark the transaction for rollback without throwing an immediate exception. The transaction will then be rolled back when the `execute` method completes.

### Comparing Declartive vs Programmatic Approaches
| Feature | Declarative (`@Transactional`) | Programmatic (`TransactionTemplate`/`PlatformTransactionManager`)                       |
|---------|--------------------------------|-----------------------------------------------------------------------------------------|
| **Simplicity** | High. Annotations are concise and easy to read. | Lower. Requires more boilerplate code.                                                  |
| **Control** | Less fine-grained. Relies on AOP interception. Limited by proxy model. | High. Explicit control over transaction boundaries, commit/rollback logic               |
| **Readability** | Excellent. Transaction boundaries are immediately visible. | Can reduce readability if abused. Transaction logic is interleaved with business logic. |
| **Maintainability** | High. Changes to transactional behavior often involve just modifying annotations. | Lower. Requires modifying more code, potential for manual errors.                       |
| **Use Cases** | Most common. Ideal for defining consistent transactional behavior for entire methods/classes. | Specific scenarios: conditional commits/rollbacks, managing transactions across non-Spring components. |
| **Testability** | Easy to test, as Spring handles the transaction. | Can be slightly more complex to test, as you are managing the transaction lifecycle directly. |

### Best Practices 
1. **Keep Transactions at the Service Layer**: The service layer is the ideal place to define transaction boundaries. It encapsulates business logic and coordinates operations across multiple repositories. Data access objects (DAOs/Repositories) should typically not manage transactions themselves.
2. **Avoid Long-Lived Transactions**: Long transactions hold database resources (locks) for extended periods, reducing concurrency and potentially leading to deadlocks. Break down complex operations into smaller, independent transactions if possible.
3. **Align Isolation Levels with Business Needs**: Choose the lowest possible isolation level that satisfies your application's data consistency requirements. Overly strict isolation levels (like SERIALIZABLE) can significantly degrade performance. READ_COMMITTED is often a good balance for many applications.
4. **Understand `rollbackFor` and `noRollbackFor`**: Be explicit about which exceptions should trigger a rollback, especially for checked exceptions.
5. **Use `readOnly = true` for Queries**: Optimize read-only operations to improve performance and concurrency.
6. **Be Mindful of Self-Invocation**: If you encounter unexpected transactional behavior, check for internal method calls within the same object. Refactor if necessary.
7. **Prioritize Declarative over Programmatic**: Use `@Transactional` whenever possible. Reserve programmatic transaction management for complex or exceptional scenarios where declarative approach falls short.

### How Transactions Behave with JPA EntityManager, Lazy Loading, and Exception Handling
#### JPA `EntityManager` Flushes
In JPA (and Hibernate), the `EntityManager` maintains a persistence context (a first-level cache) where managed entities reside. Changes to managed entities are not immediately written to the database. Instead, they are typically synchronized (flushed) to the database at specific points:
- **Before a query**: To ensure queries see the latest changes.
- **When** `EntityManager.flush()` **is explicitly called**: Developers can force a flush.
- **Before a transaction commit**: This is the most common and important one. Spring's transaction manager, working with JpaTransactionManager, ensures that the EntityManager is flushed before the transaction is committed. If the transaction rolls back, any uncommitted changes in the persistence context are discarded.

#### Lazy Loading
JPA's lazy loading mechanism defers the loading of associated entities until they are actually accessed. This is crucial for performance, preventing the loading of entire object graphs unnecessarily.
- **Transactions are essential for lazy loading**: Lazy-loaded associations can only be initialized within an active transactional context. When you access a lazily loaded collection or single entity (e.g., `order.getOrderItems()`), JPA needs an open `EntityManager` and an active transaction to fetch the data from the database.
- `LazyInitializationException`: If you try to access a lazily loaded association outside of a transactional context (e.g., in a DTO mapping layer after the service method has committed), you will encounter a `LazyInitializationException`.
- **"Open Session In View" (OSIV) Pattern**: This pattern aims to keep the `EntityManager` (and thus the transaction, though not strictly committing it) open until the view rendering phase, preventing `LazyInitializationException` in web applications. While convenient, OSIV can lead to long-lived transactions, performance issues, and make it harder to reason about transaction boundaries. It's often recommended to fetch all necessary data within the service layer's transaction or use DTOs to transfer only required data.

#### Exception Handling
Spring's `@Transactional` annotation handles exceptions by default:
- **Runtime Exceptions (** `RuntimeException` **and** `Error` **)**: By default, if any `RuntimeException` or `Error` propagates out of a `@Transactional` method, the transaction will be rolled back.
- **Checked Exceptions**: By default, checked exceptions do not trigger a rollback. This is based on the assumption that checked exceptions represent business-level exceptions that might be recoverable or handled without needing a full transaction rollback. If you need a checked exception to roll back the transaction, use `rollbackFor` attribute of `@Transactional`.
- **Internal Exception Handling**: If an exception is caught and handled within the @Transactional method, and not re-thrown, the transaction will proceed to commit normally. If you catch an exception and decide that the transaction should still be rolled back, you must either re-throw a `RuntimeException` or explicitly call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` if you're using programmatic transaction management (or a similar mechanism with `@Transactional` through the `TransactionAspectSupport`)

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.interceptor.TransactionAspectSupport;

@Service
public class ExceptionHandlingService {

    private final ProductRepository productRepository;

    public ExceptionHandlingService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Transactional
    public void processProductWithConditionalRollback(Product product) {
        try {
            productRepository.save(product);
            // Simulate a condition that might require a rollback, but is handled
            if (product.getName().contains("Invalid")) {
                System.out.println("Product name is invalid, marking for rollback.");
                // This line explicitly tells Spring to roll back the current transaction
                TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
            } else {
                System.out.println("Product processed successfully.");
            }
        } catch (Exception e) {
            System.err.println("Caught exception during product processing: " + e.getMessage());
            // If we re-throw a RuntimeException, it will naturally roll back
            throw new RuntimeException("Failed to process product: " + e.getMessage(), e);
        }
    }
}
```

In the `processProductWithConditionalRollback` example, if the product name contains "Invalid", `setRollbackOnly()` is called. Even though no exception is immediately thrown, the transaction will be rolled back when the method exits. If a `RuntimeException` is re-thrown in the `catch` block, the transaction will also be rolled back.