# Asynchronous Programming in Spring Boot with @Async

## What Asynchronous Execution Means in Java and How Spring Enables It with @Async
Spring provides a powerful and convenient declarative way to achieve asynchronous execution through the `@Async` annotation. When a method is annotated with `@Async`, Spring intercepts the method call and executes it on a separate thread, managed by a Spring `TaskExecutor`. This allows the original calling thread to proceed without blocking.

To enable `@Async` in a Spring Boot application, you need to use the `@EnableAsync` annotation on one of your `@Configuration` classes, typically your main application class
```java
// Main Spring Boot Application Class
@SpringBootApplication
@EnableAsync
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

### The Declarative Approach: Using @Async with @EnableAsync
The `@Async` annotation is applied directly to methods you want to execute asynchronously. Spring, behind the scenes, uses AOP (Aspect-Oriented Programming) to create a proxy around the bean containing the `@Async` method. When the method is invoked, the proxy intercepts the call and dispatches it to a `TaskExecutor` for execution on a separate thread.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

@Service
public class EmailService {

    private static final Logger logger = LoggerFactory.getLogger(EmailService.class);

    @Async
    public void sendEmail(String recipient, String subject, String body) {
        logger.info("Sending email to {} with subject: {} - Thread: {}", recipient, subject, body, Thread.currentThread().getName());
        try {
            Thread.sleep(3000); // Simulate long-running email sending process
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            logger.error("Email sending interrupted", e);
        }
        logger.info("Email sent to {} - Thread: {}", recipient, Thread.currentThread().getName());
    }
}
```

And a controller to trigger it:
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class MyController {

    @Autowired
    private EmailService emailService;

    @GetMapping("/send-email")
    public String triggerEmail() {
        long startTime = System.currentTimeMillis();
        emailService.sendEmail("user@example.com", "Welcome!", "Thanks for registering!");
        long endTime = System.currentTimeMillis();
        return "Email sending triggered. Time taken to return: " + (endTime - startTime) + " ms";
    }
}
```

When you hit /send-email, you'll notice that the HTTP response is returned almost immediately (within milliseconds), even though the sendEmail method has a 3-second Thread.sleep(). This is because sendEmail is executing asynchronously on a separate thread. The logs will confirm that the email sending logic runs on a different thread than the main HTTP request thread.

## Return Types in Asynchronous Methods
Asynchronous methods can have various return types, each with its own implications for how to retrieve results and handle potential errors.

### 1. `void`
- **Behavior**: The simplest return type. The caller fires and forgets; it cannot directly receive a result or be notified of completion.
- **Result Retrieval**: Not applicable.
- **Exception Handling**: Exceptions thrown within a void @Async method are typically caught by Spring's AsyncUncaughtExceptionHandler. If no custom handler is configured, the exception will be logged but not propagated back to the caller.

#### Example (already shown in `EmailService`)
```java
@Async
public void sendEmail(String recipient, String subject, String body) {
    // ...
}
```

### 2. `Future<T>`
- **Behavior**: Represents the result of an asynchronous computation. It provides methods to check if the computation is complete, wait for its completion, and retrieve the result. Future is a basic interface for asynchronous results in Java.
- **Result Retrieval**:
  - `get()`: Blocks until the computation is complete and then retrieves the result. Can throw `InterruptedException` or `ExecutionException`.
  - `get(long timeout, TimeUnit unit)`: Blocks for a specified timeout. Throws `TimeoutException` if the result is not available within the timeout.
  - `isDone()`: Returns `true` if the computation is complete.
  - `isCancelled()`: Returns `true` if the computation was cancelled.
- **Exception Handling**: Exceptions are wrapped in an `ExecutionException` and thrown when `get()` is called.

#### Example
```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.AsyncResult;
import org.springframework.stereotype.Service;
import java.util.concurrent.Future;

@Service
public class FileProcessor {

    @Async
    public Future<String> processFile(String filePath) {
        // Simulate file processing
        try {
            Thread.sleep(5000);
            if (filePath.contains("error")) {
                throw new RuntimeException("Error processing file: " + filePath);
            }
            return new AsyncResult<>("Processed: " + filePath + " successfully!");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return new AsyncResult<>(new IllegalStateException("File processing interrupted", e));
        } catch (Exception e) {
            return new AsyncResult<>(e); // Wrap exception in AsyncResult for Future
        }
    }
}
```

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;

@RestController
public class FileController {

    @Autowired
    private FileProcessor fileProcessor;

    @GetMapping("/process-file")
    public String triggerFileProcessing() throws ExecutionException, InterruptedException {
        Future<String> resultFuture = fileProcessor.processFile("data.txt");
        // Do other things while file is being processed
        // ...

        // Later, retrieve the result (this will block until done)
        String result = resultFuture.get();
        return "File processing result: " + result;
    }

    @GetMapping("/process-file-error")
    public String triggerFileProcessingWithError() {
        Future<String> resultFuture = fileProcessor.processFile("error_data.txt");
        try {
            String result = resultFuture.get();
            return "File processing result: " + result;
        } catch (InterruptedException | ExecutionException e) {
            return "Error during file processing: " + e.getMessage();
        }
    }
}
```

### 3. `CompletableFuture<T>`
- **Behavior**: Introduced in Java 8, `CompletableFuture` is a more powerful and flexible evolution of `Future`. It supports chaining multiple asynchronous operations, combining results, and handling errors in a non-blocking, declarative way. It implements Future and provides a rich API for callbacks.
- **Result Retrieval**:
  - `get()`: Same as `Future.get()`.
  - `join()`: Similar to `get()`, but throws an unchecked CompletionException instead of ExecutionException.
  - `thenApply(Function)`: Applies a function to the result when it becomes available.
  - `thenAccept(Consumer)`: Consumes the result when it becomes available.
  - `thenRun(Runnable)`: Executes a `Runnable` when the computation completes.
  - `allOf(CompletableFuture...)`: Waits for all given `CompletableFutures` to complete.
  - `anyOf(CompletableFuture...)`: Waits for any of the given `CompletableFutures` to complete.
- **Exception Handling**: `CompletableFuture` provides excellent mechanisms for handling exceptions in a non-blocking manner:
  - `exceptionally(Function<Throwable, T>)`: Called when an exception occurs. Allows you to recover from the error by providing a default value or transforming the error into a valid result.
  - `handle(BiFunction<T, Throwable, R>)`: Called when the computation completes, regardless of whether it was successful or threw an exception. Provides both the result (if successful) and the exception (if an error occurred), allowing for more comprehensive error handling and transformation.
  - `whenComplete(BiConsumer<T, Throwable>)`: Executes a BiConsumer when the computation completes, allowing you to perform side effects (e.g., logging) without modifying the result.

#### Example
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import java.util.concurrent.CompletableFuture;

@Service
public class ReportGenerator {

    private static final Logger logger = LoggerFactory.getLogger(ReportGenerator.class);

    @Async
    public CompletableFuture<String> generateReport(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            logger.info("Generating report for user: {} - Thread: {}", userId, Thread.currentThread().getName());
            try {
                Thread.sleep(4000); // Simulate report generation
                if (userId.equals("user123_error")) {
                    throw new RuntimeException("Failed to fetch data for report: " + userId);
                }
                return "Report generated successfully for user: " + userId;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new IllegalStateException("Report generation interrupted", e);
            }
        }).exceptionally(ex -> { // Handle exception
            logger.error("Error generating report for user {}: {}", userId, ex.getMessage());
            return "Failed to generate report for user: " + userId + " due to: " + ex.getMessage();
        });
    }

    @Async
    public CompletableFuture<String> fetchUserData(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            logger.info("Fetching user data for user: {} - Thread: {}", userId, Thread.currentThread().getName());
            try {
                Thread.sleep(2000);
                if (userId.equals("user_fail_data")) {
                    throw new RuntimeException("User data not found for: " + userId);
                }
                return "Data for " + userId;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new IllegalStateException("User data fetch interrupted", e);
            }
        });
    }

    @Async
    public CompletableFuture<String> sendNotification(String message) {
        return CompletableFuture.supplyAsync(() -> {
            logger.info("Sending notification: {} - Thread: {}", message, Thread.currentThread().getName());
            try {
                Thread.sleep(1000);
                return "Notification sent: " + message;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new IllegalStateException("Notification sending interrupted", e);
            }
        });
    }
}
```

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

@RestController
public class ReportController {

    @Autowired
    private ReportGenerator reportGenerator;

    @GetMapping("/generate-report")
    public CompletableFuture<String> triggerReportGeneration(@RequestParam String userId) {
        return reportGenerator.generateReport(userId)
                .thenApply(report -> "Async Report Status: " + report) // Transform the result
                .exceptionally(ex -> "Failed to trigger report generation: " + ex.getMessage()); // Global exception handling
    }

    @GetMapping("/workflow")
    public CompletableFuture<String> complexWorkflow(@RequestParam String userId) {
        CompletableFuture<String> userDataFuture = reportGenerator.fetchUserData(userId);
        CompletableFuture<String> reportFuture = reportGenerator.generateReport(userId);

        // Chain tasks: once user data and report are ready, send a notification
        return CompletableFuture.allOf(userDataFuture, reportFuture)
                .thenApply(v -> { // v is void because allOf returns CompletableFuture<Void>
                    try {
                        String userData = userDataFuture.get();
                        String report = reportFuture.get();
                        return "Workflow completed. User Data: " + userData + ", Report: " + report;
                    } catch (InterruptedException | ExecutionException e) {
                        throw new RuntimeException("Error combining results: " + e.getMessage(), e);
                    }
                })
                .thenCompose(combinedResult -> reportGenerator.sendNotification("Workflow finished for " + userId + ". " + combinedResult))
                .handle((result, ex) -> { // Use handle for graceful recovery or logging
                    if (ex != null) {
                        return "Workflow failed for user " + userId + ": " + ex.getMessage();
                    }
                    return "Workflow final status: " + result;
                });
    }
}
```

### 4. `ListenableFuture<T>`
- **Behavior**: Spring's own `ListenableFuture` (part of spring-core) is similar to `Future` but provides the ability to register callbacks that will be invoked upon completion or failure of the computation. This is useful when you want to react to the result asynchronously without blocking the current thread, and before Java 8's `CompletableFuture` became widespread. While `CompletableFuture` is generally preferred for new code due to its richer API and adherence to the reactive programming paradigm, `ListenableFuture` can still be found in older Spring applications or specific integration scenarios.
- **Result Retrieval**:
  - `addCallback(ListenableFutureCallback<T>)`: Registers a callback that handles both success and failure.
  - `addCallback(SuccessCallback<T>, FailureCallback)`: Registers separate callbacks for success and failure.
  - It also implements `Future`, so `get()` and `isDone()` are available.
- **Exception Handling**: Handled via the `onFailure` method of `ListenableFutureCallback` or `FailureCallback`.

#### Example
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.springframework.util.concurrent.ListenableFuture;
import org.springframework.util.concurrent.ListenableFutureCallback;
import org.springframework.scheduling.annotation.AsyncResult;

@Service
public class NotificationService {

    private static final Logger logger = LoggerFactory.getLogger(NotificationService.class);

    @Async
    public ListenableFuture<String> sendSms(String phoneNumber, String message) {
        return new AsyncResult<>(CompletableFuture.supplyAsync(() -> {
            logger.info("Sending SMS to {} with message: {} - Thread: {}", phoneNumber, message, Thread.currentThread().getName());
            try {
                Thread.sleep(2000);
                if (phoneNumber.startsWith("555")) { // Simulate error for numbers starting with 555
                    throw new RuntimeException("SMS gateway error for number: " + phoneNumber);
                }
                return "SMS sent to " + phoneNumber + ": " + message;
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new IllegalStateException("SMS sending interrupted", e);
            }
        }));
    }
}
```

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.util.concurrent.ListenableFuture;
import org.springframework.util.concurrent.ListenableFutureCallback;

@RestController
public class SmsController {

    @Autowired
    private NotificationService notificationService;

    @GetMapping("/send-sms")
    public String triggerSms(@RequestParam String phoneNumber, @RequestParam String message) {
        ListenableFuture<String> smsFuture = notificationService.sendSms(phoneNumber, message);

        smsFuture.addCallback(new ListenableFutureCallback<String>() {
            @Override
            public void onSuccess(String result) {
                System.out.println("SMS sent successfully: " + result);
            }

            @Override
            public void onFailure(Throwable ex) {
                System.err.println("Failed to send SMS: " + ex.getMessage());
            }
        });

        return "SMS sending initiated for " + phoneNumber + ". Check console for status.";
    }
}
```

## Customizing Thread Pools with `ThreadPoolTaskExecutor`
By default, Spring uses a simple `SimpleAsyncTaskExecutor` for @Async methods. This executor creates a new thread for each asynchronous invocation, which can lead to excessive thread creation and resource exhaustion under high load. For production environments and fine-grained control, it's crucial to configure a custom thread pool using `ThreadPoolTaskExecutor`.

`ThreadPoolTaskExecutor` provides properties to control the thread pool's behavior:
- `corePoolSize`: The number of threads to keep in the pool, even if they are idle. This is the minimum number of threads that will be active.
- `maxPoolSize`: The maximum number of threads that can be created in the pool. When the queue is full, new tasks will create threads up to this limit.
- `queueCapacity`: The capacity of the `BlockingQueue` used to hold tasks before they are executed. When `corePoolSize` threads are busy, new tasks are added to this queue.
- `keepAliveSeconds`: The maximum time that excess idle threads will wait for new tasks before terminating.
- `threadNamePrefix`: A prefix for the names of the threads in the pool, which is very useful for debugging and logging.
- `setRejectedExecutionHandler(RejectedExecutionHandler)`: Defines the strategy for handling tasks that cannot be executed because the thread pool is saturated (queue is full and `maxPoolSize` is reached). Common handlers include `AbortPolicy` (default, throws `RejectedExecutionException`), `CallerRunsPolicy` (executes the task in the calling thread), `DiscardPolicy` (silently discards the task), and `DiscardOldestPolicy` (discards the oldest unexecuted task).

#### Example of Custom `ThreadPoolTaskExecutor` Configuration
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(10);
        executor.setThreadNamePrefix("EmailSender-");
        executor.initialize();
        return executor;
    }

    @Bean(name = "fileProcessingExecutor")
    public Executor fileProcessingExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(20);
        executor.setThreadNamePrefix("FileProcessor-");
        executor.initialize();
        return executor;
    }

    @Bean(name = "longRunningTaskExecutor")
    public Executor longRunningTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(1); // Small core size for very long tasks
        executor.setMaxPoolSize(2); // Limited max pool size to avoid overwhelming resources
        executor.setQueueCapacity(5);
        executor.setThreadNamePrefix("LongTask-");
        executor.setRejectedExecutionHandler(new java.util.concurrent.ThreadPoolExecutor.CallerRunsPolicy()); // Important for long tasks
        executor.initialize();
        return executor;
    }
}
```

### Injecting and Using Named Executors with `@Async("customExecutor")`
Once you've defined your custom `TaskExecutor` beans, you can specify which executor to use for a particular `@Async` method by providing the bean name as an argument to the annotation:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

@Service
public class EmailService {

    private static final Logger logger = LoggerFactory.getLogger(EmailService.class);

    @Async("emailExecutor") // Use the custom emailExecutor
    public void sendEmail(String recipient, String subject, String body) {
        logger.info("Sending email to {} with subject: {} - Thread: {}", recipient, subject, body, Thread.currentThread().getName());
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            logger.error("Email sending interrupted", e);
        }
        logger.info("Email sent to {} - Thread: {}", recipient, Thread.currentThread().getName());
    }
}

@Service
public class FileProcessor {

    private static final Logger logger = LoggerFactory.getLogger(FileProcessor.class);

    @Async("fileProcessingExecutor") // Use the custom fileProcessingExecutor
    public void processLargeFile(String fileName) {
        logger.info("Processing large file: {} - Thread: {}", fileName, Thread.currentThread().getName());
        try {
            Thread.sleep(7000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            logger.error("File processing interrupted", e);
        }
        logger.info("Finished processing large file: {} - Thread: {}", fileName, Thread.currentThread().getName());
    }
}
```

#### Thread Management Concerns
- **Thread Starvation**: Occurs when a thread pool is too small, and all threads are busy with long-running tasks, leaving no threads available for new tasks. This can lead to tasks being queued indefinitely or rejected. Proper tuning of `corePoolSize` and `maxPoolSize` is crucial.
- **Context Propagation**: By default, Spring's `@Async` does not automatically propagate the current thread's context (e.g., `SecurityContext`, `RequestContextHolder`, MDC for logging) to the asynchronous thread. This means if you rely on `SecurityContextHolder` to get the current authenticated user, it will be `null` in the async method.
    - **Solution for** `SecurityContext`: Use `org.springframework.security.task.DelegatingSecurityContextAsyncTaskExecutor or SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL)`. The former is generally preferred as it's more explicit and robust.
    - **Solution for MDC (Mapped Diagnostic Context) in Logging:** Manually copy MDC contents:
      ```java
      // In the calling thread, before calling async method
      Map<String, String> contextMap = MDC.getCopyOfContextMap();
    
      // In the async method
      if (contextMap != null) {
        MDC.setContextMap(contextMap);
      }
      // ... async logic ...
      MDC.clear(); // Clear context after async task
      ```

    Or, use a custom `TaskDecorator` with `ThreadPoolExecutor`:
    ```java
    // In AsyncConfig
    executor.setTaskDecorator(new MdcTaskDecorator());

    // MdcTaskDecorator.java
    public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();
        return () -> {
           if (contextMap != null) {
              MDC.setContextMap(contextMap);
           }
        try {
           runnable.run();
        } finally {
           MDC.clear();
        }
    };
    }
    }
    ```
  
- **Graceful Shutdown**: When an application shuts down, you want to ensure that all active asynchronous tasks complete gracefully before the application terminates. `ThreadPoolTaskExecutor` supports graceful shutdown. By default, it will wait for tasks to complete for a certain period. You can configure this with `setWaitForTasksToCompleteOnShutdown(true)` and `setAwaitTerminationSeconds(int)`.
- **Avoiding Blocking Calls Inside Async Methods**: The primary purpose of `@Async` is to offload blocking operations. It's counterproductive to then introduce new blocking calls (e.g., `Thread.sleep()` for very long periods, or `future.get()` without a timeout) within the asynchronous method itself, especially if it ties up a thread from a shared, limited pool. If an async method itself needs to perform further async operations, it should return a `CompletableFuture` and chain operations using its non-blocking API.

### Contrast with Programmatic Approach
While `@Async` offers a convenient declarative style, there are situations where a programmatic approach to concurrency is more suitable:
- **Complex Workflows and Chaining**: When you need to chain multiple interdependent asynchronous tasks, combine their results, or react to their completion in a more dynamic way, `CompletableFuture`'s rich API (e.g., `thenApply`, `thenCompose`, `allOf`, `anyOf`) is ideal.
- **Non-Spring Managed Objects**: If you need to perform asynchronous operations outside of Spring-managed beans.
- **Direct Control over Thread Logic**: When you require very specific thread management, such as using a custom `ExecutorService` directly or managing thread lifecycles manually.
- **Integrating with Legacy Code**: When interacting with existing codebases that use `ExecutorService` or other lower-level concurrency constructs.

### Example of Programmatic Approach
#### Using `ExecutorService` **(and Spring's** `TaskExecutor`**)**:
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;
import java.util.concurrent.Callable;

@Service
public class ManualTaskExecutorService {

    @Autowired
    @Qualifier("fileProcessingExecutor") // Inject the custom ThreadPoolTaskExecutor as ExecutorService
    private ExecutorService fileExecutorService;

    public Future<String> executeLongCalculation(int iterations) {
        return fileExecutorService.submit(new Callable<String>() {
            @Override
            public String call() throws Exception {
                // Perform long calculation
                Thread.sleep(iterations * 100L);
                return "Calculation completed after " + iterations + " iterations.";
            }
        });
    }
}
```

#### Using `CompletableFuture.supplyAsync()` / `runAsync()`:
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;

@Service
public class CompletableFutureService {

    private static final Logger logger = LoggerFactory.getLogger(CompletableFutureService.class);

    @Autowired
    @Qualifier("emailExecutor")
    private Executor emailExecutor; // Inject the specific executor

    public CompletableFuture<String> processOrderProgrammatically(String orderId) {
        return CompletableFuture.supplyAsync(() -> {
            logger.info("Processing order: {} - Thread: {}", orderId, Thread.currentThread().getName());
            try {
                Thread.sleep(2000);
                return "Order " + orderId + " processed.";
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new IllegalStateException("Order processing interrupted", e);
            }
        }, emailExecutor); // Explicitly pass the executor

        // You can then chain more operations:
        // .thenApply(result -> result + " and then something else")
        // .exceptionally(...)
    }
}
```

## Common Pitfalls and Debugging Strategies
### 1. `@Async` Not Working Due to Self-Invocation:
- **Issue**: If an `@Async` method is called from another method within the same class, the `@Async` annotation will be ignored. This is because Spring's AOP proxy for `@Async` works by intercepting external calls to the bean. Internal calls bypass the proxy.
- **Solution**: Move the `@Async` method to a separate Spring-managed service and inject that service

```java
// BEFORE (Incorrect - self-invocation)
@Service
public class MyService {
    public void doSomethingAndThenAsync() {
        // ...
        asyncMethod(); // This call will not be async!
    }

    @Async
    public void asyncMethod() { /* ... */ }
}

// AFTER (Correct - separate service)
@Service
public class MyService {
    @Autowired
    private MyAsyncService asyncService; // Inject the async service

    public void doSomethingAndThenAsync() {
        // ...
        asyncService.asyncMethod(); // This call will be async
    }
}

@Service
public class MyAsyncService {
    @Async
    public void asyncMethod() { /* ... */ }
}
```

### 2. Non-Public Methods:
- **Issue**: `@Async` only works on `public` methods. Spring's AOP proxy cannot intercept calls to non-public methods.
- **Solution**: Ensure your `@Async` methods are `public`.

### 3. Static Methods:
- **Issue**: `@Async` does not work on `static` methods. Spring AOP operates on instances of beans, not static contexts.
- **Solution**: Make the method non-static and ensure it belongs to a Spring-managed bean.

### 4. Missing Proxy Configuration (`@EnableAsync`):
- **Issue**: If you forget to add `@EnableAsync` to a `@Configuration` class, Spring will not create the necessary proxy for `@Async` methods.
- **Solution**: Add `@EnableAsync` to your main application class or a dedicated configuration class

#### 5. Calling `new MyService().asyncMethod()`:
- **Issue**: If you create a `new` instance of a service using new operator and call an `@Async` method on it, Spring's AOP proxy will not be applied, and the method will execute synchronously.
- **Solution**: Always inject Spring-managed beans using `@Autowired` (or constructor injection) and call methods on the injected instance.

#### 6. Debugging Strategies:
- **Thread Naming**: Use `executor.setThreadNamePrefix()` to give your async threads meaningful names. This makes it much easier to identify them in logs and debuggers.
- **Logging**: Add `logger.info("Current Thread: {}", Thread.currentThread().getName());` at the beginning of your async methods to confirm they are indeed running on a different thread.
- **Debugging with Breakpoints**: Place breakpoints inside your `@Async` method. When execution pauses, inspect the call stack to see if it's being called by a thread pool executor.
- **Spring Boot Actuator**: If you have Actuator enabled, you can inspect the health and metrics of your `TaskExecutor` instances at runtime (e.g., `/actuator/metrics/executor.active`, `/actuator/metrics/executor.queued`)

### Best Practices
#### Isolate CPU-bound vs I/O bound tasks:
- **I/O-bound tasks** (e.g., network calls, database queries, file I/O) spend most of their time waiting for external resources. A larger `maxPoolSize` and `queueCapacity` can be beneficial as threads spend time waiting and can thus accommodate more concurrent requests.
- **CPU-bound tasks** (e.g., complex calculations, image processing) actively consume CPU cycles. The `corePoolSize` should ideally be close to the number of available CPU cores to avoid excessive context switching. A smaller `maxPoolSize` is generally better.
- Create separate `ThreadPoolTaskExecutor` instances for different types of tasks.

#### Tuning Thread Pools Based on Workload:
- There's no one-size-fits-all solution for thread pool sizing. It depends on your application's specific workload, hardware, and the nature of your asynchronous tasks.
- **Start with reasonable defaults** and then use load testing and monitoring (e.g., CPU utilization, thread pool queue size, response times) to fine-tune the parameters.
- A common formula for CPU-bound tasks is `Number of Cores + 1`.
- For I/O-bound tasks, it can be `Number of Cores * (1 + Wait Time / Compute Time)`.

#### Use `@Async` Only in Spring-Managed Beans:
`@Async` relies on Spring's AOP infrastructure, so it must be applied to methods of beans managed by the Spring IoC container

#### Monitoring Performance in Production:
- **Thread Pool Metrics**: Monitor active threads, queued tasks, completed tasks, and rejected tasks for each of your ThreadPoolTaskExecutor instances. Spring Boot Actuator can expose these metrics.
- **System Metrics**: Keep an eye on CPU usage, memory consumption, and I/O wait times to identify bottlenecks.
- **Application-Specific Metrics**: Track the latency and success/failure rates of your asynchronous operations.

### Asynchronous Behavior and Other Spring Concerns
- **Transactions(**`@Transactional`**)**:
  - By default, the `@Transactional` context is not propagated to the `@Async` method's thread. The `@Async` method will execute in a new, separate transaction (or no transaction if not explicitly annotated).
  - If you need the async method to participate in the caller's transaction, you cannot simply use `@Async`. You would typically need to use other patterns like Spring's event listeners (`@TransactionalEventListener`) or manually manage the transaction context, or consider using synchronous execution within a shared transaction for very specific scenarios (though this defeats the purpose of async).
  - If the `@Async` method needs its own transaction, simply apply `@Transactional` to it as usual.
- **Security Context**:
  - As discussed under "Context Propagation," the `SecurityContext` is not propagated by default. Use `DelegatingSecurityContextAsyncTaskExecutor` or configure `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL` if you need the security principal to be available in async methods
- **Logging Correlation (MDC)**:
  - Also covered under "Context Propagation." Manually copy MDC contents or use a `TaskDecorator` to ensure logging statements in async methods are correlated with the original request's context.

### Testing Asynchronous Code
Testing asynchronous code requires special considerations because the execution happens concurrently and not immediately.
- `CompletableFuture.get()` or `Future.get()`: The simplest way to test `CompletableFuture` or `Future` methods is to block and retrieve the result using `get()`. This is acceptable in tests, as you want to ensure the async task completes and returns the expected value (or throws the expected exception).
- **Awaitility**: For more complex scenarios, especially when testing `void` `@Async` methods or when you need to wait for a specific state change that occurs as a side effect of an async operation, `Awaitility` is an excellent library. It allows you to express conditions that should eventually become true.
    ```xml
    <dependency>
       <groupId>org.awaitility</groupId>
       <artifactId>awaitility</artifactId>
       <version>4.2.0</version> <scope>test</scope>
    </dependency>
    ```
    ```java
    import org.junit.jupiter.api.Test;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.test.context.junit.jupiter.SpringExtension;
    import static org.awaitility.Awaitility.await;
    import static org.mockito.Mockito.*;

    import java.time.Duration;

    @SpringBootTest
    class EmailServiceIntegrationTest {

       @Autowired
       private EmailService emailService; // Assuming EmailService sends to a mock or captures logs

       @Test
       void testSendEmailAsync() {
          // Using Mockito to verify method call later
          EmailService spyEmailService = spy(emailService); // Spy on the actual service
          doNothing().when(spyEmailService).sendEmail(anyString(), anyString(), anyString()); // Don't actually send email during test

          spyEmailService.sendEmail("test@example.com", "Test Subject", "Test Body");

          // Awaitility: Wait until the sendEmail method is called on a different thread
          // This implicitly tests if the @Async works
          await().atMost(Duration.ofSeconds(5))
               .untilAsserted(() -> verify(spyEmailService, timeout(100)).sendEmail(eq("test@example.com"), eq("Test Subject"), eq("Test Body")));
       }
    }
    ```
  
    For `void` methods, you might often test their side effects (e.g., a database entry created, a message sent to a queue, a log entry). Awaitility helps you wait for these side effects to materialize.