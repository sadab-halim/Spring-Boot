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

### Non-Repeatable Read Possible
If suppose Transaction A, reads the same row several times and there is a chance that it get different value, then its known as Non-Repeatable Read Problem.

| Time | Transaction A                        | DB                        |
|------|--------------------------------------|---------------------------|
| T1   | BEGIN_TRANSACTION                    | ID: 1 <br> Status: Free   |
| T2   | Read Row ID: 1 (reads status: Free)  | ID: 1 <br> Status: Free   |
| T3   |                                      | ID: 1 <br> Status: Booked |
| T4   | Read Row ID: 1 (read status: Booked) | ID: 1 <br> Status: Free   |
| T5   | COMMIT | |

### Phantom Read Possible
If suppose Transaction A, executed same query several time 