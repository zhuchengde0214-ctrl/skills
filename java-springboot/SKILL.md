---
name: java-springboot
description: 'Get best practices for developing applications with Spring Boot.'
---

# Spring Boot Best Practices

Your goal is to help me write high-quality Spring Boot applications by following established best practices.

## Project Setup & Structure

- **Build Tool:** Use Maven (`pom.xml`) for dependency management.
- **Starters:** Use Spring Boot starters (e.g., `spring-boot-starter-web`, `spring-boot-starter-data-jpa`) to simplify dependency management.
- **Package Structure:** Organize code by layer (e.g., `com.example.app.controller`, `com.example.app.service`, `com.example.app.repository`).

## Dependency Injection & Components

- **Constructor Injection:** Always use constructor-based injection for required dependencies. This makes components easier to test and dependencies explicit.

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    // Constructor injection - preferred approach
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}

// With Lombok @RequiredArgsConstructor
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
}
```

- **Immutability:** Declare dependency fields as `private final`.
- **Component Stereotypes:** Use `@Component`, `@Service`, `@Repository`, and `@Controller`/`@RestController` annotations appropriately to define beans.

## Configuration

- **Externalized Configuration:** Use `application.yml` (or `application.properties`) for configuration. YAML is often preferred for its readability and hierarchical structure.
- **Type-Safe Properties:** Use `@ConfigurationProperties` to bind configuration to strongly-typed Java objects.

```java
@Configuration
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {
    @NotNull
    private String name;
    private Security security = new Security();
    
    public static class Security {
        private int tokenExpirationMinutes = 60;
        private String jwtSecret;
    }
    // getters and setters
}
```

- **Profiles:** Use Spring Profiles (`application-dev.yml`, `application-prod.yml`) to manage environment-specific configurations.
- **Secrets Management:** Do not hardcode secrets. Use environment variables, or a dedicated secret management tool like HashiCorp Vault or AWS Secrets Manager.

## Web Layer (Controllers)

- **RESTful APIs:** Design clear and consistent RESTful endpoints.
  - `GET /api/users` - List all users
  - `GET /api/users/{id}` - Get single user
  - `POST /api/users` - Create user
  - `PUT /api/users/{id}` - Full update
  - `DELETE /api/users/{id}` - Delete user

- **DTOs (Data Transfer Objects):** Use DTOs to expose and consume data in the API layer. Do not expose JPA entities directly to the client.

```java
// Use Java records for DTOs
public record UserDto(
    Long id,
    String username,
    String email,
    LocalDateTime createdAt
) {}

public record CreateUserRequest(
    @NotBlank @Size(min = 3, max = 50) String username,
    @NotBlank @Email String email,
    @NotBlank @Size(min = 8) String password
) {}
```

- **Validation:** Use Java Bean Validation (JSR 380) with annotations (`@Valid`, `@NotNull`, `@NotBlank`, `@Size`, `@Email`) on DTOs to validate request payloads.

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;
    
    @PostMapping
    public ResponseEntity<UserDto> createUser(@Valid @RequestBody CreateUserRequest request) {
        UserDto created = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

- **Error Handling:** Implement a global exception handler using `@ControllerAdvice` and `@ExceptionHandler` to provide consistent error responses.

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException ex) {
        log.error("Entity not found: {}", ex.getMessage());
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getMessage()));
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationErrors(
            MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .toList();
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse("Validation failed", errors));
    }
}
```

## Service Layer

- **Business Logic:** Encapsulate all business logic within `@Service` classes.
- **Statelessness:** Services should be stateless.
- **Transaction Management:** Use `@Transactional` on service methods to manage database transactions declaratively. Apply it at the most granular level necessary.
  - Use `@Transactional(readOnly = true)` for read-only operations (performance optimization)
  - Keep transactions short - don't call external APIs within transactions

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    
    @Transactional(readOnly = true)
    public UserDto findById(Long id) {
        log.debug("Finding user by id: {}", id);
        User user = userRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("User not found with id: " + id));
        return mapToDto(user);
    }
    
    @Transactional
    public UserDto create(CreateUserRequest request) {
        log.info("Creating user: {}", request.username());
        User user = new User();
        user.setUsername(request.username());
        user.setPassword(passwordEncoder.encode(request.password()));
        User saved = userRepository.save(user);
        return mapToDto(saved);
    }
}
```

## Data Layer (Repositories)

- **Spring Data JPA:** Use Spring Data JPA repositories by extending `JpaRepository` or `CrudRepository` for standard database operations.
- **Custom Queries:** For complex queries, use `@Query` or the JPA Criteria API.
- **Projections:** Use DTO projections to fetch only the necessary data from the database.

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    Optional<User> findByUsername(String username);
    boolean existsByUsername(String username);
    
    @Query("SELECT u FROM User u WHERE u.email LIKE %:domain")
    List<User> findByEmailDomain(@Param("domain") String domain);
    
    // DTO projection for performance
    @Query("SELECT new com.example.app.dto.UserSummary(u.id, u.username, u.email) " +
           "FROM User u WHERE u.active = true")
    List<UserSummary> findActiveUserSummaries();
}
```

**JPA Entity Best Practices:**
```java
@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true, length = 50)
    private String username;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(nullable = false)
    private String password;
    
    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    private LocalDateTime updatedAt;
    
    // Use LAZY fetch for associations
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "role_id")
    private Role role;
}
```

## Logging

- **Logger Declaration:** Use `@Slf4j` from Lombok or `private static final Logger logger = LoggerFactory.getLogger(MyClass.class);`
- **Parameterized Logging:** Use parameterized messages (`logger.info("Processing user {}...", userId);`) instead of string concatenation to improve performance.

```java
@Service
@Slf4j
public class UserService {
    public void processUser(Long userId) {
        log.debug("Processing user: {}", userId);
        try {
            // business logic
            log.info("Successfully processed user: {}", userId);
        } catch (Exception e) {
            log.error("Failed to process user: {}", userId, e);
            throw e;
        }
    }
}
```

- **Don't log sensitive data** (passwords, tokens, PII)
- **Log levels:** TRACE (diagnostic) → DEBUG (development) → INFO (milestones) → WARN (potential issues) → ERROR (failures)

## Testing

### Unit Tests for Controllers
- **Always use `@ExtendWith(MockitoExtension.class)`** for controller unit tests
- Mock all dependencies with `@Mock`
- Use `@InjectMocks` to inject the controller

```java
@ExtendWith(MockitoExtension.class)
class UserControllerTest {
    @Mock
    private UserService userService;
    
    @InjectMocks
    private UserController userController;
    
    @Test
    void shouldReturnUserWhenFound() {
        // Given
        UserDto expectedUser = new UserDto(1L, "john", "john@example.com", LocalDateTime.now());
        when(userService.findById(1L)).thenReturn(expectedUser);
        
        // When
        ResponseEntity<UserDto> response = userController.getUserById(1L);
        
        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isEqualTo(expectedUser);
        verify(userService).findById(1L);
    }
    
    @Test
    void shouldThrowExceptionWhenUserNotFound() {
        // Given
        when(userService.findById(1L))
            .thenThrow(new EntityNotFoundException("User not found"));
        
        // When & Then
        assertThatThrownBy(() -> userController.getUserById(1L))
            .isInstanceOf(EntityNotFoundException.class)
            .hasMessageContaining("User not found");
    }
}
```

### Integration Tests for Services
- **Use `@Import` with explicit classes** for service integration tests that load the Spring application context
- Import only the specific beans needed for the test

```java
@SpringBootTest
@Import({UserService.class, UserMapper.class})
@Transactional
class UserServiceIntegrationTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Test
    void shouldCreateAndRetrieveUser() {
        // Given
        CreateUserRequest request = new CreateUserRequest("john", "john@example.com", "pass123");
        
        // When
        UserDto created = userService.create(request);
        UserDto retrieved = userService.findById(created.id());
        
        // Then
        assertThat(retrieved.username()).isEqualTo("john");
        assertThat(retrieved.email()).isEqualTo("john@example.com");
    }
}
```

## Security

- **Spring Security:** Use Spring Security for authentication and authorization.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        return http.build();
    }

}
```

- **Input Sanitization:** Prevent SQL injection by using Spring Data JPA or parameterized queries. Prevent Cross-Site Scripting (XSS) by properly encoding output.
- **Never** hardcode secrets - use environment variables or secret management tools

## Performance & Best Practices

- **Database:** Use indexes, `@Transactional(readOnly = true)`, pagination (`Pageable`), and avoid N+1 queries with `@EntityGraph` or JOIN FETCH
- **Caching:** Use Spring Cache (`@Cacheable`, `@CacheEvict`) for expensive operations
- **MapStruct:** Use MapStruct for entity-DTO mapping instead of manual mapping

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDto toDto(User user);
    User toEntity(CreateUserRequest request);
}
```

## Anti-Patterns to Avoid

1. ❌ **Never expose JPA entities in REST APIs** - Always use DTOs
2. ❌ **Don't use field injection (`@Autowired` on fields)** - Use constructor injection
3. ❌ **Don't use `@Transactional` in controllers** - Business logic should be in services
4. ❌ **Don't query databases in loops** - Use batch operations or JOIN FETCH
5. ❌ **Don't hardcode configuration** - Use externalized configuration
6. ❌ **Don't use `Optional.get()` without checking** - Use `orElseThrow()` or `orElse()`

## Development Workflow

When creating a new feature:
1. Define the API contract (DTOs, endpoints)
2. Create/modify JPA entities if needed
3. Implement repository with required queries
4. Implement service with business logic and `@Transactional`
5. Implement controller with proper validation
6. Add global exception handling for edge cases
7. Write unit tests for controller layer
8. Write integration tests for service