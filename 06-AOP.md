# Spring Boot AOP

## AOP (Aspect Oriented Programming)
- In simple term, it helps to intercept the method invocation. And we can perform some task before and after the method.
- AOP allow us to focus on business logic by handling boilerplate and repetitive code like logging, transaction management, etc.
- So, Aspect is a module which handle this repetitive or boilerplate code.
- Helps in achieving reusability, maintainability of the code.
- User during:
  - Logging
  - Transaction Management
  - Security, etc.

### Dependency you need to add in pom.xml
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

#### Example
```java
@RestController
@RequestMapping(value = "/api/")
public class Employee {
    @GetMapping(path = "/fetchEmployee")
    public String fetchEmployee() {
        return "item fetched";
    }
}
```

```java
@Component
@Aspect
public class LoggingAspect {
    @Before("execution(public String com.conceptandcoding.learningspringboot.Employee.fetchEmployee())")
    public void beforeMethod() {
        System.out.println("Inside beforeMethod Aspect");
    }
}
```

#### Console
```
inside beforeMethod Aspect
```

## AOP Concepts
```java
@Component
@Aspect
public class LoggingAspect {
    /*
       public: access-modifers - optional and can be omitted
       return-type: optional but cannot be omitted
       com.conceptandcoding.learningspringboot.Employee.fetchEmployee(): is a pointcut
    */
    @Before("execution(public String com.conceptandcoding.learningspringboot.Employee.fetchEmployee())")
    public void beforeMethod() {
        System.out.println("Inside beforeMethod Aspect");
    }
}
```

### Pointcut
It's an expression which tells where an ADVICE should be applied.
### Types of Pointcut
1. **Execution:** matches a particular method in a particular class. <br>
   @Before("execution(public String com.conceptandcoding.learningspringboot.Employee.fetchEmployee())")
    - **(*) wildcard:** matches any single item
      - matches any return type
        ```java
        @Before("execution(* com.conceptandcoding.learningspringboot.Employee.fetchEmployee())")
            public void beforeMethod() {
            System.out.println("Inside beforeMethod Aspect");
        }
        ```
      - matches any method with single parameter string
        ```java
        @Before("execution(* com.conceptandcoding.learningspringboot.Employee.*(String))")
           public void beforeMethod() {
           System.out.println("Inside beforeMethod Aspect");
        }      
        ```
      - matches fetchEmployee method that take any single parameter
        ```java
        @Before("execution(* com.conceptandcoding.learningspringboot.Employee.fetchEmployee(*))")
           public void beforeMethod() {
           System.out.println("Inside beforeMethod Aspect");
        }      
        ```
    - **(..) wildcard:** matched 0 or more item
      - matches fetchEmployee method that take any 0 or more parameters
        ```java
        @Before("execution(execution (String com.conceptandcoding.learningspringboot.Employee.fetchEmployee(..))")
           public void beforeMethod() {
           System.out.println("Inside beforeMethod Aspect");
        }      
        ```
      - matches fetchEmployee method in 'com.conceptandcoding' package and subpackage classes
        ```java
        @Before("execution(execution (String com.conceptandcoding..fetchEmployee(..))")
           public void beforeMethod() {
           System.out.println("Inside beforeMethod Aspect");
        }      
        ```
      - matches any method in 'com.conceptandcoding' package and subpackage classes
        ```java
        @Before("execution(execution (String com.conceptandcoding..*())")
           public void beforeMethod() {
           System.out.println("Inside beforeMethod Aspect");
        }      
        ```
2. **Within:** matches all method within any class or package
   - This pointcut will run for each method in the class Employee
        ```java
        @Before("within(com.conceptandcoding.learningspringboot.Employee)")
           public void beforeMethod() {
           System.out.println("Inside beforeMethod Aspect");
        }      
        ```
   - This pointcut will run for each method in this package and subpackage 
        ```java
        @Before("within(com.conceptandcoding.learningspringboot..*)")
           public void beforeMethod() {
           System.out.println("Inside beforeMethod Aspect");
        }      
        ```   

3. **@within:** matches any method in a class which has this annotation
   ```java
   @RestController
   @RequestMapping(value = "/api/")
   public class Employee {
    @Autowired
    EmployeeUtil employeeUtil;
   
    @GetMapping(path = "/fetchEmployee")
    public String fetchEmployee() {
        employeeUtil.employeeHelperMethod();
        return "item fetched";
    }    
   }
   ```
   
   ```java
   @Aspect
   @Component
   public class LoggingAspect {
    @Before("@within(org.springframework.stereotype.Service")
    public void beforeMethod() {
        System.out.println("Inside beforeMethod aspect");
    }
   }
   ```
   
   ```java
   @Service
   public class EmployeeUtil {
    public void employeeHelperMethod() {
        System.out.println("employee helper method called");
    }
   }
   ```
   
   ```
   Inside beforeMethod aspect
   employee helper method called
   ```
4. **@annotation:** matches any method that is annotated with given annotation
   ```java
   @RestController
   @RequestMapping(value = "/api/")
   public class Employee {
    @GetMapping(path = "/fetchEmployee")
    public String fetchEmployee() {
        return "Item fetched";
    }
   }
   ```
   
   ```java
   @Component
   @Aspect
   public class LoggingAspect {
    @Before("@annotation(org.springframework.web.bind.annotation.GetMapping")
    public void beforeMethod() {
        System.out.println("Inside beforeMethod aspect");
    }
   }
   ```
5. **Args:** matches any method with particular arguments (or parameters)
   ```java
   @Component
   @Aspect
   public class LoggingAspect {
    @Before("@args(String, int)")
    public void beforeMethod() {
        System.out.println("Inside beforeMethod aspect");
    }
   }
   ```

   ```java
   @Service
   public class EmployeeUtil {
    public void employeeHelperMethod(String str, int val) {
        System.out.println("employee helper method called");
    }
   }
   ```
   
   ```
   inside beforeMethod aspect
   employee helper method called
   ```
   
   If instead of primitive type, we need object, then we can give like this:
   `@Before("args(com.conceptandcoding.learningspringboot.Employee)")`

6. **@args:** matches any method with particular parameters and that parameter class is annotated with particular annotation.
   `@Before("@args(org.springframework.stereotype.Service)")`

   ```java
   @Service
   public class EmployeeUtil {
    public void employeeHelperMethod(String str, int val) {
        System.out.println("employee helper method called");
    }
   }
   ```

   ```java
   @Component
   @Aspect
   public class LoggingAspect {
    @Before("@args(org.springframework.stereotype.Service)")
    public void beforeMethod() {
        System.out.println("Inside beforeMethod aspect");
    }
   }
   ```
   
7. **target:** matches any method on a particular instance of a class
   ```java
   @Component
   @Aspect
   public class LoggingAspect {
    @Before("target(com.conceptandcoding.learningspringboot.EmployeeUtil)")
    public void beforeMethod() {
        System.out.println("Inside beforeMethod aspect");
    }
   }
   ```

   ```
   @Before("target(com.conceptandcoding.learningspringboot.IEmployee)")
   ```
   
   ```java
   @RestController
   @RequestMapping(value = "/api")
   public class EmployeeController {
    @Autowired
    @Qualifier("testEmployee")
    IEmployee employeeObj;
   
    @GetMapping(path = "/fetchEmployee")
    public String fetchEmployee() {
        employeeObj.fetchEmployeeMethod();
        return "Item fetched";
    }
   } 
   ```
   
   ```java
   @Aspect
   @Component
   public class LoggingAspect {
    @Before("target(com.conceptandcoding.learningspringboot.IEmployee)")
    public void beforeMethod() {
        System.out.println("Inside beforeMethod aspect");
    }
   }
   ```
   
   ```java
   public interface IEmployee {
    public void fetchEmployeeMethod();
   }
   ```
   
   ```java
   @Component
   @Qualifier("testEmployee")
   public class TestEmployee implements IEmployee {
    @Override
    public void fetchEmployeeMethod() {
        System.out.println("In temp Employee fetch method");
    }
   }
   ```
   
   ```java
   @Component
   @Qualifier("permaEmployee")
   public class PermanentEmployee implements IEmployee {
    @Override
    public void fetchEmployeeMethod() {
        System.out.println("Inside permanent fetch employee method.");
    }
   }
   ```
   
   ```
   Inside beforeMethod aspect
   In temp Employee fetch method
   ```
   
---

# NEW

---

## What is AOP and Why is it Essential?
AOP is about breaking down programs into distinct parts called concerns. While object-oriented programming (OOP) primarily focuses on decomposing applications into objects based on their responsibilities, AOP introduces another level of modularity by allowing developers to encapsulate **cross-cutting concerns**. These concerns are functionalities that affect multiple points in an application, often "cutting across" different modules or layers.

**Examples of Cross-Cutting Concerns**:
- **Logging**: Recording events, debugging information, and errors.
- **Security**: Authentication, authorization, and access control.
- **Performance Monitoring**: Measuring method execution times, resource usage.
- **Auditing**: Tracking changes to data or user actions.
- **Transaction Management**: Ensuring data consistency across multiple operations.
- **Caching**: Storing frequently accessed data to improve performance.
- **Exception Handling**: Centralizing error management.

**Why Essential in Large Applications**: <br>
Without AOP, managing these cross-cutting concerns can lead to:
- Code Duplication (Scattering)
- Tight Coupling (Tangling)
- Reduced Maintainability
- Lower Readability

AOP addresses these problems by providing a way to define these concerns in a modular fashion and then "weave" them into the application where needed, without directly modifying the business logic code. This significantly reduces code duplication, improves maintainability, enhances modularity, and makes applications easier to scale and debug.

## How Spring AOP Enables this via Proxy-Based Programming
Spring AOP is implemented using proxy-based programming. When you define an aspect in Spring, Spring creates a proxy object around the target object (the object whose methods you want to intercept). When a method on the target object is called, the call is intercepted by the proxy, which then applies the advice defined in your aspect before, after, or around the execution of the original method.

#### How it Works:
- **JDK Dynamic Proxies**: If the target object implements one or more interfaces, Spring uses JDK dynamic proxies. The proxy implements the same interfaces as the target object.
- **CGLIB Proxies**: If the target object does not implement any interfaces (i.e., it's a concrete class), Spring uses CGLIB to create a subclass proxy at runtime.

This proxy-based approach means that Spring AOP operates at the method execution level. It can only intercept method calls made through the proxy. This is an important distinction and also a source of a common pitfall (self-invocation, discussed later).

## Fundamental Building Blocks of AOP
- **Aspect**: A modularization of a cross-cutting concern. It encapsulates advice (what to do) and pointcuts (where to do it). In Spring, a class annotated with @Aspect becomes an aspect.
- **Advice**: The action taken by an aspect at a particular join point. It defines "what" an aspect does. Spring AOP provides several types of advice:
  - `@Before`: The advice runs before the execution of a join point.
  - `@After`: The advice runs after the execution of a join point, regardless of whether the join point completes normally or throws an exception.
  - `@AfterReturning`: The advice runs after a join point completes successfully (i.e., returns a value without throwing an exception). You can access the return value.
  - `@AfterThrowing`: The advice runs after a join point throws an exception. You can access the thrown exception.
  - `@Around`: The most powerful advice. It completely surrounds a join point, allowing you to perform actions before and after the method execution, control whether the method executes at all, modify arguments, or modify the return value. It takes a `ProceedingJoinPoint` as an argument.
- **JoinPoint**: A point during the execution of a program, such as a method execution, an exception being thrown, or a field being modified. In Spring AOP, a join point is always the execution of a method.
- **Pointcut**: A predicate that matches join points. It defines "where" an advice should be applied. Pointcuts are expressions that select specific join points.

## Writing and Registering a Custom Aspect Class
To create an aspect in Spring Boot, you'll use the `@Aspect` annotation and typically `@Component` to make it a Spring bean so that Spring can detect and register it.

#### Example: A Simple Logging Aspect
First, ensure you have the Spring AOP dependency in your `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

Now, let's create a logging aspect:
```java
package com.example.demo.aspect;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

import java.util.Arrays;

@Aspect
@Component
public class LoggingAspect {

    // Define a pointcut for all methods in the service package
    @Pointcut("execution(* com.example.demo.service.*.*(..))")
    private void serviceMethods() {}

    // Define a pointcut for methods annotated with @Loggable
    @Pointcut("@annotation(com.example.demo.annotation.Loggable)")
    private void loggableMethods() {}

    // @Before advice
    @Before("serviceMethods()")
    public void logMethodEntry(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().toShortString();
        String args = Arrays.toString(joinPoint.getArgs());
        System.out.println("Entering method: " + methodName + " with args: " + args);
    }

    // @AfterReturning advice
    @AfterReturning(pointcut = "serviceMethods()", returning = "result")
    public void logMethodExit(JoinPoint joinPoint, Object result) {
        String methodName = joinPoint.getSignature().toShortString();
        System.out.println("Exiting method: " + methodName + " with result: " + result);
    }

    // @AfterThrowing advice
    @AfterThrowing(pointcut = "serviceMethods()", throwing = "exception")
    public void logMethodException(JoinPoint joinPoint, Throwable exception) {
        String methodName = joinPoint.getSignature().toShortString();
        System.err.println("Exception in method: " + methodName + ": " + exception.getMessage());
    }

    // @Around advice for performance monitoring
    @Around("loggableMethods()")
    public Object logExecutionTime(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        Object result = null;
        try {
            result = proceedingJoinPoint.proceed(); // Execute the target method
        } finally {
            long endTime = System.currentTimeMillis();
            long executionTime = endTime - startTime;
            String methodName = proceedingJoinPoint.getSignature().toShortString();
            System.out.println(methodName + " executed in " + executionTime + "ms");
        }
        return result;
    }
}
```

And a custom annotation for `@Loggable`:
```java
package com.example.demo.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Loggable {
}
```

Now, let's create a sample service and controller to demonstrate:
```java
package com.example.demo.service;

import com.example.demo.annotation.Loggable;
import org.springframework.stereotype.Service;

@Service
public class ProductService {

    public String getProductById(String id) {
        if ("error".equals(id)) {
            throw new RuntimeException("Product not found for ID: " + id);
        }
        return "Product with ID: " + id;
    }

    @Loggable
    public String calculatePrice(String productName, double quantity) {
        // Simulate some complex calculation
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "Price for " + productName + " (" + quantity + " units): " + (quantity * 10.5);
    }
}
```

```java
package com.example.demo.controller;

import com.example.demo.service.ProductService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ProductController {

    @Autowired
    private ProductService productService;

    @GetMapping("/products/{id}")
    public ResponseEntity<String> getProduct(@PathVariable String id) {
        try {
            String product = productService.getProductById(id);
            return ResponseEntity.ok(product);
        } catch (RuntimeException e) {
            return ResponseEntity.badRequest().body(e.getMessage());
        }
    }

    @GetMapping("/products/price/{name}/{quantity}")
    public ResponseEntity<String> calculatePrice(@PathVariable String name, @PathVariable double quantity) {
        String price = productService.calculatePrice(name, quantity);
        return ResponseEntity.ok(price);
    }
}
```

When you run this Spring Boot application and access `/products/123` or `/products/price/Laptop/2`, you'll see the logging output from the aspect without any logging code within the `ProductService` itself. If you access `/products/error`, you'll see the `@AfterThrowing` advice in action.

## Defining Pointcut Expressions
Pointcut expressions are crucial for targeting specific join points. They use the AspectJ pointcut designators.

### Common Pointcut Designators:
- `execution`: For matching method execution join points. This is the primary one used in Spring AOP.
  - `execution(public * com.example.demo.service.*.*(..))`: Any public method in any class within `com.example.demo.service` package.
  - `execution(* com.example.demo.service.ProductService.*(..))`: Any method in `ProductService`.
  - `execution(* com.example.demo..*.*(..))`: Any method in any subpackage of `com.example.demo`.
  - `execution(* *.*(String, ..))`: Any method with at least one `String` argument followed by any other arguments.
  - `execution(* set*(..))`: Any method whose name starts with "set".
- `within`: For matching join points within certain types (classes or interfaces).
  - `within(com.example.demo.service.*)`: Any join point within any class in `com.example.demo.service` package.
  - `within(com.example.demo..*)`: Any join point within any class in `com.example.demo` package or its subpackages.
- `this`: Matches join points where the proxy (Spring AOP proxy) is an instance of a given type.
- `target`: Matches join points where the target object (the object being advised) is an instance of a given type.
- `args`: Matches join points where the arguments are of a certain type or quantity.
  - `args(String, ...)`: Matches methods that take a `String` as the first argument.
- `@annotation`: Matches join points where the executing method has a specific annotation.
  - `@annotation(com.example.demo.annotation.Loggable)`: Matches methods annotated with `@Loggable`.
- `@target`: Matches join points where the target object's class has a specific annotation.
- `@args`: Matches join points where the runtime type of the arguments has a specific annotation.

### Combining Pointcut Expressions:
You can combine pointcut expressions using logical operators: `&&` (and), `||` (or), `!` (not).
```java
@Pointcut("execution(* com.example.demo.service.*.*(..)) && !execution(* com.example.demo.service.ProductService.calculatePrice(..))")
private void allServiceMethodsExceptCalculatePrice() {}
```

## Real-World Examples
#### a) Logging Method Execution Time (using `@Around`)
Demonstrated in the `LoggingAspect` above with the `@Loggable` annotation. This is incredibly useful for performance profiling

#### b) Securing Controller Endpoints (using `@Around` or `@Before`)
Imagine you have custom roles or permissions not covered by Spring Security's out-of-the-box annotations.
```java
package com.example.demo.aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

// Assume you have a SecurityContextHolder or similar for current user info
// import com.example.demo.security.SecurityContextHolder;

@Aspect
@Component
public class SecurityAspect {

    // Custom annotation for authorization
    // @Target(ElementType.METHOD) @Retention(RetentionPolicy.RUNTIME) public @interface HasPermission { String value(); }

    @Around("@annotation(com.example.demo.annotation.HasPermission)")
    public Object checkPermission(ProceedingJoinPoint joinPoint, HasPermission hasPermission) throws Throwable {
        String requiredPermission = hasPermission.value();
        // In a real application, you'd fetch user permissions from SecurityContextHolder
        // For demonstration, let's hardcode
        String currentUserPermission = "admin"; // Replace with actual user permission logic

        if (currentUserPermission.equals(requiredPermission)) {
            return joinPoint.proceed(); // User has permission, proceed with method execution
        } else {
            throw new SecurityException("Access Denied: User does not have " + requiredPermission + " permission.");
        }
    }
}
```

```java
package com.example.demo.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface HasPermission {
    String value();
}
```

```java
package com.example.demo.controller;

import com.example.demo.annotation.HasPermission;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AdminController {

    @HasPermission("admin")
    @GetMapping("/admin/data")
    public ResponseEntity<String> getAdminData() {
        return ResponseEntity.ok("Sensitive admin data accessed.");
    }
}
```

Accessing `/admin/data` would trigger the `SecurityAspect` and deny access if the `currentUserPermission` doesn't match "admin".

#### c) Catching and Processing Exceptions (`@AfterThrowing`)
Already shown in the initial `LoggingAspect`. This centralizes exception logging and can be extended to send alerts, transform exceptions, or perform cleanup.

#### d) Modifying Request/Response Payloads (`@Around`)
This is more complex and often involves working with Spring's `HandlerMethodArgumentResolver` and `HttpMessageConverter` for controllers, but AOP can be used for general method argument/return value manipulation.
```java
// Example: Encrypting/Decrypting Sensitive Data in a Service Layer
@Around("execution(* com.example.demo.service.UserService.getUserDetails(..))")
public Object encryptDecryptUserData(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
    Object[] args = proceedingJoinPoint.getArgs();
    // Assuming the first arg is an encrypted user ID
    String encryptedUserId = (String) args[0];
    String decryptedUserId = decrypt(encryptedUserId); // Your decryption logic

    // Modify arguments before proceeding
    Object[] newArgs = {decryptedUserId};
    Object result = proceedingJoinPoint.proceed(newArgs); // Execute with modified args

    // Assuming the result is sensitive user data, encrypt it before returning
    if (result instanceof UserDetails) {
        UserDetails userDetails = (UserDetails) result;
        // Encrypt specific fields in userDetails object before returning
        // userDetails.setEmail(encrypt(userDetails.getEmail()));
        // userDetails.setPhoneNumber(encrypt(userDetails.getPhoneNumber()));
    }
    return result;
}
```

## Powerful Use Cases for `@Around` Advice
`@Around` advice gives you complete control over the execution of the advised method.
- **Transaction Timing**: Similar to performance monitoring, but specific to database transactions. You can start a timer before a transactional method and log its duration, especially useful for long-running transactions.
- **Retry Logic**: If a method call fails due to transient issues (e.g., network timeout, database deadlock), `@Around` can implement retry logic.
```java
@Around("@annotation(com.example.demo.annotation.Retryable)")
public Object retryOperation(ProceedingJoinPoint proceedingJoinPoint, Retryable retryable) throws Throwable {
    int maxAttempts = retryable.value();
    int attempt = 0;
    Throwable lastException = null;

    while (attempt < maxAttempts) {
        try {
            return proceedingJoinPoint.proceed();
        } catch (Throwable ex) {
            lastException = ex;
            attempt++;
            System.out.println("Attempt " + attempt + " failed for " + proceedingJoinPoint.getSignature().toShortString() + ". Retrying...");
            if (attempt < maxAttempts) {
                Thread.sleep(retryable.delay()); // Wait before retrying
            }
        }
    }
    throw lastException; // Re-throw if all attempts fail
}
```

```java
// Custom annotation for retry
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Retryable {
    int value() default 3; // Max attempts
    long delay() default 1000; // Delay in milliseconds between retries
}
```

```java
// Custom annotation for retry
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Retryable {
    int value() default 3; // Max attempts
    long delay() default 1000; // Delay in milliseconds between retries
}
```

```java
// In your service:
@Retryable(value = 5, delay = 500)
public String performRiskyOperation() {
    // Simulate a flaky operation
    if (Math.random() < 0.7) { // 70% chance of failure
        throw new RuntimeException("Operation failed temporarily!");
    }
    return "Operation successful!";
}
```

- **Input/Output Transformation**: As seen in the sensitive data example, `@Around` can modify method arguments before execution and alter return values afterward. This is useful for data encryption/decryption, data sanitization, or standardizing input/output formats.

## Common Pitfalls and Limitations of Proxy-Based AOP
#### Self-Invocation (Internal Method Calls):
This is the most common pitfall. If a method within the same object calls another method within the same object, the AOP advice will not be applied to the internally called method. This is because the internal call is a direct method invocation on this reference, bypassing the Spring AOP proxy.
```java
@Service
public class MyService {
    @Loggable
    public void outerMethod() {
        System.out.println("Outer method logic");
        innerMethod(); // Self-invocation: AOP won't apply to innerMethod
    }

    @Loggable
    public void innerMethod() {
        System.out.println("Inner method logic");
    }
}
```

**Workarounds**: <br>
- Inject the service into itself (not recommended, can lead to circular dependencies).
- Call the method on the proxy directly (more complex, requires `AopContext.currentProxy()`).
- Refactor the code so that the advised method is called from another Spring bean, thus going through the proxy.
- Consider using full AspectJ for more powerful weaving capabilities.

#### Limitations of Proxy-Based AOP:
- **Method Execution Only**: Spring AOP only works for method execution join points. It cannot intercept field access, constructor calls, or static method calls.
- **Public Method Limitation (JDK Proxies)**: If using JDK dynamic proxies (when the target implements interfaces), only public methods can be advised. CGLIB proxies (for concrete classes) can advise non-public methods.
- **Final Classes/Methods**: Spring AOP (CGLIB) cannot proxy final classes or final methods, as it relies on subclassing.

#### When Full AspectJ is Needed:
- **Field Access Advising**: If you need to intercept read/write operations on fields.
- **Constructor Advising**: If you need to perform actions during object construction.
- **Static Method Advising**: If you need to advise static methods.
- **No Proxy Limitations**: AspectJ performs byte-code weaving at compile-time or load-time, meaning it directly modifies the .class files. This overcomes the self-invocation problem and the public method limitation.
- **More Complex Pointcut Expressions**: AspectJ offers a richer set of pointcut designators.

For most Spring Boot applications, Spring AOP (proxy-based) is sufficient. Only when you hit its limitations should you consider integrating full AspectJ.

## Best Practices
- **Keep Aspects Focused**: Each aspect should ideally address a single cross-cutting concern. Don't mix logging, security, and transaction management in one aspect. This improves modularity and maintainability.
- **Externalize Reusable Pointcuts**: As demonstrated with `serviceMethods()` and `loggableMethods()`, define common pointcut expressions in `@Pointcut` methods and reuse them across different advice methods within the same aspect or even other aspects. This makes your pointcut expressions more readable and easier to manage.
- **Apply AOP in the Service Layer Instead of Controllers**:
  - **Why not Controllers?** Controllers are primarily responsible for handling web requests and responses. Applying AOP directly to controllers can sometimes lead to issues with argument binding, return value serialization, and might not be the most appropriate layer for certain cross-cutting concerns (e.g., transaction management).
  - **Why Service Layer?** The service layer typically encapsulates the core business logic. Applying AOP here ensures that the cross-cutting concerns are applied uniformly across all business operations, regardless of how they are invoked (e.g., via a REST API, a message queue, or a scheduled job). This promotes a clean separation of concerns and makes your business logic independent of the access mechanism.
- **Use the Right Advice Type**: Choose the most appropriate advice type for your use case. @Around is powerful but can be complex. Prefer `@Before`, `@AfterReturning`, or `@AfterThrowing` when you just need to perform an action at a specific point without controlling the method execution.
- **Avoid Overuse**: While AOP is powerful, don't use it for every minor cross-cutting concern. Sometimes, a simple utility method or inheritance is more suitable. Overusing AOP can make the control flow harder to understand.
- **Test Your Aspects**: Just like any other code, your aspects need to be thoroughly tested to ensure they behave as expected and don't introduce unintended side effects.