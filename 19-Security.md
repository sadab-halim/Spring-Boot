# Spring Security

## Spring Security
### Overview: What is Spring Security?
Spring Security is a powerful and highly customizable authentication and access-control framework for Java applications. It's an integral part of the Spring ecosystem, providing robust security solutions for a wide range of applications, from simple web applications to complex enterprise systems

**Why it's the preferred choice:** <br>
- **Comprehensive Security Features**: It offers a wide array of security features out-of-the-box, including:
  - Authentication mechanisms (form-based, HTTP Basic, OAuth2, JWT, LDAP, etc.)
  - Authorization at the URL and method level
  - Protection against common vulnerabilities (CSRF, session fixation, clickjacking)
  - Session management
  - Remember-me functionality

- **Declarative Security**: It promotes a declarative security approach, allowing developers to define security rules using annotations (e.g., `@PreAuthorize`) or configuration, leading to cleaner and more maintainable code.

### Spring Security Architecture
Spring Security's architecture is built around a series of filters that intercept incoming requests. This filter chain forms the backbone of its request processing.

#### The Filter Chain
When a request arrives at a Spring application, it first passes through a series of `javax.servlet.Filter` (or `jakarta.servlet.Filter` in Jakarta EE) instances. Spring Security inserts its own filters into this chain, typically represented by a `FilterChainProxy`.

The `FilterChainProxy` is the main Spring Security filter. It delegates to a series of `SecurityFilterChain` objects. Each `SecurityFilterChain` can have a different set of security filters, allowing for fine-grained control over which filters apply to which requests.

#### Key Architectural Components:
1. `SecurityFilterChain`:
   - An interface that defines a chain of `Filter` instances that will be applied to a specific set of HTTP requests.
   - A Spring Boot application often has a default `SecurityFilterChain` configured, but you can define multiple chains, each with different security rules, to apply to different URL patterns.
   - For example, you might have one SecurityFilterChain for `/api/**` that uses JWT authentication and another for `/admin/**` that requires form login
2. `AuthenticationManager`:
   - The central component responsible for authenticating a user.
   - It's an interface with a single method: `authenticate(Authentication authentication)`.
   - It doesn't perform the actual authentication itself but delegates to one or more `AuthenticationProvider` instances.
   - If authentication is successful, it returns a fully populated `Authentication` object. If not, it throws an `AuthenticationException`
3. `AuthenticationProvider`:
   - Performs the actual authentication logic for a specific type of authentication.
   - Examples include `DaoAuthenticationProvider` (for username/password from a `UserDetailsService`), `JwtAuthenticationProvider`, `OAuth2LoginAuthenticationProvider`, etc.
   - An `AuthenticationManager` can be configured with multiple `AuthenticationProvider`s, and it will try each provider until one successfully authenticates the user or all fail
4. `UserDetailsService`:
   - An interface used by `AuthenticationProvider`s (specifically `DaoAuthenticationProvider`) to retrieve user details from a data source (e.g., database, LDAP).
   - It has one method: `loadUserByUsername(String username)`.
   - It returns a `UserDetails` object, which contains information like the username, password, authorities (roles), and account status (enabled, locked, expired, etc.).
   - You implement this interface to tell Spring Security how to load your application's users.
5. `Authentication` **(Object)**:
   - A core interface representing the authentication request or the currently authenticated principal.
   - Before authentication, it typically holds the user's credentials (e.g., username and password).
   - After successful authentication, it holds the principal (the authenticated user's identity), their granted authorities (roles), and any other relevant security information.
   - It is stored in the `SecurityContext`
6. `SecurityContext`:
   - Holds the `Authentication` object of the currently authenticated principal.
   - It's crucial for security context propagation.
   - By default, it's stored in a `ThreadLocal` in web applications, meaning the `Authentication` object is available throughout the request processing for that specific thread
7. `SecurityContextHolder`:
   - A utility class that provides access to the `SecurityContext`.
   - You can retrieve the current `Authentication` object using `SecurityContextHolder.getContext().getAuthentication()`.
   - This allows you to access the authenticated user's details and roles anywhere in your application code

#### How Components Interact During Request Processing
1. **Request Arrival**: An HTTP request arrives at the application.
2. `FilterChainProxy`: The request first hits the `FilterChainProxy` (Spring Security's main filter).
3. `SecurityFilterChain` **Selection**: The `FilterChainProxy` determines which `SecurityFilterChain` to apply based on the request's URL pattern.
4. **Filter Execution**: The filters within the selected `SecurityFilterChain` are executed in a specific order. Common filters include:
   - `UsernamePasswordAuthenticationFilter`: Intercepts form login requests, extracts username and password.
   - `BasicAuthenticationFilter`: Handles HTTP Basic authentication.
   - `BearerTokenAuthenticationFilter`: Handles JWT or OAuth2 bearer tokens.
   - `ExceptionTranslationFilter`: Catches authentication and access-denied exceptions and initiates appropriate responses (e.g., redirect to login page, send 401/403).
   - `FilterSecurityInterceptor`: Performs authorization decisions based on the authenticated `Authentication` object and configured access rules.
5. **Authentication Triggered**: When an authentication-related filter (e.g., `UsernamePasswordAuthenticationFilter`) receives a request that requires authentication (e.g., a login attempt), it creates an `Authentication` object (e.g., `UsernamePasswordAuthenticationToken`) and passes it to the `AuthenticationManager`.
6. `AuthenticationManager` **Delegation**: The `AuthenticationManager` iterates through its configured `AuthenticationProvider`s, attempting to authenticate the `Authentication` object.
7. `AuthenticationProvider` & `UserDetailsService`: If a `DaoAuthenticationProvider` is used, it calls the configured `UserDetailsService` to load the `UserDetails` for the given username. It then compares the provided password with the stored password (after decoding, if applicable).
8. **Successful Authentication**: If authentication succeeds, the `AuthenticationProvider` returns a fully populated `Authentication` object (containing `UserDetails` and authorities) to the `AuthenticationManager`.
9. `Authentication` **Stored in** `SecurityContext`: The `AuthenticationManager` then places this authenticated `Authentication` object into the `SecurityContext` via `SecurityContextHolder`. This context is typically bound to the current thread.
10. **Authorization**: Later filters in the chain (e.g., `FilterSecurityInterceptor`) use the `Authentication` object from the `SecurityContext` to perform authorization checks against configured access rules for the requested resource.
11. **Request Continues/Denied**: If authorization passes, the request proceeds to the application's controllers. If it fails, an access-denied exception is thrown, handled by `ExceptionTranslationFilter`.

## Step-by-Step Configuration in a Spring Boot Application
Spring Boot simplifies Spring Security configuration significantly through auto-configuration.

### Basic In-Memory Authentication
This is useful for development or simple applications.

#### 1. Add Dependency:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

#### 2. Basic Configuration (Java Config):
Spring Boot 3.x+ uses `jakarta.servlet` and a slightly different security configuration approach (Lambda DSL for `SecurityFilterChain`).

```java
// src/main/java/com/example/securitydemo/config/SecurityConfig.java
package com.example.securitydemo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity // Enables Spring Security's web security features
public class SecurityConfig {

    // Define a PasswordEncoder bean
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // Strong password hashing
    }

    // Configure in-memory user details
    @Bean
    public UserDetailsService userDetailsService(PasswordEncoder passwordEncoder) {
        UserDetails user = User.withUsername("user")
            .password(passwordEncoder.encode("password")) // Encode the password!
            .roles("USER")
            .build();

        UserDetails admin = User.withUsername("admin")
            .password(passwordEncoder.encode("adminpass"))
            .roles("ADMIN", "USER")
            .build();

        return new InMemoryUserDetailsManager(user, admin);
    }

    // Configure the SecurityFilterChain
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/public/**").permitAll() // Allow public access
                .requestMatchers("/admin/**").hasRole("ADMIN") // Only ADMIN role can access
                .anyRequest().authenticated() // All other requests require authentication
            )
            .formLogin(form -> form
                .loginPage("/login") // Custom login page (if you have one)
                .defaultSuccessUrl("/welcome", true) // Redirect after successful login
                .permitAll() // Allow everyone to access the login page
            )
            .logout(logout -> logout
                .permitAll()
            );
        return http.build();
    }
}
```

#### 3. Create a Basic Controller and HTML Page:
```java
// src/main/java/com/example/securitydemo/controller/HomeController.java
package com.example.securitydemo.controller;

import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HomeController {

    @GetMapping("/")
    public String home() {
        return "Welcome to the home page!";
    }

    @GetMapping("/public/hello")
    public String publicHello() {
        return "Hello from the public endpoint!";
    }

    @GetMapping("/admin/dashboard")
    public String adminDashboard() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        String username = authentication.getName();
        return "Welcome to the admin dashboard, " + username + "!";
    }

    @GetMapping("/user/profile")
    public String userProfile() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        String username = authentication.getName();
        return "Welcome to your profile, " + username + "!";
    }

    @GetMapping("/login")
    public String login() {
        return "Please log in!"; // In a real app, this would return an HTML login form
    }

    @GetMapping("/welcome")
    public String welcome() {
        return "You are successfully logged in!";
    }
}
```

Now, if you run this, `/public/hello` will be accessible, `/admin/dashboard` will require `ADMIN` role, and other paths will require authentication (using a default Spring Security login page if you don't provide `/login.html`)

### Securing REST APIs (Stateless Authentication with JWTs)
For REST APIs, stateless authentication using JWTs is common as it avoids session management on the server.

#### 1. Add Dependencies
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
</dependency>
```

#### 2. JWT Utility Class
```java
// src/main/java/com/example/securitydemo/jwt/JwtUtil.java
package com.example.securitydemo.jwt;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import java.security.Key;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

@Component
public class JwtUtil {

    @Value("${jwt.secret}") // Store secret in application.properties/yml
    private String secret;

    @Value("${jwt.expiration}")
    private long expiration; // in milliseconds

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
    }

    private Boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        // You can add roles or other claims here
        claims.put("roles", userDetails.getAuthorities().stream()
                               .map(grantedAuthority -> grantedAuthority.getAuthority())
                               .toList());
        return createToken(claims, userDetails.getUsername());
    }

    private String createToken(Map<String, Object> claims, String subject) {
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(subject)
                .setIssuedAt(new Date(System.currentTimeMillis()))
                .setExpiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(getSigningKey(), SignatureAlgorithm.HS256)
                .compact();
    }

    public Boolean validateToken(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
    }

    private Key getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secret);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

#### 3. JWT Authentication Filter
This filter will intercept requests, extract the JWT, validate it, and set the `Authentication` in the `SecurityContext`

```java
// src/main/java/com/example/securitydemo/jwt/JwtRequestFilter.java
package com.example.securitydemo.jwt;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class JwtRequestFilter extends OncePerRequestFilter {

    @Autowired
    private UserDetailsService userDetailsService; // Our custom or default UserDetailsService

    @Autowired
    private JwtUtil jwtUtil;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        final String authorizationHeader = request.getHeader("Authorization");

        String username = null;
        String jwt = null;

        if (authorizationHeader != null && authorizationHeader.startsWith("Bearer ")) {
            jwt = authorizationHeader.substring(7);
            username = jwtUtil.extractUsername(jwt);
        }

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);

            if (jwtUtil.validateToken(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken usernamePasswordAuthenticationToken = new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                usernamePasswordAuthenticationToken
                        .setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(usernamePasswordAuthenticationToken);
            }
        }
        chain.doFilter(request, response);
    }
}
```

#### 4. Update `SecurityConfig` for JWT Authentication
```java
// src/main/java/com/example/securitydemo/config/SecurityConfig.java (updated)
package com.example.securitydemo.config;

import com.example.securitydemo.jwt.JwtRequestFilter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity; // For @PreAuthorize
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy; // For stateless session
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter; // For JWT

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true) // Enable method-level security
public class SecurityConfig {

    @Autowired
    private UserDetailsService userDetailsService; // Autowire our UserDetailsService (from In-Memory or custom)

    @Autowired
    private JwtRequestFilter jwtRequestFilter; // Autowire our JWT filter

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    // Configure AuthenticationManager
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration authenticationConfiguration) throws Exception {
        return authenticationConfiguration.getAuthenticationManager();
    }

    // Configure DaoAuthenticationProvider with UserDetailsService and PasswordEncoder
    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable()) // Disable CSRF for stateless REST APIs
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/authenticate").permitAll() // Allow unauthenticated access for login
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // Make session stateless
            )
            .authenticationProvider(authenticationProvider()) // Set our custom authentication provider
            .addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class); // Add JWT filter before UPFA
        return http.build();
    }
}
```

5. **Add Login Endpoint**:
```java
// src/main/java/com/example/securitydemo/controller/AuthController.java
package com.example.securitydemo.controller;

import com.example.securitydemo.jwt.JwtUtil;
import com.example.securitydemo.model.AuthenticationRequest;
import com.example.securitydemo.model.AuthenticationResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private UserDetailsService userDetailsService;

    @Autowired
    private JwtUtil jwtUtil;

    @PostMapping("/authenticate")
    public ResponseEntity<?> createAuthenticationToken(@RequestBody AuthenticationRequest authenticationRequest) throws Exception {
        try {
            authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(authenticationRequest.getUsername(), authenticationRequest.getPassword())
            );
        } catch (Exception e) {
            throw new Exception("Incorrect username or password", e);
        }

        final UserDetails userDetails = userDetailsService.loadUserByUsername(authenticationRequest.getUsername());
        final String jwt = jwtUtil.generateToken(userDetails);

        return ResponseEntity.ok(new AuthenticationResponse(jwt));
    }
}
```

6. Create Request / Response Models
```java
// src/main/java/com/example/securitydemo/model/AuthenticationRequest.java
package com.example.securitydemo.model;

public class AuthenticationRequest {
    private String username;
    private String password;

    // Getters and Setters
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
}

// src/main/java/com/example/securitydemo/model/AuthenticationResponse.java
package com.example.securitydemo.model;

public class AuthenticationResponse {
    private final String jwt;

    public AuthenticationResponse(String jwt) {
        this.jwt = jwt;
    }

    public String getJwt() { return jwt; }
}
```

7. Add JWT Secret to `application.properties`
```
# application.properties
jwt.secret=a_super_secret_key_that_is_at_least_256_bits_long_and_base64_encoded_for_production
jwt.expiration=3600000 # 1 hour in milliseconds
```

Now, you can send a POST request to `/authenticate` with username/password, get a JWT, and then use that JWT in the `Authorization` header (`Bearer <token>`) for subsequent requests to secured endpoints

### OAuth2 (Client Credentials or Authorization Code Flow)
Integrating OAuth2 (e.g., with Google, GitHub) for login. Here's a simplified example for client setup:

#### 1. Add Dependency
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

#### 2. Configure OAuth2 Clients in `application.properties`
For Google (Authorization Code Flow):
```
# application.properties
spring.security.oauth2.client.registration.google.client-id=YOUR_GOOGLE_CLIENT_ID
spring.security.oauth2.client.registration.google.client-secret=YOUR_GOOGLE_CLIENT_SECRET
spring.security.oauth2.client.registration.google.scope=openid,profile,email
spring.security.oauth2.client.registration.google.redirect-uri={baseUrl}/oauth2/callback/{registrationId}
```

#### 3. Update `SecurityConfig`
Spring Boot's auto-configuration takes care of much of the OAuth2 integration once the properties are set. You might just need to ensure the login page (or endpoints) allows for OAuth2 redirects.

```java
// In SecurityConfig.java, within securityFilterChain:
.oauth2Login(oauth2 -> oauth2
    .loginPage("/login") // Can still use your custom login page
    .defaultSuccessUrl("/welcome", true)
    .failureUrl("/login?error")
    // .userInfoEndpoint(userInfo -> userInfo.userService(customOAuth2UserService())) // Custom user service if needed
)
```

#### 4. Custom `OAuth2UserService` (Optional but often needed):
To map OAuth2 user attributes to your internal UserDetails and roles
```java
// src/main/java/com/example/securitydemo/oauth2/CustomOAuth2UserService.java
package com.example.securitydemo.oauth2;

import org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest;
import org.springframework.security.oauth2.core.OAuth2AuthenticationException;
import org.springframework.security.oauth2.core.user.DefaultOAuth2User;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Service;

import java.util.Collections;
import java.util.Map;

@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2User oauth2User = super.loadUser(userRequest);

        // You would typically save/update user in your DB here
        // And map roles based on email, etc.
        String userNameAttributeName = userRequest.getClientRegistration()
                                                  .getProviderDetails()
                                                  .getUserInfoEndpoint()
                                                  .getUserNameAttributeName();

        // For simplicity, granting a default ROLE_USER
        return new DefaultOAuth2User(
                Collections.singletonList(() -> "ROLE_USER"), // Assign roles here
                oauth2User.getAttributes(),
                userNameAttributeName
        );
    }
}
```

Then register this custom service in `SecurityConfig`: <br>
```java
// In SecurityConfig, inside oauth2Login:
// .userInfoEndpoint(userInfo -> userInfo.userService(new CustomOAuth2UserService()))
```

### Method-Level Security with `@PreAuthorize`
This allows you to secure individual methods in your service or controller layer.

#### 1. Enable Method Security
Add `@EnableMethodSecurity` (or `@EnableGlobalMethodSecurity` for older versions) to your `SecurityConfig`.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true) // Crucial for @PreAuthorize
public class SecurityConfig {
    // ...
}
```

#### 2. Use `@PreAuthorize` on Methods
```java
// src/main/java/com/example/securitydemo/service/MyService.java
package com.example.securitydemo.service;

import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;

@Service
public class MyService {

    @PreAuthorize("hasRole('ADMIN')")
    public String getAdminData() {
        return "Secret admin data!";
    }

    @PreAuthorize("hasAnyRole('USER', 'ADMIN')")
    public String getUserAndAdminData() {
        return "Data accessible by users and admins!";
    }

    @PreAuthorize("#username == authentication.name") // SpEL for argument-based security
    public String getMyPersonalData(String username) {
        return "Personal data for " + username;
    }
}
```

### Integrating Custom Login Flows
You can provide your own HTML login page and handle the form submission.

#### 1. Update `SecurityConfig`s `formLogin`:
```java
// In SecurityConfig.java, inside securityFilterChain:
.formLogin(form -> form
    .loginPage("/login") // Specify your custom login page URL
    .loginProcessingUrl("/perform_login") // URL where your login form POSTs
    .defaultSuccessUrl("/welcome", true) // Redirect after success
    .failureUrl("/login?error") // Redirect on failure
    .permitAll()
)
```

#### 2. Create a Custom Login Controller (Optional, but good for rendering):
```java
// src/main/java/com/example/securitydemo/controller/LoginController.java
package com.example.securitydemo.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class LoginController {

    @GetMapping("/login")
    public String showLoginPage() {
        return "login"; // This resolves to src/main/resources/templates/login.html (Thymeleaf/JSP)
    }
}
```

#### 3. Create `login.html` (e.g., using Thymeleaf):
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Login</title>
</head>
<body>
    <h2>Login</h2>
    <div th:if="${param.error}">
        Invalid username and password.
    </div>
    <div th:if="${param.logout}">
        You have been logged out.
    </div>
    <form th:action="@{/perform_login}" method="post">
        <div>
            <label> Username: <input type="text" name="username"/> </label>
        </div>
        <div>
            <label> Password: <input type="password" name="password"/> </label>
        </div>
        <div>
            <input type="submit" value="Sign In"/>
        </div>
    </form>
    <br/>
    <a href="/oauth2/authorization/google">Login with Google</a>
</body>
</html>
```

Remember to add Thymeleaf dependency for this: <br>
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

## Best Practices for Customizing Security Behavior
- **Use Java Configuration**: Always prefer Java-based configuration (`@Configuration` classes) over XML.
- **Strong Password Encoding**: Always use `BCryptPasswordEncoder` or `Pbkdf2PasswordEncoder` for hashing passwords. Never store plain text passwords.
- **Stateless vs. Stateful Authentication**:
  - **Stateful**: Uses server-side sessions (e.g., JSESSIONID cookies). Suitable for traditional web applications where the server maintains session state. Spring Security's default `formLogin` is stateful. CSRF protection is crucial.
  - **Stateless**: No server-side sessions. Each request is self-contained (e.g., JWT in Authorization header). Ideal for REST APIs, mobile apps, and microservices. Improves scalability. Requires proper token validation and potentially token revocation mechanisms. CSRF protection is generally not needed if entirely stateless
- **Role-Based Access Control (RBAC)**: Define clear roles and assign them to users. Use `hasRole()`, `hasAuthority()`, `@PreAuthorize` for authorization.
- **Custom** `UserDetailsService`: For real-world applications, you'll need to implement your own `UserDetailsService` to load users from a database, LDAP, or another identity store.
- **Error Handling**: Implement custom error pages or handlers for `401 Unauthorized` and `403 Forbidden` responses for a better user experience. Spring Security's `ExceptionTranslationFilter` helps with this
- **HTTPS Everywhere**: Always use HTTPS to protect credentials and sensitive data in transit.
- **Secure Headers**: Spring Security can automatically add security-related HTTP headers (e.g., X-Content-Type-Options, X-Frame-Options, HSTS).
- **CSRF Protection**: Essential for stateful web applications that use form submissions. Spring Security enables it by default. For REST APIs, if entirely stateless (JWTs), you can disable it.
- **Least Privilege Principle**: Grant users only the minimum permissions necessary to perform their tasks.

## Security Context Propagation
As mentioned, the `SecurityContext` holds the `Authentication` object. In web applications, by default, Spring Security uses a `ThreadLocal` strategy to store the `SecurityContext`. This means:
- When a request comes in, the `SecurityContext` is populated for the current thread.
- Any code executed within that thread (e.g., service methods, controllers) can access the `Authentication` object using `SecurityContextHolder.getContext().getAuthentication()`.
- At the end of the request, the `ThreadLocal` is cleared to prevent memory leaks and ensure that the context from one request doesn't leak into another thread (in thread pools).

### Use Cases
- Auditing: Log which user performed an action.
- Conditional Logic: Show/hide UI elements based on user roles.
- Data Filtering: Retrieve data specific to the authenticated user

#### Example:
```java
// Anywhere in your application code within a request context
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
if (authentication != null && authentication.isAuthenticated()) {
    String username = authentication.getName();
    System.out.println("Current user: " + username);
    authentication.getAuthorities().forEach(authority -> System.out.println("Role: " + authority.getAuthority()));
}
```

## Common Pitfalls
1. **Misconfigured Filter Order**: Spring Security filters have a specific order. If you add custom filters, ensure they are placed correctly in the `SecurityFilterChain` using `addFilterBefore()` or `addFilterAfter()`. Incorrect order can lead to unexpected behavior (e.g., an unauthenticated request hitting a secured resource before the authentication filter runs).
2. **CSRF Issues**: Disabling CSRF protection (`.csrf(csrf -> csrf.disable())`) in stateful applications is a major security vulnerability. Only disable it if you truly understand the implications (e.g., for stateless REST APIs with JWTs). For stateful apps, ensure your forms include the CSRF token.
3. **Plain Text Passwords**: A fundamental error. Always encode passwords using a strong hashing algorithm.
4. **Incorrect `UserDetailsService` Implementation**: Failing to load roles/authorities correctly, or not handling `UsernameNotFoundException`.
5. **Overly Permissive Rules**: `permitAll()` or `hasAnyRole()` used too broadly can expose sensitive endpoints. Always follow the principle of least privilege.
6. **Ignoring Security Headers**: Not configuring essential security headers can leave your application vulnerable to clickjacking, XSS, etc.
7. **Session Fixation**: Spring Security handles this by default by changing the session ID upon successful authentication, but custom login flows might need extra care.
8. **CORS Issues**: When a browser sends a request to a different origin, CORS preflight requests (OPTIONS method) are made. Spring Security might block these if not configured correctly. Ensure your `SecurityFilterChain` allows `OPTIONS` requests or integrates with Spring's CORS configuration.

## Testing Secured Endpoints

---

# NEW

---

---

## Fundamentals of Spring Security
Spring Security is a powerful and highly customizable authentication and access-control framework. It is the de-facto standard for securing Spring-based applications.

### Spring Security Architecture
At its core, Spring Security's architecture revolves around a chain of Filter implementations. When a request comes into a Spring Boot application, it passes through this filter chain. Each filter has a specific responsibility, such as authentication, authorization, or managing sessions.

**Key Principles**:
- **Delegation**: Spring Security delegates security decisions to various components.
- **Pluggable**: You can easily swap out or add components to customize security behavior.
- **Intercepting Filters**: Security is enforced through a chain of servlet filters

### Core Components
#### `SecurityFilterChain`
The `SecurityFilterChain` is the heart of Spring Security's web security. It's a list of `Filter` instances that Spring Security will apply to incoming HTTP requests. You define this chain in your configuration to specify which URLs are protected, how authentication should occur, and what authorization rules apply.

**How it works**:
When an HTTP request arrives, Spring Security iterates through the `SecurityFilterChain`. Each `Filter` in the chain can perform an action (e.g., check for a valid session, authenticate a user, authorize access to a resource) and then either pass the request to the next filter or terminate the chain (e.g., by redirecting to a login page or sending an error response).

**Example (simplified)**:
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/public/**").permitAll() // Allow public access to /public/**
                .requestMatchers("/admin/**").hasRole("ADMIN") // Only ADMIN role for /admin/**
                .anyRequest().authenticated() // All other requests require authentication
            )
            .formLogin(Customizer.withDefaults()) // Enable form-based login
            .httpBasic(Customizer.withDefaults()); // Enable HTTP Basic authentication
        return http.build();
    }

    // ... other beans like UserDetailsService, PasswordEncoder
}
```

#### `AuthenticationManager`
The `AuthenticationManager` is a central interface that performs authentication. It has a single method: `authenticate(Authentication authentication)`
- **Input**: It takes an `Authentication` object (a token representing the user's credentials, e.g., `UsernamePasswordAuthenticationToken`)
- **Output**: If authentication is successful, it returns a fully populated `Authentication` object (containing the authenticated `Principal` and `GrantedAuthority` objects). If unsuccessful, it throws an `AuthenticationException`

Typically, you don't directly interact with `AuthenticationManager` in your application code for authentication flow, as Spring Security's filters handle this. However, it's crucial for understanding the delegation model. `ProviderManager` is the default implementation of `AuthenticationManager`, which delegates to a list of `AuthenticationProvider` instances

#### `UserDetailsService`
The `UserDetailsService` is a core interface that loads user-specific data during authentication. Its single method `loadUserByUsername(String username)` is responsible for finding a user by their username (or other identifier).
- **Input**: A username.
- **Output**: A `UserDetails` object, which contains information about the user (username, password, authorities, account status like enabled/locked/expired). If the user is not found, it should throw a `UsernameNotFoundException`

You'll implement this interface to integrate your application's user storage (e.g., database, LDAP) with Spring Security.

**Example Implementation**:
```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository; // Assuming you have a UserRepository

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found with username: " + username));

        return new org.springframework.security.core.userdetails.User(
                user.getUsername(),
                user.getPassword(),
                user.getRoles().stream()
                    .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
                    .collect(Collectors.toList())
        );
    }
}
```

#### `PasswordEncoder`
`PasswordEncoder` is a crucial component for securely storing and verifying user passwords. **Never store plain-text passwords**. Spring Security mandates the use of a `PasswordEncoder` since Spring Security 5.

**Common Implementations**: <br>
- `BCryptPasswordEncoder`: Highly recommended for password hashing due to its adaptive nature.
- `Argon2PasswordEncoder`
- `Pbkdf2PasswordEncoder`
- `SCryptPasswordEncoder`

**Configuration Example**: <br>
```java
@Configuration
public class PasswordEncoderConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

#### `Authentication` object and `SecurityContextHolder`
- `Authentication` **object**: Represents the current authentication request or a successfully authenticated user. It contains the `Principal` (the authenticated user, often a `UserDetails` object), `Credentials` (password, token), and `Authorities` (roles/permissions).
- `SecurityContext`: Holds the `Authentication` object for the current security context (typically per thread for web applications).
- `SecurityContextHolder`: A static helper class that provides access to the `SecurityContext`. This allows you to retrieve the currently authenticated user's details anywhere in your application.

**Accessing Current User**: <br>
```java
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
if (authentication != null && authentication.isAuthenticated()) {
String username = authentication.getName();
// UserDetails userDetails = (UserDetails) authentication.getPrincipal(); // Cast if you need more details
// ...
}
```

## Implementing Various Security Mechanisms
Let's walk through different authentication mechanisms you can implement in Spring Boot.

### Form-Based Authentication
This is the most common authentication mechanism for web applications, where users are presented with a login form.

**When to use**: <br>
Ideal for traditional web applications with a user interface where users log in directly via a browser.

**How it works**: <br>
1. User attempts to access a protected resource.
2. Spring Security intercepts the request and redirects to a login page (default `/login` or a custom one).
3. User submits credentials via the login form (typically POST to `/login`).
4. `UsernamePasswordAuthenticationFilter` intercepts this request, creates an `Authentication` object, and delegates to `AuthenticationManager`.
5. `AuthenticationManager` (via `AuthenticationProvider` and `UserDetailsService`) validates credentials.
6. On success, `Authentication` object is stored in `SecurityContextHolder`, and the user is redirected to the originally requested URL.
7. On failure, user is redirected back to the login page with an error message.

**Configuration**: <br>
```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/public/**", "/login", "/error").permitAll() // Allow public access to login page
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login") // Specify custom login page
                .loginProcessingUrl("/perform_login") // URL to process login form (default is /login)
                .defaultSuccessUrl("/dashboard", true) // Redirect after successful login
                .failureUrl("/login?error=true") // Redirect on login failure
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/perform_logout") // URL to trigger logout (default is /logout)
                .logoutSuccessUrl("/login?logout=true") // Redirect after successful logout
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID")
                .permitAll()
            );
        return http.build();
    }

    // UserDetailsService and PasswordEncoder beans as shown previously
}
```

**Controller for Login/Dashboard (Thmyeleaf example)**: <br>
```java
@Controller
public class MyWebController {

    @GetMapping("/login")
    public String login() {
        return "login"; // Renders login.html
    }

    @GetMapping("/dashboard")
    public String dashboard(Principal principal, Model model) {
        model.addAttribute("username", principal.getName());
        return "dashboard"; // Renders dashboard.html
    }

    @GetMapping("/public/home")
    public String publicHome() {
        return "public_home"; // Renders public_home.html
    }
}
```

`login.xml` (Thymeleaf)
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Login</title>
</head>
<body>
    <h2>Login</h2>
    <div th:if="${param.error}" style="color: red;">
        Invalid username or password.
    </div>
    <div th:if="${param.logout}" style="color: green;">
        You have been logged out.
    </div>
    <form th:action="@{/perform_login}" method="post">
        <div>
            <label for="username">Username:</label>
            <input type="text" id="username" name="username"/>
        </div>
        <div>
            <label for="password">Password:</label>
            <input type="password" id="password" name="password"/>
        </div>
        <div>
            <input type="submit" value="Sign In"/>
        </div>
    </form>
</body>
</html>
```

## Basic Authentication
HTTP Basic Authentication is a simple challenge-response authentication mechanism. The client sends credentials (username and password) encoded in Base64 in the `Authorization` header with every request.

**When to use**: <br>
Suitable for securing APIs where a browser-based login is not required, or for simple machine-to-machine communication in trusted environments. It's **not recommended for public-facing web applications** as credentials are sent with every request and Base64 encoding is not encryption.

**How it works**: <br>
1. Client makes a request to a protected resource without credentials.
2. Server responds with `401 Unauthorized` and `WWW-Authenticate: Basic realm="Realm Name"` header.
3. Client (e.g., browser or `curl`) prompts for username/password.
4. Client sends `Authorization: Basic <base64_encoded_username_password>` header with subsequent requests.
5. Spring Security's `BasicAuthenticationFilter` intercepts, decodes, and delegates to `AuthenticationManager`.

**Configuration**:
```java
@Configuration
@EnableWebSecurity
public class BasicAuthSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults()); // Enable HTTP Basic authentication
        return http.build();
    }

    // UserDetailsService and PasswordEncoder beans are still needed
}
```

**Example API Endpoint**:
```java
@RestController
@RequestMapping("/api")
public class BasicAuthController {

    @GetMapping("/hello")
    public String hello() {
        return "Hello, authenticated user!";
    }

    @GetMapping("/public/info")
    public String publicInfo() {
        return "This is public information.";
    }
}
```

**Using ** `curl` ** to test**:
```
# Accessing protected endpoint without credentials
curl http://localhost:8080/api/hello
# Expected: 401 Unauthorized

# Accessing protected endpoint with credentials
curl -u user:password http://localhost:8080/api/hello
# Expected: Hello, authenticated user! (assuming 'user' with 'password' is configured)

# Accessing public endpoint
curl http://localhost:8080/api/public/info
# Expected: This is public information.
```

## JWT-based Stateless Authentication
JSON Web Tokens (JWTs) are a popular choice for building stateless APIs, especially in microservices architectures. A JWT contains claims (information about the user) and is signed to ensure its integrity.

**When to use**: <br>
Ideal for REST APIs, single-page applications (SPAs), and mobile applications where server-side sessions are undesirable (for scalability, microservices, etc.). They enable statelessness, as the server doesn't need to store session information

**How it works**:
1. User sends credentials (e.g., username/password) to a `/login` endpoint.
2. Server authenticates the user (via `AuthenticationManager` and `UserDetailsService`).
3. If successful, the server generates a JWT containing user details (e.g., ID, roles, expiration time) and signs it with a secret key.
4. The JWT is returned to the client.
5. Client stores the JWT (e.g., in `localStorage` or `sessionStorage` for web, secure storage for mobile).
6. For subsequent requests to protected resources, the client includes the JWT in the `Authorization: Bearer <token>` header.
7. Server intercepts the request, extracts the JWT, validates its signature, and parses its claims.
8. If valid, an `Authentication` object is created from the JWT claims and placed in `SecurityContextHolder`.
9. Request proceeds to the secured endpoint.

**Configuration (** `spring-boot-starter-oauth2-resource-server` **provides JWT parsing)**:
```xml
// pom.xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>com.nimbusds</groupId>
    <artifactId>nimbus-jose-jwt</artifactId>
    <version>9.31</version> 
</dependency>
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity // For method-level security with JWT
public class JwtSecurityConfig {

   // You'll typically use a private/public key pair for signing/verification in production
   // For simplicity, a symmetric key can be used for development/testing
   // A shared secret (e.g., from application.properties)
   @Value("${jwt.secret:supersecretkey}")
   private String jwtSecret;

   @Bean
   public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
      http
              .csrf(csrf -> csrf.disable()) // Disable CSRF for stateless APIs
              .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // No sessions
              .authorizeHttpRequests(authorize -> authorize
                      .requestMatchers("/api/auth/**").permitAll() // Authentication endpoint
                      .requestMatchers("/api/public/**").permitAll()
                      .anyRequest().authenticated()
              )
              .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.decoder(jwtDecoder()))); // Configure JWT resource server
      return http.build();
   }

   // Custom UserDetailsService and PasswordEncoder still needed for initial login

   @Bean
   public JwtDecoder jwtDecoder() {
      // For symmetric key (development only)
      SecretKey secretKey = new SecretKeySpec(jwtSecret.getBytes(), "HmacSHA256");
      return NimbusJwtDecoder.withSecretKey(secretKey).build();

      // For asymmetric key (RSA), typically use Jwks from an Authorization Server:
      // return NimbusJwtDecoder.withJwkSetUri("http://localhost:9000/oauth2/jwks").build();
   }

   @Bean
   public JwtEncoder jwtEncoder() {
      // For symmetric key (development only)
      SecretKey secretKey = new SecretKeySpec(jwtSecret.getBytes(), "HmacSHA256");
      JWK jwk = new OctetSequenceKey.Builder(secretKey).build();
      JWKSource<SecurityContext> jwkSource = new ImmutableJWKSet<>(new JWKSet(jwk));
      return new NimbusJwtEncoder(jwkSource);

      // For asymmetric key (RSA):
      // RSAKey rsaKey = generateRsa(); // You'd load your private key here
      // JWKSource<SecurityContext> jwkSource = new ImmutableJWKSet<>(new JWKSet(rsaKey));
      // return new NimbusJwtEncoder(jwkSource);
   }
}
```

`AuthController` **for token issuance**:
```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtEncoder jwtEncoder;

    public AuthController(AuthenticationManager authenticationManager, JwtEncoder jwtEncoder) {
        this.authenticationManager = authenticationManager;
        this.jwtEncoder = jwtEncoder;
    }

    @PostMapping("/token")
    public String getToken(@RequestBody LoginRequest loginRequest) {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(loginRequest.username(), loginRequest.password()));

        Instant now = Instant.now();
        long expiry = 36000L; // 10 hours

        String scope = authentication.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.joining(" "));

        JwtClaimsSet claims = JwtClaimsSet.builder()
                .issuer("self")
                .issuedAt(now)
                .expiresAt(now.plusSeconds(expiry))
                .subject(authentication.getName())
                .claim("scope", scope)
                .build();

        return this.jwtEncoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
    }

    public record LoginRequest(String username, String password) {}
}
```

`application.properties` **(for JWT secret)**:
```
jwt.secret=your-super-secret-key-that-is-at-least-256-bits-long
```

**Testing with** `curl`:
1. **Get Token**:
   ```
   curl -X POST -H "Content-Type: application/json" -d '{"username":"user", "password":"password"}' http://localhost:8080/api/auth/token
   ```
   
2. **Access protected resource with token**:
   ```
   TOKEN="<your_jwt_token_here>"
   curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/hello
   ```

## OAuth2 Authentication
OAuth2 is an authorization framework that enables an application to obtain limited access to a user's resources on another HTTP service.

**When to use**:
- **External Authorization Servers (e.g., Google, GitHub)**: For allowing users to log in with their existing accounts from popular identity providers. This simplifies user registration and login, leveraging the provider's security.
- **Internal Authorization Servers (e.g., Spring Authorization Server)**: For complex enterprise applications or microservices where you need to manage your own user identities and issue tokens for various clients (e.g., mobile app, web app, other microservices).

### OAuth2 with External Authorization Servers (Client Application)
Spring Boot makes it very easy to integrate with external OAuth2 providers using `spring-boot-starter-oauth2-client`. Your Spring Boot application acts as an OAuth2 client.

**How it works (Authorization Code Flow)**:
1. User attempts to access a protected resource in your Spring Boot application.
2. Your application redirects the user to the external OAuth2 provider's authorization page (e.g., Google login).
3. User authenticates with the provider and grants consent to your application.
4. Provider redirects the user back to your application with an `authorization_code`.
5. Your application exchanges the `authorization_code` for an `access_token` (and optionally a `refresh_token` and `id_token`) directly with the provider's token endpoint.
6. Your application uses the `access_token` to fetch user details from the provider's user info endpoint.
7. An `Authentication` object is created and stored in `SecurityContextHolder`, and the user is logged in.

**Dependencies**:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

**Configuration (** `application.yml` **for Google as an example)**:
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: your-google-client-id.apps.googleusercontent.com
            client-secret: your-google-client-secret
            scope: openid, profile, email
        provider:
          google:
            authorization-uri: https://accounts.google.com/o/oauth2/auth
            token-uri: https://oauth2.googleapis.com/token
            user-info-uri: https://www.googleapis.com/oauth2/v3/userinfo
            jwk-set-uri: https://www.googleapis.com/oauth2/v3/certs
            user-name-attribute: sub # The attribute in the user info response that represents the user's name
```

**Security Configuration**:
```java
@Configuration
@EnableWebSecurity
public class OAuth2LoginSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/").permitAll() // Allow access to home page
                .anyRequest().authenticated() // All other requests require authentication
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/oauth2/authorization/google") // Custom login page (optional)
                .defaultSuccessUrl("/oauth2-dashboard", true) // Redirect after successful OAuth2 login
            );
        return http.build();
    }
}
```

**Controller**:
```java
@Controller
public class OAuth2WebController {

    @GetMapping("/")
    public String index() {
        return "index"; // Simple home page with a login link
    }

    @GetMapping("/oauth2-dashboard")
    public String oauth2Dashboard(OAuth2AuthenticationToken authentication, Model model) {
        OAuth2User oauth2User = authentication.getPrincipal();
        model.addAttribute("username", oauth2User.getAttribute("name"));
        model.addAttribute("email", oauth2User.getAttribute("email"));
        return "oauth2-dashboard";
    }
}
```

`index.html` **(Thymeleaf - simple login link)**:
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Home</title>
</head>
<body>
    <h2>Welcome!</h2>
    <p><a th:href="@{/oauth2/authorization/google}">Login with Google</a></p>
</body>
</html>
```

#### OAuth2 with Internal Authorization Servers (Spring Authorization Server)
For building your own OAuth2 Authorization Server, Spring provides the Spring Authorization Server project. This is suitable when you need full control over the authorization process, client registration, and token issuance within your own ecosystem.

**Dependencies (for Authorization Server)**:
```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-oauth2-authorization-server</artifactId>
    <version>1.2.3</version> </dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

**Basic Setup (Authorization Server Side - highly simplified)**:
```java
@Configuration
@EnableWebSecurity
public class AuthorizationServerSecurityConfig {

    @Bean
    @Order(1) // Higher precedence for Authorization Server specific security
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            .oidc(Customizer.withDefaults()); // Enable OpenID Connect 1.0
        http
            // Redirect to /login page for authentication
            .exceptionHandling((exceptions) -> exceptions
                .authenticationEntryPoint(
                    new LoginUrlAuthenticationEntryPoint("/login"))
            )
            .oauth2ResourceServer(oauth2ResourceServer ->
                oauth2ResourceServer.jwt(Customizer.withDefaults()));
        return http.build();
    }

    @Bean
    @Order(2) // Lower precedence for general application security
    public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authorize) -> authorize
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults()); // Form login for authentication to the AS itself
        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        // Define users for the Authorization Server
        UserDetails user = User.withUsername("authserver_user")
            .passwordEncoder(passwordEncoder()::encode)
            .password("password")
            .roles("USER")
            .build();
        return new InMemoryUserDetailsManager(user);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient registeredClient = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("my-client-app")
            .clientSecret(passwordEncoder().encode("secret"))
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("http://127.0.0.1:8080/login/oauth2/code/my-client-app") // Client app's redirect URI
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .clientSettings(ClientSettings.builder().requireProofKeyForCodeExchange(true).build())
            .build();

        return new InMemoryRegisteredClientRepository(registeredClient);
    }

    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        // For production, use a secure way to manage RSA keys
        KeyPair keyPair = generateRsaKey();
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
        RSAKey rsaKey = new RSAKey.Builder(publicKey)
                .privateKey(privateKey)
                .keyID(UUID.randomUUID().toString())
                .build();
        JWKSet jwkSet = new JWKSet(rsaKey);
        return new ImmutableJWKSet<>(jwkSet);
    }

    private static KeyPair generateRsaKey() {
        try {
            KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
            keyPairGenerator.initialize(2048);
            return keyPairGenerator.generateKeyPair();
        } catch (Exception ex) {
            throw new IllegalStateException(ex);
        }
    }

    @Bean
    public ProviderSettings providerSettings() {
        return ProviderSettings.builder().issuer("http://localhost:9000").build(); // Your AS issuer URL
    }
}
```

**Client Applicatoin Side (using the Internal Authorization Server)**: <br>
The configuration for the client application will be similar to the external OAuth2 client, but with the specific URLs of your Spring Authorization Server.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          my-client-app:
            client-id: my-client-app
            client-secret: secret
            scope: openid, profile
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            authorization-grant-type: authorization_code
        provider:
          my-auth-server:
            issuer-uri: http://localhost:9000 # The issuer URL of your Authorization Server
```

## Securing Endpoints and Services
Beyond basic authentication, you need fine-grained control over which users can access specific resources.

### URL-Based Security (via `authorizeHttpRequests`)
This is configured within `SecurityFilterChain` using `http.authorizeHttpRequests()`
- `permitAll()`: Allows access to anyone (even unauthenticated users).
- `authenticated()`: Requires the user to be authenticated.
- `hasRole('ROLE_NAME')`: Requires the user to have a specific role (e.g., `hasRole('ADMIN')` checks for `ROLE_ADMIN`). Spring Security automatically prefixes roles with `ROLE_` if you use `hasRole()`. If you store roles without the prefix, use `hasAuthority('ROLE_NAME')`.
- `hasAnyRole('ROLE1', 'ROLE2')`: Requires the user to have at least one of the specified roles.
- `hasAuthority('PERMISSION')`: Requires the user to have a specific authority (permission).
- `hasAnyAuthority('PERMISSION1', 'PERMISSION2')`: Requires at least one of the specified authorities.
- `denyAll()`: Denies access to everyone.

**Example**:
```java
@Configuration
@EnableWebSecurity
public class EndpointSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/users/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/products/view").hasAuthority("READ_PRODUCT")
                .requestMatchers(HttpMethod.POST, "/products").hasAuthority("CREATE_PRODUCT")
                .anyRequest().authenticated()
            );
            // ... other security configurations (formLogin, httpBasic, etc.)
        return http.build();
    }
}
```

#### Method-Level Security and Role-Based Access Control
Method-level security allows you to protect individual methods within your service or controller layer, providing very granular access control

To enable method-level security, add `@EnableMethodSecurity` to your `@Configuration` class

**Annotations**:
- `@PreAuthorize`: Evaluated before the method is invoked. This is the most flexible and commonly used annotation as it supports Spring Expression Language (SpEL).
- `@PostAuthorize`: Evaluated after the method has been invoked, allowing access to the method's return value. Useful for securing returned objects.
- `@Secured`: A simpler annotation to check for specific roles (requires `@EnableMethodSecurity(securedEnabled = true)`).
- `@PostFilter`: Filters the collection returned by the method.
- `@PreFilter`: Filters the arguments passed into the method.

**Example with ** `@PreAuthorize` **(SpEL)**:
```java
@Service
@EnableMethodSecurity // Enable method-level security on your configuration class
public class ProductService {

    @PreAuthorize("hasRole('ADMIN') or hasAuthority('CREATE_PRODUCT')")
    public Product createProduct(Product product) {
        // Logic to create product
        return product;
    }

    @PreAuthorize("hasRole('ADMIN') or (hasRole('USER') and #productId == authentication.principal.id)")
    public Product getProductById(Long productId) {
        // Logic to retrieve product by ID
        return new Product(productId, "Sample Product", 100.0);
    }

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteProduct(Long productId) {
        // Logic to delete product
    }

    // Example with custom permission evaluator (see below)
    @PreAuthorize("hasPermission(#productId, 'Product', 'READ')")
    public Product viewProduct(Long productId) {
        // Logic to view product, using custom permission
        return new Product(productId, "Custom Product", 50.0);
    }
}
```

**Using SpEL expressions**: <br>
SpEL (Spring Expression Language) provides a powerful way to define complex security rules within `@PreAuthorize` and `@PostAuthorize`.
- `authentication`: Represents the current `Authentication` object
  - `authentication.name` or `authentication.principal.username`: Current authenticated username.
  - `authentication.principal.id` (if your `UserDetails` implementation has an id field).
  - `authentication.authorities`: Collection of `GrantedAuthority` objects.
- `hasRole('ROLE_NAME')`: Checks if the user has the specified role.
- `hasAnyRole('ROLE1', 'ROLE2')`: Checks for any of the roles.
- `hasAuthority('PERMISSION')`: Checks for a specific authority/permission.
- `hasPermission(targetDomainObject, permission)`: Used with custom PermissionEvaluator.
- `isAnonymous(), isAuthenticated(), isFullyAuthenticated()`: Check authentication status.
- `#argumentName`: Access method arguments. E.g., `#productId`

#### Custom Permission Evaluators
For very complex authorization rules that go beyond simple roles (e.g., "user can only edit their own profile," "user can only access data belonging to their organization"), you can implement a custom `PermissionEvaluator`

1. **Implement** `PermissionEvaluator`:
   ```java
   @Component
   public class CustomPermissionEvaluator implements PermissionEvaluator {
   
   @Override
    public boolean hasPermission(Authentication authentication, Object targetDomainObject, Object permission) {
        if (authentication == null || !authentication.isAuthenticated() || !(permission instanceof String)) {
            return false;
        }

        if (targetDomainObject instanceof Long productId && "READ".equals(permission)) {
            // Example: Check if the current user owns this product
            // You'd typically fetch the product and check its owner against the authenticated principal
            Long currentUserId = ((UserDetails) authentication.getPrincipal()).getId(); // Assuming UserDetails has getId()
            // Assuming you have a method to get product owner
            // Long productOwnerId = productService.getOwnerId(productId);
            // return currentUserId.equals(productOwnerId);
            return true; // Placeholder for demonstration
        }
        return false;
      }

       @Override
       public boolean hasPermission(Authentication authentication, Serializable targetId, String targetType, Object permission) {
           // This overload is for checking permissions on an ID and type (e.g., "Product", 123, "READ")
           // Implement similar logic as above, but fetching the object by ID and type
           if (authentication == null || !authentication.isAuthenticated() || !(permission instanceof String)) {
               return false;
           }
           if (targetType.equals("Product") && "READ".equals(permission) && targetId instanceof Long productId) {
                // Logic to check if user can read product with productId
               return true;
           }
           return false;
       }
   }
   ```
2. **Register** `PermissionEvaluator`:
   ```java
   @Configuration
   @EnableMethodSecurity
   public class MethodSecurityConfig {
   
       @Autowired
       private CustomPermissionEvaluator customPermissionEvaluator;
   
       @Bean
       public MethodSecurityExpressionHandler methodSecurityExpressionHandler() {
           DefaultMethodSecurityExpressionHandler expressionHandler = new DefaultMethodSecurityExpressionHandler();
           expressionHandler.setPermissionEvaluator(customPermissionEvaluator);
           return expressionHandler;
       }
   }
   ```

Now you can use `hasPermission()` in your `@PreAuthorize` expressions:
`@PreAuthorize("hasPermission(#productId, 'Product', 'READ')")`

## Common Web Vulnerabilities and Prevention
Spring Security significantly helps in preventing many common web vulnerabilities, but secure coding practices are also essential

### CSRF (Cross-Site Request Forgery)
- **How it arises**: An attacker tricks a victim into submitting an unwitting malicious request to a website where they are already authenticated. Since the browser automatically sends session cookies, the malicious request appears legitimate to the server.
- **How it's exploited**: A malicious website might contain a hidden form or an image tag that sends a POST request to your application (e.g., `/transferFunds`). If the victim is logged in to your application, the request will be executed with their privileges.
- **Spring Security Prevention**: Spring Security's `CsrfFilter` generates a unique, unpredictable token for each request (the CSRF token). This token must be included in the request (e.g., in a hidden form field or a custom HTTP header) for state-changing operations (POST, PUT, DELETE). If the token is missing or invalid, the request is rejected.
   - By default, Spring Security enables CSRF protection for all mutating HTTP methods (POST, PUT, PATCH, DELETE).
   - For REST APIs, clients often send the CSRF token in a custom header (e.g., `X-CSRF-TOKEN`).
   - **Pitfall**: Disabling CSRF protection (`.csrf().disable()`) without understanding the implications, especially for session-based applications. It's generally safe to disable for stateless APIs using JWTs, as JWTs are typically sent in the `Authorization` header and are not vulnerable to CSRF in the same way.

### XSS (Cross-Site Scripting)
- **How it arises**: An attacker injects malicious client-side scripts (e.g., JavaScript) into web pages viewed by other users. This usually happens when user-supplied input is not properly validated or escaped before being rendered in the browser.
- **How it's exploited**:
  - **Reflected XSS**: Malicious script is immediately returned in an error message or search result.
  - **Stored XSS**: Malicious script is stored in the database (e.g., in a comment) and executed every time the page containing it is viewed.
  - **DOM-based XSS**: Vulnerability lies in client-side code that manipulates the DOM. Exploitation can lead to session hijacking, defacement, redirecting users, or stealing sensitive data.
- **Spring Security Prevention**: While Spring Security doesn't directly prevent XSS (it's primarily an application-level vulnerability), it provides mechanisms that complement XSS prevention:
  - **HTTP Security Headers**: Spring Security can easily configure headers like `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`, and `Content-Security-Policy` (CSP) which can mitigate XSS and related attacks.
  - **Secure Coding Practices**:
    - **Input Validation**: Always validate and sanitize user input on the server-side.
    - **Output Encoding/Escaping**: Encode all user-supplied data before rendering it in HTML, JavaScript, CSS, or URL contexts. Thymeleaf, for example, encodes output by default.

### CORS (Cross-Origin Resource Sharing) Misconfigurations
- **How it arises**: A browser's Same-Origin Policy (SOP) prevents a web page from making requests to a different domain than the one that served the web page. CORS is a mechanism that allows controlled relaxation of the SOP. Misconfigurations occur when the server allows requests from overly broad origins (`*`), or allows sensitive methods/headers from untrusted origins.
- **How it's exploited**: If CORS is misconfigured to allow `*` (any origin) or a broad range of origins for sensitive operations, an attacker's malicious website can make requests to your API, potentially performing actions on behalf of the user or stealing data if not properly authenticated/authorized.
- **Spring Security Prevention**: Spring Security offers robust CORS configuration
   **Configuration Example**:
   ```java
   @Configuration
   public class CorsConfig {
   
       @Bean
       public CorsConfigurationSource corsConfigurationSource() {
           CorsConfiguration configuration = new CorsConfiguration();
           configuration.setAllowedOrigins(Arrays.asList("http://localhost:3000", "https://your-frontend-domain.com")); // Specific allowed origins
           configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
           configuration.setAllowedHeaders(Arrays.asList("Authorization", "Content-Type", "X-CSRF-TOKEN"));
           configuration.setAllowCredentials(true); // Allow sending cookies/auth headers
           configuration.setMaxAge(3600L); // How long the pre-flight response can be cached
           UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
           source.registerCorsConfiguration("/**", configuration); // Apply to all paths
           return source;
       }
   
       // Integrate with Spring Security
       @Bean
       public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
           http
               .cors(Customizer.withDefaults()) // Enable CORS integration
               // ... other security configs
           return http.build();
       }
   }
   ```
  
**Best Practice**: Always specify explicit `allowedOrigins` rather than `*` in production environments.

### SQL Injection
- **How it arises**: An attacker inserts malicious SQL code into input fields, which is then executed by the database. This occurs when user input is directly concatenated into SQL queries without proper sanitization or parameterization.
- **How it's exploited**: Attackers can bypass authentication, extract sensitive data, modify data, or even drop tables.
  - **Example**: If an application constructs a query like `SELECT * FROM users WHERE username = '" + userInputUsername + "' AND password = '" + userInputPassword + "'`, an attacker could enter `admin' --` for username to bypass the password check.
- **Spring Security Prevention**: Spring Security does not directly prevent SQL Injection. This is a database interaction vulnerability that must be prevented at the data access layer.
- **Secure Coding Practices**:
  - **Prepared Statements/Parameterized Queries**: Always use prepared statements with parameter binding (e.g., using Spring Data JPA, JdbcTemplate with named parameters, Hibernate). This separates the SQL code from the user input, preventing the input from being interpreted as executable SQL
  - **Input Validation**: Sanitize and validate all user input to ensure it conforms to expected formats
  - **Least Privilege**: Ensure your database users have only the minimum necessary permissions

## Common Pitfalls, Best Practices, and Performance Considerations
### Common Pitfalls
- **Misconfigured Filters**: Incorrect `requestMatchers` or filter order can leave endpoints unprotected or cause unexpected behavior. Always test your security configurations thoroughly.
- **Disabling CSRF for Session-Based Apps**: A common mistake that reintroduces a major vulnerability. Only disable if you fully understand the implications and have alternative protections (e.g., stateless APIs with JWT).
- **Insecure Password Storage**: Storing plain-text or weakly hashed passwords. Always use strong, adaptive hashing algorithms like BCrypt.
- **Over-reliance on `permitAll()`**: Carelessly using `permitAll()` can inadvertently expose sensitive endpoints. Be precise with what you permit.
- **Missing Exception Handling**: Not handling authentication/authorization exceptions gracefully can lead to poor user experience or information disclosure.
- **Insecure Token Storage (for JWT/OAuth2)**: Storing JWTs in `localStorage` in the browser can be vulnerable to XSS attacks. `HttpOnly` cookies are generally safer for access tokens for web applications, but this introduces CSRF concerns again.
- **Not Setting Session Timeout**: Long-lived sessions increase the risk of session hijacking. Configure appropriate session timeouts.
- **Not Updating Dependencies**: Missing security patches from outdated Spring Security or other dependencies. Regularly update your libraries.

### Best Practices
- **Always Use HTTPS**: Encrypt all communication between client and server to prevent eavesdropping and man-in-the-middle attacks. Configure your application to redirect HTTP to HTTPS.
- **Strong Password Encoding**: Use `BCryptPasswordEncoder` (or Argon2, PBKDF2) with a sufficient strength factor. Enforce strong password policies (length, complexity).
- **CORS Headers**: Configure `CorsConfigurationSource` to be as restrictive as possible, explicitly listing allowed origins, methods, and headers. Avoid * in production.
- **Token Expiration and Renewal (for JWT/OAuth2)**:
  - **Short-lived Access Tokens**: Access tokens should have a relatively short expiration time (e.g., 5-15 minutes).
  - **Refresh Tokens**: Use longer-lived refresh tokens (stored securely) to obtain new access tokens when the current one expires, reducing the need for constant re-authentication.
  - **Token Revocation**: Implement mechanisms to revoke tokens (especially refresh tokens) if a user logs out or if a security breach is suspected
- **Secure Session Management**:
  - Use `HttpOnly` and `Secure` flags for session cookies.
  - Configure appropriate session timeouts.
  - Implement session fixation protection (Spring Security does this by default).
- **Principle of Least Privilege**: Grant users and roles only the minimum permissions required to perform their tasks.
- **Input Validation and Output Encoding**: Crucial for preventing XSS and SQL injection.
- **Secure Logging**: Avoid logging sensitive information like passwords, tokens, or personal data.
- **Security Audits and Penetration Testing**: Regularly audit your application's security configuration and conduct penetration tests.
- **Keep Dependencies Up-to-Date**: Regularly update Spring Boot, Spring Security, and all other libraries to benefit from the latest security patches.


### Performance Considerations
While Spring Security provides extensive features, performance can be a concern in high-traffic applications.

- **Filter Chain Optimization**:
  - **Order of Filters**: Spring Security automatically orders its filters, but understand the impact. More specific and less resource-intensive filters should ideally come before more general or heavier ones.
  - `requestMatchers` **Specificity**: Use specific `requestMatchers` rather than `anyRequest()` where possible to allow early termination of the filter chain for un-protected resources.
  - **Bypassing Security**: For truly public resources (e.g., static assets, health checks), consider using `WebSecurityCustomizer` to completely bypass the Spring Security filter chain for certain paths
   ```java
   @Bean
   public WebSecurityCustomizer webSecurityCustomizer() {
        return (web) -> web.ignoring().requestMatchers("/resources/**", "/static/**", "/css/**", "/js/**", "/images/**", "/actuator/health");
   }
   ```
  - **Lazy Initialization**: Spring Boot and Spring Security are generally well-optimized. Avoid unnecessary eager initialization of security-related beans if they are not always needed.
  - **Caching**:
    - `UserDetailsService` **Caching**: If `loadUserByUsername` involves a database call, consider caching user details, especially if roles/permissions don't change frequently. Spring Security provides `UserCache` and integrates with Spring's caching mechanism.
    - **Permission Evaluator Caching**: For custom `PermissionEvaluator` implementations, cache results if the permission checks are computationally expensive and inputs are repetitive.
  - **Token Validation (JWT)**: While stateless JWTs avoid session lookups, validating signatures and parsing claims for every request can add overhead. Use efficient JWT libraries (like Nimbus JOSE+JWT, which Spring Security uses) and consider caching decoded tokens for a very short duration if applicable (though this can reintroduce state).
  - **Authentication Provider Optimization**: If you have multiple `AuthenticationProvider` instances, ensure they are ordered efficiently or that the `AuthenticationManager` can quickly determine which provider can handle a given `Authentication` type.
  - **Database Queries**: Optimize your `UserDetailsService` implementation to efficiently fetch user details and authorities from the database. Use proper indexing.

