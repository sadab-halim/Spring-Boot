# Spring Boot Microservices

## Synchronous Communication Patterns with HTTP
Synchronous communication is a common pattern where a client service sends a request to a server service and waits for a response before proceeding. HTTP/REST is the most prevalent protocol for synchronous communication in microservices due to its simplicity, widespread adoption, and stateless nature.

However, in a distributed environment, relying solely on direct HTTP calls can be problematic. Several key concepts become crucial to ensure reliable and efficient synchronous communication:
- **Service Discovery**: In a dynamic microservices environment, service instances are frequently scaled up or down, and their network locations (IP addresses and ports) can change. Service discovery mechanisms allow client services to find the network location of a desired service instance without hardcoding URLs.
  - **How it works**: Services register themselves with a Service Registry (e.g., Eureka, Consul, ZooKeeper) upon startup, providing their network address. When a client wants to call a service, it queries the Service Registry to get the available instances of that service.
- **Load Balancing**: When multiple instances of a service are running, incoming requests need to be distributed across them to prevent any single instance from becoming a bottleneck and to ensure high availability.
  - **Client-Side Load Balancing**: The client service itself is responsible for selecting an instance from the available list (obtained from the service registry) and routing the request. Spring Cloud LoadBalancer (formerly Ribbon) is a popular choice for this.
  - **Server-Side Load Balancing**: A dedicated load balancer (e.g., Nginx, AWS ELB) sits in front of the service instances and distributes requests to them.
- **Resiliency**: In a distributed system, failures are inevitable. A service might be slow, unavailable, or return errors. Resiliency patterns are essential to prevent these failures from cascading throughout the system. Key patterns include:
  - **Timeouts**: Limiting the amount of time a client will wait for a response from a service.
  - **Retries**: Automatically re-attempting a failed request, especially for transient errors.
  - **Circuit Breaker**: A pattern that prevents a client from repeatedly calling a failing service, giving the service time to recover and preventing cascading failures. When failures reach a threshold, the circuit "opens," and subsequent requests fail fast without hitting the service. After a configurable time, the circuit enters a "half-open" state, allowing a few test requests to see if the service has recovered.
  - **Fallbacks**: Providing a default or alternative response when a service call fails, ensuring a graceful degradation of functionality rather than a complete system failure.

### 1. RestTemplate (Traditional Approach)
`RestTemplate` has been a staple in Spring for making synchronous HTTP requests. While it's now in maintenance mode and Spring recommends `WebClient` for reactive applications or `RestClient` for synchronous, it's still widely encountered in existing Spring Boot microservices.

#### Configuration with `@Bean`
```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        // You can customize the RestTemplate here, e.g., set timeouts, add interceptors
        RestTemplate restTemplate = new RestTemplate();
        // Optional: Set timeouts
        // SimpleClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
        // requestFactory.setConnectTimeout(5000); // 5 seconds
        // requestFactory.setReadTimeout(5000);    // 5 seconds
        // restTemplate.setRequestFactory(requestFactory);
        return restTemplate;
    }
}
```

#### Handling Requests/Responses
`RestTemplate` provides various methods for different HTTP operations (GET, POST, PUT, DELETE) and different ways to handle responses.
```java
package com.example.service;

import com.example.model.Product;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class ProductServiceClient {

    private final RestTemplate restTemplate;
    private final String PRODUCT_SERVICE_BASE_URL = "http://product-service"; // Or a direct URL if not using service discovery

    @Autowired
    public ProductServiceClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public Product getProductById(Long id) {
        // GET request with path variable
        return restTemplate.getForObject(PRODUCT_SERVICE_BASE_URL + "/products/{id}", Product.class, id);
    }

    public Product createProduct(Product product) {
        // POST request with request body
        return restTemplate.postForObject(PRODUCT_SERVICE_BASE_URL + "/products", product, Product.class);
    }

    public void updateProduct(Long id, Product product) {
        // PUT request with path variable and request body
        restTemplate.put(PRODUCT_SERVICE_BASE_URL + "/products/{id}", product, id);
    }

    public void deleteProduct(Long id) {
        // DELETE request with path variable
        restTemplate.delete(PRODUCT_SERVICE_BASE_URL + "/products/{id}", id);
    }

    // Example with ResponseEntity for more control over response headers/status
    public Product getProductByIdWithResponseEntity(Long id) {
        // getForEntity returns ResponseEntity, allowing access to headers and status code
        return restTemplate.getForEntity(PRODUCT_SERVICE_BASE_URL + "/products/{id}", Product.class, id).getBody();
    }
}
```

#### Managing Exceptions
`RestTemplate` throws `RestClientException` (and its subclasses like `HttpClientErrorException` for 4xx errors and `HttpServerErrorException` for 5xx errors) for HTTP errors. You can use `try-catch` blocks to handle these
```java
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.HttpServerErrorException;
import org.springframework.web.client.RestClientException;

// ... inside a service method
try {
    Product product = restTemplate.getForObject(PRODUCT_SERVICE_BASE_URL + "/products/{id}", Product.class, id);
    return product;
} catch (HttpClientErrorException.NotFound ex) {
    // Handle 404 Not Found specifically
    System.err.println("Product not found: " + ex.getMessage());
    return null; // Or throw a custom exception
} catch (HttpClientErrorException ex) {
    // Handle other 4xx errors
    System.err.println("Client error: " + ex.getStatusCode() + " - " + ex.getResponseBodyAsString());
    throw new RuntimeException("Failed to fetch product due to client error", ex);
} catch (HttpServerErrorException ex) {
    // Handle 5xx errors
    System.err.println("Server error: " + ex.getStatusCode() + " - " + ex.getResponseBodyAsString());
    throw new RuntimeException("Failed to fetch product due to server error", ex);
} catch (RestClientException ex) {
    // Handle other RestTemplate-related exceptions (e.g., connection issues)
    System.err.println("REST client exception: " + ex.getMessage());
    throw new RuntimeException("Failed to communicate with product service", ex);
}
```

#### Client-Side Load Balancing using Spring Cloud LoadBalancer
To enable client-side load balancing with `RestTemplate`, you need to include the `spring-cloud-starter-loadbalancer` dependency and annotate your `RestTemplate` `@Bean` with `@LoadBalanced`.

1. **Add Dependency**:
   ```xml
   <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
   </dependency>
   <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId> 
   </dependency>
   ```
2. **Configure** `RestTemplate` **with** `@LoadBalanced`:
    ```java
    package com.example.config;
   
    import org.springframework.cloud.client.loadbalancer.LoadBalanced;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.web.client.RestTemplate;
    
    @Configuration
    public class AppConfig {
    
        @Bean
        @LoadBalanced // This annotation tells Spring Cloud to intercept calls made with this RestTemplate
                      // and apply client-side load balancing using the service ID.
        public RestTemplate restTemplate() {
            return new RestTemplate();
        }
    }
    ```
   
3. **Use Service ID in URL**:
   Now, instead of using a direct URL like http://localhost:8081, you use the registered service ID of the target microservice (e.g., PRODUCT-SERVICE). Spring Cloud LoadBalancer will resolve this service ID to an actual instance URL using the service registry (e.g., Eureka).
   ```java
    package com.example.service;

    import com.example.model.Product;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.web.client.RestTemplate;
    
    @Service
    public class ProductServiceClient {
    
        private final RestTemplate restTemplate;
        private final String PRODUCT_SERVICE_ID = "PRODUCT-SERVICE"; // Use the registered service ID
    
        @Autowired
        public ProductServiceClient(@LoadBalanced RestTemplate restTemplate) { // Inject the @LoadBalanced RestTemplate
            this.restTemplate = restTemplate;
        }
    
        public Product getProductById(Long id) {
            return restTemplate.getForObject("http://" + PRODUCT_SERVICE_ID + "/products/{id}", Product.class, id);
        }
    
        // ... other methods remain similar, just use PRODUCT_SERVICE_ID
    }
   ```

   In your `application.properties` (or `application.yml`) for the client service, you would typically configure your service discovery client:
   ```properties
    # application.properties for the client service
    spring.application.name=order-service
    eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/ # URL of your Eureka server
   ```
   
   And for the product service instances:
   ```properties
   # application.properties for product-service instance 1
   spring.application.name=PRODUCT-SERVICE
   server.port=8081
   eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/

   # application.properties for product-service instance 2
   spring.application.name=PRODUCT-SERVICE
   server.port=8082
   eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
   ```

   When `OrderService` calls `http://PRODUCT-SERVICE/products/{id}`, `Spring Cloud LoadBalancer` (backed by Eureka) will intercept this, resolve `PRODUCT-SERVICE` to an available instance (e.g., `localhost:8081` or `localhost:8082`), and then execute the HTTP request.

### 2. RestClient (Modern Synchronous Replacement)
`RestClient` is a newer synchronous HTTP client introduced in Spring Framework 6 and Spring Boot 3, designed as a modern, streamlined replacement for `RestTemplate`. It provides a fluent API similar to `WebClient` but for blocking (synchronous) calls, making it more readable and easier to use while avoiding the complexities of the reactive stack if not needed.

#### Fluent API Design, Setup, and Use Cases
`RestClient` follows a builder pattern, allowing you to chain method calls to construct your HTTP requests. This fluent API significantly improves readability and reduces boilerplate code compared to `RestTemplate`.

#### Setup
You can create a `RestClient` instance using `RestClient.create()` or `RestClient.builder()`. For more complex configurations (e.g., base URL, default headers, interceptors), it's best to create it as a `@Bean`.

1. **Add Dependency**:
   `RestClient` is part of `spring-web` (which is included by `spring-boot-starter-web`), so no additional dependencies are typically needed beyond your core Spring Boot web starter.

2. **Configuration with** `@Bean` **(Recommended for centralized configuration):**
   ```java
    package com.example.config;

    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.web.client.RestClient;
    
    @Configuration
    public class RestClientConfig {
    
        @Bean
        public RestClient productServiceClient(RestClient.Builder builder) {
            return builder
                    .baseUrl("http://product-service") // Base URL for the product service
                    .defaultHeader("Accept", "application/json") // Default header
                    .build();
        }
    
        // If you need a generic RestClient without a base URL, you can also expose it
        @Bean
        public RestClient genericRestClient(RestClient.Builder builder) {
            return builder.build();
        }
    }
   ```
   
   **Note**: For client-side load balancing with `RestClient`, you'd still rely on `Spring Cloud LoadBalancer`. Instead of `@LoadBalanced` on the `RestClient` bean itself, you'd ensure your base URL uses the service ID and `Spring Cloud LoadBalancer` is on the classpath. `RestClient` transparently integrates with `Spring Cloud LoadBalancer` if it's available.

   **Use Cases**: <br>
   `RestClient` is ideal for:
    - Synchronous microservice-to-microservice communication.
    - Interacting with external REST APIs where a blocking client is sufficient.
    - When you prefer a more modern, readable API over `RestTemplate` but don't need the reactive capabilities of `WebClient`.

   **How it Improves Readability and Testing**:
   - **Readability**: The chained method calls make the request construction flow clear and concise.
     ```java
     // RestTemplate
     restTemplate.postForObject(url, requestBody, Response.class);
    
     // RestClient
     restClient.post()
     .uri(url)
     .body(requestBody)
     .retrieve()
     .body(Response.class);
     ```

     This `RestClient` syntax reads more like a natural language description of the HTTP request.
   
   - **Testing**: `RestClient` is designed with testability in mind. You can easily mock `RestClient` instances or use `MockRestServiceServer` (similar to `RestTemplate`) for unit testing. The fluent API often makes setting up expectations for mocks more intuitive

   **Example Usage**
   ```java
    package com.example.service;

    import com.example.model.Product;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.web.client.RestClient;
    import org.springframework.web.client.HttpClientErrorException;
    import org.springframework.web.client.HttpServerErrorException;
    
    @Service
    public class ProductServiceClient {
    
        private final RestClient productServiceClient; // Injected RestClient with base URL
    
        // Inject the RestClient configured for product-service
        @Autowired
        public ProductServiceClient(RestClient productServiceClient) {
            this.productServiceClient = productServiceClient;
        }
    
        public Product getProductById(Long id) {
            try {
                return productServiceClient.get()
                        .uri("/products/{id}", id) // Path relative to base URL
                        .retrieve() // Initiates the request and retrieves the response
                        .body(Product.class); // Deserializes the response body to Product.class
            } catch (HttpClientErrorException.NotFound ex) {
                System.err.println("Product not found: " + ex.getMessage());
                return null;
            } catch (HttpClientErrorException ex) {
                System.err.println("Client error: " + ex.getStatusCode() + " - " + ex.getResponseBodyAsString());
                throw new RuntimeException("Failed to fetch product due to client error", ex);
            } catch (HttpServerErrorException ex) {
                System.err.println("Server error: " + ex.getStatusCode() + " - " + ex.getResponseBodyAsString());
                throw new RuntimeException("Failed to fetch product due to server error", ex);
            }
        }
    
        public Product createProduct(Product product) {
            try {
                return productServiceClient.post()
                        .uri("/products")
                        .body(product)
                        .retrieve()
                        .body(Product.class);
            } catch (HttpClientErrorException | HttpServerErrorException ex) {
                System.err.println("Error creating product: " + ex.getStatusCode() + " - " + ex.getResponseBodyAsString());
                throw new RuntimeException("Failed to create product", ex);
            }
        }
    
        public void updateProduct(Long id, Product product) {
            try {
                productServiceClient.put()
                        .uri("/products/{id}", id)
                        .body(product)
                        .retrieve()
                        .toBodilessEntity(); // For requests without a response body
            } catch (HttpClientErrorException | HttpServerErrorException ex) {
                System.err.println("Error updating product: " + ex.getStatusCode() + " - " + ex.getResponseBodyAsString());
                throw new RuntimeException("Failed to update product", ex);
            }
        }
    
        public void deleteProduct(Long id) {
            try {
                productServiceClient.delete()
                        .uri("/products/{id}", id)
                        .retrieve()
                        .toBodilessEntity();
            } catch (HttpClientErrorException | HttpServerErrorException ex) {
                System.err.println("Error deleting product: " + ex.getStatusCode() + " - " + ex.getResponseBodyAsString());
                throw new RuntimeException("Failed to delete product", ex);
            }
        }
    }
   ```
   
   **Custom Error Handling with** `onStatus()`: <br>
   `RestClient` offers a more explicit way to handle status codes using `onStatus()`:
    ```java
    import org.springframework.http.HttpStatus;

    public Product getProductByIdWithStatusHandling(Long id) {
        return productServiceClient.get()
            .uri("/products/{id}", id)
            .retrieve()
            .onStatus(HttpStatus.NOT_FOUND::equals, (request, response) -> {
                throw new ResourceNotFoundException("Product with ID " + id + " not found.");
            })
            .onStatus(HttpStatus::is5xxServerError, (request, response) -> {
                throw new ServiceUnavailableException("Product service is currently unavailable.");
            })
            .body(Product.class);
    }
    ```

   This allows for more granular error handling directly within the fluent API chain.

### 3. FeignClient (Declarative Style)
`FeignClient` (part of Spring Cloud OpenFeign) is a declarative web service client that significantly simplifies making HTTP calls. Instead of manually constructing requests, you define an interface with annotations that map to the REST endpoints of the target service. Feign then automatically implements this interface at runtime, handling the underlying HTTP communication, serialization/deserialization, and integration with Spring Cloud features like service discovery and load balancing.

#### Declarative Style and Integration with Spring Cloud
- **Declarative**: You declare what HTTP call you want to make, not how to make it. This eliminates a lot of boilerplate code.
- **Spring Cloud Integration**: Feign integrates seamlessly with Spring Cloud Eureka (for service discovery) and Spring Cloud LoadBalancer for client-side load balancing. It also supports circuit breakers (like Resilience4j or Sentinel) for enhanced fault tolerance.

**How it Simplifies Inter-Service Calls Through Interface-Based Configuration**: <br>
Feign leverages annotations similar to Spring Web (e.g., `@GetMapping`, `@PostMapping`, `@PathVariable`, `@RequestParam`). You define an interface, annotate it with `@FeignClient`, and Feign does the rest.

1. **Add Dependencies** <br>
2. **Enable Feign Clients**: <br>
3. 