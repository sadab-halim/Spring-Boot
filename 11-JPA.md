# Spring Boot JPA

## Overview of Spring Data JPA and its Evolution
Before diving into the specifics, let's understand where Spring Data JPA fits in the ecosystem of Java database connectivity:
- **JDBC (Java Database Connectivity)**: This is the lowest-level API for interacting with databases in Java. It provides interfaces for connecting to databases, executing SQL queries, and processing results. While powerful, it requires significant boilerplate code for connection management, statement preparation, result set handling, and error handling
- **JdbcTemplate (Spring JDBC)**: Spring's `JdbcTemplate` simplifies JDBC usage by handling resource management (connections, statements, result sets) and exception translation. It reduces boilerplate code significantly compared to raw JDBC but still requires you to write SQL queries explicitly. It's a good choice when you need fine-grained control over your SQL or are working with complex, non-ORM-friendly queries
- **JPA (Java Persistence API)**: JPA is a Java specification that defines a standard for object-relational mapping (ORM). It allows you to map Java objects (entities) to database tables, and interact with the database using object-oriented concepts rather than raw SQL. JPA providers (like Hibernate, EclipseLink) implement this specification. The core idea is to persist Java objects directly, making database operations feel more natural within your application
- **Spring Data JPA**: This is a module within the Spring Data project that builds on top of JPA. It simplifies the implementation of JPA-based data access layers by providing a high-level abstraction. Instead of writing concrete implementations for common CRUD (Create, Read, Update, Delete) operations, you simply define repository interfaces, and Spring Data JPA automatically generates the necessary implementation at runtime using a JPA provider (typically Hibernate).

#### Reasons for preferring JPA in Modern Enterprise Applications:
- Reduced Boilerplate Code
- Increased Productivity
- Object-Oriented Approach
- Database Agnosticism
- Built-in Features
- Standardization

## Complete Setup Process
### 1. Dependencies
To get started with Spring Boot and JPA, you need to add the appropriate dependencies in your pom.xml (for Maven) or build.gradle (for Gradle).

#### Maven
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version> <relativePath/> </parent>
    <groupId>com.example</groupId>
    <artifactId>spring-boot-jpa-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-boot-jpa-demo</name>
    <description>Demo project for Spring Boot and JPA</description>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version> <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

#### Gradle
```

```

#### `application.properties` **(or ** `application.yml` **) for database configuration:**
```properties
# DataSource configuration for MySQL
spring.datasource.url=jdbc:mysql://localhost:3306/mydatabase?useSSL=false&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=password

# Hibernate DDL-Auto (Data Definition Language Auto)
# none: No DDL operations.
# update: Updates the schema if needed. Recommended for development.
# create: Creates the schema on startup, destroys on shutdown. Good for testing.
# create-drop: Creates schema on startup, drops on shutdown. Good for testing.
# validate: Validates the schema.
spring.jpa.hibernate.ddl-auto=update

# Show SQL queries in console (useful for debugging)
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

### 2. Entity Creation
An entity is a lightweight, persistent domain object. It represents a table in your database, and each instance of the entity corresponds to a row in that table.
```java
package com.example.springbootjpademo.entity;

import jakarta.persistence.*;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;

@Entity // Marks this class as a JPA entity
@Table(name = "users") // Maps this entity to a table named "users" in the database
@Data // Lombok annotation for getters, setters, toString, equals, hashCode
@NoArgsConstructor // Lombok annotation for no-arg constructor
@AllArgsConstructor // Lombok annotation for all-args constructor
public class User {

    @Id // Specifies the primary key of the entity
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Configures the primary key generation strategy
    private Long id;

    @Column(name = "first_name", nullable = false, length = 50) // Maps to a column named "first_name", not null, max length 50
    private String firstName;

    @Column(name = "last_name", length = 50)
    private String lastName;

    @Column(name = "email", nullable = false, unique = true) // Email must be unique
    private String email;
}
```

### 3. Repository Interfaces
Spring Data JPA provides a powerful abstraction for data access using repository interfaces. You extend one of the provided interfaces (e.g., `JpaRepository`, `CrudRepository`, `PagingAndSortingRepository`), and Spring Data JPA automatically generates the implementation
```java
package com.example.springbootjpademo.repository;

import com.example.springbootjpademo.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository // Optional, but good practice to indicate a repository component
public interface UserRepository extends JpaRepository<User, Long> {
    // Spring Data JPA automatically generates queries based on method names
    Optional<User> findByEmail(String email);
    List<User> findByLastNameContainingIgnoreCase(String lastName);
    List<User> findByFirstNameAndLastName(String firstName, String lastName);
}
```

## Deep Dive into JPA Architecture
JPA operates around several core concepts that are crucial for understanding its behavior and optimizing its usage.

### Entity Lifecycle
An entity instance transitions through various states during its lifecycle, managed by the `EntityManager`
1. **New (Transient)**: An entity instance is new when it has just been instantiated using the new operator and has no persistent identity (i.e., it's not yet associated with an EntityManager and doesn't represent a row in the database).
    ```java
    User newUser = new User("John", "Doe", "john.doe@example.com"); // New state
    ```
2. **Managed (Persistent)**: An entity instance becomes managed when it's associated with a `PersistenceContext` (and thus an `EntityManager`). Changes to a managed entity are tracked by the `EntityManager` and will be synchronized with the database at transaction commit.
   - `entityManager.persist(entity)`: Makes a new (transient) entity managed.
   - `entityManager.find(entityClass, primaryKey)`: Retrieves an entity from the database and makes it managed.
   - `entityManager.merge(entity)`: Copies the state of a detached entity onto a new managed instance.
   ```java
   User savedUser = entityManager.persist(newUser); // newUser is now managed
   User foundUser = entityManager.find(User.class, 1L); // foundUser is managed
   ```
3. **Detached**: An entity instance becomes detached when its `PersistenceContext` is closed, or when the `EntityManager` is explicitly cleared or closed, or when the entity is serialized. A detached entity is no longer managed by an `EntityManager`, and changes made to it will not be automatically synchronized with the database. To persist changes to a detached entity, it must be re-merged into a `PersistenceContext` using `entityManager.merge()`.
    ```java
    entityManager.close(); // All managed entities become detached
    ```

4. **Removed**: An entity instance is in the removed state when `entityManager.remove(entity)` is called on a managed entity. The entity is scheduled for deletion from the database at the next flush operation.
    ```java
    entityManager.remove(foundUser); // foundUser is now in removed state
    ```
   
### Persistence Context
The persistence context is a **first-level cache** where managed entity instances reside. It's essentially a synchronization mechanism between the Java objects in your application and the corresponding rows in the database
- **Role of EntityManager**: The EntityManager is the primary interface for interacting with the persistence context. It provides methods like `persist()`, `find()`, `merge()`, `remove()`, and `flush()` to manage entities within the context.
- **Transaction-Scoped Persistence Context (Default in Spring Boot)**:  In a typical Spring Boot application with `@Transactional` services, a persistence context is automatically created when a transaction begins and is closed when the transaction commits or rolls back. All operations within that transaction operate on the same persistence context. This ensures that:
  - **Identity Map**: For any given database identity (primary key), there will be at most one managed entity instance in the persistence context. If you fetch the same entity multiple times within a transaction, you'll get the same object reference.
  - **Dirty Checking**: The `EntityManager` tracks changes made to managed entities. When the transaction commits, it automatically detects these "dirty" entities and flushes their changes to the database. This saves you from explicitly calling update methods for every change.

### Transaction Boundaries
Transactions in JPA (and generally in database operations) define a logical unit of work. All operations within a transaction are treated as a single, atomic operation. If any part of the transaction fails, the entire transaction is rolled back, ensuring data consistency.

In Spring Boot, transactions are typically managed declaratively using the `@Transactional` annotation.
```java
package com.example.springbootjpademo.service;

import com.example.springbootjpademo.entity.User;
import com.example.springbootjpademo.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Transactional // All operations within this method will be part of a single transaction
    public User createUser(User user) {
        return userRepository.save(user);
    }

    @Transactional(readOnly = true) // Read-only transactions can often be optimized by the JPA provider
    public Optional<User> getUserById(Long id) {
        return userRepository.findById(id);
    }

    @Transactional
    public User updateUserEmail(Long id, String newEmail) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("User not found"));
        user.setEmail(newEmail); // Changes to managed entity are automatically tracked
        // No explicit save() call needed if transaction commits
        return user;
    }

    @Transactional
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }

    @Transactional(readOnly = true)
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }
}
```

**Important**: The `@Transactional` annotation ensures that a `PersistenceContext` is active and bound to the current thread for the duration of the method call. If an `EntityManager` is injected within this method, it will use this transaction-scoped persistence context.

## Caching in JPA
Caching is crucial for improving application performance by reducing the number of database calls. JPA supports two levels of caching:

### First-Level Cache (L1 Cache)
- **Per Persistence Context**: The first-level cache is an integral part of the `PersistenceContext` and is bound to the `EntityManager`. It's enabled by default and cannot be disabled.
- **Identity Map**: As mentioned, it acts as an identity map. When you fetch an entity by its primary key using `entityManager.find()` or `repository.findById()`, the `EntityManager` first checks its L1 cache. If the entity is already present, it returns the cached instance, avoiding a database roundtrip.
- **Scope**: The L1 cache is transactional. It lasts only for the duration of a single transaction (or the `EntityManager`'s lifecycle if not transaction-scoped). Once the transaction commits or rolls back, the L1 cache is cleared.
- **Automatic Dirty Checking**: This cache also enables JPA's dirty checking mechanism. The `EntityManager` holds the initial state of managed entities and compares it with their current state at flush time to determine if updates are needed.

### Second-Level Cache (L2 Cache)
- **Shared Cache**: The second-level cache is an optional, application-level cache that is shared across multiple `EntityManager` instances and transactions within the same `EntityManagerFactory`. It's designed to cache frequently accessed entities and query results to reduce database load significantly.
- **Pluggable Providers**: JPA specifies the L2 cache, but the actual implementation is provided by the JPA provider (e.g., Hibernate's second-level cache). You can integrate various caching solutions like Ehcache, Redis, or Caffeine as L2 cache providers.
- **Configuration**:
    1. **Enable L2 Cache**: In `application.properties`:
    2. **Add L2 Cache Provider Dependency**: For Ehcache (JCache implementation):
       ```xml
       <dependency>
          <groupId>org.hibernate.orm</groupId>
          <artifactId>hibernate-jcache</artifactId>
          <version>${hibernate.version}</version> </dependency>
       <dependency>
          <groupId>org.ehcache</groupId>
          <artifactId>ehcache</artifactId>
          <version>3.10.8</version> </dependency>
       <dependency>
          <groupId>javax.cache</groupId>
          <artifactId>cache-api</artifactId>
          <version>1.1.1</version>
       </dependency>
       ```
       (Note: For Spring Boot 3+, use `jakarta.persistence` and `jakarta.transaction` instead of `javax.persistence` and `javax.transaction`)       

    3. **Configure** `ecache.xml` **(in )** `src/main/resources`**)**:
       ```xml
       <config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns="http://www.ehcache.org/v3"
        xsi:schemaLocation="http://www.ehcache.org/v3 http://www.ehcache.org/schema/ehcache-core-3.10.xsd">

        <cache-template name="default">
            <expiry>
                <ttl unit="seconds">3600</ttl> </expiry>
            <heap unit="entries">1000</heap>
        </cache-template>
    
        <cache alias="com.example.springbootjpademo.entity.User" uses-template="default"/>
        <cache alias="com.example.springbootjpademo.entity.User.roles" uses-template="default"/> <cache alias="com.example.springbootjpademo.entity.Product">
            <expiry>
                <ttl unit="minutes">30</ttl>
            </expiry>
            <heap unit="entries">500</heap>
        </cache>
    
        </config>
       ``` 
  4. **Annotate Entities**: 
     ```java
     import jakarta.persistence.Cacheable;
     import org.hibernate.annotations.Cache;
     import org.hibernate.annotations.CacheConcurrencyStrategy;
    
     @Entity
     @Table(name = "users")
     @Cacheable // Marks the entity for L2 caching
     @Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // Specifies caching strategy
     public class User {
         // ... fields ...
     }
     ```

  `CacheConcurrencyStrategy` options:
  - `READ_ONLY`: For immutable data. Best performance.
  - `NONSTRICT_READ_WRITE`: For data that rarely changes. May occasionally return stale data.
  - `READ_WRITE`: For data that is frequently read and updated. Ensures consistency but has more overhead.
  - `TRANSACTIONAL`: For JTA environments, where transactions are managed externally.

#### Redis as L2 Cache Provider (Example)
For Redis, you'd typically use Spring Cache abstraction and a Redis cache manager, along with a Hibernate Redis cache provider (if available for your Hibernate version, or implement a custom one).

1. **Dependencies**:
   ```xml
    <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
   ```
2. **Configuration in `aplication.properties`**:
   ```
    spring.jpa.properties.hibernate.cache.use_second_level_cache=true
    spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.redis.RedisRegionFactory # if you use a specific Hibernate Redis provider
    spring.cache.type=redis
    spring.redis.host=localhost
    spring.redis.port=6379
   ```
   
3. **Spring Cache Configuration (Java Config)**:
```java
@Configuration
@EnableCaching // Enable Spring's caching abstraction
public class CacheConfig {
    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(60)) // Default TTL for all caches
                .disableCachingNullValues()
                .serializeValuesWith(SerializationPair.fromSerializer(new GenericJackson2JsonSerializer()));
    }

    @Bean
    public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
        return (builder) -> builder
                .withCacheConfiguration("users",
                        RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(10)))
                .withCacheConfiguration("products",
                        RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(30)));
    }
}
/*
 Then use `@Cacheable("users")` on your repository methods or service methods.
While Spring Cache is excellent for general application caching, integrating it directly with Hibernate's L2 cache requires a specific Hibernate Redis cache provider. Often, for L2 caching, Ehcache is a more straightforward out-of-the-box solution with Hibernate.
*/
```

## Mapping DTOs to Database Tables using JPA Annotations
While JPA entities directly map to database tables, DTOs (Data Transfer Objects) are used to transfer data between layers of an application, often for specific views or APIs. DTOs are not directly mapped to tables via JPA annotations. Instead, you map entities to DTOs.

#### Example: User Entity to UserDTO
```java
// Entity
@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String firstName;
    private String lastName;
    private String email;
}

// DTO
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserDTO {
    private Long id;
    private String fullName; // Combines first and last name
    private String email;
}

// Mapping in Service Layer (or dedicated mapper class)
import org.modelmapper.ModelMapper; // A popular library for object mapping

@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private ModelMapper modelMapper; // Configure as a Spring Bean

    public UserDTO getUserDtoById(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("User not found"));
        UserDTO userDTO = modelMapper.map(user, UserDTO.class);
        userDTO.setFullName(user.getFirstName() + " " + user.getLastName());
        return userDTO;
    }

    // You can also use JPA Projections for direct DTO fetching (covered later)
}
```

## Key Relationship Mappings
JPA supports various relationship types between entities:

### 1. OneToOne Relationship
A `OneToOne` relationship indicates that each record in one table is associated with at most one record in another table

#### Example: User and Address
A user has one address, and an address belongs to one user.

#### Diagram
```
+----------+       +---------+
|   User   |-------| Address |
+----------+       +---------+
| id (PK)  |       | id (PK) |
| name     |       | street  |
| email    |       | city    |
+----------+       | user_id | (FK)
                   +---------+
```

#### Unidirectional (User owns Address)
The `User` entity has a reference to `Address`, but `Address` does not know about `User`. The foreign key is typically in the `Address` table
```java
// User.java
@Entity
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToOne(cascade = CascadeType.ALL) // Cascade operations to Address
    @JoinColumn(name = "address_id", referencedColumnName = "id") // Foreign key in User table pointing to Address's ID
    private Address address;
    // ... getters, setters
}

// Address.java
@Entity
public class Address {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String street;
    private String city;
    // ... getters, setters
}
```

**Note**: In this unidirectional setup, the `address_id` foreign key is in the `User` table, which is less common for OneToOne where the dependent entity (Address) usually holds the FK. If User is the `owner`, `address_id` in `User` would point to `Address.id`.

**Correction for more common OneToOne unidirectional (Address owns User reference):**
```java
// User.java
@Entity
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    // ... getters, setters
}

// Address.java (owning side in this setup)
@Entity
public class Address {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String street;
    private String city;

    @OneToOne(fetch = FetchType.LAZY) // Lazy fetch recommended
    @JoinColumn(name = "user_id") // Foreign key column in Address table
    private User user; // Address knows about User
    // ... getters, setters
}
```

#### Bidirectional (User and Address know about each other):
`User` has a reference to `Address`, and `Address` has a reference back to `User`. One side is the "owning" side (manages the foreign key), and the other is the "inverse" side (`mappedBy`)
```java
// User.java (owning side, if User table has address_id)
@Entity
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToOne(cascade = CascadeType.ALL, orphanRemoval = true) // Cascade, remove address if user is removed
    @JoinColumn(name = "address_id", referencedColumnName = "id") // Foreign key column in User table
    private Address address;
    // ... getters, setters
}

// Address.java (inverse side)
@Entity
public class Address {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String street;
    private String city;

    @OneToOne(mappedBy = "address", fetch = FetchType.LAZY) // "address" refers to the field in User entity
    private User user; // Address has a reference to User, but User owns the relationship
    // ... getters, setters
}
```

**Common Pitfall**: `OneToOne` relationships need careful management of both sides to maintain consistency. When setting a relationship, you must set it on both sides if it's bidirectional. For example, `user.setAddress(address); address.setUser(user);`

### 2. OneToMany / ManyToOne Relationship
This is the most common relationship type. One entity can be associated with multiple instances of another entity, but each instance of the second entity is associated with only one instance of the first.

#### Example: Department and Employee
A  department can have many employees, but an employee belongs to only one department.

#### Diagram
```
+------------+       +----------+
| Department |-------| Employee |
+------------+       +----------+
| id (PK)    |       | id (PK)  |
| name       |       | name     |
+------------+       | salary   |
                     | dept_id  | (FK)
                     +----------+
```

#### Unidirectional (Department owns Employees):
`Department` has a collection of `Employees`, but `Employee` does not know about `Department`. The foreign key is typically in the `Employee` table. This often results in a join table if not careful, or a foreign key in the owning side which is less performant. It's generally preferred to have the Many-side own the foreign key.

```java
// Department.java (less common owning side for OneToMany unidirectional)
@Entity
public class Department {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "department_id") // Foreign key column in the Employee table
    private List<Employee> employees = new ArrayList<>();
    // ... getters, setters
}

// Employee.java
@Entity
public class Employee {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private double salary;
    // No direct reference to Department
    // ... getters, setters
}
```

**Note**: While technically possible, `OneToMany` unidirectional with `@JoinColumn` on the "one" side can lead to less optimal SQL (extra update queries) because the `Department` is managing the foreign keys in the `Employee` table. It's generally better to make the `ManyToOne` side the owning side.

#### Bidirectional (Department and Employee know about each other):
`Department` has a collection of `Employees`, and `Employee` has a reference to `Department`. The `ManyToOne` side is typically the owning side, holding the foreign key.

```java
// Department.java (inverse side for OneToMany)
@Entity
public class Department {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    private List<Employee> employees = new ArrayList<>();

    public void addEmployee(Employee employee) {
        employees.add(employee);
        employee.setDepartment(this);
    }

    public void removeEmployee(Employee employee) {
        employees.remove(employee);
        employee.setDepartment(null);
    }
    // ... getters, setters
}

// Employee.java (owning side for ManyToOne)
@Entity
public class Employee {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private double salary;

    @ManyToOne(fetch = FetchType.LAZY) // Always use LAZY for ManyToOne unless there's a strong reason not to
    @JoinColumn(name = "department_id", nullable = false) // Foreign key column in Employee table
    private Department department;
    // ... getters, setters
}
```

**Best Practice**: For `OneToMany` relationships, always make the `ManyToOne` side the owning side. This is because the foreign key usually resides in the "many" side's table, making it more intuitive for the "many" side to manage the relationship. Always ensure to manage both sides of a bidirectional relationship in your code (e.g., using helper methods like `addEmployee`/`removeEmployee`)

### 3. ManyToMany Relationship
A `ManyToMany` relationship indicates that records in both tables can be associated with multiple records in the other table. This relationship is always represented by a separate join table (or association table) in the database.

#### Example: Student and Course
A student can enroll in many courses, and a course can have many students.

#### Diagram
```
+---------+         +------------+         +--------+
| Student |---------| Student_Course |---------| Course |
+---------+         +------------+         +--------+
| id (PK) |         | student_id | (FK)   | id (PK)|
| name    |         | course_id  | (FK)   | title  |
+---------+         +------------+         +--------+
```

#### Unidirectional (Student owns Courses):
`Student` has a collection of `Courses`, but Course does not know about `Student`.

```java
// Student.java
@Entity
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE}) // Don't cascade REMOVE unless sure
    @JoinTable(name = "student_course", // Name of the join table
               joinColumns = @JoinColumn(name = "student_id"), // FK in join table pointing to Student
               inverseJoinColumns = @JoinColumn(name = "course_id")) // FK in join table pointing to Course
    private Set<Course> courses = new HashSet<>();
    // ... getters, setters
}

// Course.java
@Entity
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    // No direct reference to Student
    // ... getters, setters
}
```

#### Bidirectional (Student and Course know about each other):
Both `Student` and `Course` have collections referencing each other. One side is the owning side, the other is the inverse side using `mappedBy`. The `mappedBy` attribute is always on the inverse side and refers to the property on the owning side

```java
// Student.java (owning side)
@Entity
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(name = "student_course",
               joinColumns = @JoinColumn(name = "student_id"),
               inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>();

    public void addCourse(Course course) {
        this.courses.add(course);
        course.getStudents().add(this); // Maintain consistency on inverse side
    }

    public void removeCourse(Course course) {
        this.courses.remove(course);
        course.getStudents().remove(this); // Maintain consistency on inverse side
    }
    // ... getters, setters
}

// Course.java (inverse side)
@Entity
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    @ManyToMany(mappedBy = "courses") // "courses" refers to the field in Student entity
    private Set<Student> students = new HashSet<>();
    // ... getters, setters
}
```

**Common Pitfall**: For `ManyToMany`, `orphanRemoval = true` is typically not used, as it would imply that removing a link deletes the associated entity, which is usually not desired (e.g., removing a student from a course doesn't delete the course). Careful handling of `CascadeType.REMOVE` is crucial. Often, it's better to manage the join table explicitly, especially if it has additional attributes (e.g., `enrollment_date`). In such cases, the `ManyToMany` becomes two `OneToMany`/`ManyToOne` relationships to an intermediate entity representing the join table.

## Query Mechanisms
Spring Data JPA offers various ways to query your database.

### 1. Derived Queries (Query Methods)
Spring Data JPA automatically generates queries based on the names of your methods in repository interfaces. This is the simplest and most common way to define queries for basic CRUD and simple conditions.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByFirstName(String firstName);
    List<User> findByEmailContaining(String keyword);
    Optional<User> findByEmailAndLastName(String email, String lastName);
    List<User> findByAgeGreaterThanOrderByLastNameAsc(int age); // Example with sorting
}
```

**Limitations**: Becomes cumbersome for complex queries involving joins, aggregations, or dynamic conditions.

### 2. JPQL (Java Persistence Query Language)
JPQL is an object-oriented query language defined by the JPA specification. It's similar to SQL but operates on entities and their attributes rather than tables and columns. You define JPQL queries using the `@Query` annotation.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findUserByEmailJPQL(@Param("email") String email);

    @Query("SELECT u FROM User u JOIN FETCH u.address WHERE u.id = :id")
    Optional<User> findUserWithAddressById(@Param("id") Long id);

    @Query("UPDATE User u SET u.email = :email WHERE u.id = :id")
    @Modifying // Required for update/delete queries
    @Transactional // Required for update/delete queries outside of standard save/delete
    int updateEmailById(@Param("id") Long id, @Param("email") String email);
}
```

**Solving N+1 Problems with** `FETCH JOIN` <br>
The "N+1 problem" is a common performance pitfall where fetching a collection of parent entities results in N additional queries (one for each parent) to fetch their associated child entities, especially when using `FetchType.LAZY` for relationships

**Example of N+1**:
If `User` has a `OneToMany` relationship with `Orders` and `FetchType.LAZY` is used:
```java
// User entity
@Entity
public class User {
    // ...
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
}

// In a service method
@Transactional(readOnly = true)
public List<User> getAllUsersWithOrders() {
    List<User> users = userRepository.findAll(); // 1 query to get all users
    for (User user : users) {
        user.getOrders().size(); // N queries to fetch orders for each user (if not already initialized)
    }
    return users;
}
```

**Solution**: `FETCH JOIN`
`FETCH JOIN` allows you to fetch associated entities along with the parent entity in a single query, avoiding the N+1 problem.
```java
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT u FROM User u JOIN FETCH u.orders")
    List<User> findAllUsersWithOrders(); // Fetches users and their orders in one query

    @Query("SELECT u FROM User u JOIN FETCH u.address WHERE u.id = :id")
    Optional<User> findUserWithAddressById(@Param("id") Long id); // Good for OneToOne EAGER
}
```

**Note**: `FETCH JOIN` should be used judiciously. Fetching too much data eagerly can lead to `Cartesian Product` issues (especially with multiple `OneToMany` or `ManyToMany` collections in a single query), causing large result sets and memory problems.

#### Pagination and Sorting with `Pageable`
Spring Data JPA makes pagination and sorting incredibly easy using the `Pageable` interface.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Returns a Page of User entities, including pagination info
    Page<User> findByLastNameContainingIgnoreCase(String lastName, Pageable pageable);

    // Returns a List, but applies sorting
    List<User> findByFirstName(String firstName, Sort sort);
}

// In your service/controller
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public Page<User> findUsersPaginated(int page, int size, String sortBy) {
        Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy));
        return userRepository.findAll(pageable); // findAll also accepts Pageable
    }

    public Page<User> searchUsers(String lastName, int page, int size, String sortBy, String sortDirection) {
        Sort.Direction direction = Sort.Direction.fromString(sortDirection);
        Pageable pageable = PageRequest.of(page, size, Sort.by(direction, sortBy));
        return userRepository.findByLastNameContainingIgnoreCase(lastName, pageable);
    }
}
```

`Page<T>` contains total elements, total pages, current page number, page size, content list, etc.

### 3. Native Queries
You can execute native SQL queries directly using `@Query` with `nativeQuery = true`. Use this when JPQL or derived queries are insufficient (e.g., highly complex database-specific functions, stored procedures, or performance-critical reporting queries).

```java
public interface UserRepository extends JpaRepository<User, Long> {
    @Query(value = "SELECT * FROM users WHERE email LIKE %:domain%", nativeQuery = true)
    List<User> findUsersByEmailDomainNative(@Param("domain") String domain);

    @Query(value = "SELECT u.first_name, u.email FROM users u WHERE u.id = :id", nativeQuery = true)
    List<Object[]> findUserDetailsByIdNative(@Param("id") Long id); // Returns array of objects
}
```

**Limitations**:
- **Loss of Database Agnosticism**: Your queries become tied to a specific database dialect.
- **No Automatic ORM Mapping**: You might need to manually map results to objects (e.g., using `ResultSetMapping` or consuming `List<Object[]>`).
- **No Dirty Checking**: Changes to entities loaded via native queries are not automatically tracked by the persistence context unless you explicitly merge them.

### 4. Criteria API
The JPA Criteria API allows you to build dynamic, type-safe queries programmatically. It's an alternative to JPQL for building queries at runtime, particularly useful when the query conditions are not known in advance.

```java
// Example: Dynamically query Users by firstName and/or lastName
@Service
public class UserService {

    @PersistenceContext
    private EntityManager entityManager;

    public List<User> findUsersByCriteria(String firstName, String lastName) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> cq = cb.createQuery(User.class);
        Root<User> user = cq.from(User.class);

        List<Predicate> predicates = new ArrayList<>();

        if (firstName != null && !firstName.isEmpty()) {
            predicates.add(cb.like(user.get("firstName"), "%" + firstName + "%"));
        }
        if (lastName != null && !lastName.isEmpty()) {
            predicates.add(cb.like(user.get("lastName"), "%" + lastName + "%"));
        }

        cq.where(predicates.toArray(new Predicate[0]));

        return entityManager.createQuery(cq).getResultList();
    }
}
```

**Limitations of Criteria API in Complex Conditions:** <br>
While powerful for dynamic queries, the Criteria API can become verbose and harder to read for very complex queries with many joins, subqueries, or intricate WHERE clauses. The lack of direct SQL syntax can sometimes make debugging harder.

### 5. Specification API (Spring Data JPA)
Spring Data JPA's `Specification` API provides a cleaner, type-safe, and reusable solution for building dynamic queries, especially when dealing with complex conditions that need to be composed. It leverages the JPA Criteria API internally but offers a more fluent and maintainable external interface

To use `Specification`, your repository must extend `JpaSpecificationExecutor`
```java
public interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {
    // No methods needed here for basic specifications
}

// UserSpecification.java
public class UserSpecification {

    public static Specification<User> hasFirstName(String firstName) {
        return (root, query, criteriaBuilder) ->
                criteriaBuilder.like(root.get("firstName"), "%" + firstName + "%");
    }

    public static Specification<User> hasLastName(String lastName) {
        return (root, query, criteriaBuilder) ->
                criteriaBuilder.like(root.get("lastName"), "%" + lastName + "%");
    }

    public static Specification<User> hasEmailDomain(String domain) {
        return (root, query, criteriaBuilder) ->
                criteriaBuilder.like(root.get("email"), "%@" + domain + "%");
    }

    public static Specification<User> ageGreaterThan(int age) {
        return (root, query, criteriaBuilder) ->
                criteriaBuilder.greaterThan(root.get("age"), age);
    }
}

// In your service
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public List<User> findUsersFiltered(String firstName, String lastName, String emailDomain) {
        Specification<User> spec = Specification.where(null); // Start with a null specification

        if (firstName != null && !firstName.isEmpty()) {
            spec = spec.and(UserSpecification.hasFirstName(firstName));
        }
        if (lastName != null && !lastName.isEmpty()) {
            spec = spec.and(UserSpecification.hasLastName(lastName));
        }
        if (emailDomain != null && !emailDomain.isEmpty()) {
            spec = spec.and(UserSpecification.hasEmailDomain(emailDomain));
        }

        return userRepository.findAll(spec);
    }

    // Example combining with pagination and sorting
    public Page<User> findUsersByAgeAndName(int age, String name, Pageable pageable) {
        Specification<User> spec = Specification.where(UserSpecification.ageGreaterThan(age))
                                                .and(UserSpecification.hasFirstName(name)
                                                                       .or(UserSpecification.hasLastName(name)));
        return userRepository.findAll(spec, pageable);
    }
}
```

**Benefits of Specification API**:
- **Type-Safe**: Reduces the chance of runtime errors compared to string-based JPQL or native queries.
- **Reusable**: Individual specifications (e.g., `hasFirstName`) can be reused across different queries.
- **Composable**: Specifications can be combined using `and()`, `or()`, and `not()` methods, allowing for complex filter construction.
- **Readability**: Can be more readable than raw Criteria API for complex compositions.

## Common Pitfalls, Performance Considerations, and Best Practices
### Common Pitfalls
#### 1. Lazy Loading Issues (`LazyInitializationException``)
- **Problem**: Occurs when you try to access a lazily loaded association (e.g., a collection or related entity) outside of an active `PersistenceContext` (i.e., after the `EntityManager` has closed).
- **Solution**:
  - `@Transactional`: Ensure the entire operation that requires accessing lazy associations is within a `@Transactional` boundary.
  - `FETCH JOIN` **(JPQL) or** `EntityGraph`: Eagerly fetch necessary associations using `FETCH JOIN` in JPQL queries or `@EntityGraph` annotations.
  - **DTO Projections**: Load only the required data (flattening the graph) directly into DTOs, avoiding the need to access lazy collections

#### 2. Cascade Problems:
- **Problem**: Incorrect use of `CascadeType.ALL` can lead to unintended deletions (e.g., deleting a parent deletes all its children, which might not be desired).
- **Solution**: Be precise with `CascadeType`. Use `PERSIST` and `MERGE` frequently, but use `REMOVE` and `ALL` with extreme caution and only when the lifecycle of the child is strictly dependent on the parent (e.g., `Address` and `User` in a strong `OneToOne`). For `OneToMany`/`ManyToMany`, `orphanRemoval = true` should also be used carefully

#### 3. Inefficient Queries (N+1 Problem):
- **Problem**: As discussed, fetching a list of parent entities and then individually querying for their children leads to N+1 queries.
- **Solution**: `FETCH JOIN (JPQL)`, `@EntityGraph`, or `BatchSize` annotation on lazy collections

#### 4. Misusing `findAll()` for Large Datasets:
- **Problem**: Calling `repository.findAll()` on entities with a large number of records can lead to `OutOfMemoryError` and slow performance as it attempts to load all data into memory.
- **Solution**: Always use pagination (`Pageable`) when dealing with potentially large datasets

#### 5. Forgetting `@Modifying` for Update/Delete Queries:
- **Problem**: JPQL `@Query` methods for `UPDATE` or `DELETE` statements will not execute correctly without `@Modifying` and a transactional context.
- **Solution**: Always add `@Modifying` and ensure the method is within a `@Transactional` block

### Performance Considerations
#### 1. Query Tuning:
- **Analyze SQL**: Enable `spring.jpa.show-sql=true` and `spring.jpa.properties.hibernate.format_sql=true` to see the generated SQL. This is invaluable for identifying inefficient queries.
- **Use** `FETCH JOIN` **and** `EntityGraph` **wisely**: Optimize for common use cases.
- **Avoid** `Cartesian Product`: If fetching multiple `OneToMany` collections, consider separate queries or DTO projections.
- **Indexing**: Ensure frequently queried columns have appropriate database indexes. This is a database-level optimization, but crucial for ORM performance.

#### 2. Caching Strategy:
- **First-Level Cache**: Understand its transactional scope.
- **Second-Level Cache**: Implement and configure L2 cache (Ehcache, Redis) for frequently accessed, relatively static data to reduce database hits. Configure appropriate `CacheConcurrencyStrategy`.
- **Query Cache**: Hibernate can also cache query results, useful for frequently executed queries with the same parameters

#### 3. DTO Projections:
- **Problem**: Fetching entire entity graphs when only a subset of data is needed.
- **Solution**: Use DTO projections to fetch only the required columns, reducing data transfer over the network and memory consumption
    - **Interface-based Projections**:
      ```java
      public interface UserProjection {
        String getFirstName();
        String getEmail();
      }
      
      public interface UserRepository extends JpaRepository<User, Long> {
        List<UserProjection> findByLastName(String lastName);
      }
      ```
    - **Class-based Projections (Constructor Expression in JPQL)**:
      ```java
      public class UserDto {
        private String firstName;
        private String email;
        public UserDto (String firstName, String email) {
            this.firstName = firstName;
            this.email = email;
        }
        //getters
      }
      
      public interface UserRepository extends JpaRepository<User, Long> {
        @Query("SELECT new com.example.springbootjpademo.dto.UserDto(u.firstName, u.email) FROM User u WHERE u.lastName = :lastName")
        List<UserDto> findUserDtoByLastName(@Param("lastName") Stirng lastName);
      }
      ```    

#### 4. Batch Processing
- For large-scale inserts, updates, or deletes, consider batch processing. Hibernate can be configured to use JDBC batching to send multiple operations in a single roundtrip to the database.
- `spring.jpa.properties.hibernate.jdbc.batch_size=20`
- `spring.jpa.properties.hibernate.order_inserts=true`
- `spring.jpa.properties.hibernate.order_updates=true`

### Best Practices
1. **Service Layer for Transaction Management**: Keep @Transactional annotations primarily on the service layer methods. This ensures that a single business operation is atomic and manages the persistence context lifecycle. Avoid transactional boundaries in the controller layer.
2. **Use** `FetchType.LAZY` **by Default**: For `OneToMany`, `ManyToMany`, and sometimes `OneToOne` relationships, always default to `FetchType.LAZY`. This avoids eagerly loading potentially large collections unnecessarily. Use `FETCH JOIN` or `EntityGraph` when you explicitly need the associated data.
3. **Prefer Spring Data JPA Query Methods and JPQL**: These are typically more readable and maintainable than native queries or complex Criteria API constructions for most scenarios.
4. **Use `Optional` for Single Results**: Repository methods returning a single entity should return `Optional<Entity>` to clearly indicate that the result might be absent, forcing explicit null handling.
5. **Clean Code and Domain-Driven Design**: Keep your entities clean and focused on your domain model. Separate concerns using DTOs for API responses and request bodies.
6. **Validate Inputs**: Before persisting entities, validate user inputs to ensure data integrity and prevent security vulnerabilities.
7. **Consider Optimistic Locking (**`@Version`**)**: For concurrent updates, use `@Version` annotation on an entity field to implement optimistic locking. This prevents "lost updates" by ensuring that an entity hasn't been modified by another transaction since it was last read

```java
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private double price;

    @Version // Automatically managed by JPA for optimistic locking
    private int version;
    // ...
}
```