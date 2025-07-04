## Introduction to Spring & Spring Boot
### Overview / Core Concepts
- **Spring Framework**: A comprehensive, open-source Java framework that provides infrastructure support for developing robust Java applications. It addresses common enterprise application development challenges such as boilerplate code, tight coupling, and difficult testing. Its core principles are Inversion of Control (IoC) and Dependency Injection (DI).
- **Spring Boot vs Spring vs Spring MVC**: 
  - **Spring Framework**: The foundational framework. Provides core features like IoC, DI, AOP, transaction management, etc.
  - **Spring MVC**: A module within the Spring Framework used for building web applications (Model-View-Controller architecture). It handles web requests, rendering views, etc.
  - **Spring Boot**: Built on top of the Spring Framework, it aims to simplify the development of production-ready Spring applications. It achieves this through:
    - **Auto-configuration**: Automatically configures your Spring application based on the dependencies present in the classpath.
    - **Starters**: Pre-configured sets of dependencies that simplify build configuration.
    - **Embedded servers**: Allows you to run your application as a standalone JAR with an embedded Tomcat, Jetty, or Undertow server, eliminating the need for separate server installations.
    - **Opinionated defaults**: Provides sensible default configurations, reducing boilerplate code.
- **Spring Boot auto-configuration and starters**:
  - **Auto-configuration**: Spring Boot inspects your classpath and beans, and then automatically configures your application based on what it finds. For example, if you have `spring-boot-starter-web` on your classpath, Spring Boot will automatically configure an embedded Tomcat server and Spring MVC.
  - **Starters**: Dependency descriptors that you can include in your `pom.xml` (Maven) or `build.gradle` (Gradle). They provide a set of pre-configured, transitive dependencies for specific functionalities (e.g., `spring-boot-starter-data-jpa` for JPA, `spring-boot-starter-security` for security).
- **Spring Initializr, application.properties/yaml**:
  - **Spring Initializr**: A web-based tool (`https://start.spring.io/`) that helps you quickly generate a Spring Boot project with the necessary dependencies and project structure.
  - `application.properties` / `application.yml`: Files used for externalizing configuration in Spring Boot applications. You can define properties for various aspects of your application, such as server port, database connection, logging levels, etc. YAML (Yet Another Markup Language) is often preferred for its hierarchical structure and readability.

### Code Snippets / Configuration Examples
- `pom.xml` **with a starter**:
  ```xml
  <dependencies>
    <dependency>    
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
  </dependencies>
  ```

- `application.properties` **example**:
  ```properties
  server.port=8080
  spring.application.name=my-spring-boot-app
  ```
- `application.yml` **example**:
  ```yml
  server:
    port: 8080
  spring:
    application:
      name: my-spring-boot-app
  ```

### Best Practices / Common Pitfalls
- **Best Practices**:
  - Always use Spring Initializr for new projects.
  - Prefer `application.yml` over `application.properties` for complex configurations due to readability.
  - Understand which starters to include to avoid unnecessary dependencies
- **Common Pitfalls**:
  - Including too many unnecessary starters, leading to larger application size and potential conflicts.
  - Hardcoding configurations instead of using `application.properties`/`yml`.
  - Not understanding the auto-configuration behavior, leading to unexpected application setup.

---

## Dependency Injection & IoC
### Overview / Core Concepts
- **Inversion of Control (IoC)**: A design principle where the control of object creation and lifecycle management is transferred to a container (Spring IoC Container) rather than being managed by the objects themselves.
- **Dependency Injection (DI)**: A specific implementation of IoC where dependencies (collaborating objects) are "injected" into a component by the container, rather than the component creating or looking up its dependencies. This promotes loose coupling and testability.
- **Bean creation and management**:
  - **Spring Bean**: An object that is instantiated, assembled, and managed by the Spring IoC container.
  - The container is responsible for creating, configuring, and managing the lifecycle of these beans.
- `@Component`, `@Service`, `@Repository`, `@Controller`: These are stereotype annotations that mark a class as a Spring-managed component. They are specializations of `@Component`.
  - `@Component`: A generic stereotype for any Spring-managed component.
  - `@Service`: Indicates that a class holds business logic. It's a specialization of `@Component` for the service layer.
  - `@Repository`: Indicates that a class is a data access object (DAO) or repository. It also enables Spring's exception translation feature for persistence-related exceptions.
  - `@Controller`: Indicates that a class is a Spring MVC controller, handling web requests.
  - `@RestController`: A convenience annotation that combines `@Controller` and `@ResponseBody`. Used for building RESTful web services.
- `@Autowired`, `@Qualifier`, **construction injection vs field injection**: 
  - `@Autowired`: Spring's mechanism for automatic dependency injection. It can be used on fields, setters, and constructors.
  - `@Qualifier`: Used with `@Autowired` when there are multiple beans of the same type. It specifies which specific bean to inject by its name.
  - **Constructor Injection**: Dependencies are provided through the constructor. This is generally preferred as it ensures that the object is in a valid state upon creation and promotes immutability. It also makes testing easier.
  - **Field Injection**: Dependencies are injected directly into fields using `@Autowired`. This is concise but can hide dependencies, make testing harder, and lead to circular dependencies more easily.
  - **Setter Injection**: Dependencies are provided through setter methods. Useful for optimal dependencies or when you need to change dependencies after object creation.

### Code Snippets / Configuration Examples
#### Service with Constructor Injection:
```java
@Service
public class UserServiceImpl implements UserService {
    private final UserRepository userRepository;

    public UserServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    // ... methods
}
```

#### Repository Interface:
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // ...
}
```

#### `@Qualifier` example:
```java
@Service
public class ReportService {
    private final ReportGenerator pdfReportGenerator;

    public ReportService(@Qualifier("pdfReportGenerator") ReportGenerator pdfReportGenerator) {
        this.pdfReportGenerator = pdfReportGenerator;
    }
    // ...
}

@Component("pdfReportGenerator")
public class PdfReportGenerator implements ReportGenerator { /* ... */ }

@Component("excelReportGenerator")
public class ExcelReportGenerator implements ReportGenerator { /* ... */ }
```

### Best Practices / Common Pitfalls
- **Best Practices**:
- 

---



