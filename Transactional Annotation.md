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

### `@Transactional` annotation at Method Level
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


