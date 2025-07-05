# Exception Handling

The ability to manage errors in a structured, centralized, and user-friendly way is paramount for robust RESTful and web applications. Without effective exception handling, applications can expose sensitive details, provide inconsistent error messages, or even crash, leading to a poor user experience and security vulnerabilities. Spring Boot, building upon Spring Framework's core capabilities, offers a comprehensive and powerful set of tools to intercept, translate, and respond to exceptions consistently across the entire application.

### The Need for Structured and Centralized Error Management
In typical applications, errors can originate from various sources: invalid user input, unavailable resources, database issues, or external service failures.

### Foundational Techniques: `try-catch` Blocks
Java's `try-catch` blocks are the fundamental way to handle exceptions. In Spring Boot, you might use them within your controller or service layers for specific, localized error handling.
#### Example: Basic `try-catch` in a service layer
```java
@Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    public Product getProductById(Long id) {
        try {
            return productRepository.findById(id)
                    .orElseThrow(() -> new ResourceNotFoundException("Product not found with ID: " + id));
        } catch (DataAccessException ex) {
            // Log the exception and potentially rethrow a more specific business exception
            throw new DatabaseOperationException("Error accessing product data", ex);
        }
    }

    public Product createProduct(Product product) {
        try {
            return productRepository.save(product);
        } catch (DataIntegrityViolationException ex) {
            throw new DuplicateResourceException("Product with name '" + product.getName() + "' already exists", ex);
        }
    }
}
```

While `try-catch` blocks are essential for localized error handling, relying solely on them for all exceptions can lead to:
- **Code Duplication**: Repeated `try-catch` blocks across multiple methods.
- **Scattered Logic**: Error handling logic mixed with business logic, reducing readability and maintainability.
- **Lack of Centralization**: No single place to define how various types of exceptions are handled globally.

### Scalable and Maintainable Patterns: `@ExceptionHandler`, `@ControllerAdvice`, and `ResponseEntityExceptionHandler`
Spring Boot offers more sophisticated and centralized approaches to exception handling, allowing for cleaner code and consistent error responses.

#### `@ExceptionHandler`
The `@ExceptionHandler` annotation allows you to define methods within a controller that will handle specific exceptions thrown by handler methods within that same controller.
```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @Autowired
    private ProductService productService;

    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable Long id) {
        Product product = productService.getProductById(id);
        return ResponseEntity.ok(product);
    }

    @PostMapping
    public ResponseEntity<Product> createProduct(@RequestBody Product product) {
        Product createdProduct = productService.createProduct(product);
        return new ResponseEntity<>(createdProduct, HttpStatus.CREATED);
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFoundException(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
                HttpStatus.NOT_FOUND.value(),
                ex.getMessage(),
                System.currentTimeMillis(),
                "/api/products/{id}"
        );
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicateResourceException(DuplicateResourceException ex) {
        ErrorResponse error = new ErrorResponse(
                HttpStatus.CONFLICT.value(),
                ex.getMessage(),
                System.currentTimeMillis(),
                "/api/products"
        );
        return new ResponseEntity<>(error, HttpStatus.CONFLICT);
    }
}
```

In this example, `handleResourceNotFoundException` will catch `ResourceNotFoundException` thrown within `ProductController` and return a structured error response with `404 Not Found` status.

#### `@ControllerAdvice`
While `@ExceptionHandler` works for a single controller, `@ControllerAdvice` (or `@RestControllerAdvice` for RESTful services) provides a global mechanism for exception handling across multiple controllers. This allows you to centralize your error handling logic in one place, keeping your controllers clean and focused on business logic.
```java
// GlobalExceptionHandler.java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFoundException(ResourceNotFoundException ex, WebRequest request) {
        ErrorResponse error = new ErrorResponse(
                HttpStatus.NOT_FOUND.value(),
                ex.getMessage(),
                System.currentTimeMillis(),
                request.getDescription(false).replace("uri=", "")
        );
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicateResourceException(DuplicateResourceException ex, WebRequest request) {
        ErrorResponse error = new ErrorResponse(
                HttpStatus.CONFLICT.value(),
                ex.getMessage(),
                System.currentTimeMillis(),
                request.getDescription(false).replace("uri=", "")
        );
        return new ResponseEntity<>(error, HttpStatus.CONFLICT);
    }

    // Generic exception handler for unhandled exceptions
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(Exception ex, WebRequest request) {
        // Log the exception details for debugging
        System.err.println("An unhandled exception occurred: " + ex.getMessage());
        ex.printStackTrace();

        ErrorResponse error = new ErrorResponse(
                HttpStatus.INTERNAL_SERVER_ERROR.value(),
                "An unexpected error occurred. Please try again later.",
                System.currentTimeMillis(),
                request.getDescription(false).replace("uri=", "")
        );
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

Now, any `ResourceNotFoundException` or `DuplicateResourceException` thrown from any controller in your application will be handled by `GlobalExceptionHandler`. The `WebRequest` object can be used to extract request details like the URI.

#### `ResponseEntityExceptionHandler`
For handling Spring MVC-specific exceptions, `ResponseEntityExceptionHandler` is a convenient base class provided by Spring. It provides `@ExceptionHandler` methods for common Spring MVC exceptions (e.g., `MethodArgumentNotValidException`, `HttpMessageNotReadableException`, `HttpRequestMethodNotSupportedException`) and allows you to override them to customize the error response.
```java
// CustomResponseEntityExceptionHandler.java
@ControllerAdvice
public class CustomResponseEntityExceptionHandler extends ResponseEntityExceptionHandler {

    // You can override existing methods to customize responses for Spring MVC exceptions
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex,
                                                                  HttpHeaders headers,
                                                                  HttpStatus status,
                                                                  WebRequest request) {
        List<String> errors = ex.getBindingResult()
                .getFieldErrors()
                .stream()
                .map(error -> error.getField() + ": " + error.getDefaultMessage())
                .collect(Collectors.toList());

        ValidationErrorResponse error = new ValidationErrorResponse(
                HttpStatus.BAD_REQUEST.value(),
                "Validation Failed",
                System.currentTimeMillis(),
                request.getDescription(false).replace("uri=", ""),
                errors
        );
        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }

    @Override
    protected ResponseEntity<Object> handleHttpMessageNotReadable(HttpMessageNotReadableException ex,
                                                                  HttpHeaders headers,
                                                                  HttpStatus status,
                                                                  WebRequest request) {
        ErrorResponse error = new ErrorResponse(
                HttpStatus.BAD_REQUEST.value(),
                "Malformed JSON request body",
                System.currentTimeMillis(),
                request.getDescription(false).replace("uri=", "")
        );
        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }

    // You can still add your custom exception handlers here
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFoundException(ResourceNotFoundException ex, WebRequest request) {
        ErrorResponse error = new ErrorResponse(
                HttpStatus.NOT_FOUND.value(),
                ex.getMessage(),
                System.currentTimeMillis(),
                request.getDescription(false).replace("uri=", "")
        );
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
}
```

`ResponseEntityExceptionHandler` is particularly useful for handling common validation errors and malformed requests, providing a solid foundation for your global exception handling strategy.

### Creating and Throwing Custom Exceptions
Custom exceptions allow you to represent specific business errors in a meaningful way, making your code more readable and your error handling more precise.
```java
// ResourceNotFoundException.java
@ResponseStatus(HttpStatus.NOT_FOUND) // This annotation automatically maps to HTTP 404
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

// InvalidRequestException.java
@ResponseStatus(HttpStatus.BAD_REQUEST) // This annotation automatically maps to HTTP 400
public class InvalidRequestException extends RuntimeException {
    public InvalidRequestException(String message) {
        super(message);
    }
}

// DuplicateResourceException.java
public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) {
        super(message);
    }

    public DuplicateResourceException(String message, Throwable cause) {
        super(message, cause);
    }
}

// DatabaseOperationException.java
public class DatabaseOperationException extends RuntimeException {
    public DatabaseOperationException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### Mapping Custom Exceptions to HTTP Status Codes:
- `@ResponseStatus`: As shown above, you can directly annotate your custom exception classes with `@ResponseStatus` to automatically map them to a specific HTTP status code. When this exception is thrown, Spring will automatically set the HTTP status.
- Returning `ResponseEntity`: For more control over the response body (e.g., adding a custom error object), it's generally preferred to catch the exception in an `@ExceptionHandler` method (within `@ControllerAdvice`) and return a `ResponseEntity` with the desired HTTP status and a structured error object. This allows you to include meaningful error details.

### Handling Validation Errors
Validation is crucial for robust APIs. Spring Boot integrates with Java Bean Validation (JSR 380) to validate request bodies and path variables.
- `@Valid` or `@Validated`: Used on controller method parameters to trigger validation.
- `MethodArgumentNotValidException`: Thrown when `@Valid` or `@Validated` fails on a method argument (e.g., a `@RequestBody` object). This is typically handled by `ResponseEntityExceptionHandler`.
- `ConstraintViolationException`: Thrown for validation failures on method parameters (e.g., `@RequestParam`, `@PathVariable`) when using Spring's method validation (`@Validated` on the controller class).
```java
// Example of a DTO with validation annotations
public class ProductCreationRequest {
    @NotBlank(message = "Product name is required")
    @Size(min = 3, max = 50, message = "Product name must be between 3 and 50 characters")
    private String name;

    @NotNull(message = "Price is required")
    @Min(value = 0, message = "Price must be non-negative")
    private BigDecimal price;

    // Getters and Setters
}

@RestController
@RequestMapping("/api/products")
@Validated // Enable method validation for this controller
public class ProductController {

    @PostMapping
    public ResponseEntity<Product> createProduct(@Valid @RequestBody ProductCreationRequest request) {
        // ... business logic ...
        return new ResponseEntity<>(product, HttpStatus.CREATED);
    }

    @GetMapping("/search")
    public ResponseEntity<List<Product>> searchProducts(
            @RequestParam @Min(value = 1, message = "Min price must be at least 1") BigDecimal minPrice) {
        // ... business logic ...
        return ResponseEntity.ok(products);
    }
}
```

### Handling in `CustomResponseEntityExceptionHandler` (as shown before):
The `handleMethodArgumentNotValid` method in our `CustomResponseEntityExceptionHandler` is specifically designed to process `MethodArgumentNotValidException`. It extracts field errors and constructs a `ValidationErrorResponse`. <br>
For `ConstraintViolationException`, you would add a separate `@ExceptionHandler` method to `GlobalExceptionHandler` or `CustomResponseEntityExceptionHandler`.
```java
// In GlobalExceptionHandler or CustomResponseEntityExceptionHandler
@ExceptionHandler(ConstraintViolationException.class)
public ResponseEntity<ErrorResponse> handleConstraintViolationException(ConstraintViolationException ex, WebRequest request) {
    List<String> errors = ex.getConstraintViolations().stream()
            .map(violation -> violation.getPropertyPath() + ": " + violation.getMessage())
            .collect(Collectors.toList());

    ValidationErrorResponse error = new ValidationErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Validation Failed",
            System.currentTimeMillis(),
            request.getDescription(false).replace("uri=", ""),
            errors
    );
    return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
}
```

### Consistent Error Response Structure
A consistent error response structure is crucial for client-side consumption. A common structure includes:
- `timestamp`: When the error occurred.
- `status`: HTTP status code (e.g., 400, 404, 500).
- `error`: A brief, human-readable description of the HTTP status (e.g., "Bad Request", "Not Found").
- `message`: A more specific error message from the application (e.g., "Product not found with ID: 123", "Product name is required").
- `path`: The request URI.
- `details` (optional for validation errors): A list of specific validation errors.
```java
// ErrorResponse.java
public class ErrorResponse {
    private int status;
    private String error;
    private String message;
    private long timestamp;
    private String path;

    public ErrorResponse(int status, String message, long timestamp, String path) {
        this.status = status;
        this.error = HttpStatus.valueOf(status).getReasonPhrase();
        this.message = message;
        this.timestamp = timestamp;
        this.path = path;
    }

    // Getters
}

// ValidationErrorResponse.java (extends ErrorResponse for validation details)
public class ValidationErrorResponse extends ErrorResponse {
    private List<String> details;

    public ValidationErrorResponse(int status, String message, long timestamp, String path, List<String> details) {
        super(status, message, timestamp, path);
        this.details = details;
    }

    // Getter for details
}
```

## Advanced Topics
### Exception Handling in Asynchronous Controllers
When using `@Async` or Spring WebFlux (reactive programming), the traditional exception handling mechanisms might behave differently.
- `@Async`: If an exception is thrown from an `@Async` method, it's typically caught by the `AsyncUncaughtExceptionHandler` that you can configure. This handler can log the exception or perform other actions. It won't directly propagate back to the HTTP client in the same way as a synchronous controller. You might need to consider how to communicate these asynchronous errors back to the client, perhaps through a separate mechanism or by storing the error status in a database.
```java
// Configure AsyncUncaughtExceptionHandler
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }
}

// CustomAsyncExceptionHandler.java
public class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {
    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        System.err.println("Async Exception: " + ex.getMessage());
        System.err.println("Method name: " + method.getName());
        for (Object param : params) {
            System.err.println("Parameter value: " + param);
        }
        // Log the exception, send alert, etc.
    }
}
```
- **Spring WebFlux**: In reactive applications, exceptions are handled within the reactive stream. Operators like `onErrorResume`, `onErrorReturn`, and `onErrorStop` are used to gracefully handle errors in the Flux or Mono pipeline. `@ControllerAdvice` still works for WebFlux, and you can define `@ExceptionHandler` methods that return `Mono<ResponseEntity<ErrorResponse>>`.
```
// WebFlux example
@RestControllerAdvice
public class ReactiveExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public Mono<ResponseEntity<ErrorResponse>> handleResourceNotFoundException(ResourceNotFoundException ex, ServerWebExchange exchange) {
        ErrorResponse error = new ErrorResponse(
                HttpStatus.NOT_FOUND.value(),
                ex.getMessage(),
                System.currentTimeMillis(),
                exchange.getRequest().getURI().getPath()
        );
        return Mono.just(new ResponseEntity<>(error, HttpStatus.NOT_FOUND));
    }
}

@RestController
@RequestMapping("/api/reactive-products")
public class ReactiveProductController {

    @GetMapping("/{id}")
    public Mono<ResponseEntity<Product>> getProduct(@PathVariable Long id) {
        return Mono.just(id)
                .flatMap(productId -> productRepository.findById(productId)) // Assume reactive repository
                .switchIfEmpty(Mono.error(new ResourceNotFoundException("Product not found")))
                .map(ResponseEntity::ok)
                .onErrorResume(ResourceNotFoundException.class, ex -> Mono.just(new ResponseEntity<>(HttpStatus.NOT_FOUND))); // Local handling
    }
}
```

### Exception Handling in Filters and Interceptors
- **Filters (Servlet Filters or WebFlux Filters)**: If an exception occurs in a filter before it reaches a controller, `@ControllerAdvice` might not intercept it. You might need to add a dedicated `Filter` (e.g., `ErrorHandlingFilter`) at the beginning of your filter chain to catch exceptions and produce a proper error response. You can use a `HandlerExceptionResolver` or manually construct the `HttpServletResponse`.
- **Interceptors (Spring Handler Interceptors)**: Interceptors have `preHandle`, `postHandle`, and `afterCompletion` methods. Exceptions in `preHandle` can be caught by `@ControllerAdvice`. However, if an exception occurs in `postHandle` or `afterCompletion`, it might require specific handling within the interceptor or a custom `HandlerExceptionResolver`. The `afterCompletion` method receives an `Exception` object, allowing for post-processing of exceptions.

### Exception Handling in REST Clients (Feign or RestTemplate)
When your Spring Boot application acts as a client to other REST services, you need to handle exceptions that occur during these calls.
- `RestTemplate`: `RestTemplate` throws `RestClientException` (and its subclasses like `HttpClientErrorException` for 4xx errors and `HttpServerErrorException` for 5xx errors). You can use `try-catch` blocks or configure a `ResponseErrorHandler` for more centralized handling.
```java
// RestTemplate example with ResponseErrorHandler
@Service
public class ExternalProductService {

    private final RestTemplate restTemplate;

    public ExternalProductService(RestTemplateBuilder restTemplateBuilder) {
        this.restTemplate = restTemplateBuilder
                .errorHandler(new CustomRestTemplateResponseErrorHandler())
                .build();
    }

    public Product getExternalProduct(Long id) {
        try {
            return restTemplate.getForObject("http://external-api/products/" + id, Product.class);
        } catch (HttpClientErrorException ex) {
            if (ex.getStatusCode() == HttpStatus.NOT_FOUND) {
                throw new ResourceNotFoundException("External product not found: " + id);
            }
            throw new ExternalServiceException("Error calling external product service", ex);
        }
    }
}

// CustomRestTemplateResponseErrorHandler.java
public class CustomRestTemplateResponseErrorHandler implements ResponseErrorHandler {

    @Override
    public boolean hasError(ClientHttpResponse response) throws IOException {
        return response.getStatusCode().is4xxClientError() || response.getStatusCode().is5xxServerError();
    }

    @Override
    public void handleError(ClientHttpResponse response) throws IOException {
        if (response.getStatusCode().is4xxClientError()) {
            throw new HttpClientErrorException(response.getStatusCode(), response.getStatusText(),
                    response.getHeaders(), response.getBody().readAllBytes(), null);
        } else if (response.getStatusCode().is5xxServerError()) {
            throw new HttpServerErrorException(response.getStatusCode(), response.getStatusText(),
                    response.getHeaders(), response.getBody().readAllBytes(), null);
        }
    }
}
```
- **Feign Client**: Feign integrates well with Spring. When a Feign client makes a call, it can throw `FeignException` (or its subclasses) for non-2xx responses. You can configure a custom `ErrorDecoder` to translate these exceptions into your application-specific exceptions.
```java
// Feign client example
@FeignClient(name = "external-product-service", url = "http://external-api", configuration = FeignClientConfig.class)
public interface ExternalProductFeignClient {

    @GetMapping("/products/{id}")
    Product getProduct(@PathVariable("id") Long id);
}

// FeignClientConfig.java
@Configuration
public class FeignClientConfig {

    @Bean
    public ErrorDecoder errorDecoder() {
        return new CustomFeignErrorDecoder();
    }
}

// CustomFeignErrorDecoder.java
public class CustomFeignErrorDecoder implements ErrorDecoder {

    private final ErrorDecoder defaultErrorDecoder = new DefaultErrorDecoder();

    @Override
    public Exception decode(String methodKey, Response response) {
        if (response.status() == 404) {
            return new ResourceNotFoundException("Feign: External resource not found");
        }
        // Let the default decoder handle other errors
        return defaultErrorDecoder.decode(methodKey, response);
    }
}
```

### Logging Exceptions Effectively
Effective logging is critical for debugging and auditing.
- **Logging Frameworks**: Spring Boot uses SLF4J with Logback by default. Use `org.slf4j.Logger` for logging.
- **Logging Levels**: Use appropriate logging levels (e.g., `ERROR` for unhandled exceptions, `WARN` for recoverable errors, `INFO` for general information, DEBUG for detailed debugging).
- **Structured Logging**: Consider structured logging (e.g., JSON format) for easier parsing by log aggregators (ELK stack, Splunk).
- **Contextual Information**: Always include contextual information in your logs, such as request ID, user ID, method name, and relevant parameters.
- **Full Stack Traces**: When logging exceptions, ensure you log the full stack trace (`ex.printStackTrace()` or `logger.error("Error occurred", ex)`) to aid in diagnosis.
```java
// In GlobalExceptionHandler
private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);

@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleGenericException(Exception ex, WebRequest request) {
    logger.error("An unhandled exception occurred during request to {}: {}",
            request.getDescription(false).replace("uri=", ""), ex.getMessage(), ex); // Logs stack trace

    ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "An unexpected error occurred. Please try again later.",
            System.currentTimeMillis(),
            request.getDescription(false).replace("uri=", "")
    );
    return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
}
```

### Best Practices
1. **Avoid Exception Leaks**: Never expose internal implementation details, database errors, or stack traces directly to the client. Always translate them into user-friendly, non-sensitive error messages.
2. **Return Precise HTTP Status Codes**:
   - `200 OK`: Success (for GET, PUT, POST that returns data).
   - `201 Created`: Resource created successfully (for POST).
   - `204 No Content`: Successful request with no content to return (e.g., DELETE).
   - `400 Bad Request`: Invalid input, malformed request, validation errors (e.g., `MethodArgumentNotValidException`, `InvalidRequestException`).
   - `401 Unauthorized`: Authentication required or failed.
   - `403 Forbidden`: Authenticated, but lacks necessary permissions.
   - `404 Not Found`: Resource does not exist (e.g., `ResourceNotFoundException`).
   - `405 Method Not Allowed`: HTTP method not supported for the resource.
   - `409 Conflict`: Request conflicts with current state of the resource (e.g., DuplicateResourceException, optimistic locking failure).
   - `415 Unsupported Media Type`: Request body has an unsupported media type.
   - `429 Too Many Requests`: Rate limiting applied.
   - `500 Internal Server Error`: Generic server-side error, unhandled exception.
   - `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout`: Issues with downstream services or network.
3. **Design Centralized, Reusable Exception Handlers**: Use `@ControllerAdvice` for global exception handling. This promotes consistency and reduces boilerplate code.
4. **Create Custom Business Exceptions**: Define specific exceptions for business-level errors (e.g., `ProductNotFoundException`, `InsufficientStockException`, `UserAlreadyExistsException`). This makes your error handling more semantic.
5. **Separate Concerns**: Keep business logic clean and free from excessive error handling `try-catch` blocks. Delegate error handling to dedicated `@ExceptionHandler` methods.
6. **Don't Catch Generic** `Exception` **Unnecessarily**: Catching `Exception` too broadly can hide specific errors that you should be handling differently. Only catch `Exception` as a last resort in your global handler to provide a generic "something went wrong" message and log the full details.
7. **Consider Using** `ProblemDetail` **(Spring 6 / Spring Boot 3)**: Spring Framework 6 introduced `ProblemDetail` (RFC 7807) for a standardized way to represent API errors. This can further enhance consistency and interoperability.
```
// Example using ProblemDetail (Spring Boot 3.x)
// In GlobalExceptionHandler
@ExceptionHandler(ResourceNotFoundException.class)
public ProblemDetail handleResourceNotFoundException(ResourceNotFoundException ex, HttpServletRequest request) {
    ProblemDetail problemDetail = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    problemDetail.setTitle("Resource Not Found");
    problemDetail.setProperty("path", request.getRequestURI());
    problemDetail.setProperty("timestamp", System.currentTimeMillis());
    return problemDetail;
}
```

### Common Pitfalls
- **Catching `Exception` and Swallowing It**: Catching `Exception` without rethrowing or logging can lead to silent failures, making debugging impossible.
- **Exposing Raw Stack Traces**: A major security and usability flaw. Always filter or translate error details.
- **Inconsistent Error Formats**: Different API endpoints returning different error structures confuse clients.
- **Overly Generic Error Messages**: Messages like "An error occurred" are unhelpful. Provide specific, actionable messages.
- **Mixing Business Logic and Error Reporting**: Cluttering business methods with `try-catch` blocks for every possible error makes code hard to read and maintain.
- **Not Handling Checked Exceptions Properly**: While Spring mostly deals with unchecked exceptions in web contexts, be mindful of checked exceptions in service layers. Convert them to unchecked business exceptions if they need to propagate across architectural layers to avoid `throws` clauses everywhere.