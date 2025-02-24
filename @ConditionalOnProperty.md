# @ConditionalOnProperty

Bean is created conditionally (means maybe created may not be created)

#### The below behavior is already known
```java
@Component
public class NoSQLConnection {
    NoSQLConnection() {
        System.out.println("Initialization of NoSQLConnection");
    }
}
```

```java
@Component
public class MySQLConnection {
    MySQLConnection() {
        System.out.println("Initialization of MySQLConnection");
    }
}
```

```java
@Component
public class DBConnection {
    @Autowired
    private MySQLConnection mySQLConnection;
    
    @Autowired
    private NoSQLConnection noSQLConnection;
    
    @PostConstruct
    public void init() {
        System.out.println("DB Connection Bean Created with dependencies below: ");
        System.out.println("is MySQLConnection object Null: " + Objects.isNull(mySQLConnection));
        System.out.println("is NoSQLConnection object Null: " + Object.isNull(noSQLConnection));
    }
}
```

#### Output
```
initialization of MySQLConnection Bean
initialization of NoSQLConnection Bean
DB Connection Bean Created with dependencies below:
is MySQLConnection object Null: false
is NoSQLConnection object Null: false
```

### But how to handle the below cases?
- **Use Case 1:** We want to create only 1 Bean, either MySQLConnection or NoSQLConnection
- **Use Case 2:** We have 2 components, sharing same codebase, but 1 component need MySQLConnection and other needs NoSQLConnection

Solution is by using `@ConditionalOnProperty` Annotation

#### Solution
```java
@Component
@ConditionalOnProperty(prefix = "nosqlconnection", value = "enabled", havingValue = "true", matchIfMissing = false)
public class NoSQLConnection {
    NoSQLConnection() {
        System.out.println("Initialization of NoSQLConnection");
    }
}
```

```java
@Component
@ConditionalOnProperty(prefix = "mysqlconnection", value = "enabled", havingValue = "true", matchIfMissing = false)
public class MySQLConnection {
    MySQLConnection() {
        System.out.println("Initialization of MySQLConnection");
    }
}
```

```java
@Component
public class DBConnection {
    @Autowired(required = false)
    private MySQLConnection mySQLConnection;
    
    @Autowired(required = false)
    private NoSQLConnection noSQLConnection;
    
    @PostConstruct
    public void init() {
        System.out.println("DB Connection Bean Created with dependencies below: ");
        System.out.println("is MySQLConnection object Null: " + Objects.isNull(mySQLConnection));
        System.out.println("is NoSQLConnection object Null: " + Object.isNull(noSQLConnection));
    }
}
```

```
sqlconnection.enabled = true
```


#### Output
```
initialization of MySQLConnection Bean
initialization of NoSQLConnection Bean
DB Connection Bean Created with dependencies below:
is MySQLConnection object Null: false
is NoSQLConnection object Null: true
```

### Advantages
- Toggling of feature
- Avoid cluttering application context with unnecessary beans
- Save memory
- Reduce application startup time

### Disadvantages
- Misconfiguration can happen
- Code complexity when overused.
- Multiple bean configuration with same configuration can lead to confusion
- Complexity in managing

