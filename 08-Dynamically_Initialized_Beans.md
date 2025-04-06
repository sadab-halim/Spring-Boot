# Dynamically Initialized Beans

### Problem: Unsatisfied Dependency

```java
public interface Order {
    void createOrder();
}
```

```java
@Component
public class OnlineOrder implements Order {
    public OnlineOrder() {
        System.out.println("Online order initialized.");
    }
    
    public void createOrder() {
        System.out.println("Created online order.");
    }
}
```

```java
@Component
public class OfflineOrder implements Order {
    public OfflineOrder() {
        System.out.println("Offline order initialized.");
    }

    public void createOrder() {
        System.out.println("Created offline order.");
    }
}
```

```java
@RestController
@RequestMapping(value = "/api")
public class User {
    @Autowired
    private Order order;
    
    @PostMapping("/createOrder")
    public ResponseEntity<String> createOrder() {
        order.createOrder();
        return ResponseEntity.ok("");
    }
}
```

```
***************************
APPLICATION FAILED TO START
***************************

UnsatisfiedDependencyException: Error creating bean with name 'user'
```

### Solution

```java
public interface Order {
    void createOrder();
}
```

```java
@Component
@Qualifier("onlineOrderObject")
public class OnlineOrder implements Order {
    public OnlineOrder() {
        System.out.println("Online order initialized.");
    }
    
    public void createOrder() {
        System.out.println("Created online order.");
    }
}
```

```java
@Component
@Qualifier("offlineOrderObject")
public class OfflineOrder implements Order {
    public OfflineOrder() {
        System.out.println("Offline order initialized.");
    }

    public void createOrder() {
        System.out.println("Created offline order.");
    }
}
```

```java
@RestController
@RequestMapping(value = "/api")
public class User {
    @Autowired
    @Qualifier("onlineOrderObject")
    private Order order;
    
    @PostMapping("/createOrder")
    public ResponseEntity<String> createOrder() {
        order.createOrder();
        return ResponseEntity.ok("");
    }
}
```

```
created online order
```

But the above implementation becomes hardcoded

### Solution 1
```java
@RestController
@RequestMapping(value = "/api")
public class User {
    @Autowired
    @Qualifier("onlineOrderObject")
    private Order onlineOrderObject;

    @Autowired
    @Qualifier("offlineOrderObject")
    private Order offlineOrderObject;
    
    @PostMapping("/createOrder")
    public ResponseEntity<String> createOrder(@RequestParam boolean isOnlineOrder) {
        if(isOnlineOrder) {
            onlineOrderObject.createOrder();    
        } else {
            offlineOrderObject.createOrder();
        }
        
        return ResponseEntity.ok("");
    }
}
```

### Solution 2
```java
public interface Order {
    void createOrder();
}
```

```java
public class OnlineOrder implements Order {
    public OnlineOrder() {
        System.out.println("Online order initialized.");
    }
    
    public void createOrder() {
        System.out.println("Created online order.");
    }
}
```

```java
public class OfflineOrder implements Order {
    public OfflineOrder() {
        System.out.println("Offline order initialized.");
    }

    public void createOrder() {
        System.out.println("Created offline order.");
    }
}
```

```java
@RestController
@RequestMapping(value = "/api")
public class User {
    @Autowired
    private Order order; //will get offlineorder props --> config
    
    @PostMapping("/createOrder")
    public ResponseEntity<String> createOrder() {
        order.createOrder();
        return ResponseEntity.ok("");
    }
}
```

```java
@Configuration
public class AppConfig {
    @Bean
    public Order createOrderBean(@Value("${isOnlineOrder}") boolean isOnlineOrder) {
        if(isOnlineOrder) {
            return new OnlinOrder();
        } else {
            return new OfflineOrder();
        }
    }
}
```
application.properties 👇
```
isOnlineOrder = false
```
----------

`@Value` is used to inject values from various sources like properties/environment files variables or inline literals.

### Inline Literal Example
```java
@Bean
public Order createOrderBean(@Value("false") boolean isOnlineOrder) {
    if(isOnlineOrder) {
        return new OnlineOrder();
    } else {
        return new OfflineOrder();
    }
}
```