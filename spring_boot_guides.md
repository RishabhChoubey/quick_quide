# Complete Spring Boot Tutorial: Beginner to Advanced

## Table of Contents

1. [Introduction to Spring Boot](#introduction-to-spring-boot)
2. [Beginner Level](#beginner-level)
   - [Setting up Development Environment](#setting-up-development-environment)
   - [Creating Your First Spring Boot Application](#creating-your-first-spring-boot-application)
   - [Basic Annotations](#basic-annotations)
   - [Building REST APIs](#building-rest-apis)
   - [Configuration Basics](#configuration-basics)
3. [Intermediate Level](#intermediate-level)
   - [Data Access with JPA](#data-access-with-jpa)
   - [Validation](#validation)
   - [Exception Handling](#exception-handling)
   - [Advanced Configuration](#advanced-configuration)
   - [Profiles](#profiles)
4. [Advanced Level](#advanced-level)
   - [Spring Security](#spring-security)
   - [Testing Strategies](#testing-strategies)
   - [Microservices Architecture](#microservices-architecture)
   - [Monitoring with Actuator](#monitoring-with-actuator)
   - [Deployment Strategies](#deployment-strategies)
5. [Complete Project Examples](#complete-project-examples)
6. [Best Practices](#best-practices)

---

## Introduction to Spring Boot

Spring Boot is a powerful framework that makes it easy to create production-ready, stand-alone Spring applications. It provides:

- **Auto-configuration**: Automatically configures your application based on dependencies
- **Embedded servers**: Built-in Tomcat, Jetty, or Undertow servers
- **Production-ready features**: Health checks, metrics, externalized configuration
- **Minimal configuration**: Convention over configuration approach

---

## Beginner Level

### Setting up Development Environment

#### Prerequisites
- Java 17 or later
- IDE (IntelliJ IDEA, Eclipse, or VS Code)
- Maven or Gradle

#### Creating a Project with Spring Initializr

Visit [start.spring.io](https://start.spring.io) or use your IDE to generate a project with these dependencies:
- Spring Web
- Spring Data JPA
- H2 Database
- Spring Boot DevTools

#### Maven Configuration (pom.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.1.5</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>spring-boot-tutorial</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-boot-tutorial</name>
    <description>Complete Spring Boot Tutorial</description>
    
    <properties>
        <java.version>17</java.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### Creating Your First Spring Boot Application

#### Main Application Class

```java
package com.example.springboottutorial;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringBootTutorialApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootTutorialApplication.class, args);
    }
}
```

#### Simple Hello World Controller

```java
package com.example.springboottutorial.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
    
    @GetMapping("/hello")
    public String hello() {
        return "Hello, Spring Boot!";
    }
    
    @GetMapping("/")
    public String home() {
        return "Welcome to Spring Boot Tutorial!";
    }
}
```

### Basic Annotations

#### Core Spring Boot Annotations

```java
package com.example.springboottutorial.annotations;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.stereotype.Component;
import org.springframework.stereotype.Service;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RestController;

// Main application annotation - combines @Configuration, @EnableAutoConfiguration, @ComponentScan
@SpringBootApplication
public class AnnotationExamples {
    
    // Component annotation - generic stereotype for any Spring-managed component
    @Component
    public class MyComponent {
        public void doSomething() {
            System.out.println("Component doing something...");
        }
    }
    
    // Service annotation - specialized component for business logic
    @Service
    public class MyService {
        public String processData(String data) {
            return "Processed: " + data;
        }
    }
    
    // Repository annotation - specialized component for data access
    @Repository
    public class MyRepository {
        public void saveData(String data) {
            System.out.println("Saving: " + data);
        }
    }
    
    // Controller annotation - for web controllers returning views
    @Controller
    public class MyController {
        // Methods would return view names
    }
    
    // RestController annotation - combines @Controller and @ResponseBody
    @RestController
    public class MyRestController {
        // Methods return data directly (JSON/XML)
    }
}
```

### Building REST APIs

#### User Entity

```java
package com.example.springboottutorial.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    @Column(nullable = false)
    private String name;
    
    @Email(message = "Email should be valid")
    @NotBlank(message = "Email is required")
    @Column(nullable = false, unique = true)
    private String email;
    
    @Size(min = 6, message = "Password must be at least 6 characters")
    @Column(nullable = false)
    private String password;
    
    // Constructors
    public User() {}
    
    public User(String name, String email, String password) {
        this.name = name;
        this.email = email;
        this.password = password;
    }
    
    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    
    @Override
    public String toString() {
        return "User{id=" + id + ", name='" + name + "', email='" + email + "'}";
    }
}
```

#### User Repository

```java
package com.example.springboottutorial.repository;

import com.example.springboottutorial.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Spring Data JPA method name queries
    Optional<User> findByEmail(String email);
    List<User> findByNameContainingIgnoreCase(String name);
    boolean existsByEmail(String email);
    
    // Custom JPQL queries
    @Query("SELECT u FROM User u WHERE u.name LIKE %:name%")
    List<User> findUsersByNameContaining(@Param("name") String name);
    
    // Native SQL queries
    @Query(value = "SELECT * FROM users WHERE email = ?1", nativeQuery = true)
    Optional<User> findByEmailNative(String email);
}
```

#### User Service

```java
package com.example.springboottutorial.service;

import com.example.springboottutorial.entity.User;
import com.example.springboottutorial.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Service
@Transactional
public class UserService {
    
    private final UserRepository userRepository;
    
    @Autowired
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }
    
    public Optional<User> getUserById(Long id) {
        return userRepository.findById(id);
    }
    
    public Optional<User> getUserByEmail(String email) {
        return userRepository.findByEmail(email);
    }
    
    public User createUser(User user) {
        if (userRepository.existsByEmail(user.getEmail())) {
            throw new RuntimeException("Email already exists: " + user.getEmail());
        }
        return userRepository.save(user);
    }
    
    public User updateUser(Long id, User userDetails) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("User not found with id: " + id));
        
        user.setName(userDetails.getName());
        user.setEmail(userDetails.getEmail());
        if (userDetails.getPassword() != null && !userDetails.getPassword().isEmpty()) {
            user.setPassword(userDetails.getPassword());
        }
        
        return userRepository.save(user);
    }
    
    public void deleteUser(Long id) {
        if (!userRepository.existsById(id)) {
            throw new RuntimeException("User not found with id: " + id);
        }
        userRepository.deleteById(id);
    }
    
    public List<User> searchUsersByName(String name) {
        return userRepository.findByNameContainingIgnoreCase(name);
    }
}
```

#### User Controller

```java
package com.example.springboottutorial.controller;

import com.example.springboottutorial.entity.User;
import com.example.springboottutorial.service.UserService;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Optional;

@RestController
@RequestMapping("/api/users")
@CrossOrigin(origins = "*")
public class UserController {
    
    private final UserService userService;
    
    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    // GET /api/users - Get all users
    @GetMapping
    public ResponseEntity<List<User>> getAllUsers() {
        List<User> users = userService.getAllUsers();
        return ResponseEntity.ok(users);
    }
    
    // GET /api/users/{id} - Get user by ID
    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable Long id) {
        Optional<User> user = userService.getUserById(id);
        return user.map(ResponseEntity::ok)
                  .orElse(ResponseEntity.notFound().build());
    }
    
    // POST /api/users - Create new user
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        try {
            User createdUser = userService.createUser(user);
            return ResponseEntity.status(HttpStatus.CREATED).body(createdUser);
        } catch (RuntimeException e) {
            return ResponseEntity.badRequest().build();
        }
    }
    
    // PUT /api/users/{id} - Update user
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(@PathVariable Long id, 
                                         @Valid @RequestBody User userDetails) {
        try {
            User updatedUser = userService.updateUser(id, userDetails);
            return ResponseEntity.ok(updatedUser);
        } catch (RuntimeException e) {
            return ResponseEntity.notFound().build();
        }
    }
    
    // DELETE /api/users/{id} - Delete user
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        try {
            userService.deleteUser(id);
            return ResponseEntity.noContent().build();
        } catch (RuntimeException e) {
            return ResponseEntity.notFound().build();
        }
    }
    
    // GET /api/users/search?name=... - Search users by name
    @GetMapping("/search")
    public ResponseEntity<List<User>> searchUsers(@RequestParam String name) {
        List<User> users = userService.searchUsersByName(name);
        return ResponseEntity.ok(users);
    }
    
    // GET /api/users/email/{email} - Get user by email
    @GetMapping("/email/{email}")
    public ResponseEntity<User> getUserByEmail(@PathVariable String email) {
        Optional<User> user = userService.getUserByEmail(email);
        return user.map(ResponseEntity::ok)
                  .orElse(ResponseEntity.notFound().build());
    }
}
```

### Configuration Basics

#### Application Properties (application.yml)

```yaml
# Server Configuration
server:
  port: 8080
  servlet:
    context-path: /api

# Database Configuration
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password: password
  
  # JPA Configuration
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: create-drop
    show-sql: true
    properties:
      hibernate:
        format_sql: true
  
  # H2 Console (for development)
  h2:
    console:
      enabled: true
      path: /h2-console
  
  # Logging Configuration
logging:
  level:
    com.example.springboottutorial: DEBUG
    org.springframework.web: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE

# Application-specific configuration
app:
  name: Spring Boot Tutorial
  version: 1.0.0
```

---

## Intermediate Level

### Data Access with JPA

#### Advanced Entity Relationships

```java
package com.example.springboottutorial.entity;

import jakarta.persistence.*;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

// Author Entity
@Entity
@Table(name = "authors")
public class Author {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    private String email;
    
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Book> books = new ArrayList<>();
    
    // Constructors, getters, setters
    public Author() {}
    
    public Author(String name, String email) {
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
    
    public List<Book> getBooks() { return books; }
    public void setBooks(List<Book> books) { this.books = books; }
    
    public void addBook(Book book) {
        books.add(book);
        book.setAuthor(this);
    }
}

// Book Entity
@Entity
@Table(name = "books")
public class Book {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String title;
    
    private String isbn;
    
    @Column(name = "published_date")
    private LocalDateTime publishedDate;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;
    
    @ManyToMany(mappedBy = "books")
    private List<Category> categories = new ArrayList<>();
    
    // Constructors, getters, setters
    public Book() {}
    
    public Book(String title, String isbn, LocalDateTime publishedDate) {
        this.title = title;
        this.isbn = isbn;
        this.publishedDate = publishedDate;
    }
    
    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    
    public String getIsbn() { return isbn; }
    public void setIsbn(String isbn) { this.isbn = isbn; }
    
    public LocalDateTime getPublishedDate() { return publishedDate; }
    public void setPublishedDate(LocalDateTime publishedDate) { this.publishedDate = publishedDate; }
    
    public Author getAuthor() { return author; }
    public void setAuthor(Author author) { this.author = author; }
    
    public List<Category> getCategories() { return categories; }
    public void setCategories(List<Category> categories) { this.categories = categories; }
}

// Category Entity
@Entity
@Table(name = "categories")
public class Category {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String name;
    
    private String description;
    
    @ManyToMany
    @JoinTable(
        name = "book_categories",
        joinColumns = @JoinColumn(name = "category_id"),
        inverseJoinColumns = @JoinColumn(name = "book_id")
    )
    private List<Book> books = new ArrayList<>();
    
    // Constructors, getters, setters
    public Category() {}
    
    public Category(String name, String description) {
        this.name = name;
        this.description = description;
    }
    
    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    
    public List<Book> getBooks() { return books; }
    public void setBooks(List<Book> books) { this.books = books; }
}
```

#### Custom Repository Implementation

```java
package com.example.springboottutorial.repository;

import com.example.springboottutorial.entity.Book;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDateTime;
import java.util.List;

public interface BookRepository extends JpaRepository<Book, Long> {
    
    // Query methods
    List<Book> findByAuthorName(String authorName);
    List<Book> findByTitleContainingIgnoreCase(String title);
    List<Book> findByPublishedDateBetween(LocalDateTime startDate, LocalDateTime endDate);
    
    // Complex queries with joins
    @Query("SELECT b FROM Book b JOIN b.author a WHERE a.name = :authorName")
    List<Book> findBooksByAuthorName(@Param("authorName") String authorName);
    
    @Query("SELECT b FROM Book b JOIN b.categories c WHERE c.name = :categoryName")
    List<Book> findBooksByCategory(@Param("categoryName") String categoryName);
    
    // Pagination and sorting
    Page<Book> findByTitleContainingIgnoreCase(String title, Pageable pageable);
    
    // Custom native query
    @Query(value = "SELECT * FROM books WHERE published_date > :date ORDER BY published_date DESC", 
           nativeQuery = true)
    List<Book> findRecentBooks(@Param("date") LocalDateTime date);
}
```

### Validation

#### Custom Validation Annotations

```java
package com.example.springboottutorial.validation;

import jakarta.validation.Constraint;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import jakarta.validation.Payload;

import java.lang.annotation.*;

// Custom validation annotation
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ISBNValidator.class)
@Documented
public @interface ValidISBN {
    String message() default "Invalid ISBN format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// ISBN Validator Implementation
class ISBNValidator implements ConstraintValidator<ValidISBN, String> {
    
    @Override
    public void initialize(ValidISBN constraintAnnotation) {
        // Initialization logic if needed
    }
    
    @Override
    public boolean isValid(String isbn, ConstraintValidatorContext context) {
        if (isbn == null || isbn.trim().isEmpty()) {
            return true; // Let @NotBlank handle null/empty validation
        }
        
        // Remove hyphens and spaces
        String cleanIsbn = isbn.replaceAll("[\\s-]", "");
        
        // Check if it's ISBN-10 or ISBN-13
        return isValidISBN10(cleanIsbn) || isValidISBN13(cleanIsbn);
    }
    
    private boolean isValidISBN10(String isbn) {
        if (isbn.length() != 10) return false;
        
        int sum = 0;
        for (int i = 0; i < 9; i++) {
            if (!Character.isDigit(isbn.charAt(i))) return false;
            sum += (isbn.charAt(i) - '0') * (10 - i);
        }
        
        char lastChar = isbn.charAt(9);
        if (lastChar == 'X') {
            sum += 10;
        } else if (Character.isDigit(lastChar)) {
            sum += (lastChar - '0');
        } else {
            return false;
        }
        
        return sum % 11 == 0;
    }
    
    private boolean isValidISBN13(String isbn) {
        if (isbn.length() != 13) return false;
        
        int sum = 0;
        for (int i = 0; i < 13; i++) {
            if (!Character.isDigit(isbn.charAt(i))) return false;
            int digit = isbn.charAt(i) - '0';
            sum += (i % 2 == 0) ? digit : digit * 3;
        }
        
        return sum % 10 == 0;
    }
}
```

#### DTO with Validation

```java
package com.example.springboottutorial.dto;

import com.example.springboottutorial.validation.ValidISBN;
import jakarta.validation.constraints.*;

import java.time.LocalDateTime;

public class BookCreateRequest {
    
    @NotBlank(message = "Title is required")
    @Size(min = 1, max = 200, message = "Title must be between 1 and 200 characters")
    private String title;
    
    @ValidISBN
    private String isbn;
    
    @NotNull(message = "Author ID is required")
    @Positive(message = "Author ID must be positive")
    private Long authorId;
    
    @PastOrPresent(message = "Published date cannot be in the future")
    private LocalDateTime publishedDate;
    
    private String description;
    
    // Constructors
    public BookCreateRequest() {}
    
    public BookCreateRequest(String title, String isbn, Long authorId, LocalDateTime publishedDate) {
        this.title = title;
        this.isbn = isbn;
        this.authorId = authorId;
        this.publishedDate = publishedDate;
    }
    
    // Getters and Setters
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    
    public String getIsbn() { return isbn; }
    public void setIsbn(String isbn) { this.isbn = isbn; }
    
    public Long getAuthorId() { return authorId; }
    public void setAuthorId(Long authorId) { this.authorId = authorId; }
    
    public LocalDateTime getPublishedDate() { return publishedDate; }
    public void setPublishedDate(LocalDateTime publishedDate) { this.publishedDate = publishedDate; }
    
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
}
```

### Exception Handling

#### Global Exception Handler

```java
package com.example.springboottutorial.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.context.request.WebRequest;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {
    
    // Handle validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationExceptions(
            MethodArgumentNotValidException ex) {
        
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach((error) -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        
        ErrorResponse errorResponse = new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Validation failed",
            LocalDateTime.now(),
            errors
        );
        
        return ResponseEntity.badRequest().body(errorResponse);
    }
    
    // Handle resource not found
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFoundException(
            ResourceNotFoundException ex, WebRequest request) {
        
        ErrorResponse errorResponse = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            LocalDateTime.now(),
            Map.of("path", request.getDescription(false))
        );
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(errorResponse);
    }
    
    // Handle business logic exceptions
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(
            BusinessException ex, WebRequest request) {
        
        ErrorResponse errorResponse = new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            ex.getMessage(),
            LocalDateTime.now(),
            Map.of("path", request.getDescription(false))
        );
        
        return ResponseEntity.badRequest().body(errorResponse);
    }
    
    // Handle general exceptions
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(
            Exception ex, WebRequest request) {
        
        ErrorResponse errorResponse = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "An unexpected error occurred",
            LocalDateTime.now(),
            Map.of("path", request.getDescription(false), "error", ex.getMessage())
        );
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(errorResponse);
    }
}

// Error Response DTO
class ErrorResponse {
    private int status;
    private String message;
    private LocalDateTime timestamp;
    private Map<String, String> details;
    
    public ErrorResponse(int status, String message, LocalDateTime timestamp, Map<String, String> details) {
        this.status = status;
        this.message = message;
        this.timestamp = timestamp;
        this.details = details;
    }
    
    // Getters and Setters
    public int getStatus() { return status; }
    public void setStatus(int status) { this.status = status; }
    
    public String getMessage() { return message; }
    public void setMessage(String message) { this.message = message; }
    
    public LocalDateTime getTimestamp() { return timestamp; }
    public void setTimestamp(LocalDateTime timestamp) { this.timestamp = timestamp; }
    
    public Map<String, String> getDetails() { return details; }
    public void setDetails(Map<String, String> details) { this.details = details; }
}

// Custom Exceptions
class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }
}
```

### Advanced Configuration

#### Configuration Properties

```java
package com.example.springboottutorial.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app")
public class AppConfigProperties {
    
    private String name;
    private String version;
    private Database database = new Database();
    private Security security = new Security();
    
    // Getters and Setters
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getVersion() { return version; }
    public void setVersion(String version) { this.version = version; }
    
    public Database getDatabase() { return database; }
    public void setDatabase(Database database) { this.database = database; }
    
    public Security getSecurity() { return security; }
    public void setSecurity(Security security) { this.security = security; }
    
    // Nested configuration classes
    public static class Database {
        private int maxConnections = 10;
        private int timeout = 30;
        
        public int getMaxConnections() { return maxConnections; }
        public void setMaxConnections(int maxConnections) { this.maxConnections = maxConnections; }
        
        public int getTimeout() { return timeout; }
        public void setTimeout(int timeout) { this.timeout = timeout; }
    }
    
    public static class Security {
        private boolean enabled = true;
        private String jwtSecret = "defaultSecret";
        private long jwtExpiration = 86400; // 24 hours
        
        public boolean isEnabled() { return enabled; }
        public void setEnabled(boolean enabled) { this.enabled = enabled; }
        
        public String getJwtSecret() { return jwtSecret; }
        public void setJwtSecret(String jwtSecret) { this.jwtSecret = jwtSecret; }
        
        public long getJwtExpiration() { return jwtExpiration; }
        public void setJwtExpiration(long jwtExpiration) { this.jwtExpiration = jwtExpiration; }
    }
}
```

### Profiles

#### Profile-specific Configuration

```yaml
# application.yml (default profile)
spring:
  profiles:
    active: development

app:
  name: Spring Boot Tutorial
  version: 1.0.0

---
# application-development.yml
spring:
  config:
    activate:
      on-profile: development
  
  datasource:
    url: jdbc:h2:mem:devdb
    username: sa
    password: password
  
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  
  h2:
    console:
      enabled: true

logging:
  level:
    com.example.springboottutorial: DEBUG
    org.springframework.web: DEBUG

---
# application-production.yml
spring:
  config:
    activate:
      on-profile: production
  
  datasource:
    url: jdbc:postgresql://localhost:5432/springboot_tutorial
    username: ${DB_USERNAME:admin}
    password: ${DB_PASSWORD:password}
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  
logging:
  level:
    com.example.springboottutorial: INFO
    org.springframework.web: WARN
  file:
    name: /var/log/springboot-tutorial.log

---
# application-test.yml
spring:
  config:
    activate:
      on-profile: test
  
  datasource:
    url: jdbc:h2:mem:testdb
    username: sa
    password: password
  
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: false
```

---

## Advanced Level

### Spring Security

#### Security Configuration

```java
package com.example.springboottutorial.config;

import com.example.springboottutorial.security.JwtAuthenticationEntryPoint;
import com.example.springboottutorial.security.JwtAuthenticationFilter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    
    @Autowired
    private UserDetailsService userDetailsService;
    
    @Autowired
    private JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    
    @Autowired
    private JwtAuthenticationFilter jwtAuthenticationFilter;
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration authConfig) throws Exception {
        return authConfig.getAuthenticationManager();
    }
    
    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
            .exceptionHandling(exception -> exception.authenticationEntryPoint(jwtAuthenticationEntryPoint))
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/h2-console/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        
        // Add JWT filter
        http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
        
        // For H2 Console
        http.headers(headers -> headers.frameOptions().disable());
        
        return http.build();
    }
}
```

#### JWT Utility

```java
package com.example.springboottutorial.security;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.util.Date;

@Component
public class JwtUtils {
    
    @Value("${app.jwtSecret:defaultSecretKey}")
    private String jwtSecret;
    
    @Value("${app.jwtExpirationMs:86400000}")
    private int jwtExpirationMs;
    
    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(jwtSecret.getBytes());
    }
    
    public String generateJwtToken(Authentication authentication) {
        UserPrincipal userPrincipal = (UserPrincipal) authentication.getPrincipal();
        
        return Jwts.builder()
                .setSubject(userPrincipal.getUsername())
                .setIssuedAt(new Date())
                .setExpiration(new Date((new Date()).getTime() + jwtExpirationMs))
                .signWith(getSigningKey(), SignatureAlgorithm.HS256)
                .compact();
    }
    
    public String getUserNameFromJwtToken(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody()
                .getSubject();
    }
    
    public boolean validateJwtToken(String authToken) {
        try {
            Jwts.parserBuilder().setSigningKey(getSigningKey()).build().parseClaimsJws(authToken);
            return true;
        } catch (MalformedJwtException e) {
            System.err.println("Invalid JWT token: " + e.getMessage());
        } catch (ExpiredJwtException e) {
            System.err.println("JWT token is expired: " + e.getMessage());
        } catch (UnsupportedJwtException e) {
            System.err.println("JWT token is unsupported: " + e.getMessage());
        } catch (IllegalArgumentException e) {
            System.err.println("JWT claims string is empty: " + e.getMessage());
        }
        
        return false;
    }
}
```

#### Authentication Controller

```java
package com.example.springboottutorial.controller;

import com.example.springboottutorial.dto.LoginRequest;
import com.example.springboottutorial.dto.LoginResponse;
import com.example.springboottutorial.dto.SignupRequest;
import com.example.springboottutorial.entity.User;
import com.example.springboottutorial.security.JwtUtils;
import com.example.springboottutorial.service.UserService;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    @Autowired
    private AuthenticationManager authenticationManager;
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private JwtUtils jwtUtils;
    
    @PostMapping("/login")
    public ResponseEntity<LoginResponse> authenticateUser(@Valid @RequestBody LoginRequest loginRequest) {
        
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        loginRequest.getEmail(),
                        loginRequest.getPassword())
        );
        
        SecurityContextHolder.getContext().setAuthentication(authentication);
        String jwt = jwtUtils.generateJwtToken(authentication);
        
        return ResponseEntity.ok(new LoginResponse(jwt, "Bearer"));
    }
    
    @PostMapping("/signup")
    public ResponseEntity<String> registerUser(@Valid @RequestBody SignupRequest signUpRequest) {
        
        if (userService.getUserByEmail(signUpRequest.getEmail()).isPresent()) {
            return ResponseEntity.badRequest().body("Error: Email is already taken!");
        }
        
        // Create new user
        User user = new User(
                signUpRequest.getName(),
                signUpRequest.getEmail(),
                passwordEncoder.encode(signUpRequest.getPassword())
        );
        
        userService.createUser(user);
        
        return ResponseEntity.ok("User registered successfully!");
    }
}
```

### Testing Strategies

#### Unit Tests

```java
package com.example.springboottutorial.service;

import com.example.springboottutorial.entity.User;
import com.example.springboottutorial.repository.UserRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Arrays;
import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @InjectMocks
    private UserService userService;
    
    private User testUser;
    
    @BeforeEach
    void setUp() {
        testUser = new User("John Doe", "john@example.com", "password123");
        testUser.setId(1L);
    }
    
    @Test
    void getAllUsers_ShouldReturnAllUsers() {
        // Given
        List<User> users = Arrays.asList(testUser);
        when(userRepository.findAll()).thenReturn(users);
        
        // When
        List<User> result = userService.getAllUsers();
        
        // Then
        assertEquals(1, result.size());
        assertEquals(testUser.getName(), result.get(0).getName());
        verify(userRepository).findAll();
    }
    
    @Test
    void getUserById_WhenUserExists_ShouldReturnUser() {
        // Given
        when(userRepository.findById(1L)).thenReturn(Optional.of(testUser));
        
        // When
        Optional<User> result = userService.getUserById(1L);
        
        // Then
        assertTrue(result.isPresent());
        assertEquals(testUser.getName(), result.get().getName());
        verify(userRepository).findById(1L);
    }
    
    @Test
    void createUser_WhenEmailDoesNotExist_ShouldCreateUser() {
        // Given
        when(userRepository.existsByEmail(testUser.getEmail())).thenReturn(false);
        when(userRepository.save(testUser)).thenReturn(testUser);
        
        // When
        User result = userService.createUser(testUser);
        
        // Then
        assertEquals(testUser.getName(), result.getName());
        verify(userRepository).existsByEmail(testUser.getEmail());
        verify(userRepository).save(testUser);
    }
    
    @Test
    void createUser_WhenEmailExists_ShouldThrowException() {
        // Given
        when(userRepository.existsByEmail(testUser.getEmail())).thenReturn(true);
        
        // When & Then
        RuntimeException exception = assertThrows(RuntimeException.class, 
                () -> userService.createUser(testUser));
        
        assertEquals("Email already exists: " + testUser.getEmail(), exception.getMessage());
        verify(userRepository).existsByEmail(testUser.getEmail());
        verify(userRepository, never()).save(testUser);
    }
}
```

#### Integration Tests

```java
package com.example.springboottutorial.controller;

import com.example.springboottutorial.entity.User;
import com.example.springboottutorial.repository.UserRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureWebMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import static org.hamcrest.Matchers.hasSize;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureWebMvc
@ActiveProfiles("test")
@Transactional
class UserControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    private User testUser;
    
    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
        testUser = new User("John Doe", "john@example.com", "password123");
    }
    
    @Test
    void getAllUsers_ShouldReturnEmptyList_WhenNoUsers() throws Exception {
        mockMvc.perform(get("/api/users"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(0)));
    }
    
    @Test
    void createUser_ShouldCreateUser_WhenValidData() throws Exception {
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(testUser)))
                .andExpect(status().isCreated())
                .andExpected(jsonPath("$.name").value(testUser.getName()))
                .andExpected(jsonPath("$.email").value(testUser.getEmail()));
    }
    
    @Test
    void createUser_ShouldReturnBadRequest_WhenInvalidData() throws Exception {
        testUser.setEmail("invalid-email");
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(testUser)))
                .andExpect(status().isBadRequest());
    }
}
```

### Microservices Architecture

#### Service Discovery with Eureka

```java
// Eureka Server Application
@SpringBootApplication
@EnableEurekaServer
public class ServiceDiscoveryApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceDiscoveryApplication.class, args);
    }
}

// Eureka Client Configuration
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

#### Feign Client

```java
package com.example.springboottutorial.client;

import com.example.springboottutorial.dto.NotificationRequest;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

@FeignClient(name = "notification-service", url = "${notification.service.url:http://localhost:8082}")
public interface NotificationClient {
    
    @PostMapping("/api/notifications/send")
    void sendNotification(@RequestBody NotificationRequest request);
}
```

#### Circuit Breaker with Resilience4j

```java
package com.example.springboottutorial.service;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import org.springframework.stereotype.Service;

@Service
public class ExternalApiService {
    
    @CircuitBreaker(name = "external-api", fallbackMethod = "fallbackResponse")
    @Retry(name = "external-api")
    public String callExternalApi(String request) {
        // Call to external API
        // This could fail and trigger circuit breaker
        return externalApiClient.call(request);
    }
    
    public String fallbackResponse(String request, Exception ex) {
        return "Fallback response due to: " + ex.getMessage();
    }
}
```

### Monitoring with Actuator

#### Actuator Configuration

```yaml
# Actuator configuration
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env,beans
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      show-components: always
  health:
    circuitbreakers:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
```

#### Custom Health Indicator

```java
package com.example.springboottutorial.health;

import org.springframework.boot.actuator.health.Health;
import org.springframework.boot.actuator.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        try {
            // Check database connectivity
            boolean databaseUp = checkDatabaseConnection();
            
            if (databaseUp) {
                return Health.up()
                        .withDetail("database", "Available")
                        .withDetail("connections", "10/50")
                        .build();
            } else {
                return Health.down()
                        .withDetail("database", "Not Available")
                        .build();
            }
        } catch (Exception e) {
            return Health.down()
                    .withDetail("database", "Error: " + e.getMessage())
                    .build();
        }
    }
    
    private boolean checkDatabaseConnection() {
        // Implement actual database connectivity check
        return true;
    }
}
```

### Deployment Strategies

#### Docker Configuration

```dockerfile
# Dockerfile
FROM openjdk:17-jdk-slim

WORKDIR /app

COPY target/spring-boot-tutorial-*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=production
      - DB_HOST=postgres
      - DB_USERNAME=admin
      - DB_PASSWORD=password
    depends_on:
      - postgres
    networks:
      - app-network

  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: springboot_tutorial
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

---

## Complete Project Examples

### E-commerce Microservice

#### Product Service

```java
package com.example.ecommerce.product.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "products")
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotBlank
    @Column(nullable = false)
    private String name;
    
    @Column(length = 1000)
    private String description;
    
    @NotNull
    @DecimalMin(value = "0.0", inclusive = false)
    @Digits(integer = 10, fraction = 2)
    private BigDecimal price;
    
    @Min(0)
    private Integer stockQuantity;
    
    private String category;
    private String imageUrl;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
    
    // Constructors, getters, setters...
}

// Product Service
@Service
@Transactional
public class ProductService {
    
    private final ProductRepository productRepository;
    
    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }
    
    public Page<Product> getAllProducts(Pageable pageable) {
        return productRepository.findAll(pageable);
    }
    
    public Page<Product> getProductsByCategory(String category, Pageable pageable) {
        return productRepository.findByCategoryIgnoreCase(category, pageable);
    }
    
    public Product createProduct(Product product) {
        return productRepository.save(product);
    }
    
    public Product updateStock(Long productId, Integer quantity) {
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new ProductNotFoundException("Product not found: " + productId));
        
        if (product.getStockQuantity() < quantity) {
            throw new InsufficientStockException("Not enough stock available");
        }
        
        product.setStockQuantity(product.getStockQuantity() - quantity);
        return productRepository.save(product);
    }
}

// Product Controller
@RestController
@RequestMapping("/api/products")
@Validated
public class ProductController {
    
    private final ProductService productService;
    
    public ProductController(ProductService productService) {
        this.productService = productService;
    }
    
    @GetMapping
    public ResponseEntity<Page<Product>> getAllProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "id") String sortBy) {
        
        Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy));
        Page<Product> products = productService.getAllProducts(pageable);
        return ResponseEntity.ok(products);
    }
    
    @GetMapping("/category/{category}")
    public ResponseEntity<Page<Product>> getProductsByCategory(
            @PathVariable String category,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        
        Pageable pageable = PageRequest.of(page, size);
        Page<Product> products = productService.getProductsByCategory(category, pageable);
        return ResponseEntity.ok(products);
    }
    
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Product> createProduct(@Valid @RequestBody Product product) {
        Product createdProduct = productService.createProduct(product);
        return ResponseEntity.status(HttpStatus.CREATED).body(createdProduct);
    }
}
```

---
```java
@RestController
@RequestMapping("/api")
public class MyController {

    @GetMapping(value = "/data", produces = {"application/json", "application/xml"})
    public MyResponse getData() {
        return new MyResponse("Hello", 123);
    }
}

@XmlRootElement
public class MyResponse {
    private String message;
    private int code;

    // getters and setters
}
```
## How to Return JSON or XML from a Single Endpoint

Spring Boot supports content negotiation, allowing a single endpoint to return either JSON or XML based on the client's `Accept` header.

### 1. Add Dependencies

For Maven, add these to your `pom.xml`:

```xml
<!-- JSON (Jackson) is included by default with spring-boot-starter-web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- XML support -->
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

### 2. Create a POJO with XML Support

```java
import com.fasterxml.jackson.dataformat.xml.annotation.JacksonXmlRootElement;

@JacksonXmlRootElement(localName = "MyResponse")
public class MyResponse {
    private String message;
    private int code;

    public MyResponse() {}
    public MyResponse(String message, int code) {
        this.message = message;
        this.code = code;
    }
    public String getMessage() { return message; }
    public void setMessage(String message) { this.message = message; }
    public int getCode() { return code; }
    public void setCode(int code) { this.code = code; }
}
```

### 3. Create the Controller

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class MyController {
    @GetMapping(value = "/api/data", produces = {"application/json", "application/xml"})
    public MyResponse getData() {
        return new MyResponse("Hello", 123);
    }
}
```

### 4. How It Works
- If the client sends `Accept: application/json`, the response is JSON.
- If the client sends `Accept: application/xml`, the response is XML.

Spring Boot automatically handles conversion if dependencies are present.

### 5. Example Requests

**JSON:**
```bash
curl -H "Accept: application/json" http://localhost:8080/api/data
```
**XML:**
```bash
curl -H "Accept: application/xml" http://localhost:8080/api/data
```

### 6. Notes
- For XML, you can also use `@XmlRootElement` if you prefer JAXB.
- Make sure your POJO has getters/setters and a no-args constructor.
- This approach works for POST, PUT, etc. as well.

---

## Best Practices

### 1. Project Structure

```
src/
├── main/
│   ├── java/
│   │   └── com/example/app/
│   │       ├── Application.java
│   │       ├── config/
│   │       ├── controller/
│   │       ├── service/
│   │       ├── repository/
│   │       ├── entity/
│   │       ├── dto/
│   │       ├── exception/
│   │       └── util/
│   └── resources/
│       ├── application.yml
│       ├── application-dev.yml
│       ├── application-prod.yml
│       └── static/
└── test/
    └── java/
        └── com/example/app/
            ├── controller/
            ├── service/
            └── repository/
```

### 2. Configuration Management

- Use externalized configuration
- Implement environment-specific profiles
- Use Configuration Properties classes
- Secure sensitive information

### 3. Security Best Practices

- Implement proper authentication and authorization
- Use HTTPS in production
- Validate all inputs
- Handle security exceptions properly
- Keep dependencies updated

### 4. Performance Optimization

- Use database indexing
- Implement caching strategies
- Use pagination for large datasets
- Optimize database queries
- Monitor application metrics

### 5. Testing Strategy

- Write unit tests for business logic
- Implement integration tests for APIs
- Use test profiles
- Mock external dependencies
- Maintain good test coverage

### 6. Error Handling

- Implement global exception handling
- Use meaningful error messages
- Log errors appropriately
- Return proper HTTP status codes

### 7. Documentation

- Use OpenAPI/Swagger for API documentation
- Write clear README files
- Document complex business logic
- Maintain changelog

---

## Conclusion

This comprehensive Spring Boot tutorial covers everything from basic concepts to advanced enterprise patterns. The key to mastering Spring Boot is:

1. **Understanding the fundamentals**: Auto-configuration, dependency injection, and Spring Boot starters
2. **Building robust applications**: Proper error handling, validation, and security
3. **Following best practices**: Clean code, testing, and proper project structure
4. **Staying updated**: Spring Boot ecosystem evolves rapidly

Remember to always:
- Start simple and add complexity gradually
- Write tests for your code
- Use Spring Boot's conventions
- Monitor and secure your applications
- Keep learning and experimenting

Happy coding with Spring Boot! 🚀

---

## Docker and Kubernetes Guide for Spring Boot

### Docker Setup

#### 1. Creating a Dockerfile

```dockerfile
# Multi-stage build for optimized image size
FROM openjdk:17-jdk-slim as build

WORKDIR /workspace/app

# Copy Maven wrapper and pom.xml
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .

# Download dependencies (cached layer)
RUN ./mvnw dependency:go-offline

# Copy source code
COPY src src

# Build the application
RUN ./mvnw clean package -DskipTests

# Runtime stage
FROM openjdk:17-jre-slim

# Create non-root user for security
RUN addgroup --system spring && adduser --system spring --ingroup spring
USER spring:spring

# Set working directory
WORKDIR /app

# Copy jar from build stage
COPY --from=build /workspace/app/target/*.jar app.jar

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### 2. Docker Compose for Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    container_name: springboot-app
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DB_HOST=postgres
      - DB_USERNAME=springuser
      - DB_PASSWORD=springpass
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis
    networks:
      - app-network
    volumes:
      - ./logs:/app/logs

  postgres:
    image: postgres:15-alpine
    container_name: springboot-postgres
    environment:
      POSTGRES_DB: springbootdb
      POSTGRES_USER: springuser
      POSTGRES_PASSWORD: springpass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    container_name: springboot-redis
    ports:
      - "6379:6379"
    networks:
      - app-network
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  nginx:
    image: nginx:alpine
    container_name: springboot-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - app
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

#### 3. Docker Profile Configuration

```yaml
# application-docker.yml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/${DB_NAME:springbootdb}
    username: ${DB_USERNAME:springuser}
    password: ${DB_PASSWORD:springpass}
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
  
  redis:
    host: ${REDIS_HOST:localhost}
    port: 6379
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0

logging:
  level:
    com.example.springboottutorial: INFO
  file:
    name: /app/logs/spring-boot-app.log
  pattern:
    file: "%d{ISO8601} [%thread] %-5level %logger{36} - %msg%n"

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
```

#### 4. Docker Build and Run Commands

```bash
# Build the image
docker build -t springboot-app:latest .

# Run single container
docker run -d \
  --name springboot-app \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=docker \
  springboot-app:latest

# Using Docker Compose
docker-compose up -d

# View logs
docker-compose logs -f app

# Scale the application
docker-compose up -d --scale app=3

# Stop and remove containers
docker-compose down

# Remove volumes (data will be lost)
docker-compose down -v
```

#### 5. Nginx Configuration

```nginx
# nginx/nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream springboot-app {
        server app:8080;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://springboot-app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /actuator/health {
            proxy_pass http://springboot-app/actuator/health;
            access_log off;
        }
    }
}
```

### Kubernetes Deployment

#### 1. Namespace Configuration

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: springboot-app
  labels:
    name: springboot-app
```

#### 2. ConfigMap and Secrets

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: springboot-config
  namespace: springboot-app
data:
  application.yml: |
    spring:
      profiles:
        active: kubernetes
      datasource:
        url: jdbc:postgresql://postgres-service:5432/springbootdb
        username: springuser
        driver-class-name: org.postgresql.Driver
      jpa:
        hibernate:
          ddl-auto: validate
        show-sql: false
      redis:
        host: redis-service
        port: 6379
    logging:
      level:
        com.example.springboottutorial: INFO
    management:
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus

---
# k8s/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: springboot-secrets
  namespace: springboot-app
type: Opaque
data:
  db-password: c3ByaW5ncGFzcw==  # springpass in base64
  redis-password: ""
```

#### 3. PostgreSQL Deployment

```yaml
# k8s/postgres.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: springboot-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: springbootdb
        - name: POSTGRES_USER
          value: springuser
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: springboot-secrets
              key: db-password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: springboot-app
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: springboot-app
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

#### 4. Redis Deployment

```yaml
# k8s/redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: springboot-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "128Mi"
            cpu: "125m"
          limits:
            memory: "256Mi"
            cpu: "250m"

---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: springboot-app
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
  type: ClusterIP
```

#### 5. Spring Boot Application Deployment

```yaml
# k8s/app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-app
  namespace: springboot-app
  labels:
    app: springboot-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: springboot-app
  template:
    metadata:
      labels:
        app: springboot-app
    spec:
      containers:
      - name: springboot-app
        image: springboot-app:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: springboot-secrets
              key: db-password
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
      volumes:
      - name: config-volume
        configMap:
          name: springboot-config

---
apiVersion: v1
kind: Service
metadata:
  name: springboot-service
  namespace: springboot-app
spec:
  selector:
    app: springboot-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
```

#### 6. Ingress Configuration

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: springboot-ingress
  namespace: springboot-app
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - your-domain.com
    secretName: springboot-tls
  rules:
  - host: your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: springboot-service
            port:
              number: 80
```

#### 7. Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: springboot-hpa
  namespace: springboot-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: springboot-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

#### 8. Service Monitor for Prometheus

```yaml
# k8s/service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: springboot-metrics
  namespace: springboot-app
  labels:
    app: springboot-app
spec:
  selector:
    matchLabels:
      app: springboot-app
  endpoints:
  - port: http
    path: /actuator/prometheus
    interval: 30s
```

#### 9. Deployment Commands

```bash
# Create namespace
kubectl apply -f k8s/namespace.yaml

# Apply configurations
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml

# Deploy database and cache
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/redis.yaml

# Wait for dependencies to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n springboot-app --timeout=300s
kubectl wait --for=condition=ready pod -l app=redis -n springboot-app --timeout=300s

# Deploy application
kubectl apply -f k8s/app-deployment.yaml

# Apply ingress and HPA
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/hpa.yaml

# Check deployment status
kubectl get pods -n springboot-app
kubectl get services -n springboot-app
kubectl get ingress -n springboot-app

# View logs
kubectl logs -f deployment/springboot-app -n springboot-app

# Scale manually
kubectl scale deployment springboot-app --replicas=5 -n springboot-app

# Port forward for testing
kubectl port-forward service/springboot-service 8080:80 -n springboot-app
```

#### 10. Health Checks Configuration

Add to your Spring Boot application:

```java
// Health checks for Kubernetes
@Component
public class KubernetesHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        // Check application health
        boolean isHealthy = checkApplicationHealth();
        
        if (isHealthy) {
            return Health.up()
                    .withDetail("status", "Application is running")
                    .withDetail("pod", System.getenv("HOSTNAME"))
                    .build();
        } else {
            return Health.down()
                    .withDetail("status", "Application is not ready")
                    .build();
        }
    }
    
    private boolean checkApplicationHealth() {
        // Implement your health check logic
        return true;
    }
}
```

```yaml
# application-kubernetes.yml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

### Best Practices

#### 1. Security
- Use non-root users in containers
- Scan images for vulnerabilities
- Use secrets for sensitive data
- Implement network policies
- Use service accounts with minimal permissions

#### 2. Resource Management
- Set appropriate resource requests and limits
- Use horizontal pod autoscaling
- Monitor resource usage
- Implement pod disruption budgets

#### 3. Monitoring and Logging
- Use centralized logging (ELK stack)
- Implement distributed tracing
- Set up alerts for critical metrics
- Use health checks effectively

#### 4. CI/CD Integration
```yaml
# .github/workflows/deploy.yml
name: Build and Deploy
on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
    
    - name: Build with Maven
      run: mvn clean package -DskipTests
    
    - name: Build Docker image
      run: |
        docker build -t ${{ secrets.REGISTRY_URL }}/springboot-app:${{ github.sha }} .
        docker tag ${{ secrets.REGISTRY_URL }}/springboot-app:${{ github.sha }} ${{ secrets.REGISTRY_URL }}/springboot-app:latest
    
    - name: Push to registry
      run: |
        echo ${{ secrets.REGISTRY_PASSWORD }} | docker login ${{ secrets.REGISTRY_URL }} -u ${{ secrets.REGISTRY_USERNAME }} --password-stdin
        docker push ${{ secrets.REGISTRY_URL }}/springboot-app:${{ github.sha }}
        docker push ${{ secrets.REGISTRY_URL }}/springboot-app:latest
    
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/springboot-app springboot-app=${{ secrets.REGISTRY_URL }}/springboot-app:${{ github.sha }} -n springboot-app
```

This comprehensive guide covers Docker containerization and Kubernetes orchestration for Spring Boot applications, including best practices for production deployments.
