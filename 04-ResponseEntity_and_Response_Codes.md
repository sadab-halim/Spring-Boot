# ResponseEntity and Response Codes

## The Basics of HTTP Response Handling and Why It's Essential
HTTP is a request-response protocol. When a client (e.g., a web browser, a mobile app, or another service) sends an HTTP request to a server, the server processes it and sends back an HTTP response. This response isn't just about the data; it's also about how the request was handled.

A typical HTTP response consists of:
1. **Status Line**: Includes the HTTP version and a three-digit status code, followed by a reason phrase (e.g., `HTTP/1.1 200 OK`).
2. **Headers**: Key-value pairs providing metadata about the response (e.g., `Content-Type: application/json`, `Cache-Control: no-cache`).
3. **Body (Optional)**: The actual data payload of the response (e.g., a JSON object, an HTML page, an image).

### Why controlling response structure and status codes is essential in building RESTful APIs:
- Clarity and Understandability
- Client-Side Logic
- Error Handling
- API Documentation and Discoverability
- Caching
- Idempotency
  
## Introducing ResponseEntity
`ResponseEntity` is a class in Spring Framework (part of `org.springframework.http`) that encapsulates the entire HTTP response. It allows you to return not just the data (the response body), but also the HTTP status code and any custom headers. This gives you unparalleled control over how your API responds to client requests.

When you return a `ResponseEntity` from a Spring controller method, Spring's message converters will automatically handle the serialization of the `ResponseEntity`'s body into the appropriate format (e.g., JSON, XML) based on the `Content-Type` header and the client's Accept header.

### `ResponseEntity` vs. Returning Objects/Strings Directly
You might wonder why you can't just return a plain Java object (POJO) or a `String` directly from your controller methods. While Spring Boot with `@RestController` will automatically serialize POJOs to JSON (or XML) and set a `200 OK` status, this approach lacks flexibility:
 - Limited Control over Status Cod
 - No Header Control
 - Ambiguity for Empty Responses

`ResponseEntity` addresses these limitations by providing explicit control over all aspects of the HTTP response.

### How `ResponseEntity` is Used in Controller Methods
`ResponseEntity` is typically used with a builder pattern, making it highly readable and flexible.

Here's the basic structure:
```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class MyController {

    @GetMapping("/hello")
    public ResponseEntity<String> sayHello() {
        return new ResponseEntity<>("Hello from Spring Boot!", HttpStatus.OK);
    }
}
```

Or, using the more common builder pattern:

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class MyController {

    @GetMapping("/greet")
    public ResponseEntity<String> greetUser() {
        return ResponseEntity.ok("Greetings, user!"); // Shorthand for 200 OK
    }
}
```

The `T` in `ResponseEntity<T>` represents the type of the response body. It can be any Java object, including custom POJOs, `List`s, `String`s, or even `Void` if there's no body.

### Examples of Setting Common Response Codes
#### 1. `200 OK` - Success
Used for a successful request where the response body contains the requested resource.
```java
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ProductController {

    // Assuming a ProductService exists
    // private final ProductService productService;

    @GetMapping("/products/{id}")
    public ResponseEntity<Product> getProductById(@PathVariable Long id) {
        // Product product = productService.findById(id);
        Product product = new Product(id, "Example Product", 29.99); // Mock data

        if (product != null) {
            return ResponseEntity.ok(product); // Returns 200 OK with product data
        } else {
            return ResponseEntity.notFound().build(); // Returns 404 Not Found
        }
    }
}

class Product { // Simple POJO for demonstration
    private Long id;
    private String name;
    private double price;

    public Product(Long id, String name, double price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }
    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public double getPrice() { return price; }
    public void setPrice(double price) { this.price = price; }
}
```

#### 2. `201 Created` - Resource Creation
Used when a new resource has been successfully created on the server as a result of a POST request. It's a best practice to include a Location header pointing to the URI of the newly created resource.

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.support.ServletUriComponentsBuilder;
import java.net.URI;

@RestController
public class UserController {

    // private final UserService userService;

    @PostMapping("/users")
    public ResponseEntity<User> createUser(@RequestBody User newUser) {
        // User createdUser = userService.save(newUser);
        User createdUser = new User(1L, newUser.getName(), newUser.getEmail()); // Mock creation

        // Construct the URI for the newly created resource
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(createdUser.getId())
                .toUri();

        return ResponseEntity.created(location).body(createdUser); // Returns 201 Created with Location header and user data
    }
}

class User { // Simple POJO for demonstration
    private Long id;
    private String name;
    private String email;

    public User() {}
    public User(Long id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }
    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

#### 3. `204 No Content` - Successful Request with No Body
Used when a request has been successfully processed, but there's no content to return in the response body. This is common for `DELETE` operations or `PUT` updates where the client doesn't need the updated resource back.

```java
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ItemController {

    // private final ItemService itemService;

    @DeleteMapping("/items/{id}")
    public ResponseEntity<Void> deleteItem(@PathVariable Long id) {
        // boolean deleted = itemService.deleteById(id);
        boolean deleted = true; // Mock deletion

        if (deleted) {
            return ResponseEntity.noContent().build(); // Returns 204 No Content
        } else {
            return ResponseEntity.notFound().build(); // Returns 404 Not Found if item not found
        }
    }
}
```

Notice `ResponseEntity<Void>.` This indicates that the response body will be empty.

#### 4. `400 Bad Request` - Client Error (Malformed Request, Validation Errors)
Indicates that the server cannot process the request due to something that is perceived to be a client error (e.g., malformed syntax, invalid request parameters, validation failures).

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;
import java.util.HashMap;
import java.util.Map;

@RestController
public class OrderController {

    @PostMapping("/orders")
    public ResponseEntity<?> createOrder(@RequestBody OrderRequest orderRequest) {
        if (orderRequest.getProductId() == null || orderRequest.getQuantity() <= 0) {
            Map<String, String> errors = new HashMap<>();
            if (orderRequest.getProductId() == null) {
                errors.put("productId", "Product ID is required.");
            }
            if (orderRequest.getQuantity() <= 0) {
                errors.put("quantity", "Quantity must be greater than zero.");
            }
            return ResponseEntity.badRequest().body(errors); // Returns 400 Bad Request with error details
        }
        // Proceed with order creation
        // ...
        return ResponseEntity.status(HttpStatus.CREATED).body("Order created successfully.");
    }
}

class OrderRequest { // Simple POJO for demonstration
    private Long productId;
    private int quantity;

    public OrderRequest() {}
    public OrderRequest(Long productId, int quantity) {
        this.productId = productId;
        this.quantity = quantity;
    }
    // Getters and Setters
    public Long getProductId() { return productId; }
    public void setProductId(Long productId) { this.productId = productId; }
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
}
```

#### 5. `404 Not Found` - Resource Not Found
Indicates that the server cannot find the requested resource. This is typically used when a client tries to access a resource that does not exist at the given URI.

```java
// Already shown in getProductById and deleteItem examples
// ...
// return ResponseEntity.notFound().build();
```

#### 6. `409 Conflict` - Resource Conflict
Indicates that the request could not be completed due to a conflict with the current state of the target resource. This is often used in scenarios like:
 - Trying to create a resource that already exists (e.g., a user with a duplicate email).
 - Concurrent updates where the client's version of the resource is outdated.
```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AccountController {

    // private final AccountService accountService;

    @PostMapping("/accounts")
    public ResponseEntity<?> createAccount(@RequestBody Account newAccount) {
        // Assume accountService.accountExists(newAccount.getEmail()) checks for existing account
        boolean accountExists = newAccount.getEmail().equals("existing@example.com"); // Mock check

        if (accountExists) {
            return ResponseEntity.status(HttpStatus.CONFLICT).body("An account with this email already exists.");
        }
        // Proceed with account creation
        // ...
        return ResponseEntity.status(HttpStatus.CREATED).body(newAccount);
    }
}

class Account { // Simple POJO for demonstration
    private String email;
    private String password;

    public Account() {}
    public Account(String email, String password) {
        this.email = email;
        this.password = password;
    }
    // Getters and Setters
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
}
```

#### 7. `500 Internal Server Error` - Server-Side Error
Indicates that the server encountered an unexpected condition that prevented it from fulfilling the request. This is a generic error response for unhandled exceptions or unexpected server-side issues.
```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DataController {

    @GetMapping("/data")
    public ResponseEntity<String> fetchData() {
        try {
            // Simulate an unexpected server error
            if (Math.random() > 0.5) {
                throw new RuntimeException("Simulated database connection error!");
            }
            return ResponseEntity.ok("Data fetched successfully.");
        } catch (Exception e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body("An unexpected error occurred: " + e.getMessage());
        }
    }
}
```

Note: For `500 Internal Server Error`, it's generally better to handle this centrally using `@ControllerAdvice`, which we'll discuss next.

### Deep Dive into Real-World Use Cases
#### Returning `201 Created` with a `Location` Header
As demonstrated with `createUser`, sending a `201 Created` status code for resource creation is a cornerstone of RESTful design. The `Location` header is critical because it tells the client exactly where to find the newly created resource, enabling them to fetch it directly without needing to construct the URL themselves.

```java
import org.springframework.web.servlet.support.ServletUriComponentsBuilder;
import java.net.URI;

// ... inside a @RestController method
@PostMapping("/users")
public ResponseEntity<User> createUser(@RequestBody User newUser) {
    User createdUser = userService.save(newUser); // Actual service call

    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(createdUser.getId())
            .toUri();

    return ResponseEntity.created(location).body(createdUser);
}
```

`ServletUriComponentsBuilder` is a utility class provided by Spring MVC to easily construct URIs relative to the current request.

#### Using `ResponseEntity.noContent()` for Delete Operations
For `DELETE` operations, if the operation is successful and there's no data to return, `204 No Content` is the semantically correct response. `ResponseEntity.noContent().build()` is the concise way to achieve this.

```java
@DeleteMapping("/items/{id}")
public ResponseEntity<Void> deleteItem(@PathVariable Long id) {
    boolean deleted = itemService.deleteById(id);
    if (deleted) {
        return ResponseEntity.noContent().build();
    } else {
        return ResponseEntity.notFound().build();
    }
}
```

### Returning Custom Objects or Error Payloads with `ResponseEntity`
You can return any Java object as the body of your `ResponseEntity`. This is incredibly useful for providing structured error messages or complex data structures.

#### Custom Error Payload Example:
First, define a generic error response structure:
```java
// ErrorResponse.java
public class ErrorResponse {
    private String timestamp;
    private int status;
    private String error;
    private String message;
    private String path;

    public ErrorResponse(int status, String error, String message, String path) {
        this.timestamp = java.time.LocalDateTime.now().toString();
        this.status = status;
        this.error = error;
        this.message = message;
        this.path = path;
    }
    // Getters
    public String getTimestamp() { return timestamp; }
    public int getStatus() { return status; }
    public String getError() { return error; }
    public String getMessage() { return message; }
    public String getPath() { return path; }
}
```

Then, use it in your controller:

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;
import jakarta.servlet.http.HttpServletRequest;

@RestController
public class AnotherController {

    @GetMapping("/users/{id}")
    public ResponseEntity<?> getUser(@PathVariable Long id, HttpServletRequest request) {
        // Simulate user not found
        if (id < 1) { // Example of invalid ID
            ErrorResponse error = new ErrorResponse(
                    HttpStatus.BAD_REQUEST.value(),
                    HttpStatus.BAD_REQUEST.getReasonPhrase(),
                    "User ID cannot be less than 1.",
                    request.getRequestURI()
            );
            return ResponseEntity.badRequest().body(error);
        }

        // Simulate a specific user not found
        if (id == 99L) {
            ErrorResponse error = new ErrorResponse(
                    HttpStatus.NOT_FOUND.value(),
                    HttpStatus.NOT_FOUND.getReasonPhrase(),
                    "User with ID " + id + " not found.",
                    request.getRequestURI()
            );
            return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
        }

        // Return a successful response
        User user = new User(id, "John Doe", "john.doe@example.com");
        return ResponseEntity.ok(user);
    }
}
```

#### Wrapping Validation Errors or Exception Messages using Standard Error Structures
For validation errors, it's common to return `400 Bad Request` with a detailed error payload. Spring's validation API (JSR 303/349 with Hibernate Validator) integrates well here.
Consider a request body with `@Valid` annotation:

```java
import jakarta.validation.Valid;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ValidationController {

    @PostMapping("/validate")
    public ResponseEntity<String> validateUser(@Valid @RequestBody UserForm userForm) {
        // If validation passes, this method is executed
        return ResponseEntity.ok("User form is valid.");
    }
}

class UserForm {
    @NotBlank(message = "Name is required")
    private String name;

    @Min(value = 18, message = "Age must be at least 18")
    private int age;

    @Email(message = "Invalid email format")
    @NotBlank(message = "Email is required")
    private String email;

    // Getters and Setters
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

When validation fails, Spring will throw `MethodArgumentNotValidException`. To catch this and return a `400 Bad Request` with custom error details, you'd typically use `@ControllerAdvice`.

#### `ResponseEntity` with Spring’s Global Exception Handling (`@ControllerAdvice`)
`@ControllerAdvice` (or `@RestControllerAdvice`) is a powerful annotation that allows you to define global exception handlers for your Spring application. It centralizes error handling logic, ensuring consistent error responses across all your controllers. Within `@ControllerAdvice`, you'll often use `ResponseEntity` to construct the error response.

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import jakarta.servlet.http.HttpServletRequest;
import java.util.HashMap;
import java.util.Map;

@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationExceptions(
            MethodArgumentNotValidException ex, HttpServletRequest request) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
                errors.put(error.getField(), error.getDefaultMessage()));

        ErrorResponse errorResponse = new ErrorResponse(
                HttpStatus.BAD_REQUEST.value(),
                HttpStatus.BAD_REQUEST.getReasonPhrase(),
                "Validation failed",
                request.getRequestURI()
        );
        // You might want to add a 'details' field to ErrorResponse for validation errors
        // errorResponse.setDetails(errors);

        return ResponseEntity.badRequest().body(errorResponse);
    }

    @ExceptionHandler(ResourceNotFoundException.class) // Custom exception
    public ResponseEntity<ErrorResponse> handleResourceNotFoundException(
            ResourceNotFoundException ex, HttpServletRequest request) {
        ErrorResponse errorResponse = new ErrorResponse(
                HttpStatus.NOT_FOUND.value(),
                HttpStatus.NOT_FOUND.getReasonPhrase(),
                ex.getMessage(),
                request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(errorResponse);
    }

    @ExceptionHandler(RuntimeException.class) // Catch-all for unexpected errors
    public ResponseEntity<ErrorResponse> handleAllUncaughtExceptions(
            RuntimeException ex, HttpServletRequest request) {
        ErrorResponse errorResponse = new ErrorResponse(
                HttpStatus.INTERNAL_SERVER_ERROR.value(),
                HttpStatus.INTERNAL_SERVER_ERROR.getReasonPhrase(),
                "An unexpected error occurred: " + ex.getMessage(),
                request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(errorResponse);
    }
}

// Example custom exception
class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

This setup ensures that all validation errors are consistently returned as `400 Bad Request` with meaningful error messages, and `ResourceNotFoundException` (a custom exception you'd define) leads to a `404 Not Found`. Any other uncaught `RuntimeException` will result in a `500 Internal Server Error`, providing a fallback.

#### `ResponseEntity` and Swagger/OpenAPI Integration
When documenting your API using Swagger (now Open API), `ResponseEntity` plays a crucial role in accurately describing the possible responses for each endpoint. Libraries like `springdoc-openapi` automatically infer much of the API documentation from your Spring code, including the return types of `ResponseEntity`.

For more detailed documentation, especially for different error responses, you can use Swagger/OpenAPI annotations like `@ApiResponse` and `@Operation`.

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ProductDocumentationController {

    @Operation(summary = "Get a product by its ID")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "200", description = "Found the product",
                    content = { @Content(mediaType = "application/json",
                            schema = @Schema(implementation = Product.class)) }),
            @ApiResponse(responseCode = "400", description = "Invalid ID supplied",
                    content = @Content(mediaType = "application/json",
                            schema = @Schema(implementation = ErrorResponse.class))),
            @ApiResponse(responseCode = "404", description = "Product not found",
                    content = @Content(mediaType = "application/json",
                            schema = @Schema(implementation = ErrorResponse.class)))
    })
    @GetMapping("/products/{id}")
    public ResponseEntity<?> getProductByIdForDoc(@PathVariable Long id, HttpServletRequest request) {
        if (id <= 0) {
            return ResponseEntity.badRequest().body(new ErrorResponse(400, "Bad Request", "Product ID must be positive", request.getRequestURI()));
        }
        if (id == 1L) { // Simulate finding product
            return ResponseEntity.ok(new Product(id, "Documented Product", 100.0));
        } else { // Simulate not finding product
            return ResponseEntity.status(HttpStatus.NOT_FOUND).body(new ErrorResponse(404, "Not Found", "Product not found", request.getRequestURI()));
        }
    }
}
```

In this example, `@ApiResponses` is used to declare different possible HTTP responses, linking them to specific status codes, descriptions, and the schema of the response body, making your API documentation comprehensive.

### Best Practices for Using `ResponseEntity`
#### 1. Use Appropriate and Meaningful Status Codes:
- **2xx (Success)**: Use `200 OK` for general success, `201 Created` for new resource creation (with `Location` header), `204 No Content` for successful operations with no response body (like deletes or updates that don't need to return the resource).
- **4xx (Client Errors)**: Use `400 Bad Request` for invalid input (validation errors, malformed JSON), `401 Unauthorized` for missing/invalid authentication, `403 Forbidden` for insufficient authorization, `404 Not Found` for non-existent resources, `405 Method Not Allowed` if the HTTP method is not supported, `409 Conflict` for state conflicts (e.g., duplicate unique keys).
- **5xx (Server Errors)**: Use `500 Internal Server Error` for unexpected server-side issues, `503 Service Unavailable` for temporary server overload/maintenance.

 #### 2. Avoid Ambiguous Responses: 
Don't always return `200 OK` even for error scenarios. This makes it harder for clients to programmatically handle errors and blurs the lines between success and failure. For example, returning `200 OK` with a message "User not found" instead of `404 Not Found` is a common anti-pattern.

#### 3. Structure Response Bodies Consistently:
- For successful responses, the body should contain the requested resource or a confirmation message.
- For error responses, define a consistent error payload structure (like the `ErrorResponse` example) that includes details like a timestamp, status code, a short error type, and a more descriptive message. This helps clients parse and display errors uniformly.

#### 4. Centralize Error Handling with `@ControllerAdvice`: 
As shown, global exception handling is crucial for maintaining consistency and reducing boilerplate code in individual controller methods.

#### 5. Use `ResponseEntity.ok()` and other static methods: 
For common scenarios (`ok()`, `badRequest()`, `notFound()`, `noContent()`, `created()`), use the static helper methods for cleaner and more readable code.

#### 6. Consider `ResponseEntity<Void>` for Empty Bodies: 
When there's no meaningful content to return in the body, use `ResponseEntity<Void>` to explicitly indicate that the body is absent.

#### 7. Document Your Responses: 
Use Swagger/OpenAPI annotations (`@ApiResponse`) to clearly document all possible response codes and their corresponding body structures.

### Common Pitfalls
#### 1. Always Returning `200 OK` Even for Error Scenarios: 
This is perhaps the most common mistake. It forces clients to parse the response body to determine if an error occurred, making API consumption brittle and less intuitive.

#### 2. Misusing Status Codes:
- Using `500 Internal Server Error` for client-side errors (e.g., validation failures). `4xx` codes are for client errors.
- Using `404 Not Found` for business logic errors (e.g., "product out of stock" when the product exists). A `400 Bad Request` or a more specific 4xx code might be better, or even a `200 OK` with a status indicating the business constraint if the operation was partially successful or informative.

#### 3. Inconsistent Error Payload Structure: 
If every error response has a different JSON structure, clients will struggle to parse and display error messages consistently.

#### 4. Returning Sensitive Information in Error Messages: 
Avoid exposing stack traces, internal server details, or sensitive data in production error responses. Error messages should be informative enough for the client to understand the issue but not reveal system vulnerabilities.



