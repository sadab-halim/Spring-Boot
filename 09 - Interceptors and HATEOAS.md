# Interceptors
## **Module 1: Introduction and Core Concepts**
### **What is an Interceptor?**
In simple terms, an **Interceptor** is an object that automatically intercepts and modifies requests and responses without altering the core business logic of your application. It's a mechanism that allows you to run code *before* a primary method is invoked, *after* it completes, or both.

This is a core concept of **Aspect-Oriented Programming (AOP)**, which focuses on separating "cross-cutting concerns" from your main application logic. Cross-cutting concerns are functionalities required in many places but are not central to any single one of them—think logging, security checks, or data transformation.

> **Analogy: The Security Checkpoint**
>
> Imagine a high-security building where every visitor must go to a specific office (the "business logic"). Before anyone can reach their destination office, they must pass through a security checkpoint at the entrance.
>
> *   The **Visitor** is the incoming `request`.
> *   The **Office** is the `controller` or `service` method that does the actual work.
> *   The **Security Guard** is the `interceptor`.
>
> The guard's job isn't related to the work happening in any specific office. Instead, the guard performs a standard check on everyone—verifying credentials (authentication), checking their bag (validation), and logging their entry time (logging). This checkpoint "intercepts" the visitor's path to add a necessary, separate function. Similarly, an interceptor "catches" a request on its way to a controller to perform these kinds of reusable tasks.

### **Why Were Interceptors Created?**

Interceptors were created to solve a very common and frustrating problem in software development: **code scattering and tangling**.

Without interceptors, logic for common tasks like logging or authentication would have to be manually added to every single method that needs it.

* **Problem 1: Code Scattering:** The same lines of logging code are scattered across dozens or hundreds of different methods. If you need to change how logging works, you have to hunt down and modify every single instance.
* **Problem 2: Code Tangling:** Your core business logic (e.g., `calculateBilling()`) becomes tangled up with non-essential boilerplate code (e.g., `log.info("Starting billing calculation...")`, `if (!user.isAuthenticated())`). This makes the primary logic harder to read, test, and maintain.

**Interceptors solve this by centralizing cross-cutting concerns.** You write the logging or security logic once, in one place (the interceptor), and the framework automatically applies it wherever it's needed. This keeps your business logic clean, focused, and lean.

### **Core Architecture \& Philosophy**

The architecture of interceptors is typically based on the **"Chain of Responsibility"** and **"Inversion of Control (IoC)"** principles.

1. **High-Level Architecture (Chain of Responsibility):** When a request comes in, it doesn't go directly to the controller. Instead, it's passed to a chain of one or more interceptors.
    * The first interceptor in the chain does its work (e.g., checks for an authentication token).
    * If the check passes, it forwards the request to the next interceptor in the chain.
    * This continues until the last interceptor forwards the request to the actual target method (the controller).
    * After the controller finishes and produces a response, the response travels *back* through the chain in the reverse order, allowing each interceptor to process or modify the response (e.g., adding a header or logging the response time).
2. **Core Philosophy (Inversion of Control):** You don't manually call the interceptor from your business logic. Instead, you declare the interceptor and define which request paths it applies to. The framework (like Spring or ASP.NET) takes control and ensures the interceptor is invoked at the right time. This "inversion" of control is what decouples the cross-cutting concern from the business logic.

This architecture ensures that your components are **highly decoupled and reusable**, which is a cornerstone of modern, maintainable software design.

***

## **Module 2: The Core Curriculum (Beginner)**

In this module, we will address two fundamental topics: the crucial difference between Filters and Interceptors, and how to build your very own custom interceptor. Mastering this section is key to avoiding common misunderstandings and writing clean, effective code.

### **1. Filters vs. Interceptors**

This is one of the most common points of confusion for developers and a frequent interview question. Both Filters and Interceptors seem to do similar things—intercept requests—but they operate at different levels of the application stack and have different capabilities.

#### **In-Depth Explanation**

* **Filters (e.g., `javax.servlet.Filter` in Java/Spring)** are part of the Servlet specification. They are not managed by the application context (like Spring's `ApplicationContext`) but by the Servlet container (like Tomcat or Jetty). This means they operate *outside* the core application framework. Filters are perfect for tasks that need to happen before the request even reaches the framework's dispatcher, which is responsible for routing the request to your controller.
    * **Scope:** They can intercept *any* incoming HTTP request for a given URL pattern, even requests for static resources like images or CSS files. They have no knowledge of which controller or method will eventually handle the request.
    * **Capabilities:** They can modify the incoming request and outgoing response, but they cannot directly interact with the application's beans (services, repositories) or access context-specific information like which handler method will be executed.
* **Interceptors (e.g., `org.springframework.web.servlet.HandlerInterceptor` in Spring)** are a feature of the application framework itself (like Spring MVC). They operate *within* the application context, sitting between the framework's dispatcher and your controller.
    * **Scope:** They only intercept requests that are mapped to a specific handler (i.e., a controller method). They have access to the `HandlerMethod` object, which gives them detailed context about the controller and method that will process the request.
    * **Capabilities:** Because they are part of the application context, they can use dependency injection to access any other bean (e.g., a logging service, a user repository). They also offer more granular control with methods that run before the handler, after the handler, and after the view is rendered.

> **Analogy: Building Security vs. Office Assistant**
>
> *   A **Filter** is like the **main security guard at the building's front door**. They check everyone who enters the building, regardless of which company or floor they are visiting. They don't know or care about the specifics of your meeting; they just perform a generic check (e.g., checking for an ID badge).
>
> *   An **Interceptor** is like the **personal assistant sitting outside a specific executive's office**. They only deal with visitors who have an appointment with *that* executive. They know who the executive is (the handler method) and can perform more specific tasks, like pulling up the relevant meeting files (accessing other beans) before the visitor goes in.

#### **Comparison Table**

| Feature | Filter | Interceptor |
| :-- | :-- | :-- |
| **Level** | Servlet Container Level (e.g., Tomcat) | Framework Level (e.g., Spring MVC) |
| **Lifecycle** | Tied to the Servlet container | Tied to the framework's application context |
| **Context** | Cannot access handler-specific info | Can access the target controller and method |
| **Dependency Injection** | Cannot directly inject application beans | Can inject and use any application bean |
| **Use Cases** | Request/Response encoding, GZIP compression, coarse-grained authentication checks | Fine-grained authentication, logging, auditing, modifying the model before view rendering |
| **Execution Order** | Filter Chain -> DispatcherServlet -> Interceptor Chain -> Controller |  |

### **2. Custom Interceptors**

Now let's build one. We'll create a simple interceptor that logs the time it takes to process a request. This is a classic and highly practical use case. We'll use the Spring Boot `HandlerInterceptor` interface as our example.

#### **In-Depth Explanation**

To create a custom interceptor in Spring, you typically implement the `HandlerInterceptor` interface, which has three main methods you can override:

1. **`preHandle()`**: Executed *before* the target controller method is called. You can use this to perform checks. If this method returns `false`, the execution chain stops, and the controller is never called. This is perfect for authentication or validation logic.
2. **`postHandle()`**: Executed *after* the controller method runs but *before* the view is rendered. This allows you to modify the `ModelAndView` object, adding or changing attributes that the view will use. This method is not called if `preHandle()` returns `false`.
3. **`afterCompletion()`**: Executed *after* the complete request has finished and the view has been rendered. This is the ideal place for cleanup tasks or logging, as it will always be called, even if an exception occurred in the controller.

#### **Code Example: A Performance Logging Interceptor**

Let's create an interceptor to measure and log how long a request takes to be handled by the controller.

```java
// File: src/main/java/com/example/demo/interceptor/PerformanceInterceptor.java

package com.example.demo.interceptor;

import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Component // Make this interceptor a bean managed by Spring
public class PerformanceInterceptor implements HandlerInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(PerformanceInterceptor.class);

    // Use ThreadLocal to safely store start time per request thread
    private final ThreadLocal<Long> startTime = new ThreadLocal<>();

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 1. Executed before the actual handler (controller method)
        long start = System.currentTimeMillis();
        startTime.set(start); // Store the start time for this request

        logger.info("Request Start: [URI: {}] [Method: {}]", request.getRequestURI(), request.getMethod());
        return true; // Return true to continue the execution chain
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        // 2. Executed after the handler method completes, but before view rendering
        // This is useful for adding common attributes to the model
        logger.info("Handler finished. Preparing to render view for URI: {}", request.getRequestURI());
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        // 3. Executed after the complete request is finished (view is rendered)
        long end = System.currentTimeMillis();
        long start = startTime.get();
        long duration = end - start;

        logger.info("Request End: [URI: {}] [Status: {}] [Duration: {}ms]",
                request.getRequestURI(),
                response.getStatus(),
                duration);

        // Clean up the ThreadLocal to prevent memory leaks in application servers
        startTime.remove();

        if (ex != null) {
            logger.error("An exception occurred during request processing: {}", ex.getMessage());
        }
    }
}
```


#### **Best Practice: Registering the Interceptor**

Creating the interceptor class isn't enough; you must tell the framework when and where to apply it. In Spring, you do this by creating a configuration class that implements `WebMvcConfigurer`.

```java
// File: src/main/java/com/example/demo/config/WebConfig.java

package com.example.demo.config;

import com.example.demo.interceptor.PerformanceInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private PerformanceInterceptor performanceInterceptor; // Inject our interceptor bean

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // Register the performance interceptor
        // .addPathPatterns("/**") applies it to all URL paths
        // .excludePathPatterns("/login", "/error") can be used to skip specific paths
        registry.addInterceptor(performanceInterceptor)
                .addPathPatterns("/api/**"); // Only apply to paths under /api/
    }
}
```

This configuration tells Spring to use your `PerformanceInterceptor` for all requests whose URL starts with `/api/`. This is the power of interceptors: precise, configurable, and completely decoupled from the controllers they monitor.

***

## **Module 3: The Core Curriculum (Intermediate)**

In this section, we'll dive into HATEOAS. This isn't a specific technology like an interceptor but rather a powerful constraint of the REST architectural style. Understanding it is crucial for designing robust, long-lasting APIs.

### **HATEOAS API**

HATEOAS stands for **Hypermedia as the Engine of Application State**. It’s a bit of a mouthful, but the core idea is revolutionary. It's the key ingredient that makes an API truly "RESTful."

#### **In-Depth Explanation**

At its heart, HATEOAS is a principle that says a client should be able to navigate your API and discover all possible actions it can take just by following links provided in the responses from the server.

The client starts with a single entry point URL and, from there, the server's responses guide it on what to do next. This decouples the client from the server. The client doesn't need to have a hardcoded list of all possible API endpoints. The server can evolve its URI structure without breaking clients, as long as it keeps the link relations (`rel`) consistent.

* **Application State:** This refers to the current state of a resource. For example, an `Order` resource could be in a `PENDING`, `SHIPPED`, or `CANCELLED` state.
* **Engine:** The "engine" that drives transitions between these states is **hypermedia**—specifically, the links within the API responses.

> **Analogy: Navigating a Website vs. Using a Map**
>
> *   **A non-HATEOAS API** is like being given a paper map of a city. You have a fixed set of streets and locations. If the city builds a new road or a landmark moves, your map is now outdated and potentially broken. You are tightly coupled to the city's layout at the time the map was printed.
>
> *   **A HATEOAS API** is like browsing the World Wide Web. You land on a homepage (the API entry point). You don't know all the URLs on the site beforehand. Instead, you discover them by clicking on links ("Log In," "View Products," "Add to Cart"). The website (the server) guides you through the possible actions. If the site adds a new "Track Your Order" page, it simply adds a new link to the main page. Your browser (the client) doesn't need to be updated; it just sees the new link and knows it can click it.

**How does this relate to Interceptors?** While HATEOAS is a design principle and Interceptors are a technical mechanism, they can work together powerfully. An interceptor (specifically using `postHandle`) could be used to dynamically inspect a response object before it's sent to the client. Based on the data in the response and the user's permissions, the interceptor could inject the appropriate HATEOAS links. For instance, an admin user might see a `delete` link on a resource, while a regular user would not. This logic can be centralized in an interceptor instead of being scattered across controllers.

#### **Code Example: Building a HATEOAS-driven Response**

Let's transform a standard API response into a HATEOAS-compliant one using Spring HATEOAS.

First, imagine a simple controller returning a `Book` object.

**1. The `pom.xml` Dependency (for Maven):**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

**2. The Standard (Non-HATEOAS) Controller:**

```java
// A simple POJO for our Book resource
public class Book {
    private long id;
    private String title;
    private String author;
    // ... getters and setters
}

@RestController
@RequestMapping("/books")
public class BookController {

    @GetMapping("/{id}")
    public Book getBookById(@PathVariable long id) {
        // In a real app, this would come from a service/database
        return new Book(id, "The Pragmatic Programmer", "Andy Hunt & Dave Thomas");
    }
}
```

The response would be plain JSON, with no context or discoverability:

```json
{
  "id": 1,
  "title": "The Pragmatic Programmer",
  "author": "Andy Hunt & Dave Thomas"
}
```

A client receiving this knows nothing about what to do next. How do I get all books? How do I delete this one? The client must have this knowledge pre-programmed.

**3. The Refactored (HATEOAS) Controller:**

Now, let's use Spring HATEOAS to add hypermedia links. We wrap our `Book` object in an `EntityModel`, which is a container for the resource data plus a collection of links.

```java
// Import necessary classes from Spring HATEOAS
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

import org.springframework.hateoas.EntityModel;
import org.springframework.hateoas.Link;
import org.springframework.http.ResponseEntity;
// ... other imports

@RestController
@RequestMapping("/books")
public class HateoasBookController {

    @GetMapping("/{id}")
    public ResponseEntity<EntityModel<Book>> getBookById(@PathVariable long id) {
        Book book = new Book(id, "The Pragmatic Programmer", "Andy Hunt & Dave Thomas");

        // Create a link to this specific book (a "self" link)
        Link selfLink = linkTo(methodOn(HateoasBookController.class).getBookById(id)).withSelfRel();

        // Create a link to the collection of all books
        Link allBooksLink = linkTo(methodOn(HateoasBookController.class).getAllBooks()).withRel("all-books");
        
        // Wrap the book object and its links in an EntityModel
        EntityModel<Book> resource = EntityModel.of(book, selfLink, allBooksLink);

        return ResponseEntity.ok(resource);
    }

    @GetMapping
    public /* ... */ getAllBooks() {
        // ... logic to return all books
    }
}
```

Now, the JSON response is far more powerful and descriptive:

```json
{
  "id": 1,
  "title": "The Pragmatic Programmer",
  "author": "Andy Hunt & Dave Thomas",
  "_links": {
    "self": {
      "href": "http://localhost:8080/books/1"
    },
    "all-books": {
      "href": "http://localhost:8080/books"
    }
  }
}
```


#### **Best Practices**

* **Provide a `self` link:** Every resource representation should include a link pointing to itself. This is the canonical URL for that resource.
* **Use Standard Link Relations:** Whenever possible, use IANA-registered link relations like `self`, `next`, `previous`, `collection`. This improves consistency and allows for generic clients. For custom actions, create your own descriptive relation names (e.g., `cancel-order`, `add-to-cart`).
* **Links Represent State:** The links you provide should reflect the current state of the resource. An order that is `SHIPPED` should not contain a `cancel` link. This embeds business logic into your API's structure, making it smarter and safer for clients.
* **Keep Clients Dumb:** The client should not be constructing URLs. It should only be following the URLs provided by the server. This is the essence of decoupling.

***

## **Module 5: Expert - Interview Mastery**

This module is your playbook for success. We'll cover conceptual questions to test your depth, coding challenges to test your practical skills, and system design scenarios to test your architectural thinking.

### **Common Interview Questions (Theory)**

Here are the top conceptual questions related to our curriculum. Your goal is to answer them concisely and accurately, demonstrating deep understanding.

1. **What is the fundamental difference between a Filter and an Interceptor?**
    * **Answer:** A Filter is a Servlet-level construct, operating at the container level (e.g., Tomcat) before the request even hits the application framework's dispatcher. An Interceptor is a framework-level construct (e.g., Spring MVC) that runs within the application context, giving it access to controllers and other application beans.
2. **When would you choose a Filter over an Interceptor?**
    * **Answer:** You'd use a Filter for concerns that are completely agnostic to the application's business logic, such as GZIP compression, content-type filtering, or modifying the raw HTTP request/response. It's for coarse-grained, framework-independent tasks.
3. **Describe the three lifecycle methods of a Spring `HandlerInterceptor` and their primary use case.**
    * **Answer:**
        * `preHandle()`: Runs *before* the controller. Used for gating logic like authentication or validation. Can stop the request processing chain.
        * `postHandle()`: Runs *after* the controller but *before* view rendering. Used for modifying the `ModelAndView` or adding common model attributes.
        * `afterCompletion()`: Runs *after* the view has been rendered. Used for cleanup tasks like logging request duration or releasing resources. It's always called, even if an exception occurs.
4. **How do you ensure an Interceptor is thread-safe when storing state during a request?**
    * **Answer:** You must use a `ThreadLocal` variable. Since a single interceptor instance handles multiple requests concurrently on different threads, storing request-specific state (like a start time) in an instance variable would lead to race conditions. `ThreadLocal` ensures each thread gets its own isolated copy of the variable.
5. **How can you control which requests an Interceptor applies to?**
    * **Answer:** In the `WebMvcConfigurer` configuration class, when registering the interceptor using `InterceptorRegistry`, you use the `addPathPatterns()` method to specify URL patterns to include and `excludePathPatterns()` to specify patterns to ignore.
6. **What problem does HATEOAS solve?**
    * **Answer:** It solves the problem of tight coupling between a client and a server. By providing discoverable links in API responses, the server can evolve its URI structure and available actions without breaking clients, as clients learn to navigate the API dynamically rather than relying on hardcoded endpoints.
7. **In a HATEOAS response, what is the purpose of the "rel" attribute in a link?**
    * **Answer:** The `rel` (relation) attribute describes the *semantic meaning* of the link. It tells the client *what the link represents*. For example, `rel="self"` indicates the resource's canonical URL, while `rel="edit"` indicates a URL to modify the resource. This allows clients to make decisions based on link semantics, not the URL itself.
8. **Can you have multiple Interceptors? If so, how is their execution order determined?**
    * **Answer:** Yes. Their execution order is determined by the order in which they are registered in the `InterceptorRegistry`. You can also explicitly set an `order()` value on the registration. `preHandle` methods execute in registration order, while `postHandle` and `afterCompletion` execute in the reverse order.
9. **How would an Interceptor prevent a controller method from ever being executed?**
    * **Answer:** By returning `false` from its `preHandle()` method. This immediately halts the execution chain, and the framework sends the response directly to the client without invoking the controller or any subsequent `postHandle` methods.
10. **What is the key difference between the information available in `postHandle` versus `afterCompletion`?**
    * **Answer:** In `postHandle`, you have access to the `ModelAndView` object, allowing you to manipulate the model before the view is rendered. In `afterCompletion`, the view is already rendered, but you gain access to an `Exception` object if one was thrown during processing, making it ideal for centralized exception logging.

### **Common Interview Questions (Practical/Coding)**

#### **1. Coding Task: Implement a Tenant-ID Verification Interceptor**

**Problem:** In a multi-tenant application, every API request to `/api/**` must contain a `X-Tenant-ID` header. Create an interceptor that validates the presence of this header. If it's missing or empty, reject the request with a `400 Bad Request` status and a clear error message.

**Ideal Solution:**

```java
// File: TenantIdInterceptor.java
@Component
public class TenantIdInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String tenantId = request.getHeader("X-Tenant-ID");

        if (tenantId == null || tenantId.trim().isEmpty()) {
            // Header is missing or empty, reject the request.
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST); // 400
            response.getWriter().write("{\"error\": \"X-Tenant-ID header is missing.\"}");
            response.setContentType("application/json");
            return false; // Stop the execution chain
        }

        // Add tenantId to the request attributes so the controller can use it
        request.setAttribute("tenantId", tenantId);
        return true; // Continue to the controller
    }
}

// File: WebConfig.java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private TenantIdInterceptor tenantIdInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // Apply this interceptor only to secured API endpoints
        registry.addInterceptor(tenantIdInterceptor).addPathPatterns("/api/**");
    }
}
```

**Thought Process:** The `preHandle` method is the correct place for this validation. Returning `false` is the key to stopping unauthorized requests early. Setting the response status and writing a message provides clear feedback to the client. Attaching the ID as a request attribute is a clean way to pass data from the interceptor to the controller without altering method signatures.

#### **2. Coding Task: Add HATEOAS Links to a Collection Resource**

**Problem:** You have a controller that returns a list of `Employee` objects. Modify it so that the response includes a `self` link for the entire collection, and each individual `Employee` object in the list also has its own `self` link.

**Ideal Solution:**

```java
// Assume Employee and EmployeeController exist.
// We are focusing on the HATEOAS-enabled controller method.
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

@RestController
@RequestMapping("/employees")
public class EmployeeHateoasController {
    
    // Assume employeeService.findAll() returns List<Employee>
    @Autowired
    private EmployeeService employeeService;

    @GetMapping
    public ResponseEntity<CollectionModel<EntityModel<Employee>>> getAllEmployees() {
        // 1. Fetch all employees
        List<Employee> employees = employeeService.findAll();

        // 2. Create a self-link for each individual employee
        List<EntityModel<Employee>> employeeModels = employees.stream()
                .map(employee -> {
                    Link selfLink = linkTo(methodOn(EmployeeHateoasController.class)
                            .getEmployeeById(employee.getId())).withSelfRel();
                    return EntityModel.of(employee, selfLink);
                })
                .collect(Collectors.toList());

        // 3. Create a link for the entire collection
        Link collectionLink = linkTo(methodOn(EmployeeHateoasController.class)
                .getAllEmployees()).withSelfRel();

        // 4. Wrap the list of models in a CollectionModel
        CollectionModel<EntityModel<Employee>> responseModel = CollectionModel.of(employeeModels, collectionLink);

        return ResponseEntity.ok(responseModel);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<EntityModel<Employee>> getEmployeeById(@PathVariable Long id) {
        // ... logic to get a single employee and add HATEOAS links
    }
}
```

**Thought Process:** The key is using `CollectionModel` for lists of resources and `EntityModel` for single resources. The `stream().map()` pattern is an elegant way to transform each data object into its hypermedia-aware representation. Building links with `linkTo(methodOn(...))` provides type-safe, refactor-friendly URL generation.

## **System Design Scenarios**

### **1. System Design: A Pluggable Auditing System for Microservices**

**Problem:** Design a system that can be easily added to any microservice to automatically log all write operations (POST, PUT, DELETE). The audit log must capture who performed the action, what resource was affected, and the timestamp.

**High-Level Solution:**

* **Component 1: The Audit Interceptor Library:**
    * Create a shared Java library containing a custom `AuditInterceptor`.
    * The interceptor's `preHandle` method would check if the request method is a write operation (POST, PUT, DELETE).
    * It extracts the user's identity from the `Authorization` header (e.g., from a JWT).
    * The `afterCompletion` method is used to ensure the log is sent even if the request fails. It gathers the request URI, the user ID, the HTTP status, and a timestamp.
    * This interceptor doesn't write to a database directly. Instead, it serializes this audit data into a JSON event and sends it asynchronously to a message broker like **Apache Kafka** or **RabbitMQ**.
* **Component 2: Microservice Integration:**
    * Each microservice includes this library as a dependency. The standard `WebMvcConfigurer` registers the `AuditInterceptor` to apply to all relevant endpoints. This is a "plug-and-play" solution.
* **Component 3: The Central Audit Service:**
    * A separate, standalone service consumes the audit events from the message queue.
    * This service is responsible for persisting the audit logs into a scalable, search-optimized data store like **Elasticsearch**. This allows for powerful querying and dashboarding (e.g., "show me all failed delete operations by user X in the last 24 hours").

**Design Trade-offs:**

* **Synchronous vs. Asynchronous Logging:** We chose asynchronous logging via a message queue. This prevents the auditing system from adding latency to the primary user request or causing the request to fail if the audit service is down. The trade-off is eventual consistency for the audit trail.
* **Payload Logging:** We are not logging the request/response body to avoid storing sensitive data (PII) and to keep log sizes manageable. If payload logging were required, it would need a robust data scrubbing/anonymization layer.


### **2. System Design: A Tiered, Rate-Limited Public API**

**Problem:** Design the gateway for a public API that provides financial data. The API has three tiers: Free (API key, 10 req/min), Pro (API key, 100 req/min), and Enterprise (mTLS, unlimited). The gateway must enforce these rules.

**High-Level Solution:**
This problem is best solved at the **API Gateway** level (e.g., Spring Cloud Gateway, Kong, Apigee), which uses a Filter-chain pattern.

* **Chain of Filters in the API Gateway:**

1. **Authentication Filter:**
        * This is the first filter. It inspects the request for credentials. If it finds an `X-Api-Key` header, it calls a fast-caching `Auth Service` to validate the key and retrieve the associated user ID and tier (`Free` or `Pro`).
        * If it finds a client-side TLS certificate (mTLS), it validates it and identifies the tier as `Enterprise`.
        * If authentication fails, the filter rejects the request with a `401 Unauthorized`.
        * If successful, it adds headers to the request before forwarding it downstream (e.g., `X-User-ID: 123`, `X-User-Tier: Pro`).
2. **Rate Limiting Filter:**
        * This filter runs next. It reads the `X-User-ID` and `X-User-Tier` headers.
        * It uses a distributed cache like **Redis** with a sliding window algorithm to enforce rate limits. The key could be something like `ratelimit:{userId}`.
        * If the request is over the limit, it rejects it with a `429 Too Many Requests`. It also adds informative response headers like `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`.
        * It ignores requests with `X-User-Tier: Enterprise`.
3. **Routing Filter:**
        * Finally, the gateway routes the validated, enriched request to the appropriate downstream microservice (e.g., the `MarketData-Service`).
* **HATEOAS for Upgrades:**
    * The downstream `MarketData-Service`, upon receiving a request with `X-User-Tier: Free`, can use HATEOAS in its response. It would add a link like: `{"rel": "upgrade-to-pro", "href": "https://api.example.com/subscriptions/upgrade"}`. This guides free users toward conversion without the client needing to know the upgrade URL.

**Design Trade-offs:**

* **Centralized vs. Decentralized Enforcement:** By placing this logic in a central API Gateway, we keep the downstream microservices simple. They don't need to know about authentication or rate limiting; they just focus on their business logic. The trade-off is that the gateway becomes a critical, stateful component that must be highly available and scalable.

***

