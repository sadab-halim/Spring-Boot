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