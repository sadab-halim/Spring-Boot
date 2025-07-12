# Spring Boot Annotations

## Core Annotations (Application Entry and Configuration)
These annotations are fundamental for setting up and running a Spring Boot application.
### `@SpringBootApplication`
This is a convenience annotation that marks the main class of a Spring Boot application. It's a combination of three other crucial annotations:
- `@Configuration`: Tags the class as a source of bean definitions for the Spring application context. Spring will use this class to find and register beans.
- `@EnableAutoConfiguration`: Tells Spring Boot to automatically configure your application based on the JAR dependencies that you have added. For example, if `spring-boot-starter-web` is on the classpath, it automatically configures Tomcat and Spring MVC.
- `@ComponentScan`: Instructs Spring to scan the current package and its sub-packages for components (classes annotated with `@Component`, `@Service`, `@Repository`, `@Controller`, etc.) and register them as beans in the application context.

#### Real-world Example
```java
// src/main/java/com/example/myapp/MyApplication.java
package com.example.myapp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

**Common Use Case**: The entry point for almost every Spring Boot application. <br>
**Best Practices**: Place it in the root package of your application to ensure all components are scanned. <br>
**Potential Pitfalls**: If placed in a sub-package, it might not scan components in parent packages unless explicitly configured.

### `@Configuration`
Indicates that a class declares one or more `@Bean` methods and may be processed by the Spring container to generate bean definitions and service requests for those beans.

#### Real-world Example
```java
// src/main/java/com/example/myapp/config/AppConfig.java
package com.example.myapp.config;

import com.example.myapp.service.MyService;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean
    public MyService myService() {
        return new MyService();
    }
}
```

**Common Use Case**: Defining complex bean configurations, third-party library integrations, or application-specific configurations that can't be easily auto-configured.

### `@ComponentScan`
Configures component scanning directives for use with `@Configuration` classes. It can be used to customize the package scanning behavior.

#### Real-world Example
```java
// src/main/java/com/example/myapp/MyApplication.java
package com.example.myapp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;

@SpringBootApplication
@ComponentScan(basePackages = {"com.example.myapp", "com.example.anothermodule.service"})
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

**Common Use Case**: When your components are scattered across different root packages, or when you want to exclude specific packages from scanning. <br>
**Best Practices**: Generally, rely on the default behavior of `@SpringBootApplication` unless you have specific reasons to customize. <br>
**Potential Pitfalls**: Overly broad scanning can lead to longer startup times and unintended bean registrations.

### `@EnableAutoConfiguration`
Enables Spring Boot's auto-configuration mechanism. This annotation is crucial for the "just run" philosophy of Spring Boot.

#### Real-world Example: (Typically used implicitly via @SpringBootApplication)
```java
// This is how it works internally within @SpringBootApplication
@Configuration
@EnableAutoConfiguration
@ComponentScan
public class MyStandaloneApplicationConfig {
    // ...
}
```

**Common Use Case**: Allows Spring Boot to automatically set up configurations based on the dependencies present in your `pom.xml` (or `build.gradle`). For example, if you add `spring-boot-starter-data-jpa`, it will automatically configure a `DataSource` and `EntityManagerFactory`. <br>
**Best Practices**: Understand that auto-configuration can be overridden. If you define your own beans, they will typically take precedence over auto-configured ones. <br>
**Potential Pitfalls**: Can sometimes pull in unnecessary dependencies or configure things you don't need, which can be mitigated by excluding specific auto-configurations.

## Stereotype Annotations (Component Scanning and DI)
These annotations are specializations of `@Component` and provide semantic meaning to your classes, aiding in readability and organization. They also serve as markers for component scanning. <br>
`@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`

All these annotations are specializations of `@Component`. When Spring performs component scanning, it looks for classes annotated with any of these to register them as beans in the application context.
- `@Component`: A generic stereotype for any Spring-managed component. It's the most general-purpose annotation for marking a class as a Spring bean.
- `@Service`: Indicates that an annotated class is a "Service" (e.g., business logic). It's a specialization of `@Component` for the service layer.
- `@Repository`: Indicates that an annotated class is a "Repository" (e.g., data access object). It's a specialization of `@Component` that also enables persistence exception translation (translates technology-specific exceptions like `SQLException` into Spring's unchecked `DataAccessException` hierarchy).
- `@Controller`: Indicates that an annotated class is a "Controller" (e.g., handles web requests and returns a view name or redirects). It's a specialization of @Component for the web layer, typically used in traditional Spring MVC applications returning view names.
- `@RestController`: A convenience annotation that combines `@Controller` and `@ResponseBody`. It's used for building RESTful web services that directly return data (e.g., JSON or XML) instead of view names.

### Differences and Semantic Intent:
While all of them ultimately make a class a Spring bean, their primary differences lie in their **semantic intent** and the **additional features** they might enable:
- `@Component`: General-purpose. Use when none of the more specific stereotypes fit.
- `@Service`: For business logic. Good for clarity and organization.
- `@Repository`: For data access. Enables exception translation, which is a significant benefit for data layer consistency.
- `@Controller`: For web layer, traditionally returning view names.
- `@RestController`: For RESTful web services, returning data directly.

### Component Scanning Mechanism and Scope:
Spring's `@ComponentScan` annotation (which is part of `@SpringBootApplication`) is responsible for finding these stereotyped classes. Once found, Spring registers them as singleton beans by default, meaning only one instance of each class is created and managed by the Spring container.

#### Real-world Examples:
```java
// src/main/java/com/example/myapp/component/MyUtilityComponent.java
package com.example.myapp.component;

import org.springframework.stereotype.Component;

@Component
public class MyUtilityComponent {
    public String doSomethingUseful() {
        return "Utility task performed.";
    }
}
```
```java
// src/main/java/com/example/myapp/service/ProductService.java
package com.example.myapp.service;

import org.springframework.stereotype.Service;
import com.example.myapp.repository.ProductRepository;
import org.springframework.beans.factory.annotation.Autowired;

@Service
public class ProductService {

    private final ProductRepository productRepository;

    @Autowired
    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public String getProductDetails(Long id) {
        return productRepository.findById(id).orElse("Product not found");
    }
}
```
```java
// src/main/java/com/example/myapp/repository/ProductRepository.java
package com.example.myapp.repository;

import org.springframework.stereotype.Repository;

@Repository
public class ProductRepository {
    public String findById(Long id) {
        // Simulate database access
        if (id == 1L) {
            return "Laptop";
        }
        return null;
    }
}
```
```java
// src/main/java/com/example/myapp/controller/WebController.java
package com.example.myapp.controller;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class WebController {

    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("message", "Welcome to the web app!");
        return "home"; // Resolves to src/main/resources/templates/home.html
    }
}
```
```java
// src/main/java/com/example/myapp/controller/ProductRestController.java
package com.example.myapp.controller;

import com.example.myapp.service.ProductService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ProductRestController {

    private final ProductService productService;

    @Autowired
    public ProductRestController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping("/api/products/{id}")
    public String getProduct(@PathVariable Long id) {
        return productService.getProductDetails(id);
    }
}
```

**Best Practices**: Always use the most specific stereotype annotation that applies to your class. This improves code readability and allows Spring to apply specific features (like exception translation for `@Repository`).

## Dependency Injection Annotations
These annotations are at the heart of Spring's Inversion of Control (IoC) container, enabling automatic dependency management.

### `@Autowired`
The primary annotation for automatic dependency injection. It can be used for field, setter, and constructor injection. Spring will automatically find a matching bean in the application context and inject it.
- **Field Injection**: (Least recommended for larger applications)
  ```java
  // In MyService.java
  @Autowired
  private ProductRepository productRepository;
  ```

- **Setter Injection**: (Allows for optional dependencies or post-construction logic)
  ```java
  // In MyService.java
  private ProductRepository productRepository;
  
  @Autowired
  public void setProductRepository(ProductRepository productRepository) {
    this.productRepository = productRepository;
  }
  ```

- **Constructor Injection**: (Most recommended for mandatory dependencies, promotes immutability and testability)
  ```java
  // In MyService.java
  private final ProductRepository productRepository;
  
  @Autowired // From Spring 4.3 onwards, @Autowired is optional for single constructor  
  public MyService(ProductRepository productRepository) {
    this.productRepository = productRepository;
  }
  ```

**Common Use Case**: Injecting services into controllers, repositories into services, or any component needing another Spring-managed bean. <br>
**Best Practices**: Prefer constructor injection for mandatory dependencies. It makes dependencies explicit, helps with immutability, and makes testing easier by allowing direct instantiation without Spring. Avoid field injection in production code due to potential issues with circular dependencies and testability. <br>
**Potential Pitfalls**:
- `NoUniqueBeanDefinitionException`: If Spring finds multiple beans of the same type.
- `NoSuchBeanDefinitionException`: If Spring cannot find any bean of the required type.

### `@Qualifier`
Used in conjunction with `@Autowired` to resolve ambiguity when multiple beans of the same type are present in the application context. It specifies which specific bean should be injected.

#### Real-world Example:
```java
// src/main/java/com/example/myapp/service/EmailService.java
package com.example.myapp.service;

public interface EmailService {
    void sendEmail(String to, String subject, String body);
}
```

```java
// src/main/java/com/example/myapp/service/impl/SmtpEmailService.java
package com.example.myapp.service.impl;

import com.example.myapp.service.EmailService;
import org.springframework.stereotype.Service;

@Service("smtpEmailService") // Give a specific name
public class SmtpEmailService implements EmailService {
    @Override
    public void sendEmail(String to, String subject, String body) {
        System.out.println("Sending SMTP email to " + to + ": " + subject);
    }
}
```

```java
// src/main/java/com/example/myapp/service/impl/MockEmailService.java
package com.example.myapp.service.impl;

import com.example.myapp.service.EmailService;
import org.springframework.stereotype.Service;

@Service("mockEmailService") // Give another specific name
public class MockEmailService implements EmailService {
    @Override
    public void sendEmail(String to, String subject, String body) {
        System.out.println("Mock: Sending email to " + to + ": " + subject);
    }
}
```

```java
// src/main/java/com/example/myapp/controller/NotificationController.java
package com.example.myapp.controller;

import com.example.myapp.service.EmailService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class NotificationController {

    private final EmailService emailService;

    @Autowired
    public NotificationController(@Qualifier("smtpEmailService") EmailService emailService) {
        this.emailService = emailService;
    }

    @GetMapping("/notify")
    public String sendNotification() {
        emailService.sendEmail("user@example.com", "Hello", "Welcome!");
        return "Notification sent via SMTP";
    }
}
```

**Common Use Case**: When you have multiple implementations of an interface or multiple beans of the same type and need to specify which one to use. <br>
**Best Practices**: Use meaningful qualifier names. Consider using `@Primary` for the default bean if most injections should use a specific implementation. <br>

### `@Value`
Used to inject property values into Spring-managed beans. It can retrieve values from property files (e.g., `application.properties`), system properties, environment variables, or SpEL expressions.
#### Real-world Example:
```java
// src/main/resources/application.properties
app.name=My Spring Boot App
app.version=1.0.0
app.admin.email=admin@example.com
```

```java
// src/main/java/com/example/myapp/component/AppInfo.java
package com.example.myapp.component;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class AppInfo {

    @Value("${app.name}")
    private String appName;

    @Value("${app.version:unknown}") // Default value if property not found
    private String appVersion;

    @Value("${app.admin.email}")
    private String adminEmail;

    @Value("#{systemProperties['java.version']}") // SpEL for system property
    private String javaVersion;

    public void printAppInfo() {
        System.out.println("App Name: " + appName);
        System.out.println("App Version: " + appVersion);
        System.out.println("Admin Email: " + adminEmail);
        System.out.println("Java Version: " + javaVersion);
    }
}
```

**Common Use Case**: Injecting configuration properties, externalizing environment-specific values, or injecting simple String/number values. <br>
**Best Practices**: For complex configurations, consider @ConfigurationProperties over multiple @Value annotations as it provides type-safe access.

### `@Inject` and `@Resource` (JSR-330/JSR-250 alternatives)
These are standard Java annotations for dependency injection, defined by JSR-330 (`@Inject`) and JSR-250 (`@Resource`). Spring supports them as alternatives to `@Autowired`.
- `@Inject`: From JSR-330 (Dependency Injection for Java). Works similarly to `@Autowired` and can be used on fields, methods, and constructors. It does not have a `required` attribute like `@Autowired`.
- `@Resource`: From JSR-250 (Common Annotations for Java). Primarily used for injection by name. If a name is specified, it will look for a bean with that name. If no name is specified, it defaults to injecting by name of the field/setter method, then by type.

#### Real-world Example:
```java
// Using @Inject (requires javax.inject or jakarta.inject dependency)
import jakarta.inject.Inject; // or javax.inject.Inject for older versions

// In MyService.java
public class MyService {

    @Inject
    private ProductRepository productRepository;

    // ...
}
```

```java
// Using @Resource
import jakarta.annotation.Resource; // or javax.annotation.Resource for older versions

// In MyService.java
public class MyService {

    @Resource(name = "anotherProductRepository") // Inject by specific bean name
    private ProductRepository productRepository;

    @Resource // Inject by field name "productRepository" or by type
    private ProductRepository anotherRepository;

    // ...
}
```

**Common Use Case**: Primarily used for portability if you want your code to be less coupled to Spring-specific annotations (though Spring is the de-facto standard for Java DI). `@Resource` is useful when injecting by name is a primary requirement. <br>
**Best Practices**: Stick to `@Autowired` for consistency within a Spring Boot project unless there's a strong reason to use the JSR alternatives (e.g., maintaining compatibility with other DI containers).

---

## Bean and Lifecycle Annotations
These annotations help in managing the creation, initialization, and destruction of Spring beans.

### @Bean
Used within a `@Configuration` class to declare a method that produces a bean to be managed by the Spring container. The method's return value will be registered as a bean.

#### Real-world Example:
```java
// src/main/java/com/example/myapp/config/MessagingConfig.java
package com.example.myapp.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import com.example.myapp.service.MessagingService;

@Configuration
public class MessagingConfig {

    @Bean
    public MessagingService smsMessagingService() {
        return new MessagingService("SMS");
    }

    @Bean
    public MessagingService emailMessagingService() {
        return new MessagingService("Email");
    }
}
```

**Common Use Case**:
- Integrating third-party libraries where you need to explicitly create and configure objects as Spring beans.
- When a class isn't annotated with a stereotype annotation (e.g., it's part of an external library).
- Defining beans that require complex setup logic.

### `@PostConstruct`, `@PreDestroy`
These are JSR-250 annotations (part of Java EE, now Jakarta EE) that Spring supports for lifecycle callbacks.
- `@PostConstruct`: Marks a method to be executed after dependency injection is done to perform any initialization.
- `@PreDestroy`: Marks a method to be executed before a bean is destroyed (e.g., when the application context is closed), typically for cleanup operations.

#### Real-world Example:
```java
// src/main/java/com/example/myapp/component/ResourceHandler.java
package com.example.myapp.component;

import org.springframework.stereotype.Component;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

@Component
public class ResourceHandler {

    private boolean connectionOpen;

    @PostConstruct
    public void init() {
        System.out.println("ResourceHandler: Initializing connection...");
        this.connectionOpen = true;
    }

    public void processData() {
        if (connectionOpen) {
            System.out.println("ResourceHandler: Processing data.");
        } else {
            System.out.println("ResourceHandler: Connection not open.");
        }
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("ResourceHandler: Closing connection...");
        this.connectionOpen = false;
    }
}
```

**Common Use Case**:
- `@PostConstruct`: Loading initial data, establishing connections, performing setup that requires injected dependencies.
- `@PreDestroy`: Releasing resources, closing connections, flushing caches.

### `InitializingBean`, `DisposableBean`
These are Spring-specific interfaces that provide similar lifecycle callback functionality to `@PostConstruct` and `@PreDestroy`.
- `InitializingBean`: Implements the `afterPropertiesSet()` method, which is invoked after all properties have been set. 
- `DisposableBean`: Implements the `destroy()` method, which is invoked when the bean is destroyed.

#### Real-world Example:
```java
// src/main/java/com/example/myapp/component/LegacyResourceHandler.java
package com.example.myapp.component;

import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.stereotype.Component;

@Component
public class LegacyResourceHandler implements InitializingBean, DisposableBean {

    public LegacyResourceHandler() {
        System.out.println("LegacyResourceHandler: Constructor called.");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("LegacyResourceHandler: afterPropertiesSet() called (equivalent to @PostConstruct).");
        // Perform initialization here
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("LegacyResourceHandler: destroy() called (equivalent to @PreDestroy).");
        // Perform cleanup here
    }

    public void doWork() {
        System.out.println("LegacyResourceHandler: Doing work.");
    }
}
```

**Common Use Case**: Primarily used for older Spring applications or when you prefer interface-based callbacks. <br>
**Best Practices**: For new Spring Boot applications, generally prefer `@PostConstruct` and `@PreDestroy` as they are annotation-driven, less intrusive, and part of a standard.

### `@Lazy`
Indicates that a bean should be initialized only when it's first accessed, rather than at application startup.
#### Real-world Example:
```java
// src/main/java/com/example/myapp/service/ExpensiveService.java
package com.example.myapp.service;

import org.springframework.context.annotation.Lazy;
import org.springframework.stereotype.Service;

@Service
@Lazy // This service will only be initialized when first requested
public class ExpensiveService {
    public ExpensiveService() {
        System.out.println("ExpensiveService initialized.");
    }

    public String performHeavyOperation() {
        return "Heavy operation completed.";
    }
}
```

```java
// In another component that injects ExpensiveService
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class ServiceConsumer {

    private final ExpensiveService expensiveService;

    @Autowired
    public ServiceConsumer(@Lazy ExpensiveService expensiveService) { // @Lazy also works on injection point
        this.expensiveService = expensiveService;
        System.out.println("ServiceConsumer created, but ExpensiveService not yet initialized.");
    }

    public void triggerExpensiveOperation() {
        System.out.println("Triggering expensive operation...");
        System.out.println(expensiveService.performHeavyOperation());
    }
}
```

**Common Use Case**: For services that are not always needed or consume significant resources, improving application startup time. <br>
**Potential Pitfalls**: Can sometimes mask initialization errors that would otherwise appear at startup. Be mindful of potential `NullPointerExceptions` if a lazy-initialized bean is expected to be available immediately.

### `@Primary`
Indicates that a bean should be given preference when multiple beans of the same type are candidates for autowiring.

#### Real-world Example: (Building on the `EmailService` example)
```java
// src/main/java/com/example/myapp/service/impl/SmtpEmailService.java
package com.example.myapp.service.impl;

import com.example.myapp.service.EmailService;
import org.springframework.context.annotation.Primary;
import org.springframework.stereotype.Service;

@Service("smtpEmailService")
@Primary // This is the preferred EmailService
public class SmtpEmailService implements EmailService {
    @Override
    public void sendEmail(String to, String subject, String body) {
        System.out.println("Sending SMTP email to " + to + ": " + subject);
    }
}
```

```java
// src/main/java/com/example/myapp/controller/NotificationController.java
package com.example.myapp.controller;

import com.example.myapp.service.EmailService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class NotificationController {

    private final EmailService emailService;

    @Autowired // No @Qualifier needed now, @Primary resolves ambiguity
    public NotificationController(EmailService emailService) {
        this.emailService = emailService;
    }

    @GetMapping("/notify")
    public String sendNotification() {
        emailService.sendEmail("user@example.com", "Hello", "Welcome!");
        return "Notification sent (via Primary EmailService)";
    }
}
```

**Common Use Case**: When you have a default implementation among several alternatives. <br>
**Best Practices**: Use @Primary for the most common or default implementation. Use @Qualifier for specific, non-default injections.

--- 

## Scope and Profile Annotations
These annotations allow you to control the lifecycle scope of beans and activate beans conditionally based on the environment.

### `@Scope`
Defines the scope of a Spring bean. By default, beans are singletons.
#### Common Scopes:
- `singleton` **(Default)**: A single instance of the bean is created per Spring IoC container.
- `prototype`: A new instance of the bean is created every time it is requested.
- `request`: A new instance of the bean is created for each HTTP request (only in web-aware applications).
- `session`: A new instance of the bean is created for each HTTP session (only in web-aware applications).
- `application`: A single instance of the bean is created for the entire ServletContext (only in web-aware applications).
- `websocket`: A single instance of the bean is created for the life of a WebSocket session.

#### Real-world Example:
```java
// src/main/java/com/example/myapp/component/PrototypeBean.java
package com.example.myapp.component;

import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

@Component
@Scope("prototype")
public class PrototypeBean {
    private static int counter = 0;
    private final int instanceId;

    public PrototypeBean() {
        this.instanceId = ++counter;
        System.out.println("PrototypeBean instance " + instanceId + " created.");
    }

    public int getInstanceId() {
        return instanceId;
    }
}
```

```java
// src/main/java/com/example/myapp/controller/ScopeController.java
package com.example.myapp.controller;

import com.example.myapp.component.PrototypeBean;
import org.springframework.beans.factory.ObjectFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ScopeController {

    // Injecting ObjectFactory to get a new instance each time for prototype
    @Autowired
    private ObjectFactory<PrototypeBean> prototypeBeanObjectFactory;

    @GetMapping("/prototype")
    public String getPrototypeInstance() {
        PrototypeBean bean1 = prototypeBeanObjectFactory.getObject();
        PrototypeBean bean2 = prototypeBeanObjectFactory.getObject();
        return "Prototype instances: " + bean1.getInstanceId() + " and " + bean2.getInstanceId();
    }
}
```

#### Common Use Case:
- `prototype`: For stateful beans that should not be shared across multiple clients/threads, or for objects that need to be frequently reset.
- `request`/`session`: For web-specific data that is tied to the lifecycle of an HTTP request or session.

**Best Practices**: Be cautious with `prototype` scope in combination with singleton beans, as a singleton will only get one instance of a prototype bean (the one injected at its creation). Use `ObjectFactory` or `ApplicationContext.getBean()` for new instances.

### `@Profile`
Allows you to register beans conditionally based on the active Spring profiles. Profiles are typically set via `spring.profiles.active` property.

#### Real-world Example:
```java
// src/main/java/com/example/myapp/service/EmailService.java (Interface remains the same)
// ...

// src/main/java/com/example/myapp/service/impl/DevEmailService.java
package com.example.myapp.service.impl;

import com.example.myapp.service.EmailService;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;

@Service
@Profile("dev") // Active only when 'dev' profile is active
public class DevEmailService implements EmailService {
    @Override
    public void sendEmail(String to, String subject, String body) {
        System.out.println("DEV: Sending email to " + to + ": " + subject);
    }
}
```

```java
// src/main/java/com/example/myapp/service/impl/ProdEmailService.java
package com.example.myapp.service.impl;

import com.example.myapp.service.EmailService;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Service;

@Service
@Profile("prod") // Active only when 'prod' profile is active
public class ProdEmailService implements EmailService {
    @Override
    public void sendEmail(String to, String subject, String body) {
        System.out.println("PROD: Sending email to " + to + ": " + subject);
    }
}
```

To activate: `java -jar myapp.jar --spring.profiles.active=dev` or `set spring.profiles.active=prod` in application.properties.

**Common Use Case**: Providing different configurations or bean implementations for various environments (development, testing, production).

**Best Practices**: Use meaningful profile names. Avoid creating too many fine-grained profiles, as it can lead to complexity.

**Conditional Annotations (**`@Conditional`, `@ConditionalOnProperty`, `@ConditionalOnClass`, `@ConditionalOnMissingBean`, **etc.)**
These annotations provide powerful mechanisms for conditional bean registration, allowing beans to be included in the application context only if certain conditions are met. They are heavily used by Spring Boot's auto-configuration.
- `@Conditional`: The most generic one. Takes a `Condition` implementation that programmatically determines if a bean should be registered.
- `@ConditionalOnProperty`: Registers a bean only if a specific Spring environment property exists and optionally has a certain value.
- `@ConditionalOnClass`: Registers a bean only if the specified classes are present on the classpath.
- `@ConditionalOnMissingBean`: Registers a bean only if a bean of the specified type (or name) is not already present in the application context. This is crucial for auto-configuration, allowing users to override default beans.
- `@ConditionalOnBean`: Registers a bean only if a bean of the specified type (or name) is present in the application context.
- `@ConditionalOnMissingClass`: Registers a bean only if the specified classes are not present on the classpath.
- `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication`: Registers a bean based on whether the application is a web application or not.

#### Real-world Example (`@ConditionalOnProperty`):
```java
// src/main/java/com/example/myapp/service/FeatureToggleService.java
package com.example.myapp.service;

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Service;

@Service
@ConditionalOnProperty(name = "features.new-feature.enabled", havingValue = "true")
public class FeatureToggleService {
    public void executeNewFeature() {
        System.out.println("Executing the new feature!");
    }
}
```

To enable: `features.new-feature.enabled=true` in `application.properties`.

#### Real-world Example (`@ConditionalOnMissingBean`):
```java
// src/main/java/com/example/myapp/config/DefaultDataSourceConfig.java
package com.example.myapp.config;

import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;

@Configuration
public class DefaultDataSourceConfig {

    @Bean
    @ConditionalOnMissingBean // Only create this DataSource if no other DataSource bean exists
    public DataSource myDefaultDataSource() {
        System.out.println("Creating default in-memory H2 DataSource.");
        // Example: return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2).build();
        return null; // Placeholder
    }
}
```

**Common Use Case**: Building flexible and extensible applications, especially for libraries and auto-configurations. Allows features to be enabled/disabled based on configuration or classpath presence.

**Best Practices**: Understand the order of evaluation for multiple conditional annotations on a single class. For complex custom conditions, implement the `Condition` interface.

## Web & Controller Annotations
These annotations are essential for building RESTful APIs and traditional web applications with Spring MVC.

### Request Mapping Annotations
These map incoming HTTP requests to handler methods in Spring controllers.
- `@RequestMapping`: The original and most versatile mapping annotation. Can be used at the class level (base path) and method level (specific path and HTTP method).
   ```java
   @Controller
   @RequestMapping("/users") // Base path for all methods in this controller
   public class UserController {

      @RequestMapping(value = "/{id}", method = RequestMethod.GET)
      public String getUserById(@PathVariable Long id) {
         return "User with ID: " + id;
      }

      @RequestMapping(value = "/create", method = RequestMethod.POST)
      public String createUser(@RequestBody User user) {
         return "User created: " + user.getName();
      }
   }
  ```

- `@GetMapping`: Shortcut for `@RequestMapping(method = RequestMethod.GET)`.
- `@PostMapping`: Shortcut for `@RequestMapping(method = RequestMethod.POST)`.
- `@PutMapping`: Shortcut for `@RequestMapping(method = RequestMethod.PUT)`.
- `@DeleteMapping`: Shortcut for `@RequestMapping(method = RequestMethod.DELETE)`.
- `@PatchMapping`: Shortcut for `@RequestMapping(method = RequestMethod.PATCH)`.

#### Real-world Example (using specific HTTP method annotations):
```java
// src/main/java/com/example/myapp/controller/ProductRestController.java
package com.example.myapp.controller;

import com.example.myapp.model.Product;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

@RestController
@RequestMapping("/api/products")
public class ProductRestController {

    private List<Product> products = new ArrayList<>(); // Simple in-memory store

    public ProductRestController() {
        products.add(new Product(1L, "Laptop", 1200.00));
        products.add(new Product(2L, "Mouse", 25.00));
    }

    @GetMapping
    public List<Product> getAllProducts() {
        return products;
    }

    @GetMapping("/{id}")
    public ResponseEntity<Product> getProductById(@PathVariable Long id) {
        Optional<Product> product = products.stream().filter(p -> p.getId().equals(id)).findFirst();
        return product.map(ResponseEntity::ok)
                      .orElseGet(() -> ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Product createProduct(@RequestBody Product product) {
        products.add(product);
        return product;
    }

    @PutMapping("/{id}")
    public ResponseEntity<Product> updateProduct(@PathVariable Long id, @RequestBody Product updatedProduct) {
        for (int i = 0; i < products.size(); i++) {
            if (products.get(i).getId().equals(id)) {
                products.set(i, updatedProduct);
                return ResponseEntity.ok(updatedProduct);
            }
        }
        return ResponseEntity.notFound().build();
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteProduct(@PathVariable Long id) {
        products.removeIf(p -> p.getId().equals(id));
    }
}
```

**Common Use Cases**: Defining REST endpoints for CRUD operations or any web-exposed functionality.

**Best Practices**: Use the specific HTTP method annotations (`@GetMapping`, etc.) for clarity. Use `@RequestMapping` when you need to specify multiple HTTP methods or more complex request matching criteria (e.g., `consumes`, `produces`).

### Parameter and Body Handling Annotations
These annotations bind request data to method parameters.
- `@RequestBody`: Binds the HTTP request body to a method parameter. Typically used with `POST`, `PUT`, or `PATCH` requests to deserialize JSON/XML into Java objects.
   ```java
   @PostMapping
   public Product createProduct(@RequestBody Product product) { ... }
   ```
- `@ResponseBody`: Indicates that the return value of a method should be bound directly to the web response body. With `@RestController`, it's implicitly applied to all methods.
   ```java
   @GetMapping("/{id}")
   @ResponseBody // Not needed with @RestController
   public Product getProductById(@PathVariable Long id) { ... }
   ```
- `@RequestParam`: Binds a web request parameter (from the URL query string) to a method parameter.
   ```java   
   @GetMapping("/search")
   public List<Product> searchProducts(@RequestParam(required = false) String name,
                                   @RequestParam(defaultValue = "0") int minPrice) {
    // ...
   }
  ```

- `@PathVariable`: Binds a URI template variable (from the path itself) to a method parameter.
   ```java
   @GetMapping("/{id}")
   public Product getProductById(@PathVariable("id") Long productId) { ... }
   ```
- `@RequestHeader`: Binds a specific HTTP header to a method parameter.
   ```java
   @GetMapping("/info")
   public String getInfo(@RequestHeader("User-Agent") String userAgent) { ... }
   ```
- `@CookieValue`: Binds an HTTP cookie to a method parameter.
   ```java
   @GetMapping("/cookie-test")
   public String getCookie(@CookieValue(value = "myCookie", defaultValue = "default") String cookieValue) { ... }
   ```
- `@ModelAttribute`: Binds a method parameter or method return value to a named model attribute, exposed to a web view. Often used in traditional MVC. Can also be used to bind request parameters to an object.
   ```java
   // For form binding
   @PostMapping("/submit-form")
   public String submitForm(@ModelAttribute("myForm") MyFormData formData, Model model) { ... }

   // For adding data to model
   @ModelAttribute("productTypes")
   public List<String> populateProductTypes() {
     return Arrays.asList("Electronics", "Books");
   }
  ```

**Common Use Cases**: Handling diverse types of incoming data for web endpoints.

**Best Practices**: Use specific annotations (`@RequestParam`, `@PathVariable`, `@RequestBody`) for clarity and type safety.

### Response and Exception Handling Annotations
These annotations help control the HTTP response status and handle exceptions.
- `@ResponseStatus`: Marks a method or exception class with a specific HTTP status code that should be returned.
   ```java
   @PostMapping
   @ResponseStatus(HttpStatus.CREATED) // Returns 201 Created on success
   public Product createProduct(@RequestBody Product product) { ... }
   
   @ResponseStatus(HttpStatus.NOT_FOUND) // Can also be on custom exception class
   public class ProductNotFoundException extends RuntimeException { ... }
   ```
- `@ExceptionHandler`: Marks a method within a `@Controller` or `@ControllerAdvice` to handle specific exceptions.
   ```java
   @RestControllerAdvice // Or @Controller
   public class GlobalExceptionHandler {

      @ExceptionHandler(ProductNotFoundException.class)
      @ResponseStatus(HttpStatus.NOT_FOUND)
      public ErrorResponse handleProductNotFound(ProductNotFoundException ex) {
        return new ErrorResponse("Product not found", ex.getMessage());
      }

      @ExceptionHandler(IllegalArgumentException.class)
      @ResponseStatus(HttpStatus.BAD_REQUEST)
      public String handleIllegalArgument(IllegalArgumentException ex) {
        return "Invalid input: " + ex.getMessage();
      }
   }
  ```
- `@ControllerAdvice`: A specialization of `@Component` that allows you to apply global configurations to controllers, such as global exception handling (`@ExceptionHandler`), global `@ModelAttribute` methods, or global `@InitBinder` methods.

**Common Use Case**: Standardizing error responses, preventing boilerplate try-catch blocks in controllers.

**Best Practices**: Use `@ControllerAdvice` for global exception handling. Create custom exception classes with `@ResponseStatus` for common application-specific errors.

### `@CrossOrigin` for CORS
Used to enable Cross-Origin Resource Sharing (CORS) on a handler method or on a controller class.

#### Real-world Example:
```java
// src/main/java/com/example/myapp/controller/CorsController.java
package com.example.myapp.controller;

import org.springframework.web.bind.annotation.CrossOrigin;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@CrossOrigin(origins = "http://localhost:3000") // Allow requests from a specific origin
public class CorsController {

    @GetMapping("/cors-data")
    public String getCorsData() {
        return "Data accessible from localhost:3000";
    }
}
```

You can also use `@CrossOrigin(origins = "*")` for all origins (not recommended for production), or specify multiple origins.

**Common Use Case**: Enabling frontend applications (running on a different domain/port) to consume APIs from your Spring Boot backend. 

**Best Practices**: Be specific with origins for security reasons. Configure CORS globally for broad policies or per controller/method for fine-grained control.

## Security Annotations (Spring Security)
Spring Security provides powerful annotations for securing your application at the web, method, and domain object levels.
### Core Security Annotations
- `@EnableWebSecurity`: Used on a `@Configuration` class to enable Spring Security's web security features. It imports the necessary Spring Security configuration.
   ```java
   // src/main/java/com/example/myapp/config/SecurityConfig.java
    package com.example.myapp.config;

    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.security.config.annotation.web.builders.HttpSecurity;
    import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
    import org.springframework.security.web.SecurityFilterChain;

    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

       @Bean
       public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
          http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .permitAll()
            )
            .logout(logout -> logout
                .permitAll());
         return http.build();
       }
    }
  ```

- `@EnableGlobalMethodSecurity` **(Deprecated - Use** `@EnableMethodSecurity`**)**: Used on a `@Configuration` class to enable Spring Security's method-level security. It allows you to use annotations like `@PreAuthorize`, `@PostAuthorize`, `@Secured`, and `@RolesAllowed` on methods.
   ```java
   Note: As of Spring Security 5.6+, `@EnableGlobalMethodSecurity` is deprecated in favor of `@EnableMethodSecurity`. The new annotation is recommended for new projects.
   // src/main/java/com/example/myapp/config/MethodSecurityConfig.java
   package com.example.myapp.config;

   import org.springframework.context.annotation.Configuration;
   import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;

   @Configuration
   @EnableMethodSecurity(prePostEnabled = true, securedEnabled = true, jsr250Enabled = true)
   public class MethodSecurityConfig {
      // No additional code here, just enabling method security
   }
   ```

-  `prePostEnabled = true`: Enables `@PreAuthorize` and `@PostAuthorize`.
-  `securedEnabled = true`: Enables `@Secured`.
- `jsr250Enabled = true`: Enables `@RolesAllowed`.

### Method Security Annotations
These control access to methods based on roles or expressions.
- `@PreAuthorize`: Evaluates a Spring Expression Language (SpEL) expression before entering the method.
   ```java
   @Service
   public class AdminService {
      @PreAuthorize("hasRole('ADMIN')")
      public String deleteUser(String userId) {
        return "User " + userId + " deleted by ADMIN.";
      }

      @PreAuthorize("hasAuthority('USER_READ') and #username == authentication.name")
      public String getUserProfile(String username) {
        return "Profile for " + username;
      }
   }
  ```

- `@PostAuthorize`: Evaluates a SpEL expression after the method has been invoked, typically used to filter or modify the return value.
   ```java
   @Service
   public class PostService {
      @PostAuthorize("returnObject.owner == authentication.name")
      public Post getPostById(Long id) {
        // Returns a Post object
        return new Post(id, "Some Title", "content", "user123");
      }
   }
  ```

- `@Secured`: A simpler annotation for role-based access control. Takes a list of role names.
   ```java
   @Service
   public class ReportingService {
     @Secured({"ROLE_ADMIN", "ROLE_REPORT_VIEWER"})
     public String generateReport() {
        return "Report generated.";
     }
   }
  ```

- `@RolesAllowed`: JSR-250 annotation, similar to @Secured, also for role-based access control.
   ```java
   import jakarta.annotation.security.RolesAllowed; // or javax.annotation.security.RolesAllowed

   @Service
   public class AuditingService {
      @RolesAllowed("ADMIN")
      public String auditSystem() {
         return "System audited.";
      }
   }
  ```

### Differences and Use Cases:
- `@PreAuthorize` / `@PostAuthorize`: Most powerful due to SpEL, allowing complex authorization logic (e.g., checking ownership, custom permissions). Preferred for new development.
- `@Secured` / `@RolesAllowed`: Simpler, role-based checks. `@Secured` is Spring-specific, `@RolesAllowed` is a standard.

### User Context and Testing Annotations
- `@AuthenticationPrincipal`: Used in `@Controller` or `@RestController` methods to directly access the currently authenticated principal (user details).
   ```java
   import org.springframework.security.core.annotation.AuthenticationPrincipal;
   import org.springframework.security.core.userdetails.UserDetails;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RestController;

   @RestController
   public class UserProfileController {

      @GetMapping("/my-profile")
      public String getMyProfile(@AuthenticationPrincipal UserDetails userDetails) {
         return "Welcome, " + userDetails.getUsername() + "! Roles: " + userDetails.getAuthorities();
      }
   }
  ```

- `@WithMockUser`: For testing Spring Security. Allows you to run a test method as if a specific user (with roles/authorities) is authenticated.
   ```java
   // src/test/java/com/example/myapp/service/AdminServiceTest.java
   package com.example.myapp.service;

   import org.junit.jupiter.api.Test;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.boot.test.context.SpringBootTest;
   import org.springframework.security.access.AccessDeniedException;
   import org.springframework.security.test.context.support.WithMockUser;

   import static org.junit.jupiter.api.Assertions.assertThrows;
   import static org.junit.jupiter.api.Assertions.assertTrue;

   @SpringBootTest
   public class AdminServiceTest {

       @Autowired
       private AdminService adminService;

       @Test
       @WithMockUser(roles = "ADMIN")
       void testDeleteUser_AsAdmin() {
          String result = adminService.deleteUser("user123");
          assertTrue(result.contains("deleted by ADMIN"));
       }

       @Test
       @WithMockUser(roles = "USER")
       void testDeleteUser_AsUser_AccessDenied() {
          assertThrows(AccessDeniedException.class, () -> adminService.deleteUser("user123"));
       }
   }
  ```

- `@PermitAll`: JSR-250 annotation. Indicates that all authenticated users are allowed to access the method.
- `@DenyAll`: JSR-250 annotation. Indicates that no authenticated user is allowed to access the method.
- `@Anonymous`: Spring Security specific. Allows unauthenticated access. (Less common for method security directly, more for endpoint configuration).

### Best Practices:
- Use method security for fine-grained authorization logic.
- Prefer `@PreAuthorize` for most use cases due to its flexibility.
- Thoroughly test security rules using `@WithMockUser` or `SecurityMockMvcRequestBuilders`.
- Don't mix `@Secured` and `@PreAuthorize` on the same method. Choose one style.

## Persistence Annotations (Spring Data JPA)
These annotations are primarily from the Java Persistence API (JPA) specification and are used with Spring Data JPA to define and map entities to a relational database.
### Entity Mapping
- `@Entity`: Declares the class as an entity bean, representing a table in the database.
- `@Table`: Specifies the primary table for the annotated entity. Can define table name, schema, unique constraints.
- `@Id`: Marks a field as the primary key of the entity.
- `@GeneratedValue`: Specifies how the primary key value is generated.
  - `strategy = GenerationType.AUTO`: Let the persistence provider choose.
  - `strategy = GenerationType.IDENTITY`: Uses database identity columns (e.g., auto-increment).
  - `strategy = GenerationType.SEQUENCE`: Uses a database sequence.
  - `strategy = GenerationType.TABLE`: Uses a separate table to store sequence values.

#### Real-world Example:
```java
// src/main/java/com/example/myapp/entity/Product.java
package com.example.myapp.entity;

import jakarta.persistence.*; // Or javax.persistence for older JPA versions

@Entity
@Table(name = "products") // Maps to 'products' table
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Auto-incrementing ID
    private Long id;

    @Column(name = "product_name", nullable = false, length = 100)
    private String name;

    @Column(precision = 10, scale = 2) // Total 10 digits, 2 after decimal
    private Double price;

    // Constructors, getters, setters
    public Product() {}

    public Product(Long id, String name, Double price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Double getPrice() { return price; }
    public void setPrice(Double price) { this.price = price; }
}
```

### Column and Relationship Mapping
- `@Column`: Specifies the mapped column for a persistent property or field. Allows defining column name, nullability, length, etc.
- `@JoinColumn`: Used in a many-to-one or one-to-one relationship to specify the foreign key column.
- `@ManyToOne`: Defines a many-to-one relationship.
- `@OneToMany`: Defines a one-to-many relationship.
- `@OneToOne`: Defines a one-to-one relationship.
- `@ManyToMany`: Defines a many-to-many relationship.

#### Real-world Example (Relationships):
```java
// src/main/java/com/example/myapp/entity/Category.java
package com.example.myapp.entity;

import jakarta.persistence.*;
import java.util.List;
import java.util.ArrayList;

@Entity
public class Category {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(mappedBy = "category", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Product> products = new ArrayList<>();

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public List<Product> getProducts() { return products; }
    public void setProducts(List<Product> products) { this.products = products; }
}

```

```java
// Modify Product.java to include ManyToOne relationship
// src/main/java/com/example/myapp/entity/Product.java
// ...
@ManyToOne(fetch = FetchType.LAZY) // Many products to one category
@JoinColumn(name = "category_id") // Foreign key column in 'products' table
private Category category;
// ...
// Add getter and setter for category
public Category getCategory() { return category; }
public void setCategory(Category category) { this.category = category; }
```

### Other Mapping Annotations
- `@Enumerated`: Specifies that a persistent property or field should be persisted as an enumerated type.
  - `EnumType.ORDINAL`: Stores the enum's ordinal value (integer).
  - `EnumType.STRING`: Stores the enum's name (string). (Recommended for readability and robustness)
- `@Temporal`: Specifies the type of the date/time field.
  - `TemporalType.DATE`: Only the date part.
  - `TemporalType.TIME`: Only the time part.
  - `TemporalType.TIMESTAMP`: Both date and time.
- `@Transient`: Marks a field that should not be persisted to the database.

#### Real-world Example:
```java
// src/main/java/com/example/myapp/entity/Order.java
package com.example.myapp.entity;

import jakarta.persistence.*;
import java.time.LocalDate;
import java.time.LocalDateTime;

enum OrderStatus {
    PENDING, PROCESSING, COMPLETED, CANCELLED
}

@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING) // Store enum as String (e.g., "PENDING")
    private OrderStatus status;

    @Temporal(TemporalType.DATE) // Only date (e.g., 2025-06-30)
    private LocalDate orderDate;

    @Temporal(TemporalType.TIMESTAMP) // Date and time (e.g., 2025-06-30 14:30:00)
    private LocalDateTime lastUpdated;

    @Transient // This field will not be persisted
    private String transientField;

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; }
    public LocalDate getOrderDate() { return orderDate; }
    public void setOrderDate(LocalDate orderDate) { this.orderDate = orderDate; }
    public LocalDateTime getLastUpdated() { return lastUpdated; }
    public void setLastUpdated(LocalDateTime lastUpdated) { this.lastUpdated = lastUpdated; }
    public String getTransientField() { return transientField; }
    public void setTransientField(String transientField) { this.transientField = transientField; }
}
```

### Spring Data JPA Specific Annotations
- `@Query`: Used on methods in a Spring Data JPA `Repository` interface to define custom queries using JPQL (Java Persistence Query Language) or native SQL.
   ```java
   // src/main/java/com/example/myapp/repository/ProductRepository.java
   package com.example.myapp.repository;

   import com.example.myapp.entity.Product;
   import org.springframework.data.jpa.repository.JpaRepository;
   import org.springframework.data.jpa.repository.Query;
   import org.springframework.data.repository.query.Param;
   import java.util.List;

   public interface ProductRepository extends JpaRepository<Product, Long> {

      List<Product> findByNameContainingIgnoreCase(String name); // Derived query

      @Query("SELECT p FROM Product p WHERE p.price < :maxPrice") // JPQL
      List<Product> findProductsBelowPrice(@Param("maxPrice") Double maxPrice);

      @Query(value = "SELECT * FROM products WHERE product_name LIKE %:name%", nativeQuery = true) // Native SQL
      List<Product> findProductsByNameNative(@Param("name") String name);
   }
  ```

- `@Modifying`: Used with `@Query` to indicate that a query is an update or delete operation, requiring a transaction.
  ```java
   import org.springframework.data.jpa.repository.Modifying;
   import org.springframework.data.jpa.repository.Query;
   import org.springframework.transaction.annotation.Transactional;

   public interface ProductRepository extends JpaRepository<Product, Long> {

      @Modifying // Indicates this is an update/delete query
      @Transactional // Required for modifying operations
      @Query("UPDATE Product p SET p.price = :newPrice WHERE p.id = :id")
      int updateProductPrice(@Param("id") Long id, @Param("newPrice") Double newPrice);
   }
  ```

- `@Transactional`: (Already discussed, but crucial here) Ensures that a method executes within a database transaction. Essential for operations that modify data (create, update, delete).

**Common Use Case**: Defining data models, performing CRUD operations, and writing custom database queries.

### Best Practices:
- Use derived query methods in Spring Data JPA when possible (e.g., `findByLastNameAndFirstName`).
- Use `@Query` for more complex queries that derived methods can't express.
- Prefer JPQL over native SQL for better portability and type safety, but use native SQL when necessary for database-specific features or performance.
- Always use `@Transactional` on service methods that involve data modification.

---

##  Transactional and Async Behavior
These annotations manage atomic operations and asynchronous method execution.

### `@Transactional`
Ensures that a method or a class executes within a transactional context. If an unchecked exception occurs, the transaction is rolled back. If it's a checked exception (by default), it's committed.
#### Key attributes:
- `propagation`: Defines how transactional boundaries are managed (e.g., `REQUIRED` (default), `REQUIRES_NEW`, `SUPPORTS`, `NOT_SUPPORTED`, `NEVER`, `MANDATORY`, `NESTED`).
- `isolation`: Defines the isolation level of the transaction (e.g., `DEFAULT`, `READ_UNCOMMITTED`, `READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE`).
- `rollbackFor` / `noRollbackFor`: Specifies exceptions that should or should not cause a rollback.
- `readOnly`: Optimizes read-only transactions (e.g., by not flushing the session). 

#### Real-world Example:
```java
// src/main/java/com/example/myapp/service/OrderService.java
package com.example.myapp.service;

import com.example.myapp.entity.Order;
import com.example.myapp.repository.OrderRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private final OrderRepository orderRepository;

    @Autowired
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional(propagation = Propagation.REQUIRED, isolation = Isolation.READ_COMMITTED)
    public Order createOrder(Order order) {
        System.out.println("Creating order: " + order.getId());
        // Simulate some business logic
        if (order.getId() == null) {
            throw new IllegalArgumentException("Order ID cannot be null.");
        }
        return orderRepository.save(order);
    }

    @Transactional(readOnly = true) // Optimize for read operations
    public Order getOrderById(Long id) {
        return orderRepository.findById(id).orElse(null);
    }

    @Transactional(rollbackFor = CustomBusinessException.class)
    public void processOrderWithCustomException(Long orderId) throws CustomBusinessException {
        // ... some logic
        if (true) { // Simulate an error
            throw new CustomBusinessException("Business rule violation!");
        }
        // ... more logic (will not be reached if exception is thrown)
    }
}
```

**Common Use Case**: Ensuring data consistency for database operations.

**Best Practices**:
- Apply `@Transactional` at the service layer, not directly on repositories or controllers.
- Keep transactional methods concise and focused on a single business operation.
- Understand `propagation` behavior to avoid unexpected transaction outcomes.
- For read-only operations, set `readOnly = true` for performance benefits.

## `@Async`
Marks a method for asynchronous execution. When an `@Async` method is called, it executes in a separate thread from the caller, allowing the caller to proceed without waiting for the method to complete.
- `@EnableAsync`: Must be placed on a `@Configuration` class to enable Spring's asynchronous method execution capability.

#### Real-world Example:
```java
// src/main/java/com/example/myapp/config/AsyncConfig.java
package com.example.myapp.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;

@Configuration
@EnableAsync
public class AsyncConfig {
    // This class enables async support
}
```

```java
// src/main/java/com/example/myapp/service/NotificationService.java
package com.example.myapp.service;

import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
public class NotificationService {

    @Async
    public CompletableFuture<String> sendEmailAsync(String to, String subject, String body) {
        System.out.println("Sending email in separate thread: " + Thread.currentThread().getName());
        try {
            Thread.sleep(2000); // Simulate long-running task
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        String result = "Email sent to " + to + " with subject: " + subject;
        System.out.println(result + " (Thread: " + Thread.currentThread().getName() + ")");
        return CompletableFuture.completedFuture(result);
    }

    @Async
    public void logActivityAsync(String activity) {
        System.out.println("Logging activity asynchronously: " + activity + " (Thread: " + Thread.currentThread().getName() + ")");
        // Simulate database write or file I/O
    }
}
```

```java
// How to call it from a controller or another service
// In a Controller or Service
import com.example.myapp.service.NotificationService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

@RestController
public class AsyncController {

    private final NotificationService notificationService;

    @Autowired
    public AsyncController(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @GetMapping("/send-notification")
    public String triggerNotification() throws ExecutionException, InterruptedException {
        System.out.println("Main thread: " + Thread.currentThread().getName() + " - Triggering notification.");
        CompletableFuture<String> emailResult = notificationService.sendEmailAsync("test@example.com", "Async Test", "This is an async email.");
        notificationService.logActivityAsync("User logged in");
        System.out.println("Main thread: " + Thread.currentThread().getName() + " - Notification triggered, continuing...");

        // You can get the result later if needed
        String result = emailResult.get(); // This will block until the email is sent
        return "Notification process initiated. Email result: " + result;
    }
}
```

**Common Use Case**: Improving responsiveness for long-running operations (e.g., sending emails, processing files, calling external APIs) by offloading them from the main request thread.
**Best Practices**:
- Methods annotated with `@Async` must be `public`.
- Self-invocation (calling an `@Async` method from within the same class) will not work because the proxy is not invoked. You need to inject the service into itself or a separate bean.
- Return `void` or `Future`/`CompletableFuture` for asynchronous methods.
- Handle exceptions in asynchronous methods carefully, as they won't propagate back to the caller's thread directly. Use `AsyncUncaughtExceptionHandler`.

---

## Validation Annotations
Used for validating data, especially incoming request bodies in web applications.
### Bean Validation (JSR 380/JSR 303)
These are standard Java annotations for defining validation rules.
- `@NotNull`: The annotated element must not be `null`.
- `@NotEmpty`: The annotated element must not be ``null`` or empty (for collections, maps, arrays, strings).
- `@NotBlank`: The annotated string must not be null and must contain at least one non-whitespace character.
- `@Size(min, max)`: The size of the annotated element must be between `min` and `max` (inclusive).
- `@Min(value)`: The annotated value must be greater than or equal to the specified minimum.
- `@Max(value)`: The annotated value must be less than or equal to the specified maximum.
- `@Email`: The annotated string must be a valid email address.
- `@Pattern(regexp)`: The annotated string must match the specified regular expression.
- `@Future` / `@FutureOrPresent` / `@Past` / `@PastOrPresent`: For dates.
- `@Positive` / `@PositiveOrZero` / `@Negative` / `@NegativeOrZero`: For numbers.

### `@Validated`, `@Valid` for Request Validation
These annotations trigger the Bean Validation process.
- `@Valid`: Triggers validation of an object (e.g., a request body). It's a standard JSR-380 annotation.
- `@Validated`: A Spring-specific annotation that can be used on classes or method parameters to enable validation. It also supports validation groups and integration with Spring's AOP.

#### Real-world Example:
```java
// src/main/java/com/example/myapp/model/UserRegistrationForm.java
package com.example.myapp.model;

import jakarta.validation.constraints.*; // Or javax.validation.constraints for older versions

public class UserRegistrationForm {

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 20, message = "Username must be between 3 and 20 characters")
    private String username;

    @NotBlank(message = "Password is required")
    @Size(min = 6, message = "Password must be at least 6 characters long")
    private String password;

    @Email(message = "Invalid email format")
    @NotBlank(message = "Email is required")
    private String email;

    @Min(value = 18, message = "You must be at least 18 years old")
    private int age;

    // Getters and Setters
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getPassword() { return password; }
    public void String setPassword(String password) { this.password = password; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}
```

```java
// src/main/java/com/example/myapp/controller/UserRegistrationController.java
package com.example.myapp.controller;

import com.example.myapp.model.UserRegistrationForm;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserRegistrationController {

    @PostMapping("/register")
    public ResponseEntity<String> registerUser(@Valid @RequestBody UserRegistrationForm registrationForm,
                                               BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            return ResponseEntity.badRequest().body("Validation errors: " + bindingResult.getAllErrors());
        }
        // Process registrationForm if validation passes
        return ResponseEntity.status(HttpStatus.CREATED).body("User registered successfully: " + registrationForm.getUsername());
    }
}
```

**Common Use Case**: Validating user input from web forms or REST API requests to ensure data integrity.

**Best Practices**:
- Use Bean Validation annotations on your DTOs (Data Transfer Objects) or domain models.
- Use `@Valid` on the method parameter to trigger validation.
- For REST APIs, consider using `@ControllerAdvice` and `@ExceptionHandler` to provide standardized error responses for `MethodArgumentNotValidException` (thrown when `@Valid` fails).

---

## Configuration and Conditional Beans
These annotations help in externalizing configuration and importing other configuration sources.

`@EnableConfigurationProperties`, `@ConfigurationProperties`, `@ConstructorBinding`

These annotations provide a type-safe way to bind external properties (e.g., from `application.properties` or `application.yml`) to Java objects.
- `@ConfigurationProperties`: Binds a set of related properties to a Java object.
- `@EnableConfigurationProperties`: Used on a `@Configuration` class to enable `@ConfigurationProperties` support for one or more classes.
- `@ConstructorBinding`: Allows `@ConfigurationProperties` to bind properties to immutable objects via constructor arguments (since Spring Boot 2.2).

#### Real-world Example:
```java
// src/main/resources/application.properties
myapp.service.url=http://api.example.com
myapp.service.username=user
myapp.service.password=pass
myapp.service.retries=3
```

```java
// src/main/java/com/example/myapp/config/ServiceProperties.java
package com.example.myapp.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.ConstructorBinding;

// For constructor binding, either use @ConstructorBinding or enable immutable binding globally
// If using older Spring Boot or don't want global immutability, use standard setters.
@ConfigurationProperties(prefix = "myapp.service")
// @ConstructorBinding // Can be placed here for immutable binding per class
public class ServiceProperties {

    private String url;
    private String username;
    private String password;
    private int retries;

    // If using @ConstructorBinding, constructor parameters must match property names
    // @ConstructorBinding // If placed here, requires all fields to be in constructor
    public ServiceProperties(String url, String username, String password, int retries) {
        this.url = url;
        this.username = username;
        this.password = password;
        this.retries = retries;
    }

    // Standard getters (setters are not needed if using @ConstructorBinding for immutable object)
    public String getUrl() { return url; }
    public String getUsername() { return username; }
    public String getPassword() { return password; }
    public int getRetries() { return retries; }

    // If not using @ConstructorBinding, provide setters for mutable binding
    /*
    public void setUrl(String url) { this.url = url; }
    public void setUsername(String username) { this.username = username; }
    public void setPassword(String password) { this.password = password; }
    public void setRetries(int retries) { this.retries = retries; }
    */
}
```

```java
// src/main/java/com/example/myapp/config/AppConfiguration.java
package com.example.myapp.config;

import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties(ServiceProperties.class) // Enable binding for ServiceProperties
public class AppConfiguration {
    // You can now inject ServiceProperties into any bean
}

// In a service that uses these properties
import com.example.myapp.config.ServiceProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class ExternalApiService {

    private final ServiceProperties serviceProperties;

    @Autowired
    public ExternalApiService(ServiceProperties serviceProperties) {
        this.serviceProperties = serviceProperties;
    }

    public void callExternalApi() {
        System.out.println("Calling API: " + serviceProperties.getUrl());
        System.out.println("With username: " + serviceProperties.getUsername());
        System.out.println("Retries: " + serviceProperties.getRetries());
    }
}
```

**Common Use Case**: Creating strongly typed configuration objects, especially when dealing with many related properties.

**Best Practices**: Prefer `@ConfigurationProperties` over multiple `@Value` annotations for better organization, type safety, and IDE support. Use `@ConstructorBinding` for immutable configuration objects.

### `@PropertySource`
Used to specify the location of additional property files to be loaded into the Spring `Environment`.

#### Real-world Example:
```java
// src/main/resources/custom.properties
custom.message=Hello from custom properties!

// src/main/java/com/example/myapp/config/CustomPropertyConfig.java
package com.example.myapp.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Configuration
@PropertySource("classpath:custom.properties") // Load custom.properties from classpath
public class CustomPropertyConfig {

    @Value("${custom.message}")
    private String message;

    @Bean
    public String customMessageBean() {
        return message;
    }
}
```

**Common Use Case**: Loading properties from files other than `application.properties` (e.g., module-specific configurations, sensitive data files not bundled by default).

**Best Practices**: Only use for additional property files. For the primary application configuration, `application.properties`/`yml` is automatically loaded.

### `@Import`, `@ImportResource`
Used to import other `@Configuration` classes or XML configuration files.
- `@Import`: Imports one or more `@Configuration` classes, `@ImportSelector`, or `ImportBeanDefinitionRegistrar`.
- `@ImportResource`: Imports Spring XML configuration files.

#### Real-world Example (`@Import`):
```java
// src/main/java/com/example/myapp/config/ServiceConfig.java
package com.example.myapp.config;

import com.example.myapp.service.NotificationService;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ServiceConfig {
    @Bean
    public NotificationService notificationService() {
        return new NotificationService();
    }
}

// src/main/java/com/example/myapp/config/AppConfig.java
package com.example.myapp.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;

@Configuration
@Import(ServiceConfig.class) // Imports beans defined in ServiceConfig
public class AppConfig {
    // All beans from ServiceConfig are now available
}
```

#### Real-world Example (`@ImportResource`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="legacyProcessor" class="com.example.myapp.component.LegacyProcessor"/>
</beans>
```

```java
// src/main/java/com/example/myapp/config/LegacyConfig.java
package com.example.myapp.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.ImportResource;

@Configuration
@ImportResource("classpath:legacy-beans.xml") // Imports beans from XML
public class LegacyConfig {
    // legacyProcessor bean is now available
}
```

**Common Use Case**: Structuring configurations into modular units, or integrating with legacy Spring applications that still use XML configuration.

**Best Practices**: Prefer `@Import` for pure Java configurations. Use `@ImportResource` only when necessary for backward compatibility.

--- 

## AOP and Eventing Annotations
These annotations enable Aspect-Oriented Programming (AOP) and event-driven communication within your application.

### AOP Annotations
Spring AOP allows you to modularize cross-cutting concerns (e.g., logging, security, transaction management) by defining "aspects."
- `@Aspect`: Declares a class as an aspect. Requires AspectJ on the classpath and `@EnableAspectJAutoProxy` on a configuration.
- `@Before`: Advice that executes before a join point.
- `@After`: Advice that executes after a join point (whether it returns normally or throws an exception).
- `@Around`: Advice that completely surrounds a join point, allowing you to perform actions before and after the method execution, and even to prevent the method from executing.
- `@AfterReturning`: Advice that executes after a join point completes successfully (returns normally).
- `@AfterThrowing`: Advice that executes after a join point throws an exception.

### Real-world Example (Logging Aspect):
```java
// src/main/java/com/example/myapp/aspect/LoggingAspect.java
package com.example.myapp.aspect;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Aspect // Marks this class as an Aspect
@Component
public class LoggingAspect {

    // Define a pointcut for all methods in service package
    @Pointcut("execution(* com.example.myapp.service.*.*(..))")
    public void serviceMethods() {}

    @Before("serviceMethods()")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Before: " + joinPoint.getSignature().getName() + " called.");
    }

    @AfterReturning(pointcut = "serviceMethods()", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        System.out.println("After Returning: " + joinPoint.getSignature().getName() + " returned " + result);
    }

    @AfterThrowing(pointcut = "serviceMethods()", throwing = "error")
    public void logAfterThrowing(JoinPoint joinPoint, Throwable error) {
        System.out.println("After Throwing: " + joinPoint.getSignature().getName() + " threw " + error.getMessage());
    }

    @Around("serviceMethods()")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        System.out.println("Around Before: " + joinPoint.getSignature().getName() + " started.");
        Object result = joinPoint.proceed(); // Execute the actual method
        long end = System.currentTimeMillis();
        System.out.println("Around After: " + joinPoint.getSignature().getName() + " executed in " + (end - start) + "ms.");
        return result;
    }
}
```

To enable AOP in a Spring Boot application, ensure you have `spring-boot-starter-aop` dependency. This typically enables `@EnableAspectJAutoProxy` automatically.

**Common Use Case**: Logging, performance monitoring, security checks, transaction management (though Spring's `@Transactional` often handles this via its own AOP).

**Best Practices**: Keep aspects focused on single cross-cutting concerns. Define pointcuts precisely to avoid unintended advice execution.

### Eventing Annotations
Spring's eventing mechanism allows for loose coupling between components.
- `@EventListener`: Marks a method as an event listener. It will be invoked when an application event of the specified type is published.
- `ApplicationEvent`: The base class for all application events. You can create custom event classes by extending this.
- `ApplicationListener`: An interface for event listeners (older, functional alternative to @EventListener).

#### Real-world Example:
```java
// src/main/java/com/example/myapp/event/ProductCreatedEvent.java
package com.example.myapp.event;

import org.springframework.context.ApplicationEvent;
import com.example.myapp.entity.Product;

public class ProductCreatedEvent extends ApplicationEvent {
    private final Product product;

    public ProductCreatedEvent(Object source, Product product) {
        super(source);
        this.product = product;
    }

    public Product getProduct() {
        return product;
    }
}
```

```java
// src/main/java/com/example/myapp/service/ProductManagementService.java
package com.example.myapp.service;

import com.example.myapp.entity.Product;
import com.example.myapp.event.ProductCreatedEvent;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

@Service
public class ProductManagementService {

    private final ApplicationEventPublisher eventPublisher;

    @Autowired
    public ProductManagementService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public Product createNewProduct(Product product) {
        // ... save product to database ...
        System.out.println("Product created in service: " + product.getName());
        eventPublisher.publishEvent(new ProductCreatedEvent(this, product)); // Publish event
        return product;
    }
}
```

```java
// src/main/java/com/example/myapp/listener/ProductEventListener.java
package com.example.myapp.listener;

import com.example.myapp.event.ProductCreatedEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class ProductEventListener {

    @EventListener
    public void handleProductCreatedEvent(ProductCreatedEvent event) {
        System.out.println("ProductEventListener: Received ProductCreatedEvent for product: " + event.getProduct().getName());
        // Perform actions like sending notification, updating cache, etc.
    }
}
```

**Common Use Case**: Decoupling components. When one component needs to inform others about a state change without direct dependencies. <br>
**Best Practices**:
- Keep events granular and focused.
- Use `@EventListener` for most cases.
- For complex scenarios, consider asynchronous event handling (e.g., `@Async` on event listener methods).

---
## Testing Annotations
Spring Boot provides a rich set of testing annotations to simplify testing various layers of your application.
- `@SpringBootTest`: The most comprehensive testing annotation. It loads the full Spring application context, making it suitable for integration tests.
  - webEnvironment` = SpringBootTest.WebEnvironment.RANDOM_PORT: Starts a web server on a random port.
  - webEnvironment` = SpringBootTest.WebEnvironment.MOCK: Provides a mock web environment without starting a real server.
  - classes = MyConfig.class: Specify a particular configuration class.
   ```java
   // src/test/java/com/example/myapp/IntegrationTest.java
   package com.example.myapp;

   import org.junit.jupiter.api.Test;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.boot.test.context.SpringBootTest;
   import org.springframework.boot.test.web.client.TestRestTemplate;
   import org.springframework.boot.test.web.server.LocalServerPort;

   import static org.assertj.core.api.Assertions.assertThat;

   @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
   public class IntegrationTest {

       @LocalServerPort
       private int port;

       @Autowired
       private TestRestTemplate restTemplate;

       @Test
       void contextLoads() {
          // Verifies that the Spring context loads successfully
        }

       @Test
       void greetingShouldReturnDefaultMessage() {
          assertThat(restTemplate.getForObject("http://localhost:" + port + "/hello", String.class))
                .contains("Hello, World!");
       }
   }
  ```

- `@WebMvcTest`: Focuses on testing Spring MVC controllers. It auto-configures Spring MVC components (like `DispatcherServlet`) but does not load the full application context. Ideal for unit/integration testing web layers without database or other backend dependencies.
  - Requires `MockMvc` for making requests.
  - Can specify which controller to test (e.g., `@WebMvcTest(MyController.class)`).
   ```java
   // src/test/java/com/example/myapp/controller/ProductRestControllerTest.java
    package com.example.myapp.controller;

    import com.example.myapp.model.Product;
    import com.fasterxml.jackson.databind.ObjectMapper;
    import org.junit.jupiter.api.Test;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
    import org.springframework.boot.test.mock.mockito.MockBean;
    import org.springframework.http.MediaType;
    import org.springframework.test.web.servlet.MockMvc;

    import static org.mockito.BDDMockito.given;
    import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
    import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
    import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
    import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

    @WebMvcTest(ProductRestController.class) // Test only this controller
    public class ProductRestControllerTest {

       @Autowired
       private MockMvc mockMvc;

       @Autowired
       private ObjectMapper objectMapper; // To convert objects to JSON

       // @MockBean will add mock instances of ProductService into the Spring context
       // This is crucial to isolate the controller test
       @MockBean
       private com.example.myapp.service.ProductService productService; // Assuming ProductRestController depends on this

       @Test
       void getProductById_ShouldReturnProduct() throws Exception {
          Product mockProduct = new Product(1L, "Test Product", 100.00);
          given(productService.getProductDetails(1L)).willReturn("Test Product"); // Mock the service call

          mockMvc.perform(get("/api/products/1")
                   .contentType(MediaType.APPLICATION_JSON))
                   .andExpect(status().isOk())
                   .andExpect(jsonPath("$.name").value("Test Product")); // Assert on JSON content
       }

       @Test
       void createProduct_ShouldReturnCreated() throws Exception {
          Product newProduct = new Product(null, "New Product", 50.00);
          String productJson = objectMapper.writeValueAsString(newProduct);

          mockMvc.perform(post("/api/products")
                   .contentType(MediaType.APPLICATION_JSON)
                   .content(productJson))
                   .andExpect(status().isCreated())
                   .andExpect(jsonPath("$.name").value("New Product"));
          }
    }
    ```

- `@DataJpaTest`: Focuses on testing JPA entities and repositories. It auto-configures an in-memory database and scans for `@Entity` and Spring Data JPA repositories. It's transactional by default and rolls back after each test.
   ```java   
    // src/test/java/com/example/myapp/repository/ProductRepositoryTest.java
    package com.example.myapp.repository;

    import com.example.myapp.entity.Product;
    import org.junit.jupiter.api.Test;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
    import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;

    import java.util.Optional;

    import static org.assertj.core.api.Assertions.assertThat;

    @DataJpaTest // Configures an in-memory database and JPA components
    public class ProductRepositoryTest {

        @Autowired
        private TestEntityManager entityManager; // Helper for persisting test data

        @Autowired
        private ProductRepository productRepository;

        @Test
        void findById_ShouldReturnProduct() {
           // Given
           Product product = new Product(null, "Laptop", 1200.00);
           entityManager.persistAndFlush(product); // Save to in-memory DB

           // When
           Optional<Product> foundProduct = productRepository.findById(product.getId());

           // Then
           assertThat(foundProduct).isPresent();
           assertThat(foundProduct.get().getName()).isEqualTo("Laptop");
        }

        @Test
        void findByNameContainingIgnoreCase_ShouldReturnMatchingProducts() {
           entityManager.persistAndFlush(new Product(null, "Laptop Pro", 1500.00));
           entityManager.persistAndFlush(new Product(null, "gaming laptop", 1800.00));

           assertThat(productRepository.findByNameContainingIgnoreCase("laptop")).hasSize(2);
        }
    }
   ```

- `@MockBean`: Adds a Mockito mock for a bean to the Spring application context. If a bean of the same type already exists, it will be replaced by the mock.
- `@SpyBean`: Adds a Mockito spy for a bean to the Spring application context. Unlike `@MockBean`, a spy calls real methods by default, allowing you to selectively mock specific methods while keeping the rest of the original functionality.
   ```java   
   // Example using @MockBean and @SpyBean
   // Assuming you have a real EmailService and a NotificationService that uses EmailService

   // In a test:
   @SpringBootTest
   public class SomeServiceTest {

       @MockBean
       private EmailService mockEmailService; // Replaces the real EmailService with a mock

       @SpyBean
       private NotificationService spyNotificationService; // Wraps the real NotificationService in a spy

       @Autowired
       private MyClientCallingService clientService; // Service that uses NotificationService

       @Test
       void testEmailSending() {
          // When clientService calls notificationService.sendEmail, it will use the mocked email service
          given(mockEmailService.sendEmail("to", "sub", "body")).willReturn(true); // Example mock
          clientService.sendEmailToUser("user@example.com");
          verify(mockEmailService, times(1)).sendEmail("user@example.com", "Welcome", "Hello");
       }

       @Test
       void testNotificationLogging() {
          // Call the real method of NotificationService but verify it
          clientService.triggerNotificationLogging("User Login");
          verify(spyNotificationService, times(1)).logActivityAsync("User Login");
          // You can also mock a specific method on the spy while other methods remain real
          // doReturn("Mocked async result").when(spyNotificationService).sendEmailAsync(any(), any(), any());
       }
   }
  ```

- `@TestConfiguration`: Used to define a nested `@Configuration` class specific to a test. Beans defined here will only be registered in the application context for that specific test.
- `@TestPropertySource`: Allows overriding or adding properties specifically for a test environment.
- `@AutoConfigureMockMvc`: Used with `@SpringBootTest` (with `WebEnvironment.MOCK`) or `@WebMvcTest` to auto-configure MockMvc, providing a way to make requests to controllers without starting a full HTTP server.

**Common Use Case**:
- `@SpringBootTest`: Full integration tests, verifying component interactions, end-to-end flows.
- `@WebMvcTest`: Testing web layer without touching backend services.
- `@DataJpaTest`: Testing persistence layer, ensuring correct entity mappings and repository behavior.
- `@MockBean` / `@SpyBean`: Isolating components for testing by mocking or spying on their dependencies.

**Best Practices**:
- Choose the most specific test slice annotation (`@WebMvcTest`, `@DataJpaTest`) to load only the necessary parts of the context, speeding up tests.
- Use `@SpringBootTest` when a broader context is required or for true end-to-end testing.
- Use `MockMvc` for testing web layers efficiently without real HTTP requests.
- Clearly distinguish between unit tests (isolated, no Spring context) and integration tests (Spring context loaded).

---

## Meta-annotations and Custom Annotations
Spring's annotation processing is powerful enough to allow you to create your own composite annotations, often called "meta-annotations."

### How to create custom annotations
To create your own annotation, you use `@Target`, `@Retention`, `@Documented`, and `@Inherited`.
- `@Target`: Specifies the contexts in which the annotation is applicable (e.g., `ElementType.TYPE`, `ElementType.METHOD`, `ElementType.FIELD`).
- `@Retention`: Specifies how long the annotation should be retained (e.g., `RetentionPolicy.RUNTIME` for annotations available at runtime via reflection).
- `@Documented`: Indicates that the annotated elements should be documented by `javadoc` and similar tools.
- `@Inherited`: Indicates that an annotation type is automatically inherited. If a class is annotated with it, its subclasses will also be annotated.

#### Example Custom Annotation:
```java
// src/main/java/com/example/myapp/annotations/Loggable.java
package com.example.myapp.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD) // Can be applied to methods
@Retention(RetentionPolicy.RUNTIME) // Available at runtime for processing
public @interface Loggable {
    String value() default "No message"; // An optional attribute
}
```

You could then create an AOP aspect to process `@Loggable` methods.

### Compose annotations (e.g., creating your own `@ServiceSecured`)
This is where meta-annotations become incredibly powerful in Spring. You can combine existing Spring annotations into a new custom annotation, reducing boilerplate and improving semantic clarity.

#### Real-world Example: `@ServiceSecured`
Let's say you frequently have service methods that are both `@Service` and `@Transactional` and need a specific security check.
```java
// src/main/java/com/example/myapp/annotations/ServiceSecured.java
package com.example.myapp.annotations;

import org.springframework.core.annotation.AliasFor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.security.access.prepost.PreAuthorize;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE) // Can be applied to classes
@Retention(RetentionPolicy.RUNTIME)
@Service // Meta-annotated with @Service
@Transactional // Meta-annotated with @Transactional
public @interface ServiceSecured {

    // You can expose attributes from the meta-annotations
    // This allows customizing @Transactional's propagation from @ServiceSecured
    @AliasFor(annotation = Transactional.class, attribute = "propagation")
    org.springframework.transaction.annotation.Propagation transactionalPropagation() default org.springframework.transaction.annotation.Propagation.REQUIRED;

    // You might also add security checks here if they are always the same,
    // or expose a @PreAuthorize expression. For simplicity, we'll assume
    // method-level @PreAuthorize will be used within the class.
    // If you wanted to apply a default @PreAuthorize to all methods of the class:
    // @AliasFor(annotation = PreAuthorize.class, attribute = "value")
    // String preAuthorizeValue() default "";
}
```

```java
// src/main/java/com/example/myapp/service/CustomerService.java
package com.example.myapp.service;

import com.example.myapp.annotations.ServiceSecured;
import org.springframework.security.access.prepost.PreAuthorize;

@ServiceSecured(transactionalPropagation = org.springframework.transaction.annotation.Propagation.REQUIRES_NEW)
public class CustomerService {

    @PreAuthorize("hasRole('ADMIN') or hasAuthority('CUSTOMER_WRITE')")
    public String createCustomer(String name) {
        System.out.println("Creating customer: " + name);
        // This method is now automatically transactional (REQUIRES_NEW) and a Spring service
        return "Customer " + name + " created.";
    }
}
```

**Common Use Case**: Creating domain-specific stereotypes, reducing redundancy, and improving readability in your codebase.

**Best Practices**:
- Only create custom composite annotations when you find recurring patterns of annotation usage.
- Use `@AliasFor` to expose and customize attributes of the meta-annotations.
- Ensure your custom annotations clearly convey their intent.

--- 

Best Practices & Pitfalls
Annotations, while powerful, can be misused.
Where annotations are commonly misused
 * Over-reliance on Field Injection for @Autowired: While convenient, it makes testing harder (can't easily instantiate without Spring), obscures dependencies, and makes circular dependencies harder to diagnose. Prefer constructor injection.
 * Missing @Transactional on methods that modify data: Leads to data inconsistency issues.
 * Broad @ComponentScan beyond the application's base package: Can slow down startup and pull in unintended beans.
 * Misunderstanding @Scope: Using prototype where singleton is sufficient, or conversely, using singleton for stateful beans that should not be shared.
 * Ignoring NoUniqueBeanDefinitionException and NoSuchBeanDefinitionException: These are critical errors indicating dependency resolution issues. Address them with @Qualifier, @Primary, or by reviewing your bean definitions.
 * Overuse of @Value for complex configurations: For multiple related properties, @ConfigurationProperties offers a type-safe and more structured approach.
 * Ignoring security implications of @CrossOrigin(origins = "*"): Too permissive in production, opens up security vulnerabilities.
Reflection cost and overuse
Spring's annotation processing heavily relies on Java Reflection. While modern JVMs and Spring's internal caching mechanisms make this overhead generally negligible for typical applications, extreme overuse or misuse can have a minor performance impact during startup.
Key considerations:
 * Startup Time: During application startup, Spring scans for and processes annotations. A very large number of beans with complex annotation configurations might slightly increase startup time.
 * Runtime Overhead: Once the application is up, the runtime overhead of already processed annotations is minimal. However, if you are frequently performing reflection-based operations (e.g., custom annotation processing at runtime without caching), that can add overhead.
In practice, for most Spring Boot applications, the benefits of annotations (readability, conciseness, maintainability) far outweigh the minimal reflection cost. Focus on good design and best practices, rather than obsessing over micro-optimizations related to annotation reflection.
Annotation-driven vs programmatic configuration
Spring offers both annotation-driven and programmatic (Java code-based) configuration.
 * Annotation-driven (@Configuration, @Bean, @Autowired, etc.):
   * Pros: Highly declarative, concise, easier to read for many developers, excellent IDE support, less boilerplate. This is the default and recommended approach in Spring Boot.
   * Cons: Can sometimes be less flexible for highly dynamic or conditional bean creation scenarios (though @Conditional annotations address many of these).
 * Programmatic Configuration (e.g., using BeanDefinitionRegistryPostProcessor, BeanFactoryPostProcessor, ApplicationContextAware):
   * Pros: Maximum flexibility, allows for dynamic bean registration, complex conditional logic, and fine-grained control over the bean lifecycle.
   * Cons: More verbose, harder to read, steeper learning curve, less common in typical Spring Boot applications unless building custom frameworks or highly dynamic systems.
Best Practice: Stick to annotation-driven configuration for the vast majority of your application. Reserve programmatic configuration for advanced scenarios where annotations alone cannot achieve the desired behavior.
Testing implications of annotations
Annotations significantly streamline testing in Spring Boot:
 * Reduced Setup: Test annotations like @SpringBootTest, @WebMvcTest, @DataJpaTest dramatically reduce the boilerplate code needed to set up specific parts of the application context for testing.
 * Focused Testing: Slice annotations (@WebMvcTest, @DataJpaTest) enable you to load only relevant parts of the application, making tests faster and more targeted.
 * Mocking/Spying: @MockBean and @SpyBean integrate seamlessly with Mockito, making it easy to isolate the component under test by replacing or observing its dependencies.
 * Security Testing: @WithMockUser simplifies testing security-protected endpoints and methods without needing a full authentication flow.
Best Practices for Testing with Annotations:
 * Use the Right Tool: Choose the most specific test annotation (e.g., @WebMvcTest for controllers) to minimize context loading and speed up tests.
 * Isolate Layers: Use @MockBean for dependencies outside the layer being tested.
 * Integration vs. Unit: Understand when to use a full @SpringBootTest (integration) versus a slice test (focused integration/unit).
 * Parameterized Tests: Combine annotations with JUnit 5's parameterized tests for testing various scenarios (e.g., different input validations).
By understanding and applying these annotations effectively, developers can build robust, maintainable, and scalable applications with Spring Boot.
