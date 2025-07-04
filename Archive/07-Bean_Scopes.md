# Bean Scopes
Bean scopes define the rules for creating and managing bean instances. When you request a bean from the Spring container (e.g., via `@Autowired`), Spring consults the bean's defined scope to determine whether to return an existing instance or create a new one.

## 1. Singleton: The Default and Most Common Scope
The `singleton` scope is the **default** scope in Spring. When a bean is defined as a singleton, the Spring IoC container creates **exactly one instance** of that bean per Spring container. All subsequent requests for that bean, by name or by type, will return the same single instance.

### How it works:
- The bean is instantiated when the Spring application context starts (or lazily if configured).
- This single instance is then stored in the Spring container's cache.
- Every time another bean or component requests this singleton bean, Spring returns the cached instance.

### Why it is efficient and suitable for stateless services:
- **Efficiency**: Creating a new object is a relatively expensive operation in terms of CPU cycles and memory. By reusing a single instance, `singleton` scope minimizes object creation overhead, leading to better performance.
- **Resource Conservation**: Only one instance occupies memory, which is crucial for applications with many beans.
- **Suitability for Stateless Services**: Most services in a Spring application (e.g., `Service`, `Repository`, `Controller` classes) are inherently stateless. This means they do not hold any mutable data specific to a particular client or operation. Their methods operate on input parameters and return results, without modifying internal state that would affect subsequent calls. For such services, a single shared instance is perfectly adequate and highly efficient.

### Use Case:
Almost all Spring components like `Service`, `Repository`, `Controller`, `Utility` classes that do not maintain conversational state should be singletons.

#### Example:
```java
import org.springframework.stereotype.Service;
import org.springframework.context.annotation.Scope;

@Service
@Scope("singleton") // This is the default, so it can be omitted
public class ProductService {

    private int requestCount = 0; // This would be shared across all users

    public String getProductDetails(String productId) {
        requestCount++; // This count will increment for every user accessing the service
        System.out.println("ProductService instance: " + this.hashCode() + ", Request Count: " + requestCount);
        return "Details for product: " + productId;
    }
}
```

In the above example, `ProductService` is a singleton. If multiple threads or users access `getProductDetails`, they will all operate on the same `requestCount` variable, demonstrating its shared nature. For truly stateless services, `requestCount` wouldn't exist or would be handled thread-locally.

---

## 2. Prototype: A New Instance for Every Request
In contrast to `singleton`, the `prototype` scope ensures that a **new bean instance is created every time it is requested** from the Spring container.

### How it works:
- Spring does not pre-instantiate `prototype` beans at application startup.
- When a component requests a `prototype` bean, Spring creates a brand new instance, configures it, and returns it.
- Spring does not manage the complete lifecycle of a `prototype` bean beyond its initialization. Once instantiated and handed over, the client code is responsible for managing its further lifecycle (e.g., destruction).

### When to use it:
- **Stateful Objects**: When a bean needs to maintain a unique state for each client or operation. For example, a shopping cart object for an e-commerce application, where each user has their own cart.
- **Short-Lived Objects**: Objects that are used for a specific, short-term task and then can be discarded.
- **Non-Thread-Safe Objects**: If an object is not thread-safe and needs to be used concurrently, making it `prototype` ensures each thread gets its own instance, preventing race conditions.

### How lifecycle methods behave differently than singleton:
- **Initialization**: For both `singleton` and `prototype` beans, Spring calls initialization lifecycle callbacks (e.g., `@PostConstruct`, `InitializingBean`).
- **Destruction**: For `singleton` beans, Spring manages their destruction (e.g., `@PreDestroy`, `DisposableBean`) when the container shuts down. However, for prototype beans, Spring does not manage their destruction lifecycle. Once a `prototype` instance is created and handed over, Spring loses track of it. It's the responsibility of the client code to clean up resources if needed.

### Use Case:
A `PaymentTransaction` object, where each transaction needs its own independent state.

#### Example:
```java
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

@Component
@Scope("prototype")
public class ShoppingCart {

    private String customerId;
    private List<String> items = new ArrayList<>();

    public ShoppingCart() {
        System.out.println("ShoppingCart instance created: " + this.hashCode());
    }

    @PostConstruct
    public void init() {
        System.out.println("ShoppingCart initialized: " + this.hashCode());
    }

    public void addItem(String item) {
        items.add(item);
        System.out.println("Added '" + item + "' to cart for customer: " + customerId);
    }

    public List<String> getItems() {
        return items;
    }

    public void setCustomerId(String customerId) {
        this.customerId = customerId;
    }

    @PreDestroy // This method will NOT be called by Spring for prototype beans
    public void destroy() {
        System.out.println("ShoppingCart destroyed: " + this.hashCode());
    }
}
```

If you request `ShoppingCart` multiple times, you'll get different hash codes and independent `items` lists. The `@PreDestroy` method will not be invoked by Spring.

--- 

## Web-Specific Scopes: Requiring a Web Context
For the following scopes (`request`, `session`, `application`, `websocket`), your Spring Boot application needs to be a web application, and Spring needs to have its web application context enabled. Spring Boot automatically configures this when you include `spring-boot-starter-web` (for Servlet environments) or `spring-boot-starter-webflux` (for Reactive environments).

These web-specific scopes are typically enabled by annotations like `@RequestScope`, `@SessionScope`, etc., which are convenience annotations that internally use `@Scope` with the respective scope names.

--- 

## 3. Request: One Instance Per HTTP Request
The `request` scope creates a new bean instance for **every incoming HTTP request**. This instance is available only during the lifetime of that specific HTTP request. Once the request completes, the bean instance is discarded.

### Use Cases:
- **Request-Scoped Data**: Storing data that is specific to a single HTTP request, such as a transaction ID, user preferences for the current request, or performance metrics.
- **Auditing or Logging**: Injecting a request-specific logger that includes details of the current request.
- **Form Data Processing**: When processing a web form, you might have a bean to hold the form data for that specific submission.

#### Example:
```java
import org.springframework.stereotype.Component;
import org.springframework.web.context.annotation.RequestScope;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

@Component
@RequestScope
public class RequestScopedBean {

    private String requestId;

    public RequestScopedBean() {
        System.out.println("RequestScopedBean instance created: " + this.hashCode());
    }

    @PostConstruct
    public void init() {
        this.requestId = "REQ-" + System.nanoTime();
        System.out.println("RequestScopedBean initialized with ID: " + requestId + ", Hash: " + this.hashCode());
    }

    public String getRequestId() {
        return requestId;
    }

    @PreDestroy
    public void destroy() {
        System.out.println("RequestScopedBean destroyed with ID: " + requestId + ", Hash: " + this.hashCode());
    }
}
```

```java
// In a Controller
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class MyController {

    @Autowired
    private RequestScopedBean requestScopedBean;

    @GetMapping("/request-info")
    public String getRequestInfo() {
        return "Current Request ID: " + requestScopedBean.getRequestId();
    }
}
```

Each time you hit `/request-info` in your browser, you will see a new `RequestScopedBean` instance created, initialized with a unique `requestId`, and then destroyed after the response is sent.

---

## 4. Session: One Instance Per HTTP Session
The `session` scope creates a new bean instance for `each HTTP session`. This instance lives as long as the user's HTTP session is active. If the session expires or is invalidated, the bean instance is destroyed.

### Use Cases:
- **User Session-Related State**: Storing user-specific data that persists across multiple requests within the same session. Examples include:
  - Shopping cart details (if not persisted to a database immediately).
  - User preferences or settings for the current session.
  - Authentication details after a user logs in.
- **Wizard-like Flows**: Maintaining state across a multi-step form or workflow.

#### Example:
```java
import org.springframework.stereotype.Component;
import org.springframework.web.context.annotation.SessionScope;
import java.util.ArrayList;
import java.util.List;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

@Component
@SessionScope
public class UserSessionData {

    private String username;
    private List<String> viewedProducts = new ArrayList<>();

    public UserSessionData() {
        System.out.println("UserSessionData instance created: " + this.hashCode());
    }

    @PostConstruct
    public void init() {
        System.out.println("UserSessionData initialized: " + this.hashCode());
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getUsername() {
        return username;
    }

    public void addViewedProduct(String product) {
        viewedProducts.add(product);
    }

    public List<String> getViewedProducts() {
        return viewedProducts;
    }

    @PreDestroy
    public void destroy() {
        System.out.println("UserSessionData destroyed for user: " + username + ", Hash: " + this.hashCode());
    }
}
```

```java
// In a Controller
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class SessionController {

    @Autowired
    private UserSessionData userSessionData;

    @GetMapping("/set-user")
    public String setUsername(@RequestParam String username) {
        userSessionData.setUsername(username);
        return "Username set to: " + username + " in session.";
    }

    @GetMapping("/add-product")
    public String addProduct(@RequestParam String product) {
        userSessionData.addViewedProduct(product);
        return "Product '" + product + "' added to viewed list for " + userSessionData.getUsername();
    }

    @GetMapping("/get-session-info")
    public String getSessionInfo() {
        return "User: " + userSessionData.getUsername() + ", Viewed Products: " + userSessionData.getViewedProducts();
    }
}
```

If you access `/set-user?username=Alice`, then `/add-product?product=Laptop`, then `/get-session-info` from the same browser window, you'll see the state (`username`, `viewedProducts`) maintained. If you open a different browser or an incognito window, a new `UserSessionData` instance will be created, demonstrating session isolation.

---

## 5. Application: A Single Instance Per ServletContext
The `application` scope creates a single bean instance for the entire web application, effectively acting as a singleton within the web application's `ServletContext`. This instance is created when the `ServletContext` is initialized and destroyed when the `ServletContext` is destroyed (i.e., when the web application shuts down).

### Use Cases:
- **Application-Level Caching**: Storing application-wide data that rarely changes and needs to be accessed by all users.
- **Configuration Data**: Holding global application configuration that is loaded once.
- **Shared Resources**: Managing a single instance of a resource that is shared across the entire application (e.g., a database connection pool, if not managed by a separate pool manager).
- **Global Counters/Statistics**: Maintaining application-wide statistics.

#### Example:
```java
import org.springframework.stereotype.Component;
import org.springframework.web.context.annotation.ApplicationScope;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

@Component
@ApplicationScope
public class ApplicationConfiguration {

    private String appVersion;
    private long startupTimestamp;

    public ApplicationConfiguration() {
        System.out.println("ApplicationConfiguration instance created: " + this.hashCode());
    }

    @PostConstruct
    public void init() {
        this.appVersion = "1.0.0-BETA";
        this.startupTimestamp = System.currentTimeMillis();
        System.out.println("ApplicationConfiguration initialized. Version: " + appVersion);
    }

    public String getAppVersion() {
        return appVersion;
    }

    public long getStartupTimestamp() {
        return startupTimestamp;
    }

    @PreDestroy
    public void destroy() {
        System.out.println("ApplicationConfiguration destroyed. Version: " + appVersion + ", Hash: " + this.hashCode());
    }
}
```

```java
// In a Controller
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AppInfoController {

    @Autowired
    private ApplicationConfiguration appConfig;

    @GetMapping("/app-info")
    public String getApplicationInfo() {
        return "App Version: " + appConfig.getAppVersion() +
               "<br>Startup Time: " + new java.util.Date(appConfig.getStartupTimestamp());
    }
}
```

No matter how many users access `/app-info` or how many sessions are active, they will all receive the same `appVersion` and `startupTimestamp` from the single `ApplicationConfiguration` instance.

---

## 6. WebSocket: One Bean Per WebSocket Session
The `websocket` scope is specific to Spring's WebSocket support. It creates a new bean instance for each WebSocket session. The bean lives as long as the WebSocket connection is active and is destroyed when the session ends.

### Use Cases:
- **Real-time Applications**: Maintaining conversational state or user-specific data within a single WebSocket connection.
- **Game State**: In a multiplayer game, each player's specific game state during a WebSocket session.
- **Chat Applications**: Storing user-specific details or chat history for a particular WebSocket connection.

#### Example (Conceptual - requires Spring WebSocket setup):
```java
// In a WebSocket configuration
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myWebSocketHandler(), "/ws").setAllowedOrigins("*");
    }

    @Bean
    public MyWebSocketHandler myWebSocketHandler() {
        return new MyWebSocketHandler();
    }
}

// The WebSocket-scoped bean
import org.springframework.stereotype.Component;
import org.springframework.web.socket.handler.TextWebSocketHandler;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.context.annotation.SessionScope; // Or @Scope("websocket") if using raw @Scope
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import java.io.IOException;

// Note: For WebSocket scope, you typically use @Scope("websocket") or define it programmatically.
// @SessionScope here is just for illustrative purposes as some web-specific annotations work across contexts.
// In a true WebSocket scenario, you'd integrate with WebSocketMessageBrokerConfigurer and specific STOMP handlers.

@Component
@Scope("websocket") // This is the actual annotation for WebSocket scope
public class WebSocketSessionData {

    private String sessionId;
    private String username;
    private int messageCount = 0;

    public WebSocketSessionData() {
        System.out.println("WebSocketSessionData instance created: " + this.hashCode());
    }

    @PostConstruct
    public void init() {
        // In a real scenario, you'd get the session ID from WebSocketSession
        this.sessionId = "WS-" + System.nanoTime();
        System.out.println("WebSocketSessionData initialized for session: " + sessionId + ", Hash: " + this.hashCode());
    }

    public void incrementMessageCount() {
        messageCount++;
    }

    public int getMessageCount() {
        return messageCount;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getUsername() {
        return username;
    }

    @PreDestroy
    public void destroy() {
        System.out.println("WebSocketSessionData destroyed for session: " + sessionId + ", Hash: " + this.hashCode());
    }
}

// Example WebSocket Handler (simplified)
import org.springframework.web.socket.TextMessage;
import org.springframework.beans.factory.annotation.Autowired;

public class MyWebSocketHandler extends TextWebSocketHandler {

    @Autowired
    private WebSocketSessionData sessionData; // This bean will be WebSocket-scoped

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        sessionData.setUsername("User-" + session.getId().substring(0, 5));
        System.out.println("WebSocket connection established. Session data for: " + sessionData.getUsername());
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        sessionData.incrementMessageCount();
        System.out.println("Received message from " + sessionData.getUsername() + ". Message count: " + sessionData.getMessageCount());
        session.sendMessage(new TextMessage("Echo from server: " + message.getPayload()));
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        System.out.println("WebSocket connection closed. Session data destroyed for: " + sessionData.getUsername());
    }
}
```

Each WebSocket connection will have its own unique `WebSocketSessionData` bean, allowing for independent message counts and usernames per client.

---

## Setting Scopes with `@Scope` and Other Annotations
You can set bean scopes using the `@Scope` annotation.
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

// Using @Component
@Component
@Scope("prototype")
public class MyPrototypeComponent {
    // ...
}

// Using @Bean in a @Configuration class
@Configuration
public class AppConfig {

    @Bean
    @Scope("request")
    public MyRequestBean myRequestBean() {
        return new MyRequestBean();
    }

    @Bean
    @Scope("session")
    public MySessionBean mySessionBean() {
        return new MySessionBean();
    }

    @Bean
    @Scope("application")
    public MyApplicationBean myApplicationBean() {
        return new MyApplicationBean();
    }

    // For WebSocket, if explicitly defining a bean
    @Bean
    @Scope("websocket")
    public MyWebSocketBean myWebSocketBean() {
        return new MyWebSocketBean();
    }
}
```

### Convenience Annotations for Web Scopes:
Spring provides specific annotations that are aliases for `@Scope` with the respective web scopes:
- `@RequestScope` (equivalent to `@Scope("request")`)
- `@SessionScope` (equivalent to `@Scope("session")`)
- `@ApplicationScope` (equivalent to `@Scope("application")`)
These are often preferred for readability and to clearly indicate the intent.

#### Example using convenience annotations:
```java
import org.springframework.stereotype.Component;
import org.springframework.web.context.annotation.RequestScope;
import org.springframework.web.context.annotation.SessionScope;
import org.springframework.web.context.annotation.ApplicationScope;

@Component
@RequestScope
public class RequestData { /* ... */ }

@Component
@SessionScope
public class ShoppingCartSession { /* ... */ }

@Component
@ApplicationScope
public class GlobalCache { /* ... */ }
```

--- 

## Advanced Usage and Challenges
### Dependency Injection Challenges (e.g., injecting a prototype bean into a singleton)
This is a common and important challenge. If you inject a `prototype` bean directly into a `singleton` bean, the `prototype` bean will effectively behave like a `singleton`. Why? Because the `singleton` bean is instantiated only once by Spring, and at that time, a single instance of the `prototype` bean is injected into it. The `singleton` bean will then always hold a reference to that same `prototype` instance, defeating the purpose of the `prototype` scope.

#### Example of the Problem:
```java
@Component
@Scope("prototype")
public class MessageGenerator {
    private String message;
    private final long instanceId = System.nanoTime();

    public void setMessage(String message) {
        this.message = message;
    }

    public String getMessage() {
        return "Instance " + instanceId + ": " + message;
    }
}

@Service
@Scope("singleton")
public class MessageService {

    @Autowired
    private MessageGenerator messageGenerator; // Problematic injection

    public String generateAndGetMessage(String text) {
        messageGenerator.setMessage(text);
        return messageGenerator.getMessage();
    }
}
```

If you call `messageService.generateAndGetMessage("Hello")` and then `messageService.generateAndGetMessage("World")`, you will see `Instance XXX: World` for both calls (or an inconsistent `Instance XXX: Hello` then `Instance XXX: World` depending on initial wiring and first use). The `messageGenerator` will be the same instance each time.

#### Solutions using `ObjectFactory`, `Provider`, or Scoped Proxies
Spring provides several solutions to address this challenge and ensure that a new instance of a `prototype` (or other non-singleton) bean is obtained each time it's needed within a `singleton` context.
1. `ObjectFactory<T>`:
   - Injects an `ObjectFactory` that can produce new instances of the target bean on demand.
   - You explicitly call `getObject()` on the factory whenever a new instance is required.
   ```java
   import org.springframework.beans.factory.ObjectFactory;

   @Service
   @Scope("singleton")
   public class MessageServiceWithObjectFactory {

      @Autowired
      private ObjectFactory<MessageGenerator> messageGeneratorFactory;

      public String generateAndGetMessage(String text) {
        MessageGenerator generator = messageGeneratorFactory.getObject(); // Gets a new prototype instance
        generator.setMessage(text);
        return generator.getMessage();
     }
   }
   ```

2. `Provider<T>` (JSR-330):
- Similar to `ObjectFactory` but uses the JSR-330 `Provider` interface.
- Requires the `javax.inject` (or jakarta.inject) dependency.
- The usage is identical to `ObjectFactory` (call `get()`).
   ```java
   import javax.inject.Provider; // Or jakarta.inject.Provider

   @Service
   @Scope("singleton")
   public class MessageServiceWithProvider {

      @Autowired
      private Provider<MessageGenerator> messageGeneratorProvider;

      public String generateAndGetMessage(String text) {
        MessageGenerator generator = messageGeneratorProvider.get(); // Gets a new prototype instance
        generator.setMessage(text);
        return generator.getMessage();
     }
   }
  ```

3. **Scoped Proxies** (`@Scope(value = "...", proxyMode = ScopedProxyMode.TARGET_CLASS`) or `ScopedProxyMode.INTERFACES`)
- This is the most common and often preferred solution for web-scoped beans (like `request` or `session`) and can also be used for `prototype` beans.
- Instead of injecting the actual bean instance, Spring injects a proxy to the bean.
- This proxy intercepts calls to the bean's methods. When a method is called on the proxy, it consults the current scope (e.g., current HTTP request, current session) and retrieves/creates the actual bean instance for that scope, then delegates the method call to that instance.
- `ScopedProxyMode.TARGET_CLASS`: Used when the bean is a concrete class. Spring CGLIB generates a subclass proxy.
- `ScopedProxyMode.INTERFACES`: Used when the bean implements interfaces. Spring generates a JDK dynamic proxy that implements the same interfaces.

#### Example for Prototype:
```java
import org.springframework.context.annotation.Scope;
import org.springframework.context.annotation.ScopedProxyMode;
import org.springframework.stereotype.Component;

@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class MessageGeneratorProxy {
    private String message;
    private final long instanceId = System.nanoTime();

    public void setMessage(String message) {
        this.message = message;
    }

    public String getMessage() {
        return "Instance " + instanceId + ": " + message;
    }
}

@Service
@Scope("singleton")
public class MessageServiceWithProxy {

    @Autowired
    private MessageGeneratorProxy messageGeneratorProxy; // Injects a proxy

    public String generateAndGetMessage(String text) {
        messageGeneratorProxy.setMessage(text); // Proxy ensures a new instance is used per call
        return messageGeneratorProxy.getMessage();
    }
}
```

With `ScopedProxyMode.TARGET_CLASS`, each call to `messageGeneratorProxy.setMessage()` or `getMessage()` will trigger the proxy to fetch a new `MessageGeneratorProxy` instance from the container (because it's a `prototype`), ensuring the `instanceId` and `message` are unique for that specific operation.
#### Example for Request Scope:
```java
import org.springframework.web.context.annotation.RequestScope;
import org.springframework.context.annotation.ScopedProxyMode;
import org.springframework.stereotype.Component;

@Component
@RequestScope(proxyMode = ScopedProxyMode.TARGET_CLASS) // This is how you'd use it for Request/Session scopes
public class RequestScopedData {
    private String data;
    // ... getters/setters
}

@Service
public class MyService { // This is a singleton by default

    @Autowired
    private RequestScopedData requestScopedData; // Injects a proxy

    public String processRequest(String input) {
        requestScopedData.setData("Processed: " + input);
        return requestScopedData.getData();
    }
}
```

Here, `requestScopedData` will be a proxy. When `processRequest` is called, the proxy will ensure that the correct `RequestScopedData` instance for the current HTTP request is used.

#### How Spring Boot Handles Scope Resolution Internally
Spring Boot, leveraging the core Spring Framework, handles scope resolution primarily through its `BeanFactory` and `ApplicationContext` implementations.
1. **Bean Definition**: When Spring Boot starts, it scans for components (`@Component`, `@Service`, `@Controller`, `@Repository`) and `Bean` definitions. During this process, it reads the `@Scope` annotation (or its convenience aliases) associated with each bean.
2. **Bean Registry**: These bean definitions, including their scope information, are stored in the `DefaultListableBeanFactory` (or a similar internal registry).
3. **Bean Instantiation**:
   - **Singletons**: Typically instantiated eagerly during application startup (unless lazy initialization is enabled with `@Lazy`). The single instance is cached in the `singletonObjects` map within the `DefaultSingletonBeanRegistry`.
   - **Prototypes**: Not instantiated at startup. When a `prototype` bean is requested, Spring's `AbstractBeanFactory` creates a new instance using reflection or CGLIB, applies post-processors, and returns it. No caching occurs.
   - **Web Scopes** (`request`, `session`, `application`, `websocket`):
     - Spring's web integration (e.g., `RequestContextFilter`, `SessionContextFilter`) makes the current `HttpServletRequest` and `HttpSession` objects available in the current thread's context.
     - When a web-scoped bean is requested, Spring checks if an instance for the current scope (request ID, session ID) already exists in its internal maps (`requestAttributes`, `sessionAttributes`, `applicationAttributes` within the `WebApplicationContext`).
     - If it exists, the cached instance is returned.
     - If not, a new instance is created, stored in the appropriate scope context, and returned.
     - When the request/session ends, Spring's internal mechanisms (e.g., `RequestHandledEvent`, `HttpSessionDestroyedEvent`) trigger the destruction of the corresponding scoped beans.
4. **Scoped Proxies**: When `proxyMode` is used, Spring doesn't inject the actual bean. Instead, it creates and injects a `ScopedProxyFactoryBean` (or similar internal proxy factory). This factory generates a dynamic proxy. Whenever a method is called on this proxy, the proxy looks up the actual target bean instance from the current scope context and delegates the call. This ensures that the correct instance (e.g., for the current request) is always used.

### Common Pitfalls
1. **Misuse of Stateful Prototype Beans in Singletons**: As discussed, injecting a stateful prototype directly into a singleton leads to the prototype behaving like a singleton. Always use `ObjectFactory`, `Provider`, or scoped proxies to get a fresh instance.
2. **Memory Leaks in Session/Application-Scoped Beans**:
   - **Session-scoped beans**: If you store large or numerous objects in session-scoped beans, and users maintain many active sessions, this can lead to significant memory consumption on the server. Excessive session data can lead to `OutOfMemoryError`. Consider storing only essential, small pieces of data in the session and loading larger data from a database as needed.
   - **Application-scoped beans**: While they are singletons, if they hold references to large, mutable collections that continuously grow without bounds, they can also lead to memory leaks over time. Ensure proper clearing or size limits for such collections.
3. **Thread Safety Issues with Stateful Singletons**: While the `singleton` scope is great for stateless services, if you inadvertently add mutable state to a singleton bean without proper synchronization, you will introduce thread safety issues and race conditions. This is a common bug.
4. **Serialization Issues with Web-Scoped Beans**: If you are running your application in a distributed environment (e.g., multiple Tomcat instances behind a load balancer), and you rely on session replication for failover, session-scoped beans must be `Serializable`. If they are not, session replication will fail.
5. **Overuse of Web-Specific Scopes**: While useful, not every piece of data needs to be `request` or `session` scoped. Often, passing data as method parameters or using thread-local variables for short-lived, request-specific data is more efficient and less intrusive.


### Best Practices for Choosing the Right Scope
Choosing the correct bean scope is critical for application performance, memory management, and correctness.
1. **Default to Singleton**: For the vast majority of your Spring components (services, repositories, controllers, utility classes), the `singleton` scope is the most efficient and appropriate choice. These components are typically stateless and shared across the application.
2. **Use Prototype for Unique Stateful Instances**:
   - If a bean must maintain its own independent, mutable state for each usage or operation.
   - When the bean is not thread-safe and needs to be used concurrently by multiple threads (each getting its own instance).
   - Always consider `ObjectFactory`, `Provider`, or `ScopedProxyMode` when injecting prototype beans into singletons.
3. **Leverage Web Scopes for HTTP-Specific State**:
   - **`Request` scope**: For data that is truly unique to a single HTTP request's lifecycle (e.g., request-specific transaction IDs, data collected from a form submission). It helps in isolating request-specific concerns.
   - **`Session` scope**: For user-specific data that needs to persist across multiple requests within the same user session (e.g., shopping carts, user preferences). Be mindful of memory usage and serialization.
   - **`Application` scope**: For truly global, application-wide configuration or cached data that rarely changes and is accessed by all users. Similar to a singleton but tied to the web `ServletContext`.
   - **`WebSocket` scope**: For state specific to a single WebSocket connection in real-time applications.
4. **Prioritize Statelessness**: Design your beans to be as stateless as possible. This makes them easier to test, reason about, and enables them to be safely shared as singletons. If state is absolutely necessary, encapsulate it carefully and choose the appropriate scope.
5. **Consider Memory Impact**:
   - `Session` and `Application` scoped beans, especially if they store large objects or collections, can consume significant server memory. Regularly review the data stored in these scopes.
   - `Prototype` beans, if created excessively and not properly garbage collected, can also lead to memory pressure.
6. **Concurrency Considerations**:
   - Singletons: Must be thread-safe if they contain mutable state. Use synchronization mechanisms (e.g., synchronized blocks, `java.util.concurrent` utilities) or prefer immutable state.
   - Prototype/Request/Session/WebSocket: Since each thread/request/session gets its own instance, thread safety within the bean is less of a concern, but interactions between these beans and shared singletons still require careful design.